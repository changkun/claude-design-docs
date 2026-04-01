# Claude Code: Streaming Tool Executor — Design Specification

This document analyzes the tool execution engine of Claude Code — Anthropic's agentic CLI
tool — focusing on how it receives tool_use blocks from the streaming API response,
orchestrates their execution with concurrency control, interleaves permission checking and
hooks, manages progress reporting, and assembles results back into the conversation.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Execution Architecture](#2-execution-architecture)
- [3. StreamingToolExecutor](#3-streamingtoolexecutor)
- [4. Tool Orchestration](#4-tool-orchestration)
- [5. Hook System](#5-hook-system)
- [6. Permission Checking](#6-permission-checking)
- [7. Progress Tracking](#7-progress-tracking)
- [8. Error Recovery](#8-error-recovery)
- [9. Result Rendering](#9-result-rendering)
- [10. Concurrency](#10-concurrency)
- [11. Tool Result Budget](#11-tool-result-budget)
- [12. Design Principles](#12-design-principles)

---

## 1. Overview

The Streaming Tool Executor is the engine that sits between the API streaming response and
the conversation message history. When the model emits `tool_use` content blocks during a
streaming response, these blocks must be validated, permission-checked, executed (possibly
in parallel), wrapped with pre/post hooks, and their results formatted back into
`tool_result` messages for the next API turn.

The system has two execution paths:

1. **Streaming execution** (`StreamingToolExecutor`) — tools begin executing as soon as
   their `tool_use` block arrives from the stream, before the full response is complete.
   This is the primary path, gated by a feature flag.
2. **Batch execution** (`runTools` in `toolOrchestration.ts`) — the legacy path that waits
   for the full response, partitions tools into concurrency-safe batches, and executes them
   sequentially or concurrently per batch.

Both paths funnel individual tool calls through the same execution engine
(`runToolUse` in `toolExecution.ts`), which handles validation, permission resolution,
hook orchestration, the actual `tool.call()`, and result formatting.

The executor's role in the query loop:

```
API Stream
    │
    ├── content_block_start (tool_use) ──→ StreamingToolExecutor.addTool()
    ├── content_block_delta              │
    ├── content_block_stop               │  (tools execute concurrently
    │   ...                              │   while stream continues)
    ├── message_stop                     │
    │                                    ▼
    ├── getCompletedResults() ──→ yield progress + completed results
    │                                    │
    └── getRemainingResults() ──→ yield remaining results in order
                                         │
                                         ▼
                                    Messages appended to conversation
                                    → next API turn
```

---

## 2. Execution Architecture

> **Source:** `src/services/tools/toolExecution.ts`

### 2.1 The runToolUse Entry Point

`runToolUse()` is the universal entry point for executing a single tool. It is an
`AsyncGenerator<MessageUpdateLazy, void>` — it yields a sequence of message updates
(tool results, progress events, hook outputs, error messages) as the tool progresses
through its lifecycle.

The function receives:
- `toolUse: ToolUseBlock` — the model's tool call (name, id, input)
- `assistantMessage: AssistantMessage` — the parent assistant message
- `canUseTool: CanUseToolFn` — the permission-checking callback
- `toolUseContext: ToolUseContext` — the full execution context (messages, app state,
  abort controller, MCP clients, options, etc.)

Its lifecycle:

```
runToolUse(toolUse)
    │
    ├── 1. Resolve tool definition (available tools → deprecated aliases → error)
    ├── 2. Check abort signal (yield cancel message if aborted)
    └── 3. Delegate to streamedCheckPermissionsAndCallTool()
              │
              ├── Creates a Stream<MessageUpdateLazy>
              ├── Runs checkPermissionsAndCallTool() asynchronously
              │     ├── 4. Validate input (Zod schema → semantic validation)
              │     ├── 5. Run PreToolUse hooks
              │     ├── 6. Resolve permission decision (hooks + rules + canUseTool)
              │     ├── 7. Execute tool.call() with progress callback
              │     ├── 8. Process tool result (map → persist if large)
              │     ├── 9. Run PostToolUse hooks (or PostToolUseFailure on error)
              │     └── 10. Return collected MessageUpdateLazy[]
              └── Enqueues progress events and final results into the stream
```

### 2.2 The Stream Adapter

`streamedCheckPermissionsAndCallTool()` bridges the async/callback world of tool execution
with the async-iterable world of the query loop. It creates a `Stream<MessageUpdateLazy>`
(a push-pull adapter from `src/utils/stream.ts`), runs the actual execution in a promise,
and enqueues progress events (via the `onToolProgress` callback) and final results into
the stream. This allows the caller to iterate over results as they become available,
interleaving progress messages with the eventual tool result.

### 2.3 Input Validation

Before any execution, tool input passes through two validation stages:

1. **Schema validation** — `tool.inputSchema.safeParse(input)` uses Zod to validate the
   JSON structure against the tool's declared schema. Failures produce an
   `InputValidationError` with the Zod error details.

2. **Semantic validation** — `tool.validateInput?.(parsedInput, toolUseContext)` runs
   tool-specific validation logic (e.g., checking file paths exist, validating bash
   command syntax). Returns `{result: false, message}` on failure.

A special case handles **deferred tools** (tools whose schemas are loaded on demand via
`ToolSearchTool`): if Zod validation fails and the tool's schema was not sent to the API
(it was not in the discovered-tool set), a hint is appended telling the model to call
`ToolSearch` first. Without the schema in the prompt, the model emits typed parameters
(arrays, numbers) as strings, causing client-side parsing failures.

### 2.4 The MessageUpdateLazy Type

Every step in tool execution yields `MessageUpdateLazy` objects:

```typescript
export type MessageUpdateLazy<M extends Message = Message> = {
  message: M
  contextModifier?: {
    toolUseID: string
    modifyContext: (context: ToolUseContext) => ToolUseContext
  }
}
```

The `contextModifier` is an optional function that transforms the `ToolUseContext` after
the tool completes — used by tools that change global state (e.g., `EnterPlanModeTool`
switching permission mode, `CwdChangedTool` updating the working directory). Context
modifiers are applied sequentially for non-concurrent tools and queued for later
application for concurrent tools.

---

## 3. StreamingToolExecutor

> **Source:** `src/services/tools/StreamingToolExecutor.ts`

### 3.1 Purpose

The `StreamingToolExecutor` enables tool execution to overlap with API response streaming.
Instead of waiting for the complete response before executing any tools, it starts
executing each tool as soon as its `tool_use` block is fully streamed. For a response
containing multiple tool calls, this can significantly reduce end-to-end latency.

### 3.2 State Model

Each tool is tracked as a `TrackedTool`:

```typescript
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'

type TrackedTool = {
  id: string                         // tool_use block id
  block: ToolUseBlock                // the raw API block
  assistantMessage: AssistantMessage  // parent message
  status: ToolStatus                 // lifecycle state
  isConcurrencySafe: boolean         // can run in parallel
  promise?: Promise<void>            // execution promise
  results?: Message[]                // collected result messages
  pendingProgress: Message[]         // progress messages (yielded immediately)
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>
}
```

The lifecycle is:

```
queued → executing → completed → yielded
```

- **queued**: The tool has been added but cannot execute yet (concurrency constraint).
- **executing**: The tool is actively running (`runToolUse` generator is being consumed).
- **completed**: Execution finished; results are buffered but not yet yielded to the caller.
- **yielded**: Results have been consumed by the query loop.

### 3.3 Adding Tools

`addTool(block, assistantMessage)` is called from the query loop as each `tool_use` block
arrives from the API stream. The method:

1. Looks up the tool definition. If unknown, immediately creates a completed tool with
   an error result — no queuing or execution needed.
2. Parses the input and evaluates `isConcurrencySafe` (see section 10).
3. Pushes a new `TrackedTool` with status `queued`.
4. Calls `processQueue()` to potentially start execution immediately.

### 3.4 Queue Processing

`processQueue()` iterates over all queued tools in order and starts each one that
passes the `canExecuteTool()` check:

```typescript
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

The invariant: **a non-concurrent-safe tool can only execute when nothing else is
executing, and it blocks all subsequent tools until it completes**. Concurrent-safe tools
can overlap with each other but not with non-concurrent tools.

When a non-concurrent tool is encountered and cannot execute, processing stops — tools
after it remain queued even if they are concurrent-safe, preserving execution order for
tools that might depend on prior state changes.

### 3.5 Tool Execution

`executeTool(tool)` manages a single tool's lifecycle:

1. Sets status to `executing` and updates the in-progress tool ID set (for UI display).
2. Updates the interruptible state (whether all executing tools support cancellation).
3. Checks for abort reasons before starting (sibling error, user interrupt, streaming
   fallback). If aborted, synthesizes an error message immediately.
4. Creates a **per-tool abort controller** as a child of the `siblingAbortController`.
   This controller has a critical event listener: if the tool's abort controller fires
   for reasons other than a sibling error, it bubbles up to the parent
   `toolUseContext.abortController`, ensuring the query loop's post-tool abort check
   ends the turn correctly.
5. Runs `runToolUse()` via the async generator, collecting messages and context modifiers.
6. Progress messages (`type === 'progress'`) are stored separately in `pendingProgress`
   for immediate yielding, while result messages are buffered in `results`.
7. On completion, applies context modifiers for non-concurrent tools.
8. Calls `processQueue()` again to start the next eligible tool.

### 3.6 Result Yielding

The executor provides two result-yielding methods:

**`getCompletedResults()`** — a synchronous generator called during streaming. It walks
the tool list in order, yielding:
- Pending progress messages from any tool (regardless of status) — these are yielded
  immediately to keep the UI responsive.
- Completed results in order. For non-concurrent tools, if one is still executing, it
  stops — maintaining strict ordering. For concurrent-safe tools, it can skip over
  still-executing tools and yield completed ones.

**`getRemainingResults()`** — an async generator called after the API stream completes.
It loops until all tools are yielded, processing the queue, yielding completed results,
and awaiting either tool completion or progress availability using `Promise.race()`.

### 3.7 Abort Hierarchy

The abort controller hierarchy is three levels deep:

```
toolUseContext.abortController     ← query-level (user Ctrl+C, etc.)
    └── siblingAbortController     ← executor-level (Bash error cascades)
        └── toolAbortController    ← per-tool (permission denial, etc.)
```

- The **sibling abort controller** is a child of the query-level controller. When a
  Bash tool errors, `siblingAbortController.abort('sibling_error')` is called, killing
  all sibling subprocesses immediately. Critically, this does NOT abort the parent —
  the query loop continues and collects synthetic error messages.

- Each **tool abort controller** is a child of the sibling controller. If a tool is
  aborted for reasons other than a sibling error (e.g., permission denial), the abort
  bubbles UP to the query-level controller so the turn ends properly.

### 3.8 Discarding

`discard()` is called when a streaming fallback occurs (the API response is retried
from scratch). It sets a flag that prevents all pending tools from starting and causes
`getCompletedResults()` and `getRemainingResults()` to return immediately. Already-executing
tools will receive synthetic `streaming_fallback` error messages when they next check
the abort reason.

---

## 4. Tool Orchestration

> **Source:** `src/services/tools/toolOrchestration.ts`

The orchestration module (`toolOrchestration.ts`) is the legacy (non-streaming) execution
path. It receives the complete list of `ToolUseBlock`s after the API response finishes
and executes them with the same concurrency semantics as the streaming executor.

### 4.1 Batch Partitioning

`partitionToolCalls()` groups consecutive tool calls into batches:

```typescript
type Batch = { isConcurrencySafe: boolean; blocks: ToolUseBlock[] }
```

The algorithm scans tools left-to-right. Consecutive concurrency-safe tools are merged
into one batch. Each non-concurrent-safe tool becomes its own single-element batch.

Example: given tools `[Read, Grep, Write, Read, Read]` where Read and Grep are
concurrent-safe:

```
Batch 1: [Read, Grep]    ← concurrent
Batch 2: [Write]          ← serial
Batch 3: [Read, Read]     ← concurrent
```

### 4.2 Serial vs. Concurrent Execution

**Serial execution** (`runToolsSerially`): Each tool in the batch runs one at a time.
Context modifiers are applied immediately after each tool completes, so subsequent tools
see the updated context.

**Concurrent execution** (`runToolsConcurrently`): All tools in the batch run in parallel,
bounded by `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (default 10). Uses the `all()` utility
from `src/utils/generators.ts` — a concurrent async-generator merger that respects a
maximum concurrency limit. Context modifiers are queued and applied after the entire
batch completes, in tool order.

### 4.3 runTools as the Entry Point

`runTools()` is an `AsyncGenerator<MessageUpdate, void>` that:

1. Partitions tool calls into batches.
2. For each batch, dispatches to serial or concurrent execution.
3. Yields message updates as they arrive, tracking context modifications.
4. After each batch, yields a final context update.

The query loop at `src/query.ts` chooses between `StreamingToolExecutor.getRemainingResults()`
and `runTools()` based on the `streamingToolExecution` gate:

```typescript
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

---

## 5. Hook System

> **Source:** `src/services/tools/toolHooks.ts`, `src/utils/hooks.ts`

The hook system provides extensibility points around tool execution. Hooks are
user-defined commands that fire at specific lifecycle events, allowing external scripts,
LLMs, agents, and HTTP services to observe, modify, or block tool invocations.

### 5.1 Hook Events (26 Types)

The complete set of hook events is defined in `src/entrypoints/sdk/coreSchemas.ts`:

| # | Event | When it fires |
|---|-------|---------------|
| 1 | `PreToolUse` | Before tool execution; can block, modify input, or set permission |
| 2 | `PostToolUse` | After successful tool execution; can modify MCP output |
| 3 | `PostToolUseFailure` | After tool execution fails or is interrupted |
| 4 | `PermissionDenied` | After auto-mode classifier denies a tool; can signal retry |
| 5 | `PermissionRequest` | When permission decision is needed; can provide allow/deny |
| 6 | `Notification` | On assistant text/completion events |
| 7 | `UserPromptSubmit` | Before user prompt is sent to the model |
| 8 | `SessionStart` | At session initialization |
| 9 | `SessionEnd` | At session teardown (tight 1.5s timeout) |
| 10 | `Stop` | When the model stops (no pending tool calls) |
| 11 | `StopFailure` | When an API error ends the turn (fire-and-forget) |
| 12 | `SubagentStart` | When a subagent begins execution |
| 13 | `SubagentStop` | When a subagent completes |
| 14 | `PreCompact` | Before context compaction |
| 15 | `PostCompact` | After context compaction |
| 16 | `Setup` | One-time initialization for plugins |
| 17 | `TeammateIdle` | When a teammate agent becomes idle |
| 18 | `TaskCreated` | When a background task is created |
| 19 | `TaskCompleted` | When a background task completes |
| 20 | `Elicitation` | When MCP elicitation is requested |
| 21 | `ElicitationResult` | When MCP elicitation completes |
| 22 | `ConfigChange` | When settings change |
| 23 | `WorktreeCreate` | When a git worktree is created |
| 24 | `WorktreeRemove` | When a git worktree is removed |
| 25 | `InstructionsLoaded` | When CLAUDE.md files are loaded |
| 26 | `CwdChanged` | When the working directory changes |
| -- | `FileChanged` | When watched files change on disk |

Of these, three are directly involved in tool execution: `PreToolUse`, `PostToolUse`,
and `PostToolUseFailure`. The remaining events fire at other lifecycle points but share
the same hook infrastructure.

### 5.2 Hook Implementation Types (6 Types)

Each hook is one of six implementation types, determined by the `type` discriminator
in the hook command schema:

| Type | Discriminator | Mechanism |
|------|--------------|-----------|
| **Shell** | `type: 'command'` | Spawns a child process with the hook input as JSON on stdin. Exit code 0 = success, 2 = blocking error, other = non-blocking error. Supports configurable shell (bash/zsh/sh/powershell) and timeout. |
| **LLM (Prompt)** | `type: 'prompt'` | Sends hook input to an LLM with a configurable system prompt. The LLM response is parsed as hook JSON output. Supports model selection and token limits. |
| **Agent** | `type: 'agent'` | Runs a full agentic loop with tools. More powerful than prompt hooks — the agent can read files, execute commands, etc. Supports model selection and timeout. |
| **HTTP** | `type: 'http'` | Sends hook input as a POST request to a URL. Response body is parsed as hook JSON output. Supports custom headers and timeout. |
| **Callback** | `HookCallback` | An in-process function registered via the SDK. Used by SDK hosts to implement custom permission logic without spawning external processes. |
| **Function** | `FunctionHook` | An in-process function registered per-session. Similar to callbacks but registered through `getSessionFunctionHooks()` and scoped to the session lifecycle. |

Shell hooks are the most common in user-facing configuration. Prompt and agent hooks
provide AI-powered automation. HTTP hooks enable integration with external services.
Callback and function hooks are SDK-internal mechanisms.

### 5.3 Hook Load Sources (7 Sources)

Hooks are loaded from multiple configuration sources, merged with priority ordering:

| Source | Location | Trust Level |
|--------|----------|-------------|
| `userSettings` | `~/.claude/settings.json` | User-authored |
| `projectSettings` | `.claude/settings.json` (in repo) | Project-level, requires workspace trust |
| `localSettings` | `.claude/settings.local.json` (gitignored) | Local overrides |
| `policySettings` | Enterprise policy (managed) | Highest trust, can restrict others |
| `pluginHook` | Plugin-registered hooks | Plugin-scoped |
| `sessionHook` | SDK-registered per-session hooks | Session-scoped |
| `builtinHook` | Hardcoded internal hooks | System-level |

The `policySettings` source can set `allowManagedHooksOnly: true`, which disables all
user/project/local hooks and only permits managed (enterprise) hooks. All hooks require
workspace trust in interactive mode — a defense-in-depth measure to prevent hooks in
untrusted repositories from executing before the user accepts the trust dialog.

### 5.4 Tool-Specific Hook Flow

`toolHooks.ts` exposes three async generators that wrap the lower-level hook execution
in `src/utils/hooks.ts`:

**`runPreToolUseHooks()`** yields a discriminated union:
- `{type: 'message'}` — attachment/progress messages from hook execution
- `{type: 'hookPermissionResult'}` — hook wants to allow/deny/ask permission
- `{type: 'hookUpdatedInput'}` — hook modifies tool input without deciding permission
- `{type: 'preventContinuation'}` — hook wants to stop the agentic loop after this tool
- `{type: 'stopReason'}` — human-readable reason for stopping
- `{type: 'additionalContext'}` — extra context to inject into the conversation
- `{type: 'stop'}` — abort tool execution entirely

**`runPostToolUseHooks()`** yields:
- `MessageUpdateLazy<AttachmentMessage | ProgressMessage>` — hook output messages
- `{updatedMCPToolOutput}` — for MCP tools, hooks can modify the tool output before
  it is mapped to a tool_result block

**`runPostToolUseFailureHooks()`** yields attachment/progress messages from hooks that
observe tool failures.

### 5.5 Hook Execution Timing

Hooks run in parallel per event. Wall-clock timing is tracked and logged:
- `HOOK_TIMING_DISPLAY_THRESHOLD_MS = 500` — hooks taking longer than this show an
  inline timing summary (Anthropic internal only).
- `SLOW_PHASE_LOG_THRESHOLD_MS = 2000` — hooks exceeding this threshold log a debug
  warning, matching the Bash tool's progress threshold.

The default per-hook execution timeout is `TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10 * 60 * 1000`
(10 minutes). `SessionEnd` hooks have a much tighter timeout of 1.5 seconds to avoid
blocking shutdown.

---

## 6. Permission Checking

> **Source:** `src/services/tools/toolHooks.ts` (`resolveHookPermissionDecision`),
> `src/services/tools/toolExecution.ts` (lines 599-1104)

Permission checking is deeply interleaved with tool execution. The resolution follows a
multi-stage pipeline that combines hook decisions, rule-based checks, and interactive
prompts.

### 6.1 Resolution Pipeline

```
PreToolUse hooks ──→ hookPermissionResult (allow/deny/ask/undefined)
        │
        ▼
resolveHookPermissionDecision()
        │
        ├── Hook says 'allow':
        │     ├── Tool requires user interaction & no updatedInput? → canUseTool()
        │     ├── requireCanUseTool set? → canUseTool()
        │     ├── Rule-based check (deny/ask rules in settings.json):
        │     │     ├── null (no rule) → hook allow wins, skip prompt
        │     │     ├── deny → deny overrides hook allow
        │     │     └── ask → interactive dialog despite hook approval
        │     └── Otherwise → hook allow accepted
        │
        ├── Hook says 'deny': → deny (immediate)
        │
        ├── Hook says 'ask': → canUseTool() with forceDecision
        │
        └── No hook decision: → canUseTool() (normal flow)
                │
                ├── Rule evaluation (allow/deny/ask rules)
                ├── Mode transformation (auto→classifier, dontAsk→deny)
                ├── AI classifier (auto mode)
                ├── Denial limit tracking
                └── Interactive dialog (if needed)
```

A critical invariant: **hook 'allow' does NOT bypass deny/ask rules from settings.json**.
This prevents a compromised or misconfigured hook from overriding explicit user security
policy.

### 6.2 Speculative Classifier

For Bash tool calls, `startSpeculativeClassifierCheck()` is called early — before hooks
and permission resolution — so the AI classifier runs in parallel with those checks.
The classifier result is awaited only when the permission flow reaches the `auto` mode
path, avoiding serial latency in the common case.

### 6.3 Input Mutation

Both hooks and the permission system can mutate tool input:

- **Hook `updatedInput`**: PreToolUse hooks can return modified input that replaces the
  model's original. If a hook provides `updatedInput` for an interactive tool (like
  `AskUserQuestionTool`), the hook IS the user interaction — the tool is treated as
  non-interactive for the rule-check path.

- **Permission `updatedInput`**: The permission dialog can return modified input (e.g.,
  `SedEditPermissionRequest` injects `_simulatedSedEdit` after user approval).

- **`backfillObservableInput`**: Some tools fill derived fields on a clone for hooks to
  observe (e.g., expanding `~` in file paths). The original model-provided input is
  preserved for `tool.call()` to maintain transcript stability.

### 6.4 Permission Decision Telemetry

Every permission decision is logged via OTel with a `source` label:

| Source | Meaning |
|--------|---------|
| `config` | Rule-based, mode-based, or classifier decision |
| `hook` | Hook provided the decision |
| `user_permanent` | User clicked "Always allow" (persisted to settings) |
| `user_temporary` | User clicked "Allow once" (session-scoped) |
| `user_reject` | User denied the operation |

---

## 7. Progress Tracking

> **Source:** `src/services/tools/toolExecution.ts` (lines 502-570),
> `src/services/tools/StreamingToolExecutor.ts` (lines 366-378)

### 7.1 The onProgress Callback

Tools report progress through a callback passed to `tool.call()`:

```typescript
progress => {
  onToolProgress({
    toolUseID: progress.toolUseID,
    data: progress.data,
  })
}
```

Progress data is typed as a discriminated union (`ToolProgressData`) with variants for
different tool types: `BashProgress` (stdout/stderr streaming, timing), `MCPProgress`,
`AgentToolProgress`, `WebSearchProgress`, `REPLToolProgress`, `SkillToolProgress`,
`TaskOutputProgress`, etc.

### 7.2 Progress in the Stream Adapter

`streamedCheckPermissionsAndCallTool()` wraps progress events into `ProgressMessage`
objects and enqueues them into the `Stream<MessageUpdateLazy>`. The progress callback
also logs a `tengu_tool_use_progress` analytics event for each progress emission.

### 7.3 Progress in StreamingToolExecutor

The streaming executor separates progress messages from result messages. When a tool
emits a progress message (`update.message.type === 'progress'`), it is stored in the
tool's `pendingProgress` array rather than the `results` array. A resolve callback
(`progressAvailableResolve`) is signaled to wake up `getRemainingResults()` if it is
blocked waiting for tool completion.

`getCompletedResults()` always yields pending progress messages first, regardless of
tool status or ordering constraints. This ensures the UI reflects real-time progress
(spinner updates, streaming output) even when the tool producing that progress is
blocked behind a non-concurrent sibling.

### 7.4 Hook Progress

Hooks also emit progress through the same pipeline. Hook progress messages carry
`HookProgress` data (distinct from `ToolProgressData`) and are yielded to the caller
via the same mechanism. The `startHookProgressInterval()` utility emits periodic progress
events while a hook is running, keeping the UI spinner active.

---

## 8. Error Recovery

> **Source:** `src/services/tools/toolExecution.ts` (lines 469-490, 1589-1745),
> `src/services/tools/StreamingToolExecutor.ts` (lines 153-204, 276-345)

### 8.1 Error Classification

Tool execution errors are classified for telemetry via `classifyToolError()`:

- `TelemetrySafeError` — uses the error's `telemetryMessage` (pre-vetted for safety)
- Node.js filesystem errors — logs the errno code (`ENOENT`, `EACCES`, etc.)
- Named errors (`ShellError`, `ImageSizeError`) — uses the stable `.name` property
- Generic `Error` — falls back to `'Error'`
- Non-Error throwables — `'UnknownError'`

This classification avoids logging minified constructor names (which are meaningless
3-character identifiers in production builds) while remaining telemetry-safe.

### 8.2 Bash Error Cascading

When a Bash tool errors, sibling tools are cancelled via the executor's
`siblingAbortController`. This design reflects implicit dependency chains in Bash
commands — if `mkdir` fails, subsequent commands in the same response are likely
pointless. Only Bash errors trigger this cascade; other tool types (Read, WebFetch, Grep,
etc.) are treated as independent, so one failure does not cancel siblings.

The cascade flow:

```
Bash tool errors
    │
    ├── hasErrored = true
    ├── siblingAbortController.abort('sibling_error')
    │     └── per-tool abort controllers receive the signal
    │
    └── Other tools check getAbortReason():
          ├── Queued tools → synthetic error, never start
          └── Executing tools → synthetic error on next generator yield
```

The synthetic error message includes a description of the errored tool (e.g.,
`"Cancelled: parallel tool call Bash(mkdir /tmp/foo...) errored"`), helping the model
understand what happened.

### 8.3 User Interruption

When the user presses Escape or submits a new message while tools are running, the
abort controller fires with an `'interrupt'` reason. The executor checks each tool's
`interruptBehavior`:

- `'cancel'` — tool is cancelled immediately with a synthetic `user_interrupted` error
  using `REJECT_MESSAGE`.
- `'block'` (default) — tool continues executing; the interrupt is not propagated to it.

The `updateInterruptibleState()` method tracks whether ALL executing tools are
interruptible, which determines whether the UI shows an interrupt affordance.

### 8.4 Streaming Fallback

When the API response needs to be retried (e.g., due to a network error mid-stream),
`StreamingToolExecutor.discard()` is called. This sets a flag that:

1. Prevents queued tools from starting.
2. Causes executing tools to receive `streaming_fallback` synthetic errors.
3. Makes `getCompletedResults()` and `getRemainingResults()` return immediately.

A fresh `StreamingToolExecutor` is then created for the retried response.

### 8.5 MCP Authentication Errors

`McpAuthError` is handled specially: the MCP client status is updated to `'needs-auth'`
in the app state, which updates the UI to show that the server needs re-authorization.
The error is still returned to the model as a tool_result error.

### 8.6 PostToolUseFailure Hooks

After any tool execution failure (including user interrupts), `runPostToolUseFailureHooks()`
runs hooks that observe the failure. These hooks can provide additional context, blocking
errors, or UI messages. They share the same structure as `PostToolUse` hooks but receive
the error string and an `isInterrupt` flag instead of the tool output.

---

## 9. Result Rendering

> **Source:** `src/services/tools/toolExecution.ts` (lines 1206-1588),
> `src/utils/toolResultStorage.ts`

### 9.1 Mapping Tool Output to API Format

After `tool.call()` returns, its result is mapped to the Anthropic API's
`ToolResultBlockParam` format via `tool.mapToolResultToToolResultBlockParam(result, toolUseID)`.
This mapping is tool-specific — each tool defines how its output structure serializes
to the API format (typically as a text block with formatted output).

The mapped result is cached (`mappedToolResultBlock`) and reused when hooks do not modify
the output, avoiding redundant remapping.

### 9.2 Large Result Persistence

If the mapped content exceeds the tool's `maxResultSizeChars` (clamped by the global
`DEFAULT_MAX_RESULT_SIZE_CHARS = 50,000`), the full content is persisted to disk and
replaced with a preview:

```
<persisted-output>
Output too large (150.0 KB). Full output saved to: /path/to/tool-results/abc123.txt

Preview (first 2.0 KB):
[first 2000 bytes of content]
...
</persisted-output>
```

The file is written to `<projectDir>/<sessionId>/tool-results/<toolUseId>.[txt|json]`
using the `wx` flag (write-exclusive) to avoid re-writing the same content on subsequent
API turns when microcompact replays messages.

### 9.3 Empty Result Handling

Empty tool results are a known hazard: some models (notably Capybara) interpret an empty
`tool_result` at the prompt tail as a turn boundary and emit the `\n\nHuman:` stop
sequence with zero output. To prevent this, empty results are replaced with a short marker:

```
(toolName completed with no output)
```

### 9.4 Content Block Assembly

The final message for the API includes several content blocks assembled in order:

1. **Tool result block** — the mapped (possibly persisted) tool output.
2. **Accept feedback** — if the user typed feedback while approving the tool call.
3. **Content blocks from permission** — images pasted during the permission dialog.

Each image gets a sequential `imagePasteId` for distinct rendering labels.

### 9.5 MCP Tool Ordering

MCP tools have a special ordering constraint: the tool result is added to the message
list AFTER `PostToolUse` hooks run, not before. This allows hooks to modify the MCP
output via `updatedMCPToolOutput` before the result is mapped. For non-MCP tools, the
result is added first, then hooks run — hooks see the output but cannot modify it.

---

## 10. Concurrency

> **Source:** `src/Tool.ts` (`isConcurrencySafe`),
> `src/services/tools/StreamingToolExecutor.ts`,
> `src/services/tools/toolOrchestration.ts`

### 10.1 The isConcurrencySafe Contract

Every tool must implement `isConcurrencySafe(input): boolean`. This method receives the
parsed input and returns whether the tool call can safely execute in parallel with other
concurrent-safe calls. The evaluation is per-invocation, not per-tool-type — a Bash tool
calling `cat` is concurrent-safe, while a Bash tool calling `rm -rf` is not.

Examples from the codebase:

| Tool | Concurrency Logic |
|------|------------------|
| `FileReadTool` | Always concurrent-safe (read-only) |
| `GrepTool` | Always concurrent-safe (read-only) |
| `GlobTool` | Always concurrent-safe (read-only) |
| `BashTool` | Concurrent-safe only for read-only commands (parsed via shell-quote) |
| `FileEditTool` | Never concurrent-safe (writes files) |
| `FileWriteTool` | Never concurrent-safe (writes files) |
| `AgentTool` | Never concurrent-safe (spawns subagents with side effects) |
| `AskUserQuestionTool` | Never concurrent-safe (requires user interaction) |

If `isConcurrencySafe` throws (e.g., due to a shell-quote parse failure for a complex
Bash command), the tool is conservatively treated as non-concurrent-safe.

### 10.2 Execution Semantics

The concurrency model enforces a strict ordering guarantee:

1. Non-concurrent tools execute alone — no other tool runs simultaneously.
2. Between non-concurrent tools, execution order matches the model's emission order.
3. Concurrent-safe tools between two non-concurrent tools run in parallel.
4. The maximum concurrency for the batch path is `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`
   (default 10, configurable via environment variable).

In the streaming executor, this is enforced dynamically: `processQueue()` starts
concurrent tools eagerly but halts at the first non-concurrent tool it cannot execute.
In the batch orchestrator, this is enforced structurally via `partitionToolCalls()`.

### 10.3 Context Modifier Constraints

Context modifiers (functions that transform `ToolUseContext`) are currently NOT supported
for concurrent tools. The code documents this limitation:

```typescript
// NOTE: we currently don't support context modifiers for concurrent
//       tools. None are actively being used, but if we want to use
//       them in concurrent tools, we need to support that here.
```

For non-concurrent tools, modifiers are applied immediately after the tool completes,
ensuring subsequent tools see the updated context. For the batch path, concurrent-tool
modifiers are queued and applied after the entire batch finishes, in tool order.

---

## 11. Tool Result Budget

> **Source:** `src/utils/toolResultStorage.ts`, `src/constants/toolLimits.ts`

### 11.1 Two-Level Size Enforcement

Tool results are subject to two independent size limits:

**Per-tool limit**: Each tool declares `maxResultSizeChars` (or inherits the default of
50,000). The global cap is `DEFAULT_MAX_RESULT_SIZE_CHARS = 50_000`. Results exceeding
this are persisted to disk and replaced with a preview at the `processToolResultBlock`
/ `processPreMappedToolResultBlock` stage. GrowthBook flag `tengu_satin_quoll` can
override the threshold per tool name.

Tools with `maxResultSizeChars: Infinity` (like `FileReadTool`) opt out of persistence
entirely — their size is bounded by their own `maxTokens` logic, and persisting their
output to a file the model reads back with `Read` would be circular.

**Per-message aggregate limit**: When N parallel tools each produce large results in a
single turn, their combined size can be enormous even if each individually passes the
per-tool limit. `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200,000` caps the aggregate
size per API-level user message. Overridable via GrowthBook flag `tengu_hawthorn_window`.

### 11.2 Content Replacement State

The per-message budget is managed by `ContentReplacementState`:

```typescript
type ContentReplacementState = {
  seenIds: Set<string>                    // tool_use_ids that have been evaluated
  replacements: Map<string, string>       // id → cached replacement string
}
```

Once a tool result has been seen, its fate is **frozen**: replaced results always get the
same replacement re-applied (from the cached string, with zero I/O), and unreplaced
results are never replaced later. This stability is critical for prompt cache
preservation — changing a previously-sent result would invalidate the cache prefix.

### 11.3 Enforcement Algorithm

`enforceToolResultBudget()` runs once per API turn on the full message array:

1. **Group candidates by API-level message** — consecutive user messages (not separated
   by assistant messages) are grouped together, matching how
   `normalizeMessagesForAPI` merges them on the wire.

2. **Partition by prior decision** for each group:
   - `mustReapply`: previously replaced — re-apply from cache (Map lookup, no I/O)
   - `frozen`: previously seen but unreplaced — cannot be touched
   - `fresh`: never seen — eligible for replacement

3. **Select fresh results to replace** — sort by size descending, replace the largest
   until the group's model-visible total (frozen + remaining fresh) is at or under budget.

4. **Persist and record** — selected results are persisted concurrently. New replacements
   are written to the transcript for resume reconstruction.

### 11.4 Resume Stability

On session resume, `reconstructContentReplacementState()` rebuilds the state from
transcript records. Every candidate tool_use_id in the loaded messages is marked as
seen (frozen), and replacement strings are populated from stored records. This ensures
the budget makes the same choices it made in the original session — a hard requirement
for prompt cache stability across resumes.

For subagent (fork) resumes, `reconstructForSubagentResume()` additionally uses the
parent's live replacement state to fill gaps for inherited replacements that were not
written to the subagent's own sidechain records.

---

## 12. Design Principles

### 12.1 Streaming-First Latency Optimization

The streaming executor starts tools as early as possible — during the API response
stream, not after it completes. For responses with multiple tools, the first tool may
finish executing before the last tool's block has even been fully received. This reduces
end-to-end turn latency by overlapping network I/O with tool execution.

### 12.2 Order Preservation with Opportunistic Parallelism

Results are yielded in the order tools were received, even when concurrent tools complete
out of order. Non-concurrent tools act as barriers — the executor will not yield results
past a still-executing non-concurrent tool. This preserves the model's intended execution
semantics while extracting parallelism where safe.

### 12.3 Fail-Safe Concurrency Classification

The concurrency system is conservative by design. If `isConcurrencySafe` throws, the
tool runs serially. If input parsing fails, the tool runs serially. Unknown tools are
treated as concurrent-safe only because they immediately produce error results (no actual
execution occurs). The default for `interruptBehavior` is `'block'`, not `'cancel'`.

### 12.4 Hook Allow Cannot Override Deny Rules

A critical security invariant: when a PreToolUse hook returns `{behavior: 'allow'}`,
the system still checks `checkRuleBasedPermissions()`. If the user has configured a
deny rule in settings.json for this tool, the deny rule wins. This prevents a
compromised or misconfigured hook from silently bypassing explicit user security policy.

### 12.5 Abort Signal Hierarchy

The three-level abort controller hierarchy (query → sibling → tool) provides precise
control over cancellation scope. Bash errors kill siblings but not the query. Permission
denials kill the tool and bubble up to end the query turn. User interrupts respect
per-tool interrupt behavior. Streaming fallbacks discard all pending work cleanly.

### 12.6 Stable Persistence for Cache Preservation

Both the per-tool persistence (`processToolResultBlock`) and the per-message budget
(`enforceToolResultBudget`) are designed around prompt cache stability. Decisions are
frozen once made. Replacements are cached as exact strings and re-applied from cache
on every subsequent turn — no re-computation, no file I/O, byte-identical output
guaranteed. File writes use the `wx` flag to skip re-writing unchanged content.

### 12.7 Separation of Mapping and Persistence

Tool result processing is a two-stage pipeline: first `mapToolResultToToolResultBlockParam`
converts tool-specific output to the API format, then `maybePersistLargeToolResult`
conditionally offloads large results to disk. This separation means the mapping logic
is unaware of size budgets, and the persistence logic is unaware of tool-specific
output formats. The pre-mapped block is cached so non-MCP tools (where hooks do not
modify output) avoid redundant remapping.

### 12.8 Progress as a First-Class Concern

Progress messages are separated from result messages at every level of the system.
In the streaming executor, they bypass ordering constraints — a progress update from
a tool blocked behind a non-concurrent sibling is still yielded immediately. In the
stream adapter, they are enqueued as they arrive rather than batched with the final
result. This ensures the UI always reflects what is actually happening, even during
complex multi-tool execution with concurrency barriers.
