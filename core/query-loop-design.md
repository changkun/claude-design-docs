# Query Loop — Design Document

### 1. Overview

The query loop is the core execution engine of Claude Code. It implements the agentic contract: the model produces a response, the system executes any requested tools, feeds results back, and loops until the model stops requesting tools.

The loop is structured as a **single `while(true)` loop with explicit continuation sites**, not a recursive call chain. Each iteration is a complete LLM call cycle: prepare context, call the API, stream the response, execute tools, decide whether to continue. A `State` object is reassigned at each continuation site, making control flow visible as data rather than buried in call stack depth.

The outer layer is a two-layer async generator pipeline:

```
query(params) -> queryLoop(params, consumedCommandUuids)
```

The outer generator handles command lifecycle tracking (marking consumed commands as completed only on normal return, not on throw or cancellation). The inner generator is the actual state machine that yields intermediate events and returns a terminal status.

### 2. The Single While(True) Loop

Each loop iteration follows a fixed sequence:

1. Destructure current state
2. Pre-processing: snip, microcompact, context collapse, autocompact
3. Prepare and call the LLM API (streaming)
4. Process streamed response, extract tool_use blocks
5. If no tool calls: run stop hooks, check token budget, return terminal status
6. Execute tools (parallel or serial)
7. Gather attachments (memory, skills, queued commands)
8. Build next State, continue

Each continuation site constructs a new `State` object with a `transition` field recording *why* the loop is continuing. This makes control flow machine-inspectable -- tests can assert that a specific recovery path fired by examining the transition reason rather than parsing message contents.

### 3. Continuation Sites

There are seven `continue` statements in the loop body:

| Continuation | Transition Reason | When |
|---|---|---|
| Context collapse drain | `collapse_drain_retry` | Prompt-too-long, staged collapses drained |
| Reactive compact | `reactive_compact_retry` | Prompt-too-long, emergency compaction succeeded |
| Max output tokens escalate | `max_output_tokens_escalate` | Output capped at small limit, retry at large limit |
| Max output tokens recovery | `max_output_tokens_recovery` | Output still capped, inject resume prompt |
| Stop hook blocking | `stop_hook_blocking` | Stop hook reported blocking errors |
| Token budget continuation | `token_budget_continuation` | Under 90% of token budget, inject nudge |
| Next turn | `next_turn` | Tools executed, normal continuation |

### 4. State Management

The loop carries a mutable `State` object destructured at the top of each iteration and reassigned as a whole at each continuation site. This whole-object reassignment prevents partial-update bugs and creates a visible checkpoint at each transition. The state includes:

- The message array
- Tool use context
- Auto-compact tracking state
- Max output tokens recovery counter
- A flag for whether reactive compact has been attempted
- An optional max output tokens override
- A pending tool use summary promise
- A stop hook active flag
- A turn counter
- A transition record (undefined on first iteration, set at every continuation site thereafter)

Three additional mutable variables live outside the State struct:

- **Task budget remaining**: Tracks cumulative API task budget spend across compaction boundaries. After compaction, the summarized history hides the true spend, so the client tracks it explicitly.
- **Config**: An immutable snapshot of environment variables and feature gates taken once at loop entry.
- **Budget tracker**: Mutable tracker for the token budget feature that records continuation counts and a diminishing-returns detector.

### 5. Context Assembly

Two memoized functions compute environmental context once per session:

- **System context**: Git metadata (branch, status, recent commits, user name). All git commands run in parallel. Skipped in remote sessions or when git instructions are disabled.
- **User context**: Project instructions from the four-level CLAUDE.md hierarchy (managed, user, project, local). Includes the current date string. Respects disable flags and bare mode.

At each iteration, system context is appended to the system prompt, and user context is prepended to the message array ephemerally (only for the API call, not modifying the canonical message array).

Before each API call, the message array passes through a multi-stage processing pipeline:

1. Trim to post-compaction region
2. Enforce per-message size limits
3. Remove stale segments (history snip)
4. Fine-grained compaction (microcompact)
5. Context collapse projections
6. Full summarization if needed (autocompact)

The ordering is significant: lightweight operations (snip, microcompact, collapse) run before autocompact so their savings may prevent the need for full summarization.

### 6. LLM Call Flow

Each iteration makes a single streaming API call receiving the processed messages, full system prompt, thinking config, current tool definitions (refreshed between turns for MCP), an abort signal, and runtime configuration options.

The call is wrapped in a retry loop for model fallback: if the primary model fails with a fallback-triggerable error, the loop switches to a fallback model and retries.

During streaming, the loop performs concurrent activities:

- **Tool use block extraction**: Accumulates tool_use blocks and sets a needs-follow-up flag
- **Streaming tool execution**: When enabled by feature gate, tools begin executing as their blocks arrive during streaming
- **Error withholding**: Recoverable errors (prompt-too-long, media size, max output tokens) are withheld from the yield stream until recovery logic decides their fate
- **Observable input backfill**: Tool_use blocks are cloned and enriched with derived fields for SDK consumers without mutating originals (which would break prompt caching)
- **Tombstone emission**: If streaming fallback occurs mid-response, partially-streamed messages are yielded as tombstones for UI removal

After streaming completes: post-sampling hooks fire asynchronously, abort signal is checked, and any pending tool use summary from the previous turn is awaited and yielded.

### 7. Tool Execution

#### Dual Execution Paths

- **Streaming path**: Tools begin executing as their blocks arrive during streaming. Results are buffered and emitted in tool-received order, preserving the illusion of sequential execution even when tools complete out of order.
- **Post-streaming path**: All tools execute after the full response. This is the legacy/fallback path.

#### Parallel vs. Serial Execution

Tool calls are partitioned into batches. Each batch is either concurrency-safe (multiple consecutive read-only tools in parallel, up to a configurable concurrency limit) or non-concurrent (a single tool requiring exclusive access). Context modifiers from concurrent tools are queued and applied in order after the batch completes.

#### Tool Use Summary Generation

After each tool batch, the system optionally generates a human-readable summary via a lightweight model call. The summary is generated asynchronously and yielded at the start of the next iteration, overlapping the summary generation with the main model streaming. Only generated for top-level agents.

### 8. Error Recovery

Seven distinct recovery paths, each with explicit termination conditions to prevent infinite retry loops.

#### 8.1 Context Collapse Drain

When the API returns prompt-too-long and staged context collapses exist, the loop commits pending collapse summaries. This is the cheapest recovery -- no new API call. If the previous iteration was already a collapse drain that still failed, this path is skipped and falls through to reactive compact.

#### 8.2 Reactive Compact

When collapse drain is insufficient, the loop performs emergency full summarization. A one-shot flag ensures this fires at most once per turn. If post-compact context still triggers the error, it surfaces to the user.

Also handles media size errors. If reactive compact fails, stop failure hooks fire. The loop does NOT fall through to stop hooks -- running stop hooks on prompt-too-long would create a death spiral.

#### 8.3 Max Output Tokens Escalation

When the model hits the output token cap at a small default limit, the loop retries the same request at a larger limit (e.g., 8k to 64k). No meta-message is injected. Fires once per turn. Gated behind a feature flag and only active when the user has not set an explicit max output tokens value.

#### 8.4 Max Output Tokens Recovery

After escalation fails or is not applicable, the loop injects a synthetic resume prompt telling the model to continue from where it was cut off. Assistant messages from the truncated response are preserved. A counter limits attempts (e.g., 3 maximum). After the limit, the error surfaces and the loop exits.

#### 8.5 Stop Hook Blocking

When the model produces a complete response and stop hooks report blocking errors, the errors are appended to the conversation and the loop continues so the model can adjust.

Critical subtlety: the reactive-compact-attempted flag is preserved across this continuation. Resetting it caused an infinite loop in production (compact -> still too long -> error -> stop hook blocking -> compact -> ...).

Stop hooks also handle a `preventContinuation` case where a hook explicitly signals loop termination.

#### 8.6 Token Budget Continuation

A token budget feature lets callers specify a target output budget. A budget tracker monitors cumulative output tokens. If the model has produced less than 90% of the budget and is not showing diminishing returns (fewer than 500 new tokens for two consecutive checks after 3+ continuations), a nudge is injected and the loop continues.

#### 8.7 Next Turn

Normal continuation after tools execute. Recovery counters are reset because the fresh tool-result turn means previous error conditions no longer apply. Before continuing, a max-turns limit is checked.

### 9. Attachment System

Between tool execution and the next LLM call, the loop gathers "attachments" -- additional context the model needs but did not explicitly request.

- **Queued commands**: Process-global command queue drained at configurable priority levels. Agent-scoped (main thread vs. subagents). Slash commands excluded from mid-turn drain.
- **Standard attachments**: File change notifications, plan mode instructions, skill listings, MCP resource references, task notifications.
- **Memory prefetch**: Started once per user turn, runs a side-query for relevant memory files. Resolves asynchronously. If not settled at injection point, skipped and retried next iteration. Uses explicit resource management for cleanup on all exit paths.
- **Skill discovery prefetch**: Per-iteration background check for relevant skills. Replaces a blocking path that found nothing 97% of the time.
- **Tool refresh**: Picks up newly-connected MCP servers and updates tool definitions.

### 10. QueryEngine Wrapper

A class that wraps the raw generator for SDK and headless usage. Owns the conversation lifecycle -- one instance per conversation, with state persisting across turns.

Each `submitMessage()` call:

1. Clears turn-scoped state
2. Sets working directory
3. Wraps permission decisions for SDK reporting
4. Fetches system prompt parts in parallel
5. Assembles system prompt
6. Processes user input (slash commands, file attachments, images, validation)
7. Persists transcript for resume capability
8. Enters the query loop
9. Routes yielded messages to SDK output format
10. Checks budget and retry limits
11. Yields terminal result with duration, cost, usage, and permission denials

It maintains a file state cache for post-compact restoration and memory deduplication. On compaction boundary messages, it splices pre-compaction messages for garbage collection. For one-shot usage, a convenience wrapper creates a temporary engine and returns the result.

### 11. Startup Path

The entry point orchestrates a carefully ordered startup sequence to minimize time-to-first-token:

- **Immediate side effects** (before imports complete): Entry timestamp, MDM subprocess reads, and keychain prefetch -- all fired in parallel with module evaluation
- **Context prefetch**: System and user context computed once (memoized) and reused by the query loop
- **Additional prefetches**: Pass eligibility, MCP server registry, fast mode status, cloud provider credentials -- all fire-and-forget to never block the critical path

### 12. Design Principles

1. **State as Data, Not Call Stack**: The `while(true)` loop with explicit State assignment makes control flow visible as data. Transition records enable test assertions, telemetry, and future extraction of a pure step function.

2. **Withhold-Then-Decide Error Handling**: Recoverable errors are withheld from the yield stream during streaming. If recovery succeeds, the error is never surfaced. If recovery fails, the error is explicitly yielded before loop exit. Callers see either a recovered response or a terminal error, never both.

3. **One-Shot Guards for Recovery Paths**: Each recovery path has an explicit termination condition (previous-transition check, boolean flag, counter limit, threshold + diminishing returns). This prevents unbounded error-recovery loops.

4. **Generator-Based Streaming**: Async generators give callers pull-based control with cancellation via `.return()` and automatic cleanup. The two-layer structure separates lifecycle concerns from control flow.

5. **Dependency Injection for Testability**: A narrow injection surface (4 dependencies) enables testing with fakes instead of per-module spy boilerplate. Signatures stay in sync automatically via `typeof` patterns.

6. **Feature Flag Discipline**: Compile-time tree-shaking boundaries must stay inline (not extracted into config). Runtime toggles are snapshotted once per invocation into a config object.

7. **Attachment Ordering and Timing**: Attachments gathered after tool execution but before the next API call, ensuring tools complete first, attachments see results, and prefetches have maximum time to resolve. Memory prefetch uses zero-wait: skip and retry next iteration.

8. **Abort Signal Propagation**: Checked at four points per iteration (after streaming, during streaming tool execution, after tool execution, during stop hook execution). Each point generates appropriate cleanup including synthetic tool_result blocks for orphaned tool_use blocks.

9. **Pair Preservation**: The loop never yields a tool_use block without a corresponding tool_result. Synthetic error results are generated for every orphaned tool_use on unexpected failures, preserving the invariant for API and transcript consistency.
