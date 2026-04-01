# Query Loop — Code Reference

## Part B: Code Reference

### Source Files

| File | Purpose |
|---|---|
| `src/query.ts` | The `query()` and `queryLoop()` async generator functions -- the core state machine |
| `src/query/config.ts` | `buildQueryConfig()` and `QueryConfig` type definition |
| `src/context.ts` | `getSystemContext()`, `getUserContext()` -- memoized context assembly |
| `src/utils/queryContext.ts` | `fetchSystemPromptParts()` -- parallel system prompt assembly |
| `src/services/tools/toolOrchestration.ts` | `runTools()`, `partitionToolCalls()` -- tool execution orchestrator |
| `src/QueryEngine.ts` | `QueryEngine` class -- SDK/headless wrapper around `query()` |
| `src/main.tsx` | Entry point -- startup bootstrap sequence |

### Type Definitions

**State** (loop-internal mutable state):
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

**QueryParams** (loop entry inputs):
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

**QueryDeps** (dependency injection surface):
```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

Production dependencies provided by `productionDeps()`.

**QueryConfig** (immutable per-invocation config snapshot):
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

**BudgetTracker** (token budget monitoring):
```typescript
export type BudgetTracker = {
  continuationCount: number
  lastDeltaTokens: number
  lastGlobalTurnTokens: number
  startedAt: number
}
```

**QueryEngine class fields** (`src/QueryEngine.ts`):
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

**queryLoop generator return type**:
```typescript
AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal
>
```

### Key Functions and Their Locations

| Function | File | Purpose |
|---|---|---|
| `query()` | `src/query.ts` | Outer generator, command lifecycle tracking |
| `queryLoop()` | `src/query.ts` | Inner generator, the actual state machine |
| `buildQueryConfig()` | `src/query/config.ts` | Snapshots env/gates into QueryConfig |
| `getSystemContext()` | `src/context.ts` | Memoized git metadata collection |
| `getUserContext()` | `src/context.ts` | Memoized CLAUDE.md hierarchy loading |
| `appendSystemContext()` | (context integration) | Joins system prompt parts with system context key-value pairs |
| `prependUserContext()` | (context integration) | Injects CLAUDE.md content as first user message |
| `fetchSystemPromptParts()` | `src/utils/queryContext.ts` | Parallel system prompt + user context + system context fetch |
| `getMessagesAfterCompactBoundary()` | (pre-call pipeline) | Trims messages to post-compact region |
| `applyToolResultBudget()` | (pre-call pipeline) | Enforces per-message size limits |
| `snipCompactIfNeeded()` | (pre-call pipeline) | Removes stale segments (HISTORY_SNIP) |
| `microcompactMessages()` | (pre-call pipeline) | Fine-grained compaction |
| `applyCollapsesIfNeeded()` | (pre-call pipeline) | Context collapse projections |
| `autoCompactIfNeeded()` | (pre-call pipeline) | Full summarization if needed |
| `runTools()` | `src/services/tools/toolOrchestration.ts` | Post-streaming tool execution orchestrator |
| `partitionToolCalls()` | `src/services/tools/toolOrchestration.ts` | Batches tools into concurrent/non-concurrent groups |
| `tryReactiveCompact()` | (error recovery) | Emergency compaction for prompt-too-long |
| `executeStopFailureHooks()` | (error recovery) | Fires when reactive compact fails |
| `backfillObservableInput()` | (streaming) | Enriches tool_use blocks for SDK consumers without mutating originals |
| `yieldMissingToolResultBlocks()` | `src/query.ts` | Generates synthetic error tool_result for orphaned tool_use blocks |
| `normalizeMessagesForAPI()` | (tool results) | Normalizes tool results to API-expected format |
| `getAttachmentMessages()` | (attachments) | Produces attachment messages (file changes, plans, skills, MCP, tasks) |
| `startRelevantMemoryPrefetch()` | (attachments) | Starts async memory relevance side-query |
| `startSkillDiscoveryPrefetch()` | (attachments) | Starts async skill discovery |
| `collectSkillDiscoveryPrefetch()` | (attachments) | Consumes skill discovery results |
| `refreshTools()` | (attachments) | Picks up newly-connected MCP servers |
| `getMemoryFiles()` | (user context) | Loads CLAUDE.md files |
| `getClaudeMds()` | (user context) | Loads CLAUDE.md hierarchy |
| `filterInjectedMemoryFiles()` | (user context) | Strips suspicious patterns from memory files |
| `isConcurrencySafe()` | (per-tool method) | Examines parsed input to determine read-only status |
| `processUserInput()` | (QueryEngine) | Handles slash commands, file attachments, image processing, input validation |
| `submitMessage()` | `src/QueryEngine.ts` | Per-turn entry point in QueryEngine |
| `ask()` | (QueryEngine module) | One-shot convenience wrapper |
| `cleanupComputerUseAfterTurn()` | (abort handling) | Cleanup for computer-use sessions |

### Constants and Feature Gates

| Constant/Gate | Value/Description |
|---|---|
| `ESCALATED_MAX_TOKENS` | 64k (escalation target for max output tokens) |
| `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT` | 3 (max resume attempts after output token cap) |
| `COMPLETION_THRESHOLD` | 0.9 (90% of token budget before stopping) |
| Diminishing returns threshold | 500 new tokens minimum for two consecutive checks after 3+ continuations |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` | Default 10, configurable concurrency limit for parallel tool execution |
| `tengu_otk_slot_v1` | GrowthBook feature value gating max output tokens escalation |
| `streamingToolExecution` | Statsig gate for streaming tool execution |
| `emitToolUseSummaries` | Statsig gate for tool use summary generation |
| `CONTEXT_COLLAPSE` | Feature gate for context collapse recovery |
| `HISTORY_SNIP` | Feature for stale segment removal |
| `TOKEN_BUDGET` | Feature for token budget enforcement |
| `CLAUDE_CODE_REMOTE` | Environment variable indicating remote session |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | Environment variable to disable CLAUDE.md loading |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | User-settable env var for explicit max output tokens |
| `feature('...')` | Bun compile-time tree-shaking boundaries (must stay inline) |

### Startup Side Effects (`src/main.tsx`)

Three import-time side effects, ordered to maximize parallelism:
1. `profileCheckpoint('main_tsx_entry')` -- entry timestamp
2. `startMdmRawRead()` -- fires MDM subprocess reads (plutil/reg query)
3. `startKeychainPrefetch()` -- fires macOS keychain reads (OAuth + legacy API key)

### Startup Prefetch Operations

- `prefetchPassesEligibility()` -- referral pass eligibility
- `prefetchOfficialMcpUrls()` -- MCP server registry
- `prefetchFastModeStatus()` -- fast mode availability
- `prefetchAwsCredentialsAndBedRockInfoIfSafe()` -- AWS/Bedrock credentials
- `prefetchGcpCredentialsIfSafe()` -- GCP/Vertex credentials

All fire-and-forget (`void`-called).

### Code Snippets

**yieldMissingToolResultBlocks** (pair preservation):
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

**Compaction boundary GC** (QueryEngine):
```typescript
const mutableBoundaryIdx = this.mutableMessages.length - 1
if (mutableBoundaryIdx > 0) {
  this.mutableMessages.splice(0, mutableBoundaryIdx)
}
```

**Max output tokens recovery prompt** (injected user message):
```
Output token limit hit. Resume directly -- no apology, no recap of what you
were doing. Pick up mid-thought if that is where the cut happened. Break
remaining work into smaller pieces.
```

**Parallel system prompt assembly** (`fetchSystemPromptParts`):
```typescript
const [defaultSystemPrompt, userContext, systemContext] = await Promise.all([
  getSystemPrompt(tools, mainLoopModel, ...),
  getUserContext(),
  getSystemContext(),
])
```

### Transition Reason Strings (for test assertions and telemetry)

- `'collapse_drain_retry'`
- `'reactive_compact_retry'`
- `'max_output_tokens_escalate'`
- `'max_output_tokens_recovery'`
- `'stop_hook_blocking'`
- `'token_budget_continuation'`
- `'next_turn'`

### Terminal Reason Strings (loop exit)

- `'aborted_streaming'`
- `'hook_stopped'`
- `'stop_hook_prevented'`
- `'max_turns'`

### Attachment-Related Identifiers

- `hook_stopped_continuation` -- attachment key indicating a hook stopped continuation
- `max_turns_reached` -- attachment yielded when turn count exceeds limit
- `Sleep` tool -- triggers lowered drain threshold for `'later'` priority commands
- `readFileState` / `FileStateCache` -- tracks recently-read files for deduplication
- `snipReplay` -- injection for HISTORY_SNIP feature (stays out of QueryEngine)
