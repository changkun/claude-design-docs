# Streaming Executor — Code Reference

### File Paths

| Concern | File Path |
|---------|-----------|
| Single-tool execution engine | `src/services/tools/toolExecution.ts` |
| Streaming tool executor | `src/services/tools/StreamingToolExecutor.ts` |
| Batch orchestration (legacy) | `src/services/tools/toolOrchestration.ts` |
| Tool hook integration | `src/services/tools/toolHooks.ts` |
| Low-level hook execution | `src/utils/hooks.ts` |
| Push-pull stream utility | `src/utils/stream.ts` |
| Concurrent generator utility | `src/utils/generators.ts` |
| Tool result storage/budget | `src/utils/toolResultStorage.ts` |
| Tool size limits/constants | `src/constants/toolLimits.ts` |
| Tool base class (concurrency) | `src/Tool.ts` |
| Hook event schema definitions | `src/entrypoints/sdk/coreSchemas.ts` |
| Query loop (path selection) | `src/query.ts` |

### Key Functions and Entry Points

| Function | File | Purpose |
|----------|------|---------|
| `runToolUse()` | `toolExecution.ts` | Universal single-tool entry point. Returns `AsyncGenerator<MessageUpdateLazy, void>`. |
| `streamedCheckPermissionsAndCallTool()` | `toolExecution.ts` | Bridge between async/callback execution and async-iterable query loop. Creates `Stream<MessageUpdateLazy>`. |
| `checkPermissionsAndCallTool()` | `toolExecution.ts` | Core execution logic: validate, hook, permission, call, persist, hook. |
| `resolveHookPermissionDecision()` | `toolHooks.ts` | Merges hook permission results with rule-based checks. |
| `startSpeculativeClassifierCheck()` | `toolExecution.ts` | Starts AI classifier early for Bash tools. |
| `classifyToolError()` | `toolExecution.ts` (lines 469-490) | Classifies errors for telemetry. |
| `processToolResultBlock()` | `toolResultStorage.ts` | Per-tool persistence of large results. |
| `processPreMappedToolResultBlock()` | `toolResultStorage.ts` | Persistence variant using pre-mapped block. |
| `enforceToolResultBudget()` | `toolResultStorage.ts` | Per-message aggregate budget enforcement. |
| `reconstructContentReplacementState()` | `toolResultStorage.ts` | Rebuilds replacement state on resume. |
| `reconstructForSubagentResume()` | `toolResultStorage.ts` | Fills gaps for subagent (fork) resumes. |
| `partitionToolCalls()` | `toolOrchestration.ts` | Groups consecutive tools into concurrency batches. |
| `runToolsSerially()` | `toolOrchestration.ts` | Serial batch execution. |
| `runToolsConcurrently()` | `toolOrchestration.ts` | Concurrent batch execution. |
| `runTools()` | `toolOrchestration.ts` | Top-level batch orchestration entry point. Returns `AsyncGenerator<MessageUpdate, void>`. |
| `all()` | `src/utils/generators.ts` | Concurrent async-generator merger with max concurrency limit. |
| `runPreToolUseHooks()` | `toolHooks.ts` | Runs pre-tool hooks, yields discriminated union results. |
| `runPostToolUseHooks()` | `toolHooks.ts` | Runs post-tool hooks, yields messages and optional MCP output updates. |
| `runPostToolUseFailureHooks()` | `toolHooks.ts` | Runs failure-observation hooks. |
| `startHookProgressInterval()` | `src/utils/hooks.ts` | Emits periodic progress events while hooks run. |
| `backfillObservableInput()` | `toolExecution.ts` | Fills derived fields on a clone for hook observation. |

### Key Types

```typescript
// Message update wrapper with optional context modifier
export type MessageUpdateLazy<M extends Message = Message> = {
  message: M
  contextModifier?: {
    toolUseID: string
    modifyContext: (context: ToolUseContext) => ToolUseContext
  }
}

// Streaming executor tool tracking
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'

type TrackedTool = {
  id: string
  block: ToolUseBlock
  assistantMessage: AssistantMessage
  status: ToolStatus
  isConcurrencySafe: boolean
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>
}

// Batch partitioning type
type Batch = { isConcurrencySafe: boolean; blocks: ToolUseBlock[] }

// Content replacement state for budget management
type ContentReplacementState = {
  seenIds: Set<string>
  replacements: Map<string, string>
}
```

### Key Parameters and Receiver Types for `runToolUse()`

- `toolUse: ToolUseBlock` -- the model's tool call (name, id, input)
- `assistantMessage: AssistantMessage` -- the parent assistant message
- `canUseTool: CanUseToolFn` -- permission-checking callback
- `toolUseContext: ToolUseContext` -- full execution context (messages, app state, abort controller, MCP clients, options)

### StreamingToolExecutor Methods

| Method | Behavior |
|--------|----------|
| `addTool(block, assistantMessage)` | Registers new tool from API stream, starts queue processing. |
| `processQueue()` | Iterates queued tools, starts those passing `canExecuteTool()`. |
| `canExecuteTool(isConcurrencySafe): boolean` | Returns true if no tools executing, or if all executing are concurrent-safe and this one is too. |
| `executeTool(tool)` | Full per-tool lifecycle: abort check, controller setup, generator consumption, progress separation, modifier application. |
| `getCompletedResults()` | Synchronous generator: yields progress and completed results in order. |
| `getRemainingResults()` | Async generator: loops until all yielded, uses `Promise.race()` for blocking. |
| `discard()` | Sets flag to prevent pending tools and short-circuit result methods. |
| `updateInterruptibleState()` | Tracks whether all executing tools support cancellation. |

### Line Number References (Point-in-Time)

| Section | File | Lines |
|---------|------|-------|
| Permission checking | `toolExecution.ts` | 599-1104 |
| Progress tracking | `toolExecution.ts` | 502-570 |
| Progress in streaming executor | `StreamingToolExecutor.ts` | 366-378 |
| Error classification | `toolExecution.ts` | 469-490 |
| Result rendering | `toolExecution.ts` | 1589-1745 |
| Result rendering (full range) | `toolExecution.ts` | 1206-1588 |
| Error recovery (streaming executor) | `StreamingToolExecutor.ts` | 153-204, 276-345 |

### Constants and Configuration

| Constant | Value | Location |
|----------|-------|----------|
| `DEFAULT_MAX_RESULT_SIZE_CHARS` | 50,000 | `src/constants/toolLimits.ts` |
| `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` | 200,000 | `src/constants/toolLimits.ts` |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` | 10 (default, env var) | `toolOrchestration.ts` |
| `HOOK_TIMING_DISPLAY_THRESHOLD_MS` | 500 | `src/utils/hooks.ts` |
| `SLOW_PHASE_LOG_THRESHOLD_MS` | 2,000 | `src/utils/hooks.ts` |
| `TOOL_HOOK_EXECUTION_TIMEOUT_MS` | 600,000 (10 min) | `src/utils/hooks.ts` |
| SessionEnd hook timeout | 1,500ms (1.5s) | `src/utils/hooks.ts` |
| `REJECT_MESSAGE` | (synthetic user_interrupted error constant) | `StreamingToolExecutor.ts` |

### GrowthBook Feature Flags

| Flag | Purpose |
|------|---------|
| `streamingToolExecution` | Gates the streaming executor path vs. legacy batch path |
| `tengu_satin_quoll` | Overrides per-tool max result size threshold by tool name |
| `tengu_hawthorn_window` | Overrides per-message aggregate result size limit |

### Hook Configuration Schema Discriminators

| Type Value | Implementation |
|------------|----------------|
| `type: 'command'` | Shell hook (child process, JSON stdin, exit codes) |
| `type: 'prompt'` | LLM prompt hook (model selection, token limits) |
| `type: 'agent'` | Full agentic loop hook (tools, model, timeout) |
| `type: 'http'` | HTTP POST hook (URL, headers, timeout) |
| `HookCallback` | In-process SDK callback |
| `FunctionHook` | In-process session function via `getSessionFunctionHooks()` |

### Hook Source Configuration Locations

| Source Key | File/Location |
|------------|---------------|
| `userSettings` | `~/.claude/settings.json` |
| `projectSettings` | `.claude/settings.json` (in repo) |
| `localSettings` | `.claude/settings.local.json` (gitignored) |
| `policySettings` | Enterprise policy (managed) |
| `pluginHook` | Plugin-registered |
| `sessionHook` | SDK-registered per-session |
| `builtinHook` | Hardcoded internal |

### Persistence Path Structure

Tool results persisted to disk follow this path pattern:
```
<projectDir>/<sessionId>/tool-results/<toolUseId>.[txt|json]
```

Files are written with the `wx` flag (write-exclusive, fails if file already exists).

### Telemetry Event Names

| Event | When |
|-------|------|
| `tengu_tool_use_progress` | On each progress emission from the stream adapter |
| Permission decision OTel span | On every permission resolution, with `source` label |

### Concurrency Safety by Tool Class

| Tool Class | Method |
|------------|--------|
| `FileReadTool` | `isConcurrencySafe()` returns `true` |
| `GrepTool` | `isConcurrencySafe()` returns `true` |
| `GlobTool` | `isConcurrencySafe()` returns `true` |
| `BashTool` | `isConcurrencySafe(input)` parses command via `shell-quote`, returns `true` only for read-only commands |
| `FileEditTool` | `isConcurrencySafe()` returns `false` |
| `FileWriteTool` | `isConcurrencySafe()` returns `false` |
| `AgentTool` | `isConcurrencySafe()` returns `false` |
| `AskUserQuestionTool` | `isConcurrencySafe()` returns `false` |

### Tool-Specific Behaviors

- `EnterPlanModeTool`: Uses `contextModifier` to switch permission mode.
- `CwdChangedTool`: Uses `contextModifier` to update working directory.
- `ToolSearchTool`: Deferred tool schema loading; triggers hint in validation errors.
- `FileReadTool`: Sets `maxResultSizeChars: Infinity` (opts out of persistence).
- `AskUserQuestionTool`: Classified as requiring user interaction; hook `updatedInput` can substitute for user.
- `SedEditPermissionRequest`: Permission dialog injects `_simulatedSedEdit` into input.
- MCP tools: Result added to messages AFTER PostToolUse hooks (hooks can modify output via `updatedMCPToolOutput`).

### Query Loop Path Selection Code

```typescript
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

Located in `src/query.ts`.
