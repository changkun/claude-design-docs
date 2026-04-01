# Context Management — Code Reference

### System Prompt Construction

**Source file**: `src/constants/prompts.ts`

**Key functions**:
- `getSystemPrompt()` -- returns string array
- `getSimpleIntroSection()`
- `getSimpleSystemSection()`
- `getSimpleDoingTasksSection()`
- `getActionsSection()`
- `getUsingYourToolsSection()`
- `getSimpleToneAndStyleSection()`
- `getOutputEfficiencySection()`
- `resolveSystemPromptSections()` -- async resolution of 13 dynamic sections
- `systemPromptSection(key, resolver)` -- caching wrapper, stores in `systemPromptSectionCache`, clears on `/clear` or `/compact`
- `DANGEROUS_uncachedSystemPromptSection()` -- used only for `mcp_instructions`

**Constants/markers**:
- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` -- separates static from dynamic

**Dynamic section keys** (string literals used in `systemPromptSection()`):
`session_guidance`, `memory`, `env_info_simple`, `language`, `output_style`, `mcp_instructions`, `scratchpad`, `frc`, `summarize_tool_results`, `token_budget`, `brief`, `numeric_length_anchors`, `ant_model_override`

**Feature checks**:
- `feature('PROACTIVE') || feature('KAIROS')` + `isProactiveActive()` -- triggers alternate prompt path
- `CLAUDE_CODE_SIMPLE=true` (via `--bare` flag) -- single-line prompt
- `getCachedMCConfigForFRC()` -- model ID pattern matching for microcompact support

**Model knowledge cutoffs**: Per-model lookup (Opus 4.6 -> May 2025, Sonnet 4.6 -> August 2025)

**Proactive mode prompt content**: `CYBER_RISK_INSTRUCTION` constant

---

### System Context Collection

**Source file**: `src/context.ts`

**Functions**:
- `getSystemContext()` (line 116) -- memoized via lodash `memoize`
- `getUserContext()` (line 155) -- memoized via lodash `memoize`
- `setCachedClaudeMdContent()` -- avoids import cycle: permissions -> filesystem -> permissions -> yoloClassifier

**Constants**:
- `MAX_STATUS_CHARS = 2000`

**Environment variables checked**:
- `CLAUDE_CODE_REMOTE` -- skips git collection
- `CLAUDE_CODE_DISABLE_CLAUDE_MDS` -- disables CLAUDE.md loading
- `systemPromptInjection` -- cache-breaking (ant-only debugging)

---

### Memory File Hierarchy (CLAUDE.md)

**Source file**: `src/utils/claudemd.ts`

**Key functions**:
- `getMemoryFiles()` -- memoized, walks CWD upward to root
- `getClaudeMds()` (line 1153) -- iterates files, generates per-type descriptions
- `filterInjectedMemoryFiles()` (line 1142) -- gates AutoMem/TeamMem removal
- `processConditionedMdRules()` -- two-pass conditional rule loading, uses `ignore()` library
- `resetGetMemoryFilesCache(reason)` -- reasons: `'session_start'`, `'compact'`, `'include'`
- `clearMemoryFileCaches()` -- correctness-only invalidation

**Constants**:
- `MAX_INCLUDE_DEPTH = 5`

**Worktree detection**: `findGitRoot(cwd)` vs `findCanonicalGitRoot(cwd)`

**Feature gate**: `tengu_moth_copse` -- controls whether AutoMem/TeamMem are removed from prompt (surfaced via attachments instead)

---

### Auto Memory

**Source files**: `src/memdir/memdir.ts`, `src/memdir/paths.ts`

**Key functions**:
- `getAutoMemPath()` -- memoized, keyed on `getProjectRoot()`
- `truncateEntrypointContent()` -- line-cap then byte-cap
- `validateMemoryPath()` (line 109-150 in paths.ts)
- `buildMemoryLines()` -- injects behavioral guidance into system prompt
- `isAutoMemoryEnabled()` -- priority chain
- `sanitizePathKey()`
- `realpathDeepestExisting()`
- `validateTeamMemWritePath()` -- two-pass validation

**Constants**:
- `MAX_ENTRYPOINT_LINES = 200`
- `MAX_ENTRYPOINT_BYTES = 25,000`
- `MAX_MEMORY_CHARACTER_COUNT = 40,000` (recommended max per file)

**Path resolution priority**:
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` env var
2. `autoMemoryDirectory` in settings.json (trusted sources only: policy, local, user; **not** `projectSettings`)
3. Default: `<memoryBase>/projects/<sanitized-git-root>/memory/`

**Enable/disable env vars**: `CLAUDE_CODE_DISABLE_AUTO_MEMORY`, `CLAUDE_CODE_SIMPLE`, `CLAUDE_CODE_REMOTE_MEMORY_DIR`

**Settings key**: `autoMemoryEnabled` in settings.json

**Team memory feature gate**: `TEAMMEM`

---

### Memory Types

**Source file**: `src/memdir/memoryTypes.ts`

```typescript
const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
```

**Trust constants** (string sections in memoryTypes.ts or memdir.ts):
- `TRUSTING_RECALL_SECTION`
- `MEMORY_DRIFT_CAVEAT`

---

### Query Loop

**Source file**: `src/query.ts`

**Function**: `queryLoop()` -- async generator, `while(true)` state machine

**Transition field values**: `next_turn`, `collapse_drain_retry`, `reactive_compact_retry`, `max_output_tokens_escalate`, `max_output_tokens_recovery`, `stop_hook_blocking`, `token_budget_continuation`

**Terminal reason values**: `completed`, `blocking_limit`, `model_error`, `aborted_streaming`, `aborted_tools`, `prompt_too_long`, `stop_hook_prevented`, `hook_stopped`, `max_turns`

**Error class**: `FallbackTriggeredError`

---

### Query Engine

**Source file**: `src/QueryEngine.ts`

**Class**: `QueryEngine`

**Key fields**:
- `mutableMessages: Message[]`
- `readFileState`
- `discoveredSkillNames` (turn-scoped)
- `loadedNestedMemoryPaths`
- `totalUsage`
- `permissionDenials`

**Entry point**: `submitMessage()` -- per-turn

---

### Message Types

**Source files**: `src/utils/messages.ts` (~5,500 lines), `src/types/message.ts`

**Functions**:
- `createUserMessage()` -- UUID, flags: `isMeta`, `isVirtual`, `isCompactSummary`, `origin`, `imagePasteIds`
- `normalizeMessagesForAPI()` -- 12-step pipeline
- `ensureToolResultPairing()` -- defensive validation
- `normalizeAttachmentForAPI()` -- 20+ attachment types

**System message subtypes**: `compact_boundary`, `microcompact_boundary`, `api_error`, `permission_retry`, `bridge_status`, `stop_hook_summary`, `turn_duration`, `memory_saved`, `api_metrics`, `local_command`

**Tags**: `<system-reminder>` tags, gate: `tengu_chair_sermon`

---

### Token Estimation

**Source files**: `src/services/tokenEstimation.ts`, `src/utils/tokens.ts`

**Functions**:
- `getTokenUsage()` -- exact counting from API responses
- `roughTokenCountEstimation()` -- char-to-token ratio
- `tokenCountWithEstimation()` -- canonical function, walks backward to last API response

**Constants**:
- `bytesPerToken = 4` (default)
- `bytesPerToken = 2` (JSON files)
- `MODEL_CONTEXT_WINDOW_DEFAULT = 200,000`
- Fixed `2000` tokens per image or document
- Image max: 2000x2000

**Context window determination**:
- `CLAUDE_CODE_MAX_CONTEXT_TOKENS` env var
- `[1m]` suffix -> 1M
- `max_input_tokens` from model capabilities
- `CONTEXT_1M_BETA_HEADER` + `modelSupports1M()` -> 1M
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW` env var

**Budget tracker class**: `BudgetTracker`

---

### Tool Execution

**Source files**: `src/services/tools/toolExecution.ts`, `src/services/tools/StreamingToolExecutor.ts`

**Pipeline functions**:
- `tool.inputSchema.safeParse()` -- Zod validation
- `tool.validateInput()`
- `tool.backfillObservableInput()` -- shallow clone mutation
- `runPreToolUseHooks()` -- can override permission
- `canUseTool()` -- returns allow/deny/ask
- `tool.call()` -- with `onProgress` callback
- `runPostToolUseHooks()` -- can modify MCP output
- `tool.maxResultSizeChars` -- large result persistence threshold

**Class**: `StreamingToolExecutor`
- `isConcurrencySafe()` -- determines parallel eligibility
- `getRemainingResults()` -- streams completion-order results
- `siblingAbortController.abort('sibling_error')` -- cascade abort

**Env var**: `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (default 10)

**Sub-agent**: `AgentTool`
- `createAgentWorktree()` -- worktree isolation
- `LocalAgentTask`, `RemoteAgentTask` -- async/background

---

### State Management

**Source files**: `src/state/AppStateStore.tsx`, `src/bootstrap/state.ts`

**React layer**:
- `createStore(initialState, onChange?)` -- Zustand-like
- `useAppState(selector)` (read), `useSetAppState()` (write), `useAppStateStore()` (direct)

**Bootstrap state**: Single `STATE` object in `bootstrap/state.ts` (~1,758 lines)

**CWD fields**: `originalCwd`, `projectRoot`, `cwd`

**Tool**: `EnterWorktreeTool` -- updates `cwd` mid-session

**Cost fields**: `totalCostUSD`, `totalAPIDuration`, per-model `modelUsage`

**Session fields**: `sessionId`, `parentSessionId`, `sessionProjectDir`

**Skill tracking**: `Map<string, InvokedSkillInfo>`

**Token budget**: `snapshotOutputTokensForTurn`, `getTurnOutputTokens`

**Post-compaction flag**: `pendingPostCompaction`

---

### Auto-Compact

**Source file**: `src/services/compact/autoCompact.ts`

**Functions**:
- `shouldAutoCompact()` -- evaluates threshold
- `calculateTokenWarningState()` -- warning/error/blocking states
- `autoCompactIfNeeded()` -- orchestrator

**Constants**: threshold formula `contextWindow - 13K`, blocking limit `effectiveContextWindow - 3K`

**Env var**: `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` (0-100%)

**Recursion guard query sources**: `compact`, `session_memory`, `marble_origami`

---

### Session Memory Compact

**Source file**: `src/services/compact/sessionMemoryCompact.ts`

**Function**: `adjustIndexToPreserveAPIInvariants()`

**Config (from GrowthBook `tengu_sm_compact_config`)**:
- `minTokens: 10,000`
- `minTextBlockMessages: 5`
- `maxTokens: 40,000`

---

### Full Summarization (Compact)

**Source files**: `src/services/compact/compact.ts`, `src/services/compact/prompt.ts`

**Functions**:
- `stripImagesFromMessages()` -- replaces with `[image]`/`[document]` markers
- `stripReinjectedAttachments()` -- removes `skill_discovery`/`skill_listing` types
- `streamCompactSummary()` -- forked agent path
- `truncateHeadForPTLRetry()` -- drops oldest API-round groups
- `partialCompactConversation()` -- partial compaction

**Constants**:
- `COMPACT_MAX_OUTPUT_TOKENS = 20,000`
- `MAX_PTL_RETRIES = 3`
- `MAX_COMPACT_STREAMING_RETRIES = 2`
- `NO_TOOLS_PREAMBLE`, `NO_TOOLS_TRAILER`

**Feature gate**: `tengu_compact_cache_prefix` -- forked agent cache sharing

**Interface**:
```typescript
interface CompactionResult {
  boundaryMarker: SystemMessage
  summaryMessages: UserMessage[]
  attachments: AttachmentMessage[]
  hookResults: HookResultMessage[]
  messagesToKeep?: Message[]
  preCompactTokenCount?: number
  postCompactTokenCount?: number
  truePostCompactTokenCount?: number
  compactionUsage?: CacheMetrics
}
```

**Partial compaction metadata**: `preservedSegment` with `headUuid`, `tailUuid`, `anchorUuid`

---

### Microcompact

**Source file**: `src/services/compact/microCompact.ts`

**Config (from GrowthBook `tengu_slate_heron`)**: Default 60-minute gap threshold, `keepRecent` default 5

**Compactable tool list**: Read, Bash, Grep, Glob, WebSearch, WebFetch, Edit, Write

**API fields**: `pendingCacheEdits`, `cache_reference`, `cache_edits`

**API-side config function**: `getAPIContextManagement()` -- trigger at 180K input tokens, keep 40K

---

### Reactive Compact

**Source file**: `src/services/compact/reactiveCompact.ts`

**Function**: `tryReactiveCompact()`

**Guard**: `hasAttemptedReactiveCompact` -- prevents infinite loops, reset per tool-execution turn

**Feature gate**: `REACTIVE_COMPACT`

---

### Snip Compact

**Source file**: `src/services/compact/snipCompact.ts`

**Feature gate**: `HISTORY_SNIP`

**Mechanism**: Message ID tags `[id:short-id]` appended during `normalizeMessagesForAPI()`

---

### Circuit Breaker

**Constant**: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`

---

### Memory Extraction

**Source file**: `src/services/extractMemories/extractMemories.ts`

**Functions/patterns**:
- `initExtractMemories()` -- returns closure with state
- `executeExtractMemories(context, appendSystemMessage)` -- fire-and-forget
- `drainPendingExtraction(timeoutMs)` -- called by `print.ts` before `gracefulShutdownSync`
- `hasMemoryWritesSince()` -- deduplication check
- `formatMemoryManifest(await scanMemoryFiles(memoryDir))` -- manifest injection

**Closure state**: `inFlightExtractions` (Set<Promise>), `lastMemoryMessageUuid`, `inProgress`, `turnsSinceLastExtraction`, `pendingContext`

**Fork config**:
```typescript
runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams,
  canUseTool,
  querySource: 'extract_memories',
  skipTranscript: true,
  maxTurns: 5,
})
```

**Tool permission check**: `isAutoMemPath(filePath)` for Edit/Write, `tool.isReadOnly(parsedData)` for Bash

**Feature gates**: `EXTRACT_MEMORIES` (compile-time), `tengu_passport_quail` (runtime activation), `tengu_slate_thimble` (non-interactive sessions), `tengu_bramble_lintel` (throttle config), `tengu_amber_prism` (correction hints)

---

### Auto-Dream

**Source file**: `src/services/extractMemories/autoDream.ts`

**Constants**:
- `SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000` (10 minutes)

**Config (from GrowthBook `tengu_onyx_plover`)**: `minHours` (default 24h), `minSessions` (default 5)

**Query source**: `'auto_dream'`

**Task type**: `DreamTask`

---

### KAIROS Daily Logs

**Source file**: `src/memdir/memdir.ts`

**Function**: `buildAssistantDailyLogPrompt()`

**Feature gate**: `feature('KAIROS')`

**Caching**: via `systemPromptSection('memory', ...)`

---

### Session Storage

**Source file**: `src/utils/sessionStorage.ts` (~4,500 lines)

**Path**: `~/.claude/projects/<sanitized-cwd>/<sessionId>.jsonl`

**Constants**:
- `FLUSH_INTERVAL_MS = 100`
- `MAX_CHUNK_BYTES = 100 * 1024 * 1024` (100MB)
- File permissions: mode `0o600`

**Functions**:
- `drainWriteQueue()` -- append in chunks
- `readLiteMetadata` -- scans last 64KB
- `getSessionFilesLite()` -- reads first 1KB and last 64KB

**Transcript entry fields**: `uuid`, `parentUuid`, `logicalParentUuid`, `isSidechain`, `sessionId`, `cwd`, `userType`, `timestamp`, `version`, `gitBranch`, `slug`

**Metadata entry types**: `custom-title`, `ai-title`, `last-prompt`, `tag`, `agent-name`, `agent-color`, `agent-setting`, `mode`, `worktree-state`, `pr-link`, `file-history-snapshot`, `attribution-snapshot`, `content-replacement`, `context-collapse-snapshot`, `context-collapse-commit`

---

### Session Resume

**Source files**: `src/commands/resume/`, `src/utils/sessionRestore.ts`

**Functions**:
- `applyPreservedSegmentRelinks()`
- `applySnipRemovals()`
- `checkResumeConsistency(chain)`
- `setCostStateForRestore()`
- `checkCrossProjectResume()`

**CLI flag**: `claude -r <sessionId> /path/to/project`, `--fork-session`

**Boundary scan threshold**: Files >5MB skip pre-compaction content

---

### Session Search

**Function**: `agenticSessionSearch()` -- pre-filter + Claude API relevance ranking (top 100)

**Feature gate**: `tengu_coral_fern` -- enables grep commands in memory prompt

---

### Telemetry Event Names

`tengu_memdir_loaded`, `tengu_memdir_disabled`, `tengu_compact`, `tengu_partial_compact`, `tengu_cached_microcompact`, `tengu_time_based_microcompact`, `tengu_auto_mem_tool_denied`, `tengu_extract_memories_coalesced`, `tengu_claude_md_permission_error`, `tengu_claudemd__initial_load`, `system_context_started`, `system_context_completed`, `git_status_started`, `git_status_completed`

---

### Feature Flags (Complete List)

**Compile-time** (`bun:bundle`):
`REACTIVE_COMPACT`, `CONTEXT_COLLAPSE`, `HISTORY_SNIP`, `CACHED_MICROCOMPACT`, `EXTRACT_MEMORIES`, `KAIROS`, `KAIROS_BRIEF`, `TEAMMEM`, `BREAK_CACHE_COMMAND`, `PROACTIVE`, `FORK_AGENT`, `EXPERIMENTAL_SKILL_SEARCH`, `VERIFICATION_AGENT`, `TOKEN_BUDGET`

**Runtime (GrowthBook)**:
`tengu_coral_fern`, `tengu_passport_quail`, `tengu_slate_thimble`, `tengu_moth_copse`, `tengu_herring_clock`, `tengu_amber_prism`, `tengu_bramble_lintel`, `tengu_onyx_plover`, `tengu_sm_compact_config`, `tengu_slate_heron`, `tengu_compact_cache_prefix`, `tengu_cobalt_raccoon`, `tengu_paper_halyard`, `tengu_chair_sermon`, `tengu_toolref_defer_j8m`
