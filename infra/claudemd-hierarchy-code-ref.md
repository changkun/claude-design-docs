# CLAUDE.md Hierarchy — Code Reference

### Source Files

| File | Role |
|---|---|
| `src/utils/claudemd.ts` | Core module: file discovery, @path parsing, inclusion, filtering, assembly |
| `src/utils/memory/types.ts` | Memory type definitions (Managed, User, Project, Local, AutoMem, TeamMem) |
| `src/utils/settings/managedPath.ts` | Platform-dependent managed directory resolution |
| `src/utils/settings/constants.ts` | Setting source definitions, `getEnabledSettingSources()` |
| `src/utils/settings/settings.ts` | Settings loading, policy source cascade |
| `src/utils/frontmatterParser.ts` | YAML frontmatter parsing, `paths` field extraction |
| `src/context.ts` | Context assembly pipeline, bare mode logic |
| `src/constants/prompts.ts` | System prompt structure, dynamic boundary |
| `src/components/ClaudeMdExternalIncludesDialog.tsx` | External include approval UI |
| `src/utils/doctorContextWarnings.ts` | Content size diagnostic warnings |

### Key Functions and Line Ranges

| Function/Entity | File | Lines | Description |
|---|---|---|---|
| Module-level docstring | `src/utils/claudemd.ts` | 1-26 | Priority hierarchy documentation |
| `TEXT_FILE_EXTENSIONS` | `src/utils/claudemd.ts` | 96-227 | Allowlist of ~100 text file extensions |
| HTML comment stripping | `src/utils/claudemd.ts` | 292-334 | Block-level comment removal using `marked` lexer |
| Frontmatter stripping | `src/utils/claudemd.ts` | 254-279 | YAML frontmatter removal before injection |
| `@path` parsing / `extractIncludePathsFromTokens()` | `src/utils/claudemd.ts` | 451-535 | Include path extraction from markdown tokens |
| `claudeMdExcludes` logic | `src/utils/claudemd.ts` | 540-612 | Glob-based file exclusion |
| `processMemoryFile()` | `src/utils/claudemd.ts` | 618-685 | Recursive inclusion with cycle detection |
| `processMdRules()` | `src/utils/claudemd.ts` | 697-788 | Recursive rules directory processing |
| `getMemoryFiles()` | `src/utils/claudemd.ts` | 790 | Memoized entry point for all memory file loading |
| User file includes | `src/utils/claudemd.ts` | 835 | `includeExternal: true` for user-level files |
| File discovery / directory walk | `src/utils/claudemd.ts` | 850-934 | CWD-upward walk, reversed to root-downward |
| Git worktree deduplication | `src/utils/claudemd.ts` | 860-884 | `findGitRoot()` vs `findCanonicalGitRoot()` |
| Additional directories | `src/utils/claudemd.ts` | 936-977 | `--add-dir` CLAUDE.md loading |
| `InstructionsLoaded` hook | `src/utils/claudemd.ts` | 1042-1071 | Audit/observability hook for loaded files |
| `filterInjectedMemoryFiles()` | `src/utils/claudemd.ts` | 1142-1151 | Feature-gated filtering (`tengu_moth_copse`) |
| `getClaudeMds()` | `src/utils/claudemd.ts` | 1153 | Prompt string assembly |
| Project/Local skip | `src/utils/claudemd.ts` | 1158-1161 | Kill switch via `tengu_paper_halyard` feature flag |
| `processConditionedMdRules()` | `src/utils/claudemd.ts` | 1344-1397 | Conditional rule matching using `ignore` library |
| External include approval state | `src/utils/claudemd.ts` | 1399-1430 | Approval dialog trigger and config tracking |
| `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` | `src/constants/prompts.ts` | 114 | Prompt cache split point |
| Bare mode / disable flags | `src/context.ts` | 162-167 | `CLAUDE_CODE_DISABLE_CLAUDE_MDS`, `--bare` |
| Classifier cache setter | `src/context.ts` | 176 | `setCachedClaudeMdContent()` |
| Frontmatter path parsing | `src/utils/frontmatterParser.ts` | 189-266 | Brace expansion, comma handling |
| `policySettings` always-enabled | `src/utils/settings/constants.ts` | 164 | Unconditional addition to enabled sources |
| `EditableSettingSource` exclusion | `src/utils/settings/constants.ts` | 182-184 | Policy source is read-only |
| Policy source cascade | `src/utils/settings/settings.ts` | 322-323 | First non-null source wins |

### Key Constants

| Constant | Value | Location |
|---|---|---|
| `MAX_INCLUDE_DEPTH` | 5 | `src/utils/claudemd.ts` |
| `MAX_MEMORY_CHARACTER_COUNT` | 40,000 | `src/utils/claudemd.ts` (referenced in `doctorContextWarnings.ts`) |

### Key Types

- `MemoryFileInfo` -- return type of `getMemoryFiles()`, array of instruction file metadata
- `EditableSettingSource` -- excludes `policySettings` (defined at `src/utils/settings/constants.ts:182-184`)

### Feature Flags

| Flag Name | Effect |
|---|---|
| `tengu_moth_copse` | Excludes AutoMem/TeamMem from system prompt injection (surfaced via prefetch attachments instead) |
| `tengu_paper_halyard` | Skips Project and Local type files during prompt assembly (runtime kill switch) |
| `TEAMMEM` | Gates TeamMem memory type availability |

### Environment Variables

| Variable | Effect |
|---|---|
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | Hard disable: suppresses all CLAUDE.md content |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | Enables loading CLAUDE.md from `--add-dir` directories |

### Config Properties

| Property | Scope | Purpose |
|---|---|---|
| `hasClaudeMdExternalIncludesApproved` | Per-project | User granted external include permission |
| `hasClaudeMdExternalIncludesWarningShown` | Per-project | Dialog was already presented |

### Key Internal Functions (Referenced but Not Fully Described)

| Function | Module | Purpose |
|---|---|---|
| `safelyReadMemoryFileAsync()` | `src/utils/claudemd.ts` | File read with ENOENT/EISDIR error suppression |
| `safeResolvePath()` | `src/utils/claudemd.ts` | Symlink resolution for cycle detection |
| `normalizePathForComparison()` | `src/utils/claudemd.ts` | Platform-aware path normalization (Windows drive casing) |
| `expandPath()` | `src/utils/claudemd.ts` | `~/`, `./`, `/` prefix resolution |
| `getOriginalCwd()` | (utility) | Returns the original CWD at session start |
| `findGitRoot()` | (utility) | Returns the worktree root |
| `findCanonicalGitRoot()` | (utility) | Returns the main repo root |
| `clearMemoryFileCaches()` | `src/utils/claudemd.ts` | Correctness-only cache clear (no hook) |
| `resetGetMemoryFilesCache(reason)` | `src/utils/claudemd.ts` | Full cache clear + arms InstructionsLoaded hook |
| `setCachedClaudeMdContent()` | `src/context.ts` | Caches CLAUDE.md string for classifier |
| `getRemoteManagedSettingsSyncFromCache()` | `src/utils/settings/settings.ts` | Retrieves remote policy settings from cache |
| `getUserContext()` | `src/context.ts` | Builds `{ claudeMd, currentDate }` for system prompt |

### External Libraries

| Library | Usage |
|---|---|
| `marked` | Markdown lexing for `@path` extraction and HTML comment identification (used with `gfm: false`) |
| `ignore` | Gitignore-style glob matching for conditional rule `paths` frontmatter |
| `picomatch` | Glob matching for `claudeMdExcludes` patterns |
| `lodash.memoize` | Memoization of `getMemoryFiles()` |

### Import Cycle Note

The classifier cache (`setCachedClaudeMdContent()` at `src/context.ts:176`) exists to break this import cycle:

```
src/permissions/filesystem -> src/permissions -> yoloClassifier.ts -> src/utils/claudemd.ts
```

The cached string allows the classifier to read CLAUDE.md content without importing the `claudemd` module.
