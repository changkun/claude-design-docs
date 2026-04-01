# Streaming Executor — Design Document

### 1. Overview

The Streaming Tool Executor is the engine that sits between the API streaming response and the conversation message history. When the model emits `tool_use` content blocks during a streaming response, these blocks must be validated, permission-checked, executed (possibly in parallel), wrapped with pre/post hooks, and their results formatted back into `tool_result` messages for the next API turn.

The system has two execution paths:

1. **Streaming execution** -- tools begin executing as soon as their `tool_use` block arrives from the stream, before the full response is complete. This is the primary path, gated by a feature flag.
2. **Batch execution** -- the legacy path that waits for the full response, partitions tools into concurrency-safe batches, and executes them sequentially or concurrently per batch.

Both paths funnel individual tool calls through the same single-tool execution engine, which handles validation, permission resolution, hook orchestration, the actual tool call, and result formatting.

The executor's role in the query loop:

```
API Stream
    |
    +-- content_block_start (tool_use) --> addTool()
    +-- content_block_delta              |
    +-- content_block_stop               |  (tools execute concurrently
    |   ...                              |   while stream continues)
    +-- message_stop                     |
    |                                    v
    +-- getCompletedResults() --> yield progress + completed results
    |                                    |
    +-- getRemainingResults() --> yield remaining results in order
                                         |
                                         v
                                    Messages appended to conversation
                                    -> next API turn
```

---

### 2. Single-Tool Execution Lifecycle

The universal entry point for executing a single tool is an AsyncGenerator that yields a sequence of message updates (tool results, progress events, hook outputs, error messages) as the tool progresses through its lifecycle.

It receives:
- The model's tool call (name, id, input)
- The parent assistant message
- A permission-checking callback
- The full execution context (messages, app state, abort controller, MCP clients, options, etc.)

Its lifecycle:

```
runToolUse(toolUse)
    |
    +-- 1. Resolve tool definition (available tools -> deprecated aliases -> error)
    +-- 2. Check abort signal (yield cancel message if aborted)
    +-- 3. Delegate to stream adapter
              |
              +-- Creates a push-pull stream
              +-- Runs the actual execution asynchronously:
              |     +-- 4. Validate input (Zod schema -> semantic validation)
              |     +-- 5. Run PreToolUse hooks
              |     +-- 6. Resolve permission decision (hooks + rules + canUseTool)
              |     +-- 7. Execute tool.call() with progress callback
              |     +-- 8. Process tool result (map -> persist if large)
              |     +-- 9. Run PostToolUse hooks (or PostToolUseFailure on error)
              |     +-- 10. Return collected message updates
              +-- Enqueues progress events and final results into the stream
```

#### 2.1 The Stream Adapter

A bridge between the async/callback world of tool execution and the async-iterable world of the query loop. It creates a push-pull stream, runs the actual execution in a promise, and enqueues progress events (via the progress callback) and final results into the stream. This allows the caller to iterate over results as they become available, interleaving progress messages with the eventual tool result.

#### 2.2 Input Validation

Before any execution, tool input passes through two validation stages:

1. **Schema validation** -- Zod validates the JSON structure against the tool's declared schema. Failures produce a validation error with the Zod error details.

2. **Semantic validation** -- tool-specific validation logic (e.g., checking file paths exist, validating bash command syntax). Returns a failure message on error.

A special case handles **deferred tools** (tools whose schemas are loaded on demand): if schema validation fails and the tool's schema was not sent to the API, a hint is appended telling the model to call the tool search mechanism first. Without the schema in the prompt, the model emits typed parameters as strings, causing client-side parsing failures.

#### 2.3 Message Updates with Context Modifiers

Every step in tool execution yields message update objects. Each may carry an optional context modifier -- a function that transforms the execution context after the tool completes. This is used by tools that change global state (e.g., switching permission mode, updating the working directory). Context modifiers are applied sequentially for non-concurrent tools and queued for later application for concurrent tools.

---

### 3. Streaming Tool Executor

#### 3.1 Purpose

The streaming executor enables tool execution to overlap with API response streaming. Instead of waiting for the complete response before executing any tools, it starts executing each tool as soon as its `tool_use` block is fully streamed. For a response containing multiple tool calls, this can significantly reduce end-to-end latency.

#### 3.2 State Model

Each tool is tracked with a lifecycle state:

```
queued -> executing -> completed -> yielded
```

- **queued**: The tool has been added but cannot execute yet (concurrency constraint).
- **executing**: The tool is actively running.
- **completed**: Execution finished; results are buffered but not yet yielded to the caller.
- **yielded**: Results have been consumed by the query loop.

Each tracked tool holds:
- The tool_use block ID and raw API block
- The parent assistant message
- A concurrency-safety flag
- An execution promise
- Collected result messages
- Pending progress messages (yielded immediately)
- Collected context modifiers

#### 3.3 Adding Tools

When a `tool_use` block arrives from the API stream:

1. Look up the tool definition. If unknown, immediately create a completed tool with an error result.
2. Parse the input and evaluate concurrency safety.
3. Push a new tracked tool with status `queued`.
4. Trigger queue processing to potentially start execution immediately.

#### 3.4 Queue Processing

The queue processor iterates over all queued tools in order and starts each one that passes the concurrency check:

**Invariant:** A non-concurrent-safe tool can only execute when nothing else is executing, and it blocks all subsequent tools until it completes. Concurrent-safe tools can overlap with each other but not with non-concurrent tools.

When a non-concurrent tool is encountered and cannot execute, processing stops -- tools after it remain queued even if they are concurrent-safe, preserving execution order for tools that might depend on prior state changes.

#### 3.5 Tool Execution

The tool execution manager handles a single tool's lifecycle:

1. Sets status to `executing` and updates the in-progress tool ID set.
2. Updates the interruptible state.
3. Checks for abort reasons before starting (sibling error, user interrupt, streaming fallback). If aborted, synthesizes an error message immediately.
4. Creates a **per-tool abort controller** as a child of the sibling abort controller. If the tool is aborted for reasons other than a sibling error, it bubbles up to the query-level controller.
5. Runs the tool execution generator, collecting messages and context modifiers.
6. Progress messages are stored separately for immediate yielding; result messages are buffered.
7. On completion, applies context modifiers for non-concurrent tools.
8. Re-triggers queue processing to start the next eligible tool.

#### 3.6 Result Yielding

Two result-yielding methods:

**Completed results (synchronous)** -- called during streaming. Walks the tool list in order, yielding:
- Pending progress messages from any tool (regardless of status) -- yielded immediately for UI responsiveness.
- Completed results in order. For non-concurrent tools, if one is still executing, it stops (strict ordering). For concurrent-safe tools, it can skip over still-executing tools and yield completed ones.

**Remaining results (asynchronous)** -- called after the API stream completes. Loops until all tools are yielded, processing the queue, yielding completed results, and awaiting either tool completion or progress availability using Promise.race().

#### 3.7 Abort Hierarchy

The abort controller hierarchy is three levels deep:

```
Query-level controller        <-- user Ctrl+C, etc.
    +-- Sibling controller    <-- Bash error cascades
        +-- Tool controller   <-- permission denial, etc.
```

- The **sibling controller** is a child of the query-level controller. When a Bash tool errors, it aborts all sibling subprocesses immediately. Critically, this does NOT abort the parent -- the query loop continues and collects synthetic error messages.

- Each **tool controller** is a child of the sibling controller. If a tool is aborted for reasons other than a sibling error, the abort bubbles UP to the query-level controller so the turn ends properly.

#### 3.8 Discarding

When a streaming fallback occurs (the API response is retried from scratch), a discard flag prevents all pending tools from starting and causes result-yielding methods to return immediately. Already-executing tools receive synthetic fallback error messages. A fresh executor is then created for the retried response.

---

### 4. Batch Tool Orchestration (Legacy Path)

The legacy non-streaming path receives the complete list of tool_use blocks after the API response finishes and executes them with the same concurrency semantics as the streaming executor.

#### 4.1 Batch Partitioning

Consecutive tool calls are grouped into batches. Consecutive concurrency-safe tools are merged into one batch. Each non-concurrent-safe tool becomes its own single-element batch.

Example: given tools `[Read, Grep, Write, Read, Read]` where Read and Grep are concurrent-safe:

```
Batch 1: [Read, Grep]    <-- concurrent
Batch 2: [Write]          <-- serial
Batch 3: [Read, Read]     <-- concurrent
```

#### 4.2 Serial vs. Concurrent Execution

**Serial execution**: Each tool in the batch runs one at a time. Context modifiers are applied immediately after each tool completes.

**Concurrent execution**: All tools in the batch run in parallel, bounded by a configurable maximum concurrency (default 10). Context modifiers are queued and applied after the entire batch completes, in tool order.

#### 4.3 Path Selection

The query loop chooses between the streaming executor's remaining-results method and the batch orchestration based on a feature flag.

---

### 5. Hook System

The hook system provides extensibility points around tool execution. Hooks are user-defined commands that fire at specific lifecycle events, allowing external scripts, LLMs, agents, and HTTP services to observe, modify, or block tool invocations.

#### 5.1 Hook Events

The system defines the following hook events:

| Event | When it fires |
|-------|---------------|
| `PreToolUse` | Before tool execution; can block, modify input, or set permission |
| `PostToolUse` | After successful tool execution; can modify MCP output |
| `PostToolUseFailure` | After tool execution fails or is interrupted |
| `PermissionDenied` | After auto-mode classifier denies a tool; can signal retry |
| `PermissionRequest` | When permission decision is needed; can provide allow/deny |
| `Notification` | On assistant text/completion events |
| `UserPromptSubmit` | Before user prompt is sent to the model |
| `SessionStart` | At session initialization |
| `SessionEnd` | At session teardown (tight 1.5s timeout) |
| `Stop` | When the model stops (no pending tool calls) |
| `StopFailure` | When an API error ends the turn (fire-and-forget) |
| `SubagentStart` | When a subagent begins execution |
| `SubagentStop` | When a subagent completes |
| `PreCompact` | Before context compaction |
| `PostCompact` | After context compaction |
| `Setup` | One-time initialization for plugins |
| `TeammateIdle` | When a teammate agent becomes idle |
| `TaskCreated` | When a background task is created |
| `TaskCompleted` | When a background task completes |
| `Elicitation` | When MCP elicitation is requested |
| `ElicitationResult` | When MCP elicitation completes |
| `ConfigChange` | When settings change |
| `WorktreeCreate` | When a git worktree is created |
| `WorktreeRemove` | When a git worktree is removed |
| `InstructionsLoaded` | When CLAUDE.md files are loaded |
| `CwdChanged` | When the working directory changes |
| `FileChanged` | When watched files change on disk |

Of these, three are directly involved in tool execution: `PreToolUse`, `PostToolUse`, and `PostToolUseFailure`. The remaining events fire at other lifecycle points but share the same hook infrastructure.

#### 5.2 Hook Implementation Types (6 Types)

| Type | Mechanism |
|------|-----------|
| **Shell** | Spawns a child process with hook input as JSON on stdin. Exit code 0 = success, 2 = blocking error, other = non-blocking. Configurable shell and timeout. |
| **LLM (Prompt)** | Sends hook input to an LLM with a configurable system prompt. Response parsed as hook JSON output. |
| **Agent** | Runs a full agentic loop with tools. More powerful than prompt hooks. |
| **HTTP** | Sends hook input as POST to a URL. Response body parsed as hook JSON. |
| **Callback** | In-process function registered via SDK. Used by SDK hosts for custom permission logic. |
| **Function** | In-process function registered per-session. Scoped to session lifecycle. |

#### 5.3 Hook Load Sources (7 Sources)

| Source | Trust Level |
|--------|-------------|
| User settings | User-authored |
| Project settings | Project-level, requires workspace trust |
| Local settings | Local overrides |
| Policy settings | Highest trust, can restrict others |
| Plugin hooks | Plugin-scoped |
| Session hooks | Session-scoped |
| Builtin hooks | System-level |

The policy source can set a flag to disable all user/project/local hooks and only permit managed (enterprise) hooks. All hooks require workspace trust in interactive mode -- a defense-in-depth measure to prevent hooks in untrusted repositories from executing before the user accepts the trust dialog.

#### 5.4 Tool-Specific Hook Flow

PreToolUse hooks yield a discriminated union:
- Permission results (allow/deny/ask)
- Updated tool input
- Prevention of continuation
- Stop reasons
- Additional context
- Stop execution entirely

PostToolUse hooks yield:
- Attachment/progress messages
- For MCP tools, modified tool output

PostToolUseFailure hooks yield attachment/progress messages that observe tool failures.

#### 5.5 Hook Execution Timing

- Hooks run in parallel per event.
- Hooks taking longer than 500ms show an inline timing summary (internal builds only).
- Hooks exceeding 2000ms log a debug warning.
- Default per-hook timeout: 10 minutes.
- SessionEnd hooks: 1.5 second timeout.

---

### 6. Permission Checking

Permission checking is deeply interleaved with tool execution. The resolution follows a multi-stage pipeline that combines hook decisions, rule-based checks, and interactive prompts.

#### 6.1 Resolution Pipeline

```
PreToolUse hooks --> hookPermissionResult (allow/deny/ask/undefined)
        |
        v
Permission Decision Resolution
        |
        +-- Hook says 'allow':
        |     +-- Tool requires user interaction & no updatedInput? -> canUseTool()
        |     +-- requireCanUseTool set? -> canUseTool()
        |     +-- Rule-based check (deny/ask rules in settings):
        |     |     +-- null (no rule) -> hook allow wins, skip prompt
        |     |     +-- deny -> deny overrides hook allow
        |     |     +-- ask -> interactive dialog despite hook approval
        |     +-- Otherwise -> hook allow accepted
        |
        +-- Hook says 'deny': -> deny (immediate)
        |
        +-- Hook says 'ask': -> canUseTool() with forceDecision
        |
        +-- No hook decision: -> canUseTool() (normal flow)
                |
                +-- Rule evaluation (allow/deny/ask rules)
                +-- Mode transformation (auto->classifier, dontAsk->deny)
                +-- AI classifier (auto mode)
                +-- Denial limit tracking
                +-- Interactive dialog (if needed)
```

**Critical invariant: hook 'allow' does NOT bypass deny/ask rules from user settings.** This prevents a compromised or misconfigured hook from overriding explicit user security policy.

#### 6.2 Speculative Classifier

For Bash tool calls, a speculative classifier check is started early -- before hooks and permission resolution -- so the AI classifier runs in parallel with those checks. The classifier result is awaited only when the permission flow reaches the auto mode path, avoiding serial latency in the common case.

#### 6.3 Input Mutation

Both hooks and the permission system can mutate tool input:

- **Hook updatedInput**: PreToolUse hooks can return modified input that replaces the model's original. If a hook provides updatedInput for an interactive tool, the hook IS the user interaction -- the tool is treated as non-interactive for the rule-check path.
- **Permission updatedInput**: The permission dialog can return modified input.
- **Observable input backfill**: Some tools fill derived fields on a clone for hooks to observe (e.g., expanding `~` in file paths). The original model-provided input is preserved for the tool call to maintain transcript stability.

#### 6.4 Permission Decision Telemetry

Every permission decision is logged with a source label:

| Source | Meaning |
|--------|---------|
| `config` | Rule-based, mode-based, or classifier decision |
| `hook` | Hook provided the decision |
| `user_permanent` | User clicked "Always allow" (persisted) |
| `user_temporary` | User clicked "Allow once" (session-scoped) |
| `user_reject` | User denied the operation |

---

### 7. Progress Tracking

#### 7.1 The Progress Callback

Tools report progress through a callback passed to the tool call. Progress data is typed as a discriminated union with variants for different tool types: Bash (stdout/stderr streaming, timing), MCP, Agent, WebSearch, REPL, Skill, TaskOutput, etc.

#### 7.2 Progress in the Stream Adapter

The stream adapter wraps progress events into progress messages and enqueues them into the push-pull stream. The progress callback also logs an analytics event for each emission.

#### 7.3 Progress in the Streaming Executor

The streaming executor separates progress messages from result messages. Progress messages are stored in a pending array and a resolve callback is signaled to wake up the remaining-results method if it is blocked. The completed-results method always yields pending progress messages first, regardless of tool status or ordering constraints, ensuring the UI reflects real-time progress even when the producing tool is blocked behind a non-concurrent sibling.

#### 7.4 Hook Progress

Hooks emit progress through the same pipeline. A periodic progress interval utility emits events while a hook is running, keeping the UI spinner active.

---

### 8. Error Recovery

#### 8.1 Error Classification

Tool execution errors are classified for telemetry:
- Telemetry-safe errors use a pre-vetted message
- Filesystem errors log the errno code
- Named errors use the stable name property
- Generic errors fall back to "Error"
- Non-Error throwables become "UnknownError"

This avoids logging minified constructor names (meaningless 3-character identifiers in production builds).

#### 8.2 Bash Error Cascading

When a Bash tool errors, sibling tools are cancelled via the executor's sibling abort controller. This reflects implicit dependency chains in Bash commands -- if `mkdir` fails, subsequent commands are likely pointless. Only Bash errors trigger this cascade; other tool types are treated as independent.

```
Bash tool errors
    |
    +-- hasErrored = true
    +-- siblingAbortController.abort('sibling_error')
    |     +-- per-tool abort controllers receive the signal
    |
    +-- Other tools check abort reason:
          +-- Queued tools -> synthetic error, never start
          +-- Executing tools -> synthetic error on next yield
```

The synthetic error message includes a description of the errored tool, helping the model understand what happened.

#### 8.3 User Interruption

When the user presses Escape or submits a new message while tools are running, the abort controller fires with an 'interrupt' reason. The executor checks each tool's interrupt behavior:

- `'cancel'` -- tool is cancelled immediately with a synthetic error.
- `'block'` (default) -- tool continues executing; the interrupt is not propagated to it.

An interruptible-state tracker determines whether ALL executing tools are interruptible, which controls the UI interrupt affordance.

#### 8.4 Streaming Fallback

When the API response needs to be retried, the executor's discard method:
1. Prevents queued tools from starting.
2. Causes executing tools to receive fallback synthetic errors.
3. Makes result-yielding methods return immediately.

A fresh executor is then created for the retried response.

#### 8.5 MCP Authentication Errors

MCP auth errors are handled specially: the MCP client status is updated to "needs-auth" in the app state, updating the UI. The error is still returned to the model as a tool_result error.

#### 8.6 PostToolUseFailure Hooks

After any tool execution failure (including user interrupts), failure hooks run to observe the failure. They can provide additional context, blocking errors, or UI messages. They receive the error string and an isInterrupt flag.

---

### 9. Result Rendering

#### 9.1 Mapping Tool Output to API Format

After tool execution, results are mapped to the Anthropic API's tool result format via a tool-specific mapping function. The mapped result is cached and reused when hooks do not modify the output, avoiding redundant remapping.

#### 9.2 Large Result Persistence

If mapped content exceeds the tool's maximum result size (clamped by a global default of 50,000 characters), the full content is persisted to disk and replaced with a preview:

```
<persisted-output>
Output too large (150.0 KB). Full output saved to: /path/to/tool-results/abc123.txt

Preview (first 2.0 KB):
[first 2000 bytes of content]
...
</persisted-output>
```

Files are written with a write-exclusive flag to avoid re-writing unchanged content on subsequent API turns.

#### 9.3 Empty Result Handling

Empty tool results are a known hazard: some models interpret an empty `tool_result` at the prompt tail as a turn boundary and emit the stop sequence with zero output. To prevent this, empty results are replaced with:

```
(toolName completed with no output)
```

#### 9.4 Content Block Assembly

The final message for the API includes content blocks assembled in order:
1. Tool result block (the mapped, possibly persisted output)
2. Accept feedback (if the user typed feedback while approving the tool call)
3. Content blocks from permission (images pasted during the permission dialog)

#### 9.5 MCP Tool Ordering

MCP tools have a special ordering constraint: the tool result is added to the message list AFTER PostToolUse hooks run, not before. This allows hooks to modify the MCP output before the result is mapped. For non-MCP tools, the result is added first, then hooks run -- hooks see the output but cannot modify it.

---

### 10. Concurrency

#### 10.1 The Concurrency Safety Contract

Every tool must implement a concurrency safety check that receives the parsed input and returns whether the tool call can safely execute in parallel with other concurrent-safe calls. The evaluation is per-invocation, not per-tool-type.

Examples:

| Tool Type | Concurrency Logic |
|-----------|-------------------|
| File read | Always concurrent-safe (read-only) |
| Grep | Always concurrent-safe (read-only) |
| Glob | Always concurrent-safe (read-only) |
| Bash | Concurrent-safe only for read-only commands |
| File edit | Never concurrent-safe (writes files) |
| File write | Never concurrent-safe (writes files) |
| Agent | Never concurrent-safe (spawns subagents with side effects) |
| User question | Never concurrent-safe (requires user interaction) |

If the safety check throws, the tool is conservatively treated as non-concurrent-safe.

#### 10.2 Execution Semantics

1. Non-concurrent tools execute alone -- no other tool runs simultaneously.
2. Between non-concurrent tools, execution order matches the model's emission order.
3. Concurrent-safe tools between two non-concurrent tools run in parallel.
4. Maximum concurrency for the batch path is configurable (default 10).

In the streaming executor, this is enforced dynamically. In the batch orchestrator, this is enforced structurally via partitioning.

#### 10.3 Context Modifier Constraints

Context modifiers are NOT supported for concurrent tools in the streaming executor. For non-concurrent tools, modifiers are applied immediately after completion. For the batch path, concurrent-tool modifiers are queued and applied after the entire batch finishes, in tool order.

---

### 11. Tool Result Budget

#### 11.1 Two-Level Size Enforcement

**Per-tool limit**: Each tool declares a maximum result size (or inherits the default of 50,000 characters). Results exceeding this are persisted to disk and replaced with a preview. A feature flag can override the threshold per tool name.

Tools that opt out of persistence (setting their limit to infinity) have their size bounded by their own internal logic. Persisting their output to a file the model reads back would be circular.

**Per-message aggregate limit**: When N parallel tools each produce large results, their combined size can be enormous. A cap of 200,000 characters limits the aggregate size per API-level user message. Overridable via feature flag.

#### 11.2 Content Replacement State

The per-message budget is managed by a state object that tracks:
- Which tool_use_ids have been evaluated (frozen set)
- Cached replacement strings for replaced results

Once a tool result has been seen, its fate is **frozen**: replaced results always get the same replacement re-applied (from cache, with zero I/O), and unreplaced results are never replaced later. This stability is critical for prompt cache preservation -- changing a previously-sent result would invalidate the cache prefix.

#### 11.3 Enforcement Algorithm

The budget enforcement runs once per API turn on the full message array:

1. **Group candidates by API-level message** -- consecutive user messages are grouped as the API merges them on the wire.
2. **Partition by prior decision**: must-reapply (previously replaced), frozen (previously seen but unreplaced), fresh (never seen).
3. **Select fresh results to replace** -- sort by size descending, replace the largest until the group's model-visible total is at or under budget.
4. **Persist and record** -- selected results are persisted concurrently. New replacements are written to the transcript for resume reconstruction.

#### 11.4 Resume Stability

On session resume, the replacement state is rebuilt from transcript records. Every candidate tool_use_id in the loaded messages is marked as seen (frozen), and replacement strings are populated from stored records. This ensures identical budget decisions across resumes -- a hard requirement for prompt cache stability.

For subagent (fork) resumes, the parent's live replacement state fills gaps for inherited replacements not written to the subagent's own records.

---

### 12. Design Principles

1. **Streaming-First Latency Optimization** -- Tools start during the API response stream, not after. The first tool may finish before the last tool's block has been fully received.

2. **Order Preservation with Opportunistic Parallelism** -- Results are yielded in reception order. Non-concurrent tools act as barriers. Parallelism is extracted where safe.

3. **Fail-Safe Concurrency Classification** -- Conservative defaults. If the safety check throws, run serially. Unknown tools get error results with no execution. Default interrupt behavior is "block".

4. **Hook Allow Cannot Override Deny Rules** -- A PreToolUse hook returning "allow" still faces rule-based deny checks. Compromised hooks cannot bypass explicit user security policy.

5. **Abort Signal Hierarchy** -- Three levels (query, sibling, tool) provide precise cancellation scope. Bash errors kill siblings but not the query. Permission denials end the turn. User interrupts respect per-tool behavior.

6. **Stable Persistence for Cache Preservation** -- Decisions are frozen once made. Replacements are cached as exact strings. File writes use exclusive flags. Byte-identical output guaranteed across turns.

7. **Separation of Mapping and Persistence** -- Tool-specific output mapping is unaware of size budgets. Persistence logic is unaware of tool-specific formats. The mapped block is cached to avoid redundant remapping.

8. **Progress as a First-Class Concern** -- Progress messages bypass ordering constraints. They are enqueued as they arrive, not batched. The UI always reflects what is actually happening.
