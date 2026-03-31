# Claude Code: Query Loop State Machine — Design Specification

This document analyzes the query loop architecture of Claude Code — Anthropic's
agentic CLI tool — focusing on its core execution loop: the state machine that
drives every conversation turn from user input through LLM calls, tool execution,
error recovery, and eventual termination.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Architecture](#2-architecture)
- [3. State Management](#3-state-management)
- [4. Context Assembly](#4-context-assembly)
- [5. LLM Call Flow](#5-llm-call-flow)
- [6. Tool Execution](#6-tool-execution)
- [7. Error Recovery](#7-error-recovery)
- [8. Attachment System](#8-attachment-system)
- [9. QueryEngine Wrapper](#9-queryengine-wrapper)
- [10. Startup Path](#10-startup-path)
- [11. Design Principles](#11-design-principles)

---

## 1. Overview

The query loop is the beating heart of Claude Code. It implements the agentic
contract: the model produces a response, the system executes any requested tools,
feeds results back, and loops until the model stops requesting tools. This
deceptively simple contract hides a state machine of considerable complexity —
one that handles automatic context compaction, streaming tool execution, seven
distinct error recovery paths, multi-turn attachment injection, token budget
enforcement, and hook-driven lifecycle events.

The loop lives in `src/query.ts` as an async generator function, `query()`,
that yields a stream of events (assistant messages, tool results, system
notifications, stream metadata) and returns a `Terminal` status on completion.
Its caller — either the REPL's `ask()` function or the SDK's `QueryEngine` —
consumes this stream and maps events to their respective UI or SDK wire formats.

The fundamental design decision is that the query loop is **a single
`while(true)` loop with explicit continuation sites**, not a recursive call
chain. Each iteration is a complete LLM call cycle: prepare context, call the
API, stream the response, execute tools, decide whether to continue. The `State`
object is reassigned at each continuation site, making the loop's control flow
visible as data rather than buried in call stack depth.

---

## 2. Architecture

### 2.1 The Generator Pipeline

The query loop is structured as a two-layer async generator:

```
query(params) → queryLoop(params, consumedCommandUuids)
```

The outer `query()` is a thin wrapper that delegates to `queryLoop()` via
`yield*`. Its sole purpose is command lifecycle tracking: it collects UUIDs of
consumed queued commands during the loop and notifies them as `'completed'` only
when the loop returns normally (not on throw or `.return()` cancellation). This
asymmetric lifecycle signal lets the command system distinguish "processed
successfully" from "consumed but failed."

The inner `queryLoop()` is the actual state machine. It is an
`AsyncGenerator<StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage, Terminal>`
— yielding intermediate events during execution and returning a terminal status
with a reason string on completion.

### 2.2 The Single While(True) Loop

The loop structure is a textbook state machine pattern:

```typescript
while (true) {
  // 1. Destructure current state
  let { toolUseContext } = state
  const { messages, autoCompactTracking, ... } = state

  // 2. Pre-processing: snip, microcompact, context collapse, autocompact
  // 3. Prepare and call the LLM API (streaming)
  // 4. Process streamed response, extract tool_use blocks
  // 5. If no tool calls: run stop hooks, check token budget, return Terminal
  // 6. Execute tools (parallel or serial)
  // 7. Gather attachments (memory, skills, queued commands)
  // 8. Build next State, `state = next; continue`
}
```

Each continuation site constructs a new `State` object with a `transition` field
that records *why* the loop is continuing. This makes the control flow
machine-inspectable — tests can assert that a specific recovery path fired by
examining `state.transition.reason` rather than parsing message contents.

### 2.3 Continuation Sites

There are seven `continue` statements in the loop body, each representing a
distinct reason to re-enter the LLM:

| Continuation | `transition.reason` | When |
|---|---|---|
| Context collapse drain | `collapse_drain_retry` | Prompt-too-long, staged collapses drained |
| Reactive compact | `reactive_compact_retry` | Prompt-too-long, emergency compaction succeeded |
| Max output tokens escalate | `max_output_tokens_escalate` | Output capped at 8k, retry at 64k |
| Max output tokens recovery | `max_output_tokens_recovery` | Output still capped, inject resume prompt |
| Stop hook blocking | `stop_hook_blocking` | Stop hook reported blocking errors |
| Token budget continuation | `token_budget_continuation` | Under 90% of token budget, inject nudge |
| Next turn | `next_turn` | Tools executed, normal continuation |

Each of these assigns `state = next` with a freshly constructed `State` object,
then executes `continue` to re-enter the loop.

---

## 3. State Management

### 3.1 The Mutable State Type

The loop carries a `State` object that is destructured at the top of each
iteration and reassigned at each continuation site:

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

The `transition` field is `undefined` on the first iteration and set at every
continuation site thereafter. It records the reason for continuation and any
associated metadata (e.g., `committed` count for collapse drain, `attempt`
number for max output tokens recovery).

**Key design decision**: State is reassigned as a whole object at each
continuation site rather than mutating individual fields. The comment in the
source explains: "Continue sites write `state = { ... }` instead of 9 separate
assignments." This makes it impossible to forget a field and creates a visible
checkpoint at each transition.

### 3.2 Loop-Local Mutable Variables

Three additional mutable variables live outside the `State` struct because they
either survive compaction boundaries or are constant for the loop's lifetime:

- **`taskBudgetRemaining`** (`number | undefined`): Tracks cumulative API task
  budget spend across compaction boundaries. Before compaction, the server can
  compute remaining budget from the full history. After compaction, the
  summarized history hides the true spend, so the client must track it
  explicitly and pass `remaining` in subsequent API calls.

- **`config`** (`QueryConfig`): An immutable snapshot of environment variables,
  Statsig gates, and session identifiers taken once at loop entry. Separating
  this from the per-iteration `State` keeps the loop closer to a pure reducer
  over `(state, event, config)`.

- **`budgetTracker`** (`BudgetTracker | null`): Mutable tracker for the
  `TOKEN_BUDGET` feature that records continuation counts, delta tokens between
  checks, and a diminishing-returns detector.

### 3.3 QueryParams

The loop's entry point takes `QueryParams`, which bundles the immutable inputs:

```typescript
export type QueryParams = {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  canUseTool: CanUseToolFn
  toolUseContext: ToolUseContext
  fallbackModel?: string
  querySource: QuerySource
  maxOutputTokensOverride?: number
  maxTurns?: number
  skipCacheWrite?: boolean
  taskBudget?: { total: number }
  deps?: QueryDeps
}
```

The `deps` field enables dependency injection for testing. In production,
`productionDeps()` provides the real implementations:

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

### 3.4 QueryConfig

`buildQueryConfig()` snapshots environment and Statsig state once per `query()`
invocation:

```typescript
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean
    emitToolUseSummaries: boolean
    isAnt: boolean
    fastModeEnabled: boolean
  }
}
```

The comment in `src/query/config.ts` explains the design intent: "Separating
these from the per-iteration State struct and the mutable ToolUseContext makes
future step() extraction tractable — a pure reducer can take (state, event,
config) where config is plain data."

Feature flags via `feature()` (Bun's compile-time tree-shaking boundaries) are
intentionally excluded from `QueryConfig` — they must stay inline at guarded
blocks for dead-code elimination to work.

---

## 4. Context Assembly

### 4.1 System and User Context

Two memoized async functions in `src/context.ts` compute environmental context
once per session:

**`getSystemContext()`** collects git metadata:
- Branch name, default branch, short status (capped at 2000 characters),
  recent 5 commits, user name
- All git commands run in parallel via `Promise.all()`
- Skipped in remote sessions (`CLAUDE_CODE_REMOTE`) or when git instructions
  are disabled
- Supports a `systemPromptInjection` for cache-breaking (ant-only debugging)

**`getUserContext()`** collects project instructions:
- Loads CLAUDE.md files from the four-level hierarchy (managed, user, project,
  local) via `getMemoryFiles()` and `getClaudeMds()`
- Applies `filterInjectedMemoryFiles()` to strip suspicious patterns
- Caches the result for the auto-mode classifier (avoiding an import cycle)
- Includes the current date string
- Respects `CLAUDE_CODE_DISABLE_CLAUDE_MDS` and `--bare` mode

Both use lodash `memoize` — they compute once and return the cached result for
the session's lifetime. The cache is explicitly invalidable when
`systemPromptInjection` changes.

### 4.2 Context Integration into the Loop

At the top of each loop iteration, context is integrated in two stages:

1. **System context appended to system prompt**: `appendSystemContext(systemPrompt, systemContext)`
   joins system prompt parts with key-value pairs from the system context dict.

2. **User context prepended to messages**: `prependUserContext(messagesForQuery, userContext)`
   injects CLAUDE.md content and the current date as the first user message.
   This happens at the `deps.callModel()` call site, so the original
   `messagesForQuery` array is not modified — the prepended version is
   ephemeral, used only for the API call.

### 4.3 Pre-Call Processing Pipeline

Before each API call, the message array passes through a five-stage processing
pipeline:

```
messages
  → getMessagesAfterCompactBoundary()    // trim to post-compact region
  → applyToolResultBudget()              // enforce per-message size limits
  → snipCompactIfNeeded()                // remove stale segments (HISTORY_SNIP)
  → microcompactMessages()               // fine-grained compaction
  → applyCollapsesIfNeeded()             // context collapse projections
  → autoCompactIfNeeded()                // full summarization if needed
```

The ordering is significant. Snip and microcompact run *before* autocompact so
that if their token savings bring the context under the autocompact threshold,
autocompact becomes a no-op and granular context is preserved. Context collapse
runs before autocompact for the same reason — collapses are read-time
projections that keep more structure than a flat summary.

---

## 5. LLM Call Flow

### 5.1 API Call Structure

Each loop iteration makes a single streaming API call via `deps.callModel()`.
The call receives:

- **Messages**: The post-processed message array with user context prepended
- **System prompt**: The full system prompt with system context appended
- **Thinking config**: Extended reasoning parameters (adaptive, enabled, or
  disabled)
- **Tools**: The current tool definitions (refreshed between turns for MCP)
- **Signal**: The abort controller's signal for cancellation
- **Options**: A bag of runtime configuration including model selection, fast
  mode, task budget, query tracking, MCP tools, agent definitions, and more

The call is wrapped in a retry loop for model fallback: if the primary model
fails with a `FallbackTriggeredError`, the loop switches to the fallback model
and retries. This handles high-demand scenarios where the primary model is
temporarily unavailable.

### 5.2 Streaming Response Processing

The response is consumed as an async iterable of messages. During streaming,
the loop performs several concurrent activities:

**Tool use block extraction**: As assistant messages arrive, `tool_use` content
blocks are extracted and accumulated in `toolUseBlocks[]`. The `needsFollowUp`
flag is set to `true` whenever a tool use block appears — this is the sole
loop-exit signal.

**Streaming tool execution**: When the `streamingToolExecution` gate is enabled,
a `StreamingToolExecutor` begins executing tools as their blocks arrive during
streaming, rather than waiting for the full response. Tool results that complete
during streaming are yielded immediately.

**Error withholding**: Certain recoverable errors are withheld from the yield
stream until the recovery logic determines whether they can be handled:
- Prompt-too-long errors (413) — withheld for collapse drain / reactive compact
- Media size errors — withheld for reactive compact strip-retry
- Max output tokens errors — withheld for escalation / recovery

**Observable input backfill**: Before yielding assistant messages, tool_use
blocks are cloned and their inputs are enriched via `backfillObservableInput()`
— adding derived fields that SDK stream consumers and hooks need without
mutating the original (which would break prompt caching via byte mismatch).

**Tombstone emission**: If a streaming fallback occurs mid-response, all
partially-streamed assistant messages are yielded as `tombstone` messages so
they are removed from the UI and transcript. Their thinking blocks carry
invalid signatures that would cause API errors if replayed.

### 5.3 Post-Streaming

After the streaming loop completes:

1. **Post-sampling hooks** fire asynchronously (fire-and-forget)
2. **Abort check**: If the abort signal fired during streaming, synthetic
   tool_result blocks are generated for any orphaned tool_use blocks, and the
   loop returns `{ reason: 'aborted_streaming' }`
3. **Pending tool use summary**: The summary from the previous turn (generated
   asynchronously during this turn's model streaming) is awaited and yielded

---

## 6. Tool Execution

### 6.1 Dual Execution Paths

Tool execution follows one of two paths depending on the `streamingToolExecution`
gate:

**Streaming path** (`StreamingToolExecutor`): Tools begin executing as their
`tool_use` blocks arrive during streaming. The executor tracks each tool's
status through a lifecycle: `queued → executing → completed → yielded`. Results
are buffered and emitted in the order tools were received, preserving the
illusion of sequential execution even when tools complete out of order. After
streaming ends, `getRemainingResults()` drains any tools still in flight.

**Post-streaming path** (`runTools()`): All tools execute after the full
response is received. This is the legacy path and the fallback when streaming
execution is disabled.

### 6.2 Parallel vs. Serial Execution

The `runTools()` orchestrator in `src/services/tools/toolOrchestration.ts`
partitions tool calls into batches:

```typescript
function partitionToolCalls(
  toolUseMessages: ToolUseBlock[],
  toolUseContext: ToolUseContext,
): Batch[]
```

Each batch is either:
- **Concurrency-safe**: Multiple consecutive read-only tools that can run in
  parallel (up to `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`, default 10)
- **Non-concurrent**: A single tool that requires exclusive access (writes,
  side effects)

The concurrency safety check uses each tool's `isConcurrencySafe()` method,
which examines the parsed input to determine if the operation is read-only.
Context modifiers from concurrent tools are queued and applied in order after
the batch completes, preserving sequential semantics for state updates while
parallelizing the I/O.

### 6.3 Tool Result Processing

Each tool execution produces a `Message` (typically a `UserMessage` containing
a `tool_result` content block) that is yielded to the caller and accumulated
in `toolResults[]`. Tool results are normalized via `normalizeMessagesForAPI()`
to ensure they conform to the API's expected format.

If a tool's `hook_stopped_continuation` attachment appears in the results, the
`shouldPreventContinuation` flag is set, causing the loop to return
`{ reason: 'hook_stopped' }` after all tools complete.

### 6.4 Tool Use Summary Generation

After each tool batch completes, the system optionally generates a
human-readable summary of what the tools did. This summary is generated
asynchronously via a lightweight Haiku model call and yielded at the *start*
of the next iteration (overlapping the ~1 second Haiku call with the ~5-30
second main model streaming). The summary is only generated for top-level
agents (not subagents, which don't surface in the mobile UI).

---

## 7. Error Recovery

The query loop implements seven distinct recovery paths, each with explicit
termination conditions to prevent infinite retry loops.

### 7.1 Context Collapse Drain (`collapse_drain_retry`)

> **Guard**: `state.transition?.reason !== 'collapse_drain_retry'`

When the API returns a prompt-too-long error and the `CONTEXT_COLLAPSE` feature
is enabled, the loop first attempts to drain all staged context collapses. This
is the cheapest recovery — it commits pending collapse summaries that were
deferred for read-time projection, reducing the context without a new API call.

If the previous iteration was already a collapse drain retry that still produced
a 413, this path is skipped and the loop falls through to reactive compact.

### 7.2 Reactive Compact (`reactive_compact_retry`)

> **Guard**: `hasAttemptedReactiveCompact` flag (one-shot per turn)

When collapse drain is insufficient or unavailable, the loop attempts emergency
compaction via `tryReactiveCompact()`. This performs a full summarization of the
conversation history. The `hasAttemptedReactiveCompact` flag ensures this fires
at most once per turn — if the post-compact context still triggers a 413, the
error surfaces to the user.

This path also handles media size errors (oversized images/PDFs). If the
oversized media is in the preserved tail after compaction, the next turn will
media-error again, but the one-shot guard prevents a spiral.

If reactive compact fails or is not available, the withheld error is surfaced
and `executeStopFailureHooks()` fires. The loop does *not* fall through to stop
hooks — the model never produced a valid response, so hooks have nothing
meaningful to evaluate. Running stop hooks on prompt-too-long would create a
death spiral: error -> hook blocking -> retry -> error -> ...

### 7.3 Max Output Tokens Escalation (`max_output_tokens_escalate`)

> **Guard**: `maxOutputTokensOverride === undefined` (fires once per turn)

When the model hits the output token cap and the system was using the default
capped 8k limit, the loop retries the *same* request with `ESCALATED_MAX_TOKENS`
(64k). No meta-message is injected — the model simply gets more room. This fires
once per turn (guarded by the override being `undefined`), then falls through to
multi-turn recovery if 64k also hits the cap.

Gated behind the `tengu_otk_slot_v1` GrowthBook feature value and only active
when the user has not set `CLAUDE_CODE_MAX_OUTPUT_TOKENS` explicitly.

### 7.4 Max Output Tokens Recovery (`max_output_tokens_recovery`)

> **Guard**: `maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT` (3 attempts)

After escalation fails or is not applicable, the loop injects a synthetic user
message:

```
Output token limit hit. Resume directly — no apology, no recap of what you
were doing. Pick up mid-thought if that is where the cut happened. Break
remaining work into smaller pieces.
```

The assistant messages from the truncated response are preserved so the model
can see where it was cut off. The recovery counter increments and the loop
continues. After 3 attempts, the withheld error surfaces and the loop exits.

### 7.5 Stop Hook Blocking (`stop_hook_blocking`)

> **Guard**: `hasAttemptedReactiveCompact` is preserved (not reset)

When the model produces a complete response with no tool calls and stop hooks
report blocking errors, the error messages are appended to the conversation and
the loop continues. The model sees the hook feedback and can adjust its response.

A critical subtlety: the `hasAttemptedReactiveCompact` flag is *preserved* across
this continuation. The source comment explains: "if compact already ran and
couldn't recover from prompt-too-long, retrying after a stop-hook blocking error
will produce the same result. Resetting to false here caused an infinite loop:
compact -> still too long -> error -> stop hook blocking -> compact -> ...
burning thousands of API calls."

Stop hooks also handle the `preventContinuation` case, where a hook explicitly
signals that the loop should terminate (returns `{ reason: 'stop_hook_prevented' }`).

### 7.6 Token Budget Continuation (`token_budget_continuation`)

> **Guard**: `turnTokens < budget * COMPLETION_THRESHOLD` (90%) and no
> diminishing returns

The `TOKEN_BUDGET` feature allows callers to specify a target output token
budget for a turn. The `BudgetTracker` monitors cumulative output tokens after
each model response. If the model has produced less than 90% of the budget and
is not showing diminishing returns (fewer than 500 new tokens for two
consecutive checks after 3+ continuations), a nudge message is injected and the
loop continues.

```typescript
export type BudgetTracker = {
  continuationCount: number
  lastDeltaTokens: number
  lastGlobalTurnTokens: number
  startedAt: number
}
```

The diminishing returns detector prevents infinite continuation when the model
has stopped producing meaningful output.

### 7.7 Next Turn (`next_turn`)

The normal continuation path: tools were executed, results collected, attachments
gathered. The loop assembles the full message array
(`messagesForQuery + assistantMessages + toolResults`) and continues. Recovery
counters (`maxOutputTokensRecoveryCount`, `hasAttemptedReactiveCompact`) are
reset because this is a fresh tool-result turn where previous error conditions
no longer apply.

Before continuing, the loop checks `maxTurns` — if the turn count exceeds the
limit, a `max_turns_reached` attachment is yielded and the loop returns
`{ reason: 'max_turns' }`.

---

## 8. Attachment System

Between tool execution and the next LLM call, the loop gathers "attachments" —
additional context that should be injected into the conversation for the next
turn. This system is the primary mechanism for feeding the model information it
did not explicitly request but needs to maintain coherence.

### 8.1 Queued Commands

The loop drains the process-global command queue at configurable priority levels.
The drain is agent-scoped: the main thread drains commands with
`agentId === undefined`, subagents drain only their own. Slash commands are
excluded from mid-turn drain (they go through `processSlashCommand` after the
turn ends). If a `Sleep` tool ran in this turn, the drain threshold is lowered
to include `'later'` priority commands.

### 8.2 Standard Attachments

`getAttachmentMessages()` produces a stream of attachment messages that may
include:
- File change notifications (edited files since the last turn)
- Plan mode instructions and plan file contents
- Skill listings and discovery results
- MCP resource references
- Task notifications from background agents

### 8.3 Memory Prefetch

A `MemoryPrefetch` is started once per user turn (before the loop) via
`startRelevantMemoryPrefetch()`. It runs a side-query to identify relevant
memory files based on the conversation content. The prefetch resolves
asynchronously while the model streams and tools execute.

At the attachment injection point, the loop checks if the prefetch has settled.
If so, it consumes the result, filtering out memories the model has already
accessed (via `readFileState`). If the prefetch has not settled, it is skipped
for this iteration and retried on the next — the prefetch gets as many chances
as there are loop iterations.

The prefetch uses `using` (explicit resource management) so it is disposed on
all generator exit paths, ensuring cleanup and telemetry even if the generator
is abandoned via `.return()`.

### 8.4 Skill Discovery Prefetch

Similar to memory prefetch, skill discovery runs per-iteration in the background
while the model streams. `startSkillDiscoveryPrefetch()` fires at the top of
each iteration and is consumed at the attachment injection point via
`collectSkillDiscoveryPrefetch()`. This replaces a blocking path that ran inside
`getAttachmentMessages` and found nothing 97% of the time in production.

### 8.5 Tool Refresh

After attachments are gathered, the loop calls `refreshTools()` to pick up
newly-connected MCP servers. If the tool list has changed, the `toolUseContext`
is updated with the new definitions.

---

## 9. QueryEngine Wrapper

### 9.1 Purpose and Scope

`QueryEngine` in `src/QueryEngine.ts` wraps the raw `query()` generator for
SDK and headless usage. It owns the conversation lifecycle:

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private hasHandledOrphanedPermission: boolean
  private readFileState: FileStateCache
  private discoveredSkillNames: Set<string>
  private loadedNestedMemoryPaths: Set<string>
}
```

One `QueryEngine` per conversation. Each `submitMessage()` call starts a new
turn within the same conversation. State persists across turns.

### 9.2 submitMessage Flow

The `submitMessage()` method is the per-turn entry point. Its flow:

1. **Clear turn-scoped state**: Reset `discoveredSkillNames`
2. **Set working directory**: `setCwd(cwd)`
3. **Wrap canUseTool**: Intercept permission decisions to track denials for SDK
   reporting
4. **Fetch system prompt parts**: `fetchSystemPromptParts()` loads the system
   prompt, user context, and system context in parallel. When a custom system
   prompt is provided, the default build and system context are skipped.
5. **Assemble system prompt**: Compose from default (or custom) + memory
   mechanics (if auto-mem path override is set) + append prompt
6. **Process user input**: Run through `processUserInput()` which handles slash
   commands, file attachments, image processing, and input validation
7. **Persist transcript**: Write the user's messages before entering the query
   loop so that `--resume` works even if the process is killed before the API
   responds
8. **Enter query loop**: Call `query()` with assembled parameters
9. **Process yielded messages**: Route each message type to the appropriate SDK
   output format, accumulate usage, track stop reasons, handle compaction
   boundaries
10. **Check budget and retry limits**: USD budget, structured output retry
    limits, max turns
11. **Yield result**: A terminal `SDKMessage` of type `result` with duration,
    cost, usage, and permission denials

### 9.3 File History and Read State

`QueryEngine` maintains a `readFileState` (`FileStateCache`) that tracks which
files the model has recently read. This cache serves two purposes:

1. **Post-compact restoration**: After compaction, the top 5 most recently
   accessed files are re-injected into context (see the context management
   design doc)
2. **Memory deduplication**: Memory prefetch results are filtered against this
   cache to avoid re-injecting files the model has already seen

The cache is cloned from the caller's `readFileCache` at construction and
returned via `getReadFileState()` for the caller to persist.

### 9.4 Compaction Boundary Handling

When a `compact_boundary` system message arrives from the query loop,
`QueryEngine` performs GC of pre-compaction messages:

```typescript
const mutableBoundaryIdx = this.mutableMessages.length - 1
if (mutableBoundaryIdx > 0) {
  this.mutableMessages.splice(0, mutableBoundaryIdx)
}
```

This releases pre-compaction messages for garbage collection. The query loop
internally uses `getMessagesAfterCompactBoundary()`, so only post-boundary
messages are needed going forward. This is critical for long SDK sessions where
unbounded message accumulation would cause memory exhaustion.

### 9.5 The `ask()` Convenience Wrapper

For one-shot SDK usage, the `ask()` function creates a temporary `QueryEngine`,
runs `submitMessage()`, and returns the result. It handles:

- `FileStateCache` cloning and restoration
- `snipReplay` injection for the `HISTORY_SNIP` feature (so feature-gated
  strings stay out of `QueryEngine`)

---

## 10. Startup Path

### 10.1 main.tsx Bootstrap

The entry point `src/main.tsx` orchestrates a carefully ordered startup
sequence designed to minimize time-to-first-token:

**Immediate side effects** (before any imports complete):
- `profileCheckpoint('main_tsx_entry')` — marks the entry timestamp
- `startMdmRawRead()` — fires MDM subprocess reads (plutil/reg query) in
  parallel with the remaining ~135ms of imports
- `startKeychainPrefetch()` — fires both macOS keychain reads (OAuth + legacy
  API key) in parallel, avoiding the ~65ms sequential sync spawn

These are the only three import-time side effects, and they are ordered to
maximize parallelism with the remaining module evaluation.

### 10.2 Context Prefetch

After CLI argument parsing and initialization, the startup path calls
`getSystemContext()` and `getUserContext()` (from `src/context.ts`). Since these
are memoized, they compute once and the result is reused by the query loop.

The `fetchSystemPromptParts()` function in `src/utils/queryContext.ts` runs
three operations in parallel:

```typescript
const [defaultSystemPrompt, userContext, systemContext] = await Promise.all([
  getSystemPrompt(tools, mainLoopModel, ...),
  getUserContext(),
  getSystemContext(),
])
```

When a custom system prompt is provided (SDK mode), the default system prompt
build and system context fetch are skipped entirely — the custom prompt replaces
the default, and system context would be appended to a default that is not being
used.

### 10.3 Other Prefetch Operations

The startup path fires several additional prefetch operations:
- `prefetchPassesEligibility()` — referral pass eligibility check
- `prefetchOfficialMcpUrls()` — MCP server registry fetch
- `prefetchFastModeStatus()` — fast mode availability check
- `prefetchAwsCredentialsAndBedRockInfoIfSafe()` — AWS credential prefetch for
  Bedrock users
- `prefetchGcpCredentialsIfSafe()` — GCP credential prefetch for Vertex users

These all run in parallel with the system prompt assembly and are `void`-called
(fire-and-forget) so they never block the critical path.

---

## 11. Design Principles

### 11.1 State as Data, Not Call Stack

The single `while(true)` loop with explicit `State` assignment at each
continuation site makes the control flow visible as data. Each `state.transition`
records *why* the loop continued, enabling:

- Test assertions on recovery paths without parsing message contents
- Telemetry that tracks which recovery paths fire in production
- Future extraction of a pure `step()` function where `(state, event, config) → state`

The alternative — recursive calls or deeply nested conditionals — would hide the
control flow in the call stack and make recovery path interactions much harder
to reason about.

### 11.2 Withhold-Then-Decide Error Handling

Recoverable errors (prompt-too-long, media size, max output tokens) are
*withheld* from the yield stream during the streaming loop. They are still
pushed to `assistantMessages` so recovery logic can find them, but they are
not yielded to the caller. This prevents SDK consumers from seeing intermediate
errors that the recovery loop can handle — if the error is yielded and the
consumer terminates the session, the recovery loop keeps running but nobody
is listening.

If recovery succeeds, the withheld error is never surfaced. If recovery fails,
the error is explicitly yielded before the loop returns. This creates a clean
contract: callers see either a recovered response or a terminal error, never
both.

### 11.3 One-Shot Guards for Recovery Paths

Each recovery path has an explicit termination condition:

| Path | Guard |
|---|---|
| Collapse drain | Previous transition was not already `collapse_drain_retry` |
| Reactive compact | `hasAttemptedReactiveCompact` flag |
| Max tokens escalate | `maxOutputTokensOverride === undefined` |
| Max tokens recovery | Counter < 3 |
| Token budget | < 90% threshold and no diminishing returns |

This prevents the most dangerous failure mode in agentic systems: unbounded
error-recovery loops that burn API credits. The source contains an explicit
war story in the stop-hook blocking continuation: resetting
`hasAttemptedReactiveCompact` caused an infinite loop "burning thousands of API
calls."

### 11.4 Generator-Based Streaming

The query loop is an async generator, not a callback-based event emitter. This
gives callers pull-based control: they consume events at their own pace, can
cancel via `.return()`, and get automatic cleanup via the generator's `finally`
block. The `using` keyword for `pendingMemoryPrefetch` exploits this further —
explicit resource management ties cleanup to the generator's lifetime regardless
of how it exits.

The two-layer generator structure (`query` -> `queryLoop`) separates lifecycle
concerns (command completion tracking) from control flow (the state machine)
without requiring the inner loop to know about the outer wrapper's bookkeeping.

### 11.5 Dependency Injection for Testability

The `QueryDeps` type provides a narrow injection surface for the four most
commonly mocked dependencies:

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

Tests inject fakes directly via the `deps` field in `QueryParams` instead of
per-module `spyOn` boilerplate. The `typeof fn` pattern keeps signatures in
sync with real implementations automatically. The scope is intentionally narrow
(4 deps) "to prove the pattern" — the source notes that follow-up PRs can add
more.

### 11.6 Feature Flag Discipline

The loop uses two distinct feature flag mechanisms with different semantics:

- **`feature('...')`** (Bun compile-time): Tree-shaking boundaries. The guarded
  code is literally removed from external builds. These must stay inline at
  if/ternary conditions — extracting them into `QueryConfig` would break
  dead-code elimination.

- **Statsig/GrowthBook gates**: Runtime toggles snapshotted once into
  `QueryConfig`. These can flip between sessions without a rebuild.

The `config.ts` file explains the intentional separation: feature() gates are
excluded from QueryConfig because they are tree-shaking boundaries that must
stay inline.

### 11.7 Attachment Ordering and Timing

Attachments are gathered *after* tool execution but *before* the next API call.
This ordering ensures:

1. Tool-result messages come first (the API requires tool_result messages to
   immediately follow their tool_use pairs)
2. Attachments see the results of tool execution (e.g., newly created files)
3. Prefetch operations (memory, skills) have maximum time to resolve (they
   start at loop entry and are consumed after tools complete)

The memory prefetch gets "as many chances as there are loop iterations" —
if it has not settled by the attachment injection point, it is skipped for
this iteration and retried next time. This zero-wait design ensures the
prefetch never blocks the critical path.

### 11.8 Abort Signal Propagation

The abort signal is checked at four points in each iteration:

1. After streaming completes (before any tool execution)
2. During streaming tool execution (via the executor's internal check)
3. After tool execution completes
4. During stop hook execution

Each check point generates appropriate cleanup: synthetic tool_result blocks
for orphaned tool_use blocks, interruption messages, and
`cleanupComputerUseAfterTurn()` for computer-use sessions. The
`submit-interrupt` variant (where a new user message is queued) skips the
interruption message since the queued message provides sufficient context.

### 11.9 Pair Preservation

The loop is careful to never yield a tool_use block without a corresponding
tool_result. If the API call throws unexpectedly after yielding tool_use
blocks, `yieldMissingToolResultBlocks()` generates synthetic error results
for every orphaned tool_use:

```typescript
function* yieldMissingToolResultBlocks(
  assistantMessages: AssistantMessage[],
  errorMessage: string,
) {
  for (const assistantMessage of assistantMessages) {
    const toolUseBlocks = assistantMessage.message.content.filter(
      content => content.type === 'tool_use',
    ) as ToolUseBlock[]
    for (const toolUse of toolUseBlocks) {
      yield createUserMessage({
        content: [{
          type: 'tool_result',
          content: errorMessage,
          is_error: true,
          tool_use_id: toolUse.id,
        }],
        toolUseResult: errorMessage,
        sourceToolAssistantUUID: assistantMessage.uuid,
      })
    }
  }
}
```

This preserves the invariant that the API and transcript always see matched
tool_use/tool_result pairs, preventing "orphaned tool use" errors on subsequent
turns or session resume.
