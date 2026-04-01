# Auto-Memory — Code Reference

### Source Files

| Section | Source Files |
|---|---|
| Memory directory structure | `src/memdir/paths.ts`, `src/memdir/memdir.ts` |
| MEMORY.md contract | `src/memdir/memdir.ts:34-103` |
| Background extraction agent | `src/services/extractMemories/extractMemories.ts`, `src/services/extractMemories/prompts.ts` |
| Memory taxonomy | `src/memdir/memoryTypes.ts` |
| Session memory | `src/services/SessionMemory/sessionMemory.ts`, `src/services/SessionMemory/prompts.ts`, `src/services/SessionMemory/sessionMemoryUtils.ts` |
| Memory recall | `src/memdir/findRelevantMemories.ts`, `src/memdir/memoryScan.ts`, `src/memdir/memoryAge.ts` |
| Team memory sync | `src/services/teamMemorySync/index.ts`, `src/services/teamMemorySync/watcher.ts`, `src/services/teamMemorySync/secretScanner.ts`, `src/services/teamMemorySync/teamMemSecretGuard.ts`, `src/services/teamMemorySync/types.ts`, `src/memdir/teamMemPaths.ts`, `src/memdir/teamMemPrompts.ts` |
| KAIROS memory model | `src/memdir/memdir.ts:327-370` |
| Path security | `src/memdir/paths.ts:109-150`, `src/memdir/teamMemPaths.ts` |
| Prompt injection filtering | `src/utils/claudemd.ts:1142` |
| Settings source restrictions | `src/memdir/paths.ts:173-178` |

### Functions and Methods

| Function | Location | Purpose |
|---|---|---|
| `getAutoMemPath()` | `src/memdir/paths.ts:223` | Resolves memory directory via 3-level priority chain; memoized by `getProjectRoot()` |
| `getMemoryBaseDir()` | `src/memdir/paths.ts` | Checks `CLAUDE_CODE_REMOTE_MEMORY_DIR` then falls back to `~/.claude` |
| `sanitizePath()` | `src/memdir/paths.ts` | Converts absolute path to safe directory name |
| `findCanonicalGitRoot()` | `src/memdir/paths.ts` | Resolves git worktrees to canonical root |
| `ensureMemoryDirExists()` | `src/memdir/memdir.ts:129` | Creates memory directory tree idempotently |
| `truncateEntrypointContent()` | `src/memdir/memdir.ts:57` | Line-then-byte truncation for MEMORY.md |
| `isAutoMemoryEnabled()` | `src/memdir/paths.ts:30-55` | Priority-chain enable/disable check |
| `isExtractModeActive()` | `src/memdir/paths.ts:69` | Checks compile-time + runtime gates for extraction |
| `loadMemoryPrompt()` | `src/memdir/memdir.ts:419` | Dispatches to KAIROS / team / auto-memory prompt builders |
| `buildAssistantDailyLogPrompt()` | `src/memdir/memdir.ts` | KAIROS daily-log prompt builder |
| `buildMemoryLines()` | `src/memdir/memdir.ts` | Auto-memory-only prompt builder |
| `initExtractMemories()` | `src/services/extractMemories/extractMemories.ts` | Initializes extraction agent closure |
| `hasMemoryWritesSince()` | `src/services/extractMemories/extractMemories.ts` | Detects main agent memory writes for mutual exclusion |
| `createAutoMemCanUseTool()` | `src/services/extractMemories/extractMemories.ts:171` | Tool permission function for extraction agent |
| `scanMemoryFiles()` | `src/memdir/memoryScan.ts:35` | Reads all .md files, parses frontmatter, sorts by mtime, caps at 200 |
| `selectRelevantMemories()` | `src/memdir/findRelevantMemories.ts:77` | Sonnet side-query for relevance selection |
| `findRelevantMemories()` | `src/memdir/findRelevantMemories.ts` | Orchestrates scan + select |
| `memoryAge()` | `src/memdir/memoryAge.ts` | Human-readable age string |
| `memoryFreshnessText()` | `src/memdir/memoryAge.ts` | Staleness caveat text |
| `memoryFreshnessNote()` | `src/memdir/memoryAge.ts` | System-reminder wrapped caveat |
| `validateMemoryPath()` | `src/memdir/paths.ts:109-150` | Path traversal validation |
| `validateTeamMemKey()` | `src/memdir/teamMemPaths.ts` | Team memory key validation (URL-encoded traversals, NFKC, symlinks) |
| `realpathDeepestExisting()` | `src/memdir/teamMemPaths.ts` | Symlink resolution on deepest existing ancestor |
| `isAutoMemPath()` | `src/memdir/paths.ts` | Checks if a path is within the auto-memory directory |
| `filterInjectedMemoryFiles()` | `src/utils/claudemd.ts:1142` | Prompt injection filter for memory files |
| `startTeamMemoryWatcher()` | `src/services/teamMemorySync/watcher.ts` | fs.watch-based file watcher for team memory |
| `batchDeltaByBytes()` | `src/services/teamMemorySync/index.ts` | Greedy bin-packing of PUT entries into batches |
| `checkTeamMemSecrets()` | `src/services/teamMemorySync/teamMemSecretGuard.ts` | Synchronous write-side secret guard |
| `scanForSecrets()` | `src/services/teamMemorySync/secretScanner.ts` | Upload-side secret scanner |
| `drainPendingExtraction()` | `src/services/extractMemories/extractMemories.ts` | Awaits in-flight extractions on shutdown |
| `createMemorySavedMessage()` | `src/services/extractMemories/extractMemories.ts` | Notification message injected into main conversation |
| `analyzeSectionSizes()` | `src/services/SessionMemory/sessionMemoryUtils.ts` | Per-section token count computation |
| `generateSectionReminders()` | `src/services/SessionMemory/sessionMemoryUtils.ts` | Warning generation for section size limits |
| `truncateSessionMemoryForCompact()` | `src/services/SessionMemory/sessionMemoryUtils.ts` | Truncation before compaction injection |
| `getSessionMemoryContent()` | `src/services/SessionMemory/sessionMemory.ts` | Reads session memory file |
| `getKairosActive()` | `src/memdir/memdir.ts` | Checks if KAIROS mode is active |
| `runForkedAgent()` | (not specified, likely in agent orchestration module) | Creates forked agent sharing parent's prompt cache |
| `sequential()` | (utility module) | Generic sequential execution wrapper |

### Constants

| Constant | Value | Location |
|---|---|---|
| `MAX_ENTRYPOINT_LINES` | 200 | `src/memdir/memdir.ts` |
| `MAX_ENTRYPOINT_BYTES` | 25,000 | `src/memdir/memdir.ts` |
| `MAX_MEMORY_FILES` | 200 | `src/memdir/memoryScan.ts` |
| `MAX_CONFLICT_RETRIES` | 2 | `src/services/teamMemorySync/index.ts` |
| `MAX_PUT_BODY_BYTES` | 200,000 | `src/services/teamMemorySync/index.ts` |
| `maxTurns` (extraction) | 5 | `src/services/extractMemories/extractMemories.ts` |
| Drain timeout | 60s | `src/services/extractMemories/extractMemories.ts` |
| Watcher debounce | 2s | `src/services/teamMemorySync/watcher.ts` |
| Session memory init threshold | 10,000 tokens | `src/services/SessionMemory/sessionMemory.ts` |
| Session memory update interval | 5,000 tokens | `src/services/SessionMemory/sessionMemory.ts` |
| Session memory tool call minimum | 3 | `src/services/SessionMemory/sessionMemory.ts` |
| Section token cap | ~2,000 tokens | `src/services/SessionMemory/sessionMemoryUtils.ts` |
| Total session memory cap | 12,000 tokens | `src/services/SessionMemory/sessionMemoryUtils.ts` |

### Closure-Scoped State (Extraction Agent)

```typescript
let lastMemoryMessageUuid: string | undefined   // cursor position
let inProgress: boolean                          // overlap guard
let turnsSinceLastExtraction: number             // throttle counter
let pendingContext: { context, appendSystemMessage } | undefined  // stash
const inFlightExtractions = new Set<Promise<void>>()  // drain set
```

All mutable state lives inside `initExtractMemories()`, following the same pattern as `confidenceRating.ts`.

### Environment Variables

| Variable | Purpose |
|---|---|
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | Full-path override for SDK/Cowork |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | Remote memory base directory |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | Enable/disable auto-memory (1/true = off, 0/false = on) |
| `CLAUDE_CODE_SIMPLE` | Bare/simple mode (disables auto-memory) |

### Settings Keys

| Key | Source Restrictions |
|---|---|
| `autoMemoryDirectory` | Trusted sources only: policy, flag, local, user. **Excludes** project settings |
| `autoMemoryEnabled` | Standard settings chain |

### Compile-Time Feature Flags

| Flag Name | Feature |
|---|---|
| `EXTRACT_MEMORIES` | Background memory extraction agent |
| `KAIROS` | Assistant mode daily logs and /dream |
| `TEAMMEM` | Team memory sync, combined prompts, secret scanning |
| `MEMORY_SHAPE_TELEMETRY` | Memory recall shape analytics |

### GrowthBook Runtime Gates

| Gate Name | Purpose |
|---|---|
| `tengu_passport_quail` | Extract mode activation |
| `tengu_slate_thimble` | Extract mode in non-interactive sessions |
| `tengu_moth_copse` | Skip MEMORY.md index in prompt |
| `tengu_herring_clock` | Team memory cohort tracking |
| `tengu_coral_fern` | "Searching past context" instructions |
| `tengu_amber_prism` | Memory correction hints on user rejection |
| `tengu_bramble_lintel` | Turn throttle for extraction (N turns between runs, default 1) |
| `tengu_session_memory` | Session memory feature gate |
| `tengu_sm_config` | Session memory remote configuration |

### Prompt Constants and Sections

| Constant | Purpose |
|---|---|
| `DIR_EXISTS_GUIDANCE` | System prompt text eliminating mkdir/ls before writes |
| `WHAT_NOT_TO_SAVE_SECTION` | Negative-space taxonomy definition |
| `TYPES_SECTION_INDIVIDUAL` | Type definitions for single-directory mode |
| `TYPES_SECTION_COMBINED` | Type definitions with scope routing for team mode |
| `WHEN_TO_ACCESS_SECTION` | Recall-side guidance for when to read memories |
| `TRUSTING_RECALL_SECTION` | Verification guidance for recalled memories |
| `MEMORY_DRIFT_CAVEAT` | Instruction to verify stale memories against current state |

### Telemetry Event Names

| Event | Category |
|---|---|
| `tengu_memdir_loaded` | Memory directory |
| `tengu_memdir_disabled` | Memory directory |
| `tengu_team_memdir_disabled` | Memory directory |
| `tengu_extract_memories_extraction` | Extraction |
| `tengu_extract_memories_skipped_direct_write` | Extraction |
| `tengu_extract_memories_coalesced` | Extraction |
| `tengu_extract_memories_gate_disabled` | Extraction |
| `tengu_extract_memories_error` | Extraction |
| `tengu_auto_mem_tool_denied` | Extraction |
| `tengu_session_memory_init` | Session memory |
| `tengu_session_memory_gate_disabled` | Session memory |
| `tengu_session_memory_extraction` | Session memory |
| `tengu_session_memory_file_read` | Session memory |
| `tengu_session_memory_loaded` | Session memory |
| `tengu_team_mem_sync_started` | Team memory |
| `tengu_team_mem_sync_pull` | Team memory |
| `tengu_team_mem_sync_push` | Team memory |
| `tengu_team_mem_secret_skipped` | Team memory |
| `tengu_team_mem_push_suppressed` | Team memory |
| `tengu_team_mem_entries_capped` | Team memory |

### API Endpoints (Team Memory)

```
GET  /api/claude_code/team_memory?repo={owner/repo}             -> full content + checksums
GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes -> metadata + checksums only
PUT  /api/claude_code/team_memory?repo={owner/repo}             -> upsert entries
```

### Type References

| Type/Interface | Location |
|---|---|
| `SyncState` | `src/services/teamMemorySync/types.ts` |
| `CacheSafeParams` | `src/utils/forkedAgent.ts` |

### Session Memory File Paths

- Session memory file: `~/.claude/session-memory/<session-id>.md`
- Template customization: `~/.claude/session-memory/config/template.md`
- Prompt customization: `~/.claude/session-memory/config/prompt.md`

### Session Memory Config Keys

| Config Key | Default |
|---|---|
| `minimumMessageTokensToInit` | 10,000 |
| `minimumTokensBetweenUpdate` | 5,000 |
| `toolCallsBetweenUpdates` | 3 |

### Integration Points

- `handleStopHooks` -- invokes the extraction agent as a post-sampling hook
- `registerPostSamplingHook` -- registration point for session memory extraction
- `createSubagentContext` -- creates isolated ToolUseContext for session memory
- `FileReadTool` -- used by session memory to read current memory file
- `FILE_EDIT_TOOL_NAME` -- the only tool permitted during session memory extraction
- `FileWriteTool.validateInput` / `FileEditTool.validateInput` -- call `checkTeamMemSecrets()`
- `collapseReadSearchGroups` -- render-path caller that requires memoized `getAutoMemPath()`
- `print.ts` -- calls `drainPendingExtraction()` after response flush
- `gracefulShutdownSync` -- 5s shutdown failsafe
- `systemPromptSection('memory', ...)` -- caches the memory portion of the system prompt
- `getSettingsForSource()` -- performs `realpathSync` + `readFileSync` (4 calls per resolution)

### Hash Format (Team Memory)

Server checksums use the format `sha256:<hex>`.

### Internal Security Reference

`PSR M22186` -- referenced in Section 10.1 as the security report that identified symlink-based directory escape vulnerabilities in memory path resolution.
