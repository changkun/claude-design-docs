# Prompt Cache -- Code Reference

## Source File Map

| Concept | Primary File(s) |
|---|---|
| Dynamic boundary marker | `src/constants/prompts.ts` (lines 105-115) |
| System prompt splitting | `src/utils/api.ts` (lines 296-435, function `splitSysPromptPrefix()`) |
| Static region assembly | `src/constants/prompts.ts` (lines 560-577, function `getSystemPrompt()`) |
| Session-variant quarantine | `src/constants/prompts.ts` (lines 343-347, function `getSessionSpecificGuidanceSection()` at line 352) |
| Dynamic section registry | `src/constants/prompts.ts` (lines 491-555) |
| Section memoization | `src/constants/systemPromptSections.ts` (functions `systemPromptSection()`, `DANGEROUS_uncachedSystemPromptSection()`, `resolveSystemPromptSections()`) |
| Section cache storage | `src/bootstrap/state.ts` (accessed via `getSystemPromptSectionCache()`, `setSystemPromptSectionCacheEntry()`) |
| System context memoization | `src/context.ts` (line 116, `getSystemContext = memoize(...)`) |
| User context memoization | `src/context.ts` (line 155, `getUserContext = memoize(...)`) |
| Fork agent definition | `src/tools/AgentTool/forkSubagent.ts` (line 60, `FORK_AGENT`) |
| Fork message builder | `src/tools/AgentTool/forkSubagent.ts` (line 107, `buildForkedMessages()`) |
| Fork child detection | `src/tools/AgentTool/forkSubagent.ts` (line 78, `isInForkChild()`) |
| Fork placeholder string | `src/tools/AgentTool/forkSubagent.ts` (line 93, `FORK_PLACEHOLDER_RESULT`) |
| Cache-safe params type | `src/utils/forkedAgent.ts` (line 57, `CacheSafeParams`) |
| Cache-safe params slot | `src/utils/forkedAgent.ts` (line 73, `lastCacheSafeParams`) |
| Side query function | `src/utils/sideQuery.ts` (line 107, `sideQuery()`) |
| Tool schema cache | `src/utils/toolSchemaCache.ts` (`getToolSchemaCache()`, `clearToolSchemaCache()`) |
| Tool-to-API schema | `src/utils/api.ts` (line 119, `toolToAPISchema()`) |
| Cache break detection | `src/services/api/promptCacheBreakDetection.ts` |
| Pre-call recording | `src/services/api/promptCacheBreakDetection.ts` (line 247, `recordPromptState()`) |
| Post-call checking | `src/services/api/promptCacheBreakDetection.ts` (line 437, `checkResponseForCacheBreak()`) |
| Cache deletion notifier | `src/services/api/promptCacheBreakDetection.ts` (line 673, `notifyCacheDeletion()`) |
| Compaction notifier | `src/services/api/promptCacheBreakDetection.ts` (line 689, `notifyCompaction()`) |
| Min cache miss threshold | `src/services/api/promptCacheBreakDetection.ts` (line 120, `MIN_CACHE_MISS_TOKENS = 2_000`) |
| Compaction logic | `src/services/compact/compact.ts` (direction handling at line 767+, cache comment at line 852) |
| Startup prefetch entry | `src/main.tsx` (lines 1-20) |
| MDM prefetch | `src/utils/settings/mdm/rawRead.ts` (`startMdmRawRead()`) |
| Keychain prefetch | `src/utils/secureStorage/keychainPrefetch.ts` (`startKeychainPrefetch()`, `ensureKeychainPrefetchCompleted()`) |
| Model cost definitions | `src/utils/modelCost.ts` |
| 1-hour cache TTL | `src/services/api/claude.ts` (line 393, `should1hCacheTTL()`, used at line 371) |

## Key Constants and Types

```typescript
// src/constants/prompts.ts:114-115
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'

// src/tools/AgentTool/forkSubagent.ts:93
const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'

// src/services/api/promptCacheBreakDetection.ts:120
const MIN_CACHE_MISS_TOKENS = 2_000

// src/utils/forkedAgent.ts:57-68
export type CacheSafeParams = {
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext
  forkContextMessages: Message[]
}
```

## Cost Tier Constants (from `src/utils/modelCost.ts`)

```typescript
// Sonnet models (Sonnet 3.5v2, 3.7, 4, 4.5, 4.6): lines 36-42
COST_TIER_3_15 = { inputTokens: 3, outputTokens: 15, promptCacheWriteTokens: 3.75, promptCacheReadTokens: 0.3 }

// Opus 4 and 4.1: lines 45-51
COST_TIER_15_75 = { inputTokens: 15, outputTokens: 75, promptCacheWriteTokens: 18.75, promptCacheReadTokens: 1.5 }

// Opus 4.5 and Opus 4.6 (standard): lines 54-60
COST_TIER_5_25 = { inputTokens: 5, outputTokens: 25, promptCacheWriteTokens: 6.25, promptCacheReadTokens: 0.5 }

// Opus 4.6 fast mode: lines 63-69
COST_TIER_30_150 = { inputTokens: 30, outputTokens: 150, promptCacheWriteTokens: 37.5, promptCacheReadTokens: 3 }
```

**Note:** Opus 4.6 maps to `COST_TIER_5_25` at line 124-125, with `getOpus46CostTier()` (line 94-99) selecting `COST_TIER_30_150` when fast mode is active. The spec's original cost table incorrectly used `COST_TIER_15_75`.

## Dynamic Section Registration (from `src/constants/prompts.ts:491-555`)

Complete list of registered dynamic sections:
1. `session_guidance` -- memoized
2. `memory` -- memoized
3. `ant_model_override` -- memoized (ant-only)
4. `env_info_simple` -- memoized
5. `language` -- memoized
6. `output_style` -- memoized
7. `mcp_instructions` -- **DANGEROUS uncached**, reason: `'MCP servers connect/disconnect between turns'`
8. `scratchpad` -- memoized
9. `frc` -- memoized
10. `summarize_tool_results` -- memoized
11. `numeric_length_anchors` -- memoized, ant-only conditional
12. `token_budget` -- memoized, feature-gated
13. `brief` -- memoized, feature-gated

## Static Region Function Names (from `src/constants/prompts.ts:560-571`)

```typescript
getSimpleIntroSection(outputStyleConfig)      // line 562
getSimpleSystemSection()                       // line 563
getSimpleDoingTasksSection()                   // line 566, conditional on outputStyleConfig
getActionsSection()                            // line 568
getUsingYourToolsSection(enabledTools)         // line 569
getSimpleToneAndStyleSection()                 // line 570
getOutputEfficiencySection()                   // line 571
```

## Cache Break Detection State Type (from `src/services/api/promptCacheBreakDetection.ts:28-69`)

The `PreviousState` type tracks: `systemHash`, `toolsHash`, `cacheControlHash`, `toolNames`, `perToolHashes`, `systemCharCount`, `model`, `fastMode`, `globalCacheStrategy`, `betas`, `autoModeActive`, `isUsingOverage`, `cachedMCEnabled`, `effortValue`, `extraBodyHash`, `callCount`, `pendingChanges`, `prevCacheReadTokens`, `cacheDeletionsPending`, `buildDiffableContent`.

## Key Code Comments for Cache Behavior

- Boundary marker warning at `prompts.ts:572`: `// === BOUNDARY MARKER - DO NOT MOVE OR REMOVE ===`
- Fork system prompt threading at `forkSubagent.ts:54-58`: Documents why re-calling getSystemPrompt() is avoided (GrowthBook cold-to-warm divergence).
- Session-variant quarantine at `prompts.ts:343-347`: Documents the 2^N variant problem and references PRs #24490, #24171.
- Compaction cache comment at `compact.ts:852`: `// 'up_to' prefix hits cache directly; 'from' sends all (tail wouldn't cache).`
- Fork tool pool at `forkSubagent.ts:74-76`: Documents why AgentTool is kept in the child's pool for cache-identical tool definitions.
