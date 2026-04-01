# Claude Code: Context Management System — Design Specification

This document analyzes the context management architecture of Claude Code — Anthropic's
agentic CLI tool — focusing on how it assembles, grows, compresses, and persists context
across the lifecycle of a session and across sessions.

## Table of Contents

- [1. Vision](#1-vision)
- [2. Architecture Overview](#2-architecture-overview)
- [3. Phase 1: Context Assembly](#3-phase-1-context-assembly)
  - [3.1 System Prompt Construction](#31-system-prompt-construction)
  - [3.2 System Context Collection](#32-system-context-collection)
  - [3.3 Memory File Hierarchy (CLAUDE.md)](#33-memory-file-hierarchy-claudemd)
  - [3.4 Auto Memory (Persistent Cross-Session State)](#34-auto-memory-persistent-cross-session-state)
  - [3.5 Context Assembly Summary](#35-context-assembly-summary)
- [4. Phase 2: Context Growth](#4-phase-2-context-growth)
  - [4.1 The Query Loop](#41-the-query-loop)
  - [4.2 The Query Engine Wrapper](#42-the-query-engine-wrapper)
  - [4.3 Message Types and Structure](#43-message-types-and-structure)
  - [4.4 Token Estimation](#44-token-estimation)
  - [4.5 Tool Execution and Context Expansion](#45-tool-execution-and-context-expansion)
  - [4.6 State Management](#46-state-management)
- [5. Phase 3: Context Compression](#5-phase-3-context-compression)
  - [5.1 Decision: Should We Compact?](#51-decision-should-we-compact)
  - [5.2 Strategy 1: Session Memory Compact](#52-strategy-1-session-memory-compact)
  - [5.3 Strategy 2: Full Summarization](#53-strategy-2-full-summarization)
  - [5.4 Post-Compact Restoration](#54-post-compact-restoration)
  - [5.5 Microcompact: Per-Turn Tool Result Clearing](#55-microcompact-per-turn-tool-result-clearing)
  - [5.6 Partial Compaction](#56-partial-compaction)
  - [5.7 Reactive Compaction](#57-reactive-compaction)
  - [5.8 Snip Compaction](#58-snip-compaction)
  - [5.9 Circuit Breaker](#59-circuit-breaker)
  - [5.10 Compaction Flow Summary](#510-compaction-flow-summary)
- [6. Phase 4: Memory Extraction](#6-phase-4-memory-extraction)
  - [6.1 Background Extraction Agent](#61-background-extraction-agent)
  - [6.2 The Four-Type Memory Taxonomy](#62-the-four-type-memory-taxonomy)
  - [6.3 Auto-Dream Consolidation](#63-auto-dream-consolidation)
  - [6.4 Assistant Mode Daily Logs (KAIROS)](#64-assistant-mode-daily-logs-kairos)
- [7. Phase 5: Session Persistence and Resume](#7-phase-5-session-persistence-and-resume)
  - [7.1 Transcript Format](#71-transcript-format)
  - [7.2 Session Resume](#72-session-resume)
  - [7.3 Session Search](#73-session-search)
- [8. Cross-Cutting Concerns](#8-cross-cutting-concerns)
  - [8.1 Prompt Cache Efficiency](#81-prompt-cache-efficiency)
  - [8.2 Security Boundaries](#82-security-boundaries)
  - [8.3 Telemetry and Observability](#83-telemetry-and-observability)
  - [8.4 Feature Gating](#84-feature-gating)
- [9. Design Principles](#9-design-principles)

---

## 1. Vision

Claude Code's context management solves a fundamental tension in agentic AI systems:
**the model needs rich, growing context to do useful work, but has a finite context
window**. The system's vision is to make context limits invisible to the user — sessions
of arbitrary length should "just work" without the user ever thinking about tokens,
windows, or memory.

The design achieves this through three interlocking strategies:

1. **Layered context assembly** — compose the right context from heterogeneous sources
   at the right time
2. **Automatic context compression** — summarize old context when the window fills,
   preserving what matters
3. **Persistent memory extraction** — distill durable learnings from ephemeral sessions
   into files that survive across conversations

---

## 2. Architecture Overview

Context flows through five lifecycle phases:

```
┌─────────────┐   ┌─────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Assembly    │ → │   Growth    │ → │ Compression  │ → │  Extraction  │ → │ Persistence  │
│             │   │             │   │              │   │              │   │              │
│ System      │   │ Messages    │   │ Auto-compact │   │ Background   │   │ JSONL        │
│ prompt +    │   │ accumulate  │   │ summarizes   │   │ agent writes │   │ transcripts  │
│ memory +    │   │ across      │   │ old messages  │   │ durable      │   │ enable       │
│ CLAUDE.md + │   │ tool turns  │   │ when window  │   │ memories     │   │ resume       │
│ git status  │   │             │   │ fills        │   │              │   │              │
└─────────────┘   └─────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
```

---

## 3. Phase 1: Context Assembly

### 3.1 System Prompt Construction

> **Source:** `src/constants/prompts.ts`

The system prompt is returned as a **string array** from `getSystemPrompt()`, then
flattened for the API. It is constructed with a critical architectural split: a
**dynamic boundary marker** (`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`) separates static
(cacheable) content from session-specific content.

#### Static Region (Before Boundary — Cacheable)

Seven sections that are identical across users and organizations, enabling global
prompt cache sharing:

| Section | Function | Content |
|---------|----------|---------|
| Intro | `getSimpleIntroSection()` | "You are Claude Code, Anthropic's official CLI..." |
| System | `getSimpleSystemSection()` | Permission modes, tool results, compression reminders |
| Tasks | `getSimpleDoingTasksSection()` | Code style, over-engineering avoidance, security awareness |
| Actions | `getActionsSection()` | Reversibility heuristics, destructive operation confirmation |
| Tools | `getUsingYourToolsSection()` | Dedicated tool guidance (Read/Edit/Glob/Grep vs Bash) |
| Tone | `getSimpleToneAndStyleSection()` | Emoji policy, code references, conciseness |
| Efficiency | `getOutputEfficiencySection()` | Output size constraints, communication style |

#### Dynamic Region (After Boundary — Per-Session)

Thirteen sections resolved asynchronously via `resolveSystemPromptSections()`, each
individually cached via `systemPromptSection()` until `/clear` or `/compact`:

| Section | Cached? | Content |
|---------|---------|---------|
| `session_guidance` | Yes | Conditional tool guidance (skills, ask-user, explore) |
| `memory` | Yes | Auto-memory MEMORY.md + behavioral instructions |
| `env_info_simple` | Yes | Git status, model name, knowledge cutoff, platform |
| `language` | Yes | User language preference |
| `output_style` | Yes | Custom output formatting rules |
| `mcp_instructions` | **No** | MCP server instructions (volatile — servers connect/disconnect) |
| `scratchpad` | Yes | Scratchpad directory guidance |
| `frc` | Yes | Microcompact function result clearing instructions |
| `summarize_tool_results` | Yes | Tool result note-taking guidance |
| `token_budget` | Yes | Token budget instructions (feature-gated) |
| `brief` | Yes | KAIROS brief-mode tool guidance |
| `numeric_length_anchors` | Yes | Output length constraints (ant-only) |
| `ant_model_override` | Yes | Model override suffix (ant-only) |

The only **uncached** section is `mcp_instructions`, marked with
`DANGEROUS_uncachedSystemPromptSection()` because MCP servers can connect or
disconnect between turns. All other sections use the caching wrapper, which stores
results in `systemPromptSectionCache` and clears on `/clear` or `/compact`.

#### Proactive/KAIROS Mode Variant

When `feature('PROACTIVE') || feature('KAIROS')` is active and
`isProactiveActive()` returns true, the system prompt follows an **entirely different
path** — a minimal autonomous-agent intro replacing all seven static sections:

```
"You are an autonomous agent. Use the available tools to do useful work."
+ CYBER_RISK_INSTRUCTION
+ memory prompt
+ environment info
+ proactive-specific guidance (tick/wake-up loop, sleep tool, bias toward action)
```

#### Simple (Bare) Mode

When `CLAUDE_CODE_SIMPLE=true` (via `--bare` flag), returns a single-line prompt:
`"You are an interactive agent..."` — bypassing all section construction.

#### Model-Specific Variations

- **Knowledge cutoffs**: Per-model lookup (Opus 4.6 → May 2025, Sonnet 4.6 → August 2025)
- **Microcompact support**: Gated on model ID patterns in `getCachedMCConfigForFRC()`
- **Model name display**: Suppressed in undercover mode (ant-only)
- **Frontier model info**: Latest model IDs shown to help users building AI applications

### 3.2 System Context Collection

> **Source:** `src/context.ts`

Two memoized async functions run **once per session**:

**`getSystemContext()`** (line 116):
- Collects git branch, default branch, short status, recent 5 commits, user name
- All git commands run in parallel via `Promise.all()`
- Status truncated at `MAX_STATUS_CHARS = 2000` characters
- Skipped in remote sessions (`CLAUDE_CODE_REMOTE`) or when git instructions disabled
- Supports `systemPromptInjection` for cache-breaking (ant-only debugging)

**`getUserContext()`** (line 155):
- Loads CLAUDE.md files from the 4-level hierarchy (see §3.3)
- Applies injection filtering via `filterInjectedMemoryFiles()`
- Caches result for the auto-mode classifier via `setCachedClaudeMdContent()` (avoids
  import cycle: permissions → filesystem → permissions → yoloClassifier)
- Includes current date string
- Respects `CLAUDE_CODE_DISABLE_CLAUDE_MDS` env var and `--bare` mode

Both use lodash `memoize` — compute once, return cached result for the session's
lifetime. Cache cleared when `systemPromptInjection` changes.

### 3.3 Memory File Hierarchy (CLAUDE.md)

> **Source:** `src/utils/claudemd.ts`

Context is assembled from a four-level hierarchy loaded by `getMemoryFiles()`:

```
Priority (lowest → highest, higher overrides lower):

  /etc/claude-code/CLAUDE.md               ← Managed (enterprise policy)
  ~/.claude/CLAUDE.md + ~/.claude/rules/    ← User (personal preferences)
  .claude/CLAUDE.md + .claude/rules/*.md   ← Project (repo-level instructions)
  CLAUDE.local.md                          ← Local (gitignored overrides)
```

#### Directory Walk Strategy

`getMemoryFiles()` starts at the current working directory and walks **upward** to the
filesystem root, collecting CLAUDE.md, `.claude/CLAUDE.md`, `.claude/rules/*.md`, and
`CLAUDE.local.md` at each directory level. Files are processed in **root-to-CWD order**
because the model pays more attention to later content — closest instructions win.

**Worktree detection**: Nested git worktrees are detected by comparing
`findGitRoot(cwd)` with `findCanonicalGitRoot(cwd)`. When detected, Project-type files
from the main repo above the worktree are skipped to avoid duplication.

#### @include Directives

CLAUDE.md files support `@path` syntax for composing instructions:

- `@./relative/path` — relative to file's directory
- `@~/home/path` — home directory expansion
- `@/absolute/path` — absolute path
- Paths can contain escaped spaces: `@path\ with\ spaces.md`
- Fragment stripping: `@file.md#section` → `@file.md`
- Circular references prevented via `processedPaths` Set
- Non-existent targets silently ignored
- Max include depth: `MAX_INCLUDE_DEPTH = 5`
- Text-file allowlist (~100 extensions) restricts includes to safe formats

#### Conditional Rules

Files in `.claude/rules/` can have frontmatter `paths` fields specifying glob patterns.
When present, the rule is only loaded if the current target path matches the globs.
Two-pass loading: unconditional rules load eagerly, conditional rules load via
`processConditionedMdRules()` using the `ignore()` library for glob matching.

#### Injection Prevention

`filterInjectedMemoryFiles()` at line 1142 gates AutoMem/TeamMem removal when
`tengu_moth_copse` is active (these are surfaced via attachments instead).
HTML comments are stripped from markdown content. `processedPaths` prevents both
symlink-based and direct circular references.

#### Assembly into Prompt Text

`getClaudeMds()` at line 1153 iterates over files, generates per-type descriptions
(e.g., "(project instructions, checked into the codebase)"), and joins with a header:
"Codebase and user instructions are shown below. Be sure to adhere to these
instructions. IMPORTANT: These instructions OVERRIDE any default behavior..."

#### Caching

`getMemoryFiles` is memoized. `resetGetMemoryFilesCache(reason)` clears the cache and
enables hook firing. Reasons: `'session_start'`, `'compact'`, `'include'`.
`clearMemoryFileCaches()` clears without firing hooks (correctness-only invalidation).

### 3.4 Auto Memory (Persistent Cross-Session State)

> **Source:** `src/memdir/memdir.ts`, `src/memdir/paths.ts`

The auto-memory system provides persistent, per-project memory stored at
`~/.claude/projects/<sanitized-project-root>/memory/`.

#### MEMORY.md Entrypoint

A 200-line, 25KB-capped index file **always loaded into the system prompt**. Topic
files linked from the index are read on-demand by the model.

**Truncation** (`truncateEntrypointContent()`):
- Line cap: `MAX_ENTRYPOINT_LINES = 200`
- Byte cap: `MAX_ENTRYPOINT_BYTES = 25,000`
- Line-truncates first (natural boundary), then byte-truncates at last newline
- Appends warning naming which cap fired

#### Path Resolution

`getAutoMemPath()` (memoized, keyed on `getProjectRoot()`):

1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` env var — full-path override (SDK/Cowork)
2. `autoMemoryDirectory` in settings.json — trusted sources only (policy, local, user;
   **not** projectSettings for security — see §8.2)
3. `<memoryBase>/projects/<sanitized-git-root>/memory/` — default

Git worktrees sharing the same canonical root (`findCanonicalGitRoot()`) share one
memory directory.

**Path validation** (`validateMemoryPath()`): Rejects relative paths, root/near-root
(`length < 3`), Windows drive roots, UNC paths, null bytes. Normalizes to NFC with
trailing separator.

#### Behavioral Instructions

`buildMemoryLines()` injects detailed guidance into the system prompt:

- **Four-type taxonomy**: user, feedback, project, reference (see §6.2)
- **What NOT to save**: code patterns, architecture, git history, debugging recipes
- **How to save**: two-step process (write topic file → add index entry to MEMORY.md)
- **When to access**: trust recall, proactive consultation, verify before recommending
- **Distinction**: memory = cross-session, plans/tasks = within-session
- **Searching past context**: Grep commands for memory and transcript search

#### Team Memory

Feature-gated behind `TEAMMEM`. Shared `team/` subdirectory under auto-memory path.
Security includes `sanitizePathKey()`, `realpathDeepestExisting()`, and
`validateTeamMemWritePath()` — two-pass validation using both string normalization
and real symlink resolution.

#### Enable/Disable Chain

`isAutoMemoryEnabled()` priority chain:
1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY` env var (1/true → OFF)
2. `CLAUDE_CODE_SIMPLE` (--bare) → OFF
3. CCR without `CLAUDE_CODE_REMOTE_MEMORY_DIR` → OFF
4. `autoMemoryEnabled` in settings.json
5. Default: enabled

### 3.5 Context Assembly Summary

```
┌──────────────────────────────────────────────────────────┐
│ System Prompt (static, globally cacheable)                │
│  ├── Intro + system + tasks + actions + tools + tone      │
│  └── Output efficiency                                    │
├─ ─ ─ SYSTEM_PROMPT_DYNAMIC_BOUNDARY ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
│ System Prompt (dynamic, per-session, individually cached) │
│  ├── Session guidance (skills, explore, verification)     │
│  ├── Auto-memory MEMORY.md index (≤200 lines/25KB)       │
│  ├── Memory behavioral instructions (taxonomy, save/load) │
│  ├── Environment info (git, model, platform, date)        │
│  ├── Language preference                                  │
│  ├── Output style configuration                           │
│  ├── MCP server instructions (UNCACHED — volatile)        │
│  ├── Scratchpad, FRC, token budget, brief (conditional)   │
│  └── CLAUDE.md hierarchy (filtered for injection)         │
├──────────────────────────────────────────────────────────┤
│ Conversation Messages                                     │
│  ├── [Compaction boundary + summary, if compacted]        │
│  ├── User messages + attachments (20+ types)              │
│  ├── Assistant messages + tool calls + thinking blocks    │
│  ├── Tool results (paired with tool-use requests)         │
│  └── System messages (metadata, tombstones, progress)     │
└──────────────────────────────────────────────────────────┘
```

---

## 4. Phase 2: Context Growth

### 4.1 The Query Loop

> **Source:** `src/query.ts`

The query pipeline is an async generator (`queryLoop()`) implementing the core agentic
loop via a `while(true)` state machine with **7 continue sites** and **11 terminal
return conditions**.

#### Loop Iteration Structure

Each iteration follows this sequence:

1. **Setup**: Skill discovery prefetch, stream request start, query checkpoint
2. **Pre-processing**: Apply tool result budget, snip, microcompact, context collapse
3. **Proactive auto-compact check**: If tokens exceed threshold, compact before API call
4. **Model selection**: Pick model based on permission mode and token count
5. **API streaming**: Call model, process streamed blocks, collect tool calls
6. **Post-streaming**: Recovery checks, stop hooks, budget checks
7. **Tool execution**: Run tools, collect results, generate summaries
8. **Continue decision**: Inject tool results, advance turn count, loop

#### Continue Transitions

The `transition` field records why the loop continued:

| Reason | Trigger |
|--------|---------|
| `next_turn` | Normal tool execution → next API call |
| `collapse_drain_retry` | Drained context collapses after PTL error |
| `reactive_compact_retry` | Emergency compaction after PTL error |
| `max_output_tokens_escalate` | Escalated output cap 8K → 64K |
| `max_output_tokens_recovery` | Injected "resume directly..." message (up to 3x) |
| `stop_hook_blocking` | Stop hook injected blocking error |
| `token_budget_continuation` | Budget allows further work |

#### Terminal Conditions

| Reason | Condition |
|--------|-----------|
| `completed` | Normal turn end or API error |
| `blocking_limit` | Proactive token limit exceeded |
| `model_error` | Uncaught exception in API call |
| `aborted_streaming` | User abort during streaming |
| `aborted_tools` | User abort during tool execution |
| `prompt_too_long` | PTL recovery exhausted |
| `stop_hook_prevented` | Hook returned `preventContinuation: true` |
| `hook_stopped` | Hook set `shouldPreventContinuation` |
| `max_turns` | Turn count exceeds `maxTurns` limit |

#### Error Withholding

Prompt-too-long, max_output_tokens, and media errors are **withheld** from the SDK
during streaming. Recovery is attempted first; the error surfaces only if recovery
is exhausted. This prevents intermediate errors from terminating sessions in SDKs
that kill on any error.

#### Model Fallback

A `FallbackTriggeredError` handler switches to a fallback model mid-stream. Tombstone
messages clean up orphaned partial responses. Thinking signatures are stripped if the
fallback model doesn't support them.

### 4.2 The Query Engine Wrapper

> **Source:** `src/QueryEngine.ts`

`QueryEngine` wraps the query loop for SDK/headless mode:

- **`mutableMessages: Message[]`** — session history, persists across `submitMessage()` calls
- **`readFileState`** — recently-accessed files cache, persists across turns for
  post-compact restoration and memory prefetch deduplication
- **`discoveredSkillNames`** — turn-scoped (cleared each `submitMessage()`)
- **`loadedNestedMemoryPaths`** — prevents circular memory includes
- **`totalUsage`** — accumulated API usage across turns
- **`permissionDenials`** — tracked per-turn, reported in result

`submitMessage()` is the per-turn entry point: clears turn-scoped state, fetches
model-specific system prompt parts, validates thinking config, injects custom/append
system prompts, processes user input (slash commands, attachments), and invokes the
query loop.

### 4.3 Message Types and Structure

> **Source:** `src/utils/messages.ts` (~5,500 lines), `src/types/message.ts`

#### Core Message Types

- **UserMessage**: Text + attachments. Created via `createUserMessage()` with UUID,
  `isMeta` flags, `isVirtual`, `isCompactSummary`, `origin` tracking, `imagePasteIds`.
- **AssistantMessage**: Text + tool_use + thinking blocks. Can carry `isApiErrorMessage`
  and `isVirtual` flags.
- **SystemMessage**: 14+ subtypes including `compact_boundary`, `microcompact_boundary`,
  `api_error`, `permission_retry`, `bridge_status`, `stop_hook_summary`,
  `turn_duration`, `memory_saved`, `api_metrics`, `local_command`.
- **AttachmentMessage**: 20+ types (file, directory, plan, skills, memory, team context,
  skill discovery, queued commands, teammate mailbox).
- **TombstoneMessage**: Deletion markers preserving conversation structure.
- **ProgressMessage**: Streaming tool execution updates.
- **ToolUseSummaryMessage**: SDK-only batch summaries.

#### API Normalization Pipeline

`normalizeMessagesForAPI()` is a **12-step transformation pipeline**:

1. Reorder attachments (bubble up until tool_result or assistant)
2. Filter virtual messages
3. Strip problematic document/image blocks based on preceding API errors
4. Merge consecutive user messages (API requirement)
5. Normalize tool inputs (strip non-API fields)
6. Strip tool_reference blocks (when tool search disabled)
7. Inject turn boundary text after tool_reference messages
8. Relocate tool_reference siblings (when gate enabled)
9. Filter orphaned thinking-only assistant messages
10. Strip trailing thinking from last assistant message
11. Ensure non-empty assistant content (insert placeholder if needed)
12. Smoosh system-reminder siblings into tool_result blocks

#### Tool Result Pairing

`ensureToolResultPairing()` performs defensive validation:
- Strips duplicate tool_use IDs (cross-message dedup)
- Strips orphaned server_tool_use/mcp_tool_use blocks
- Inserts synthetic error tool_result blocks for unmatched tool_use blocks
- Strips orphaned tool_result blocks referencing non-existent tool_use blocks
- Creates placeholder user message for role alternation if needed

#### Attachment Normalization

`normalizeAttachmentForAPI()` converts 20+ attachment types into UserMessage tuples:
- **file**: FileReadTool use + result with content/images/PDFs/notebooks
- **directory**: Bash `ls` use + result
- **plan_file_reference**: Plan file path + contents
- **invoked_skills**: Skill contents with instructions
- **nested_memory**: Memory file contents
- **relevant_memories**: One user message per memory with header
- **queued_command**: Command input with origin tracking
- **teammate_mailbox**: Formatted teammate messages (swarm mode)
- All wrapped in `<system-reminder>` tags (when smoosh gate enabled)

### 4.4 Token Estimation

> **Source:** `src/services/tokenEstimation.ts`, `src/utils/tokens.ts`

#### Two Parallel Approaches

**Exact counting** (from API responses): `getTokenUsage()` extracts actual usage from
assistant messages. Only available AFTER an API call completes. Sums
`input_tokens + cache_creation_input_tokens + cache_read_input_tokens + output_tokens`.

**Rough estimation** (for prospective decisions): `roughTokenCountEstimation()` uses
a char-to-token ratio. Default `bytesPerToken = 4`, except JSON files which use
`bytesPerToken = 2` (dense token structure with many single-char tokens).

**`tokenCountWithEstimation()`** — the canonical function for context window
measurement. Walks backward to the last API response (exact count), then estimates
the delta for new messages added since.

#### Context Window Determination

Hierarchical resolution (first-match wins):
1. `CLAUDE_CODE_MAX_CONTEXT_TOKENS` env var (explicit cap)
2. `[1m]` suffix in model string → 1M tokens
3. Model capability metadata (`max_input_tokens`)
4. `CONTEXT_1M_BETA_HEADER` + `modelSupports1M()` → 1M
5. Default: `MODEL_CONTEXT_WINDOW_DEFAULT = 200,000`

Effective window: `contextWindow - min(maxOutputTokens, 20,000)` (reserves space for
compaction summary output). Further limited by `CLAUDE_CODE_AUTO_COMPACT_WINDOW`.

#### Image and Document Estimation

Fixed `2000` tokens per image or document. Images auto-resize to max 2000×2000.
PDF base64 encoding would naive-estimate ~325K tokens per MB, but actual API charge
is ~2000 tokens — using the fixed constant prevents massive overestimation.

#### Token Budget Feature

Users can specify budgets: `+500k`, `use 2M tokens`. Tracked via `BudgetTracker`
with continuation counting and diminishing-returns detection (3+ continuations with
<500 token deltas triggers completion).

### 4.5 Tool Execution and Context Expansion

> **Source:** `src/services/tools/toolExecution.ts`, `src/services/tools/StreamingToolExecutor.ts`

#### Tool Execution Pipeline

Every tool invocation passes through a 5-phase pipeline:

**Phase 1 — Lookup**: Find tool by name, check aliases, return "unknown tool" error
if not found.

**Phase 2 — Permission & Pre-Hooks**:
1. Input validation via `tool.inputSchema.safeParse()`
2. Custom validation via `tool.validateInput()`
3. Backfill observable input via `tool.backfillObservableInput()` (shallow clone mutation)
4. Pre-tool hooks (`runPreToolUseHooks()` — can override permission)
5. Speculative bash classifier (parallel with other checks)
6. Permission decision via `canUseTool()` → allow/deny/ask

**Phase 3 — Execution**: `tool.call()` with progress reporting via `onProgress` callback.

**Phase 4 — Post-Hooks**: `runPostToolUseHooks()` — can modify MCP output, prevent
continuation, add context.

**Phase 5 — Result Persistence**: Large results (exceeding `tool.maxResultSizeChars`)
persisted to disk, replaced with file path preview.

#### Streaming Tool Executor

`StreamingToolExecutor` enables **concurrent tool execution during model streaming**:

- **Safe tools** (`isConcurrencySafe() === true`): Execute in parallel
- **Unsafe tools**: Execute alone (exclusive lock), order preserved
- Max concurrency: `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || 10`
- Results streamed in completion order via `getRemainingResults()`
- Bash errors cascade to siblings via `siblingAbortController.abort('sibling_error')`

#### Sub-Agent Spawning

`AgentTool` creates sub-agents with rich configuration:
- MCP server initialization (string references share parent connections, inline defs
  create agent-specific clients)
- Fork subagents with worktree isolation (`createAgentWorktree()`)
- Async/background launch via `LocalAgentTask` or `RemoteAgentTask`
- Teammate spawning for multi-agent orchestration (flat roster, no nesting)

### 4.6 State Management

> **Source:** `src/state/AppStateStore.tsx`, `src/bootstrap/state.ts`

#### Two-Layer Architecture

**React State (AppState)** — reactive UI updates via `useSyncExternalStore`:
- 150+ fields tracking UI, permissions, tools, agents, MCP, plugins, costs
- Zustand-like store: `createStore(initialState, onChange?)` with pub/sub
- Hooks: `useAppState(selector)` (read), `useSetAppState()` (write, no re-render),
  `useAppStateStore()` (direct access)

**Bootstrap State (MODULE-LEVEL)** — non-React globals persisting across compaction:
- Single `STATE` object in `bootstrap/state.ts` (~1,758 lines)
- **Three CWD concepts**:
  - `originalCwd` — initial CWD, **never updated** (sets project identity)
  - `projectRoot` — stable root, set once at startup via `--worktree`
  - `cwd` — current CWD, updated by `EnterWorktreeTool` mid-session
- Cost/duration accumulators (`totalCostUSD`, `totalAPIDuration`, per-model `modelUsage`)
- Session identity (`sessionId`, `parentSessionId`, `sessionProjectDir`)
- Invoked skills (`Map<string, InvokedSkillInfo>` — preserved across compaction)
- Token budget state (`snapshotOutputTokensForTurn`, `getTurnOutputTokens`)
- OpenTelemetry meters and counters

#### State Survival Across Compaction

**Preserved**: `invokedSkills`, `planSlugCache`, `modelUsage`, cost accumulators,
`mainLoopModelOverride`, `sessionId`.

**Reset**: Turn-level metrics, thinking/speculation state, `pendingPostCompaction` flag
(set after compact, cleared after next API success).

**Session metadata re-append**: After compaction, metadata (title, tag, agent name, PR
link) is re-appended to the transcript tail to stay within the 64KB window that
`readLiteMetadata` scans.

---

## 5. Phase 3: Context Compression

### 5.1 Decision: Should We Compact?

> **Source:** `src/services/compact/autoCompact.ts`

`shouldAutoCompact()` evaluates:

1. **Token threshold**: `tokenCountWithEstimation(messages) >= (contextWindow - 13K)`
2. **Recursion guard**: Won't compact during compact, session_memory, or marble_origami queries
3. **Feature suppression**: `REACTIVE_COMPACT` or `CONTEXT_COLLAPSE` may suppress
4. **Snip accounting**: Subtract tokens freed by pending snips
5. **Env override**: `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` (0-100%) overrides threshold

Token warning states computed by `calculateTokenWarningState()`:
- **Warning**: threshold - 20K tokens
- **Error**: threshold - 20K tokens (same, red-state UI)
- **Blocking limit**: `effectiveContextWindow - 3K` (hard stop, user must `/compact`)

### 5.2 Strategy 1: Session Memory Compact

> **Source:** `src/services/compact/sessionMemoryCompact.ts`

A lightweight strategy that avoids an API call entirely:

- Keeps recent messages meeting BOTH minimums:
  - `minTokens: 10,000` (configurable via GrowthBook `tengu_sm_compact_config`)
  - `minTextBlockMessages: 5` (interactive messages with actual text)
- Hard cap: `maxTokens: 40,000`
- Floors at last compact boundary (preserves disk discontinuity invariant)

**Pair preservation** via `adjustIndexToPreserveAPIInvariants()`:
- Collects all tool_result IDs from kept messages
- Walks backward to find matching tool_use blocks in assistant messages
- Handles streaming-split messages (same `message.id` but different `uuid`s)
- Never splits a tool chain or thinking block sequence

**Tried first** in `autoCompactIfNeeded()`. Falls back to full summarization on failure.

### 5.3 Strategy 2: Full Summarization

> **Source:** `src/services/compact/compact.ts`

When session memory compact is insufficient:

#### Pre-Processing

- `stripImagesFromMessages()`: Replaces image/document blocks with `[image]`/`[document]`
  markers — prevents the compaction API call itself from hitting prompt-too-long
- `stripReinjectedAttachments()`: Removes skill_discovery/skill_listing types — these
  will be re-surfaced post-compact anyway

#### Summarization Prompt

The compact prompt (in `prompt.ts`) instructs the model to produce a structured summary
with **9 required sections**:

1. Primary Request and Intent
2. Key Technical Concepts (bullet list)
3. Files and Code Sections (with full snippets)
4. Errors and fixes (with resolution)
5. Problem Solving (solved and ongoing)
6. All user messages (non-tool-use only)
7. Pending Tasks
8. Current Work (precise description from recent messages)
9. Optional Next Step (with direct quotes)

The prompt includes `NO_TOOLS_PREAMBLE` and `NO_TOOLS_TRAILER` — aggressive "CRITICAL:
Respond with TEXT ONLY" instructions because Sonnet 4.6+ sometimes attempts tool calls
during summarization despite the trailer.

Output capped at `COMPACT_MAX_OUTPUT_TOKENS = 20,000`. If the compact request itself
overflows, `truncateHeadForPTLRetry()` drops oldest API-round groups and retries
(up to `MAX_PTL_RETRIES = 3`).

#### Cache-Sharing Optimization

`streamCompactSummary()` tries a **forked agent path** first (when
`tengu_compact_cache_prefix` enabled) to reuse the main conversation's cached prompt
prefix. Falls back to regular streaming if the fork fails. Implements
`MAX_COMPACT_STREAMING_RETRIES = 2` with exponential backoff and 30-second keep-alive
signals to prevent WebSocket idle timeouts.

#### CompactionResult

```typescript
interface CompactionResult {
  boundaryMarker: SystemMessage         // Compact boundary with metadata
  summaryMessages: UserMessage[]        // AI-generated conversation summary
  attachments: AttachmentMessage[]      // File/skill/plan/agent attachments
  hookResults: HookResultMessage[]      // Session start hook messages
  messagesToKeep?: Message[]            // Preserved segment (partial compact)
  preCompactTokenCount?: number
  postCompactTokenCount?: number
  truePostCompactTokenCount?: number
  compactionUsage?: CacheMetrics
}
```

### 5.4 Post-Compact Restoration

After compaction, **strategically re-inject** context lost in summarization:

| Restored Item | Budget | Details |
|---|---|---|
| Recently-accessed files (top 5) | 50K tokens total, 5K each | `readFileState` cache, most-recent-first, skips plan/memory files |
| Active skills | 25K tokens total, 5K each | Most-recent-first, truncated heads (where instructions live) |
| Plan file | Unbounded | If plan exists for current session |
| Plan mode instructions | Minimal | Ensures model continues in plan mode |
| Background agent status | Minimal | Running agents (prevent duplicate spawn), finished agents (results pending) |
| Deferred tools delta | Minimal | Tools announced since last boundary |
| Agent listing delta | Minimal | Agents discovered since last boundary |
| MCP instructions delta | Minimal | MCP server changes since last boundary |
| Session start hooks | Minimal | Restores CLAUDE.md context |

### 5.5 Microcompact: Per-Turn Tool Result Clearing

> **Source:** `src/services/compact/microCompact.ts`

A fine-grained, per-turn optimization that clears old tool results without full
compaction:

#### Time-Based Microcompact

When the gap since last assistant message exceeds a threshold (default 60 minutes,
from GrowthBook `tengu_slate_heron`), clears old tool results:
- Keeps last `keepRecent` results (default 5)
- Content-clears others with `'[Old tool result content cleared]'`
- Mutates message content directly (cache is cold anyway)
- Resets cached MC state

#### Cached Microcompact (Cache-Editing API)

For active sessions with warm cache:
- Collects compactable tool IDs (Read, Bash, Grep, Glob, WebSearch, WebFetch, Edit, Write)
- Registers tool results grouped by user message
- Calls `getToolResultsToDelete()` to determine which to clear
- Creates `cache_edits` blocks queued in `pendingCacheEdits`
- Does NOT mutate local message content — `cache_reference`/`cache_edits` added at API layer
- Preserves prompt cache while reducing context

#### API-Side Context Management

`getAPIContextManagement()` builds server-side clearing configs:
- **Tool result clearing**: Trigger at 180K input tokens, keep 40K, clear Read/Bash/Grep/Glob/Web
- **Tool use clearing**: Same trigger, clear everything EXCEPT Edit/Write/NotebookEdit
- **Thinking clearing**: Keep last 1 turn if >1h idle, otherwise keep all

### 5.6 Partial Compaction

> **Source:** `src/services/compact/compact.ts`, `partialCompactConversation()`

Targeted compaction around a selected message index:

- **Direction `from`**: Summarizes messages AFTER pivot, keeps earlier. Preserves
  prompt cache prefix.
- **Direction `up_to`**: Summarizes messages BEFORE pivot, keeps later. Invalidates
  cache since summary precedes kept messages.

Strips old compact boundaries from `messagesToKeep` (prevents double-pruning).
Annotates boundary with `preservedSegment` metadata (headUuid, tailUuid, anchorUuid)
for resume relinking.

### 5.7 Reactive Compaction

> **Source:** `src/services/compact/reactiveCompact.ts`

When the API returns a "prompt too long" error, the query loop attempts recovery:

1. **First**: Drain context-collapse queue (if CONTEXT_COLLAPSE enabled)
2. **Second**: `tryReactiveCompact()` — emergency compaction with `hasAttempted` guard
3. **If both fail**: Surface the withheld error

Gated behind `REACTIVE_COMPACT` feature flag. `hasAttemptedReactiveCompact` prevents
infinite loops (reset on each new tool-execution turn).

### 5.8 Snip Compaction

> **Source:** `src/services/compact/snipCompact.ts`

Finer-grained approach: identifies and snips specific segments (large tool results no
longer relevant). Uses message ID tags (`[id:short-id]`) appended during
`normalizeMessagesForAPI()` for the model to reference specific segments. Gated behind
`HISTORY_SNIP` feature flag.

### 5.9 Circuit Breaker

`autoCompactIfNeeded()` stops after `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`.
BQ data (2026-03-10): 1,279 sessions had 50+ consecutive failures (up to 3,272),
wasting ~250K API calls/day — the circuit breaker prevents this. Counter resets on
any successful compaction.

### 5.10 Compaction Flow Summary

```
┌─ autoCompactIfNeeded() ───────────────────────────────────────────┐
│                                                                    │
│  Circuit breaker: 3+ consecutive failures? → skip                  │
│         │                                                          │
│         ▼                                                          │
│  Session memory compact (lightweight, no API call)                 │
│  ├── Uses lastSummarizedMessageId boundary                        │
│  ├── Keeps recent messages (10K-40K tokens, 5+ text blocks)       │
│  ├── Preserves tool_use/tool_result pairs                         │
│  └── Success? → done                                              │
│         │ failure                                                  │
│         ▼                                                          │
│  Full summarization (API call, forked agent for cache sharing)     │
│  ├── Strip images/documents, re-injectable attachments            │
│  ├── 9-section structured summary (≤20K output tokens)            │
│  ├── PTL retry: truncate head, retry up to 3x                    │
│  ├── Post-compact restore: files, skills, plan, agents            │
│  └── Success? → done                                              │
│         │ failure                                                  │
│         ▼                                                          │
│  Increment consecutiveFailures → retry next turn or circuit break │
│                                                                    │
├─ Meanwhile, on API "prompt too long" error: ──────────────────────┤
│  1. Drain context-collapse queue                                  │
│  2. Reactive compact (emergency, one-shot guard)                  │
│  3. Surface error if both fail                                    │
│                                                                    │
├─ Every turn, before API call: ────────────────────────────────────┤
│  microcompactMessages()                                           │
│  ├── Time-based trigger (>60min gap → clear old tool results)     │
│  └── Cached MC path (cache-editing API, warm cache preserved)     │
│                                                                    │
└───────────────────────────────────────────────────────────────────┘
```

---

## 6. Phase 4: Memory Extraction

### 6.1 Background Extraction Agent

> **Source:** `src/services/extractMemories/extractMemories.ts`

A **forked agent** runs asynchronously after each model response, analyzing the session
for durable learnings.

#### Lifecycle

- **Trigger**: `handleStopHooks` fires when model stops (no pending tool calls)
- **Execution**: Fire-and-forget via `void executeExtractMemories(context, appendSystemMessage)`
- **Drain**: `drainPendingExtraction(timeoutMs)` called by `print.ts` before
  `gracefulShutdownSync` — ensures completion with 60-second soft timeout
- **State**: Closure-scoped inside `initExtractMemories()`:
  - `inFlightExtractions` (Set<Promise>)
  - `lastMemoryMessageUuid` (cursor position)
  - `inProgress` (overlap guard)
  - `turnsSinceLastExtraction` (throttling counter)
  - `pendingContext` (stashed for trailing runs)

#### Fork Configuration

```typescript
runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams,        // Share parent's prompt cache
  canUseTool,             // Restricted permissions (see below)
  querySource: 'extract_memories',
  skipTranscript: true,   // No transcript race conditions
  maxTurns: 5,            // Hard cap (well-behaved: 2-4 turns)
})
```

#### Restricted Permissions

| Tool | Access |
|---|---|
| Read, Grep, Glob | Unrestricted |
| REPL | When enabled; inner primitives re-gated |
| Bash | Read-only commands only (`tool.isReadOnly(parsedData)`) |
| Edit, Write | Only within auto-memory directory (`isAutoMemPath(filePath)`) |
| MCP, Agent, all others | Denied |

#### Deduplication and Throttling

- **Mutual exclusion**: `hasMemoryWritesSince()` checks if main agent wrote memories →
  skip extraction for that range (cursor advances past)
- **Overlap guard**: `inProgress` boolean, new requests during extraction are stashed
  as trailing runs (only latest context kept)
- **Throttle**: `turnsSinceLastExtraction` compared to `tengu_bramble_lintel` (default 1)
- **Trailing runs bypass throttle**: Process already-committed work

#### Memory Manifest Pre-Injection

`formatMemoryManifest(await scanMemoryFiles(memoryDir))` is injected into the
extraction prompt — lists all existing memory files with type, timestamp, and
description so the agent doesn't spend a turn on `ls`.

### 6.2 The Four-Type Memory Taxonomy

> **Source:** `src/memdir/memoryTypes.ts`

```typescript
const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
```

| Type | Scope | Purpose | Examples |
|------|-------|---------|----------|
| **user** | Always private | Role, goals, knowledge, preferences | "data scientist investigating logging", "10 years Go, new to React" |
| **feedback** | Private (team if project-wide) | Corrections AND confirmations on approach | "don't mock databases" + why + how to apply |
| **project** | Private or team | Ongoing work, goals NOT derivable from code/git | "freeze merges 2026-03-05", "auth rewrite for compliance" |
| **reference** | Usually team | Pointers to external systems | "bugs tracked in Linear INGEST", "Grafana latency board" |

#### Frontmatter Format

```markdown
---
name: {{memory name}}
description: {{one-line description — used for future relevance, be specific}}
type: {{user, feedback, project, reference}}
---

{{content — for feedback/project: rule/fact, then **Why:** and **How to apply:**}}
```

#### What NOT to Save

Code patterns, conventions, architecture, file paths, git history, debugging
solutions, CLAUDE.md content, ephemeral task details. Even when user asks "save this
PR list", push back and ask what was surprising or non-obvious.

#### Trust and Verification

`TRUSTING_RECALL_SECTION`: "A memory that names a specific function, file, or flag is
a claim it existed when written. It may have been renamed, removed, or never merged."
Before recommending: check file exists, grep for function/flag, verify before user acts.

`MEMORY_DRIFT_CAVEAT`: "Memory records can become stale. Before answering based on
memory, verify against current state. If recalled memory conflicts with current
information, trust what you observe now — update or remove the stale memory."

### 6.3 Auto-Dream Consolidation

> **Source:** `src/services/extractMemories/autoDream.ts`

A nightly consolidation engine that distills accumulated memory:

#### Three-Stage Gating

1. **Time gate**: `hoursSince >= minHours` (default 24h, from `tengu_onyx_plover`)
2. **Session gate**: `sessionCount >= minSessions` (default 5 sessions)
3. **Lock gate**: Acquire per-process consolidation lock

Session scan throttled to `SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000` (10 minutes)
to avoid expensive checks every turn.

#### Execution

Same forked-agent pattern as extraction but with `querySource: 'auto_dream'`.
Progress tracked via `DreamTask` with UI visibility. Watches forked agent messages
for text blocks (reasoning), collapses tool_use to counts, collects Edit/Write paths.

#### /remember Command

User-invocable skill (ant-only) that reviews all memory layers and proposes:
- Promotions: auto-memory → CLAUDE.md or CLAUDE.local.md
- Cleanup: duplicates, outdated entries, conflicts
- Presents ALL proposals before changes — review tool, not auto-action

### 6.4 Assistant Mode Daily Logs (KAIROS)

> **Source:** `src/memdir/memdir.ts`, `buildAssistantDailyLogPrompt()`

For perpetual sessions (gated behind `feature('KAIROS')`):

```
memory/
├── MEMORY.md                          ← distilled index (nightly)
├── logs/
│   └── 2026/
│       └── 03/
│           ├── 2026-03-30.md          ← timestamped bullets
│           └── 2026-03-31.md          ← today's log
└── topic files (distilled from logs)
```

- Agent appends timestamped bullets to daily log file
- Date path described as pattern in prompt (not literal) — cached by
  `systemPromptSection('memory', ...)`, NOT invalidated on date change
- Nightly `/dream` skill distills logs → topic files + MEMORY.md
- MEMORY.md serves as read-only distilled index

---

## 7. Phase 5: Session Persistence and Resume

### 7.1 Transcript Format

> **Source:** `src/utils/sessionStorage.ts` (~4,500 lines)

Sessions are serialized as **JSONL** (JSON Lines) — one JSON object per line,
append-only, never rewritten in place.

**Path**: `~/.claude/projects/<sanitized-cwd>/<sessionId>.jsonl`

**Entry types**: `user`, `assistant`, `system`, `attachment` (transcript messages),
plus metadata entries: `custom-title`, `ai-title`, `last-prompt`, `tag`,
`agent-name`, `agent-color`, `agent-setting`, `mode`, `worktree-state`, `pr-link`,
`file-history-snapshot`, `attribution-snapshot`, `content-replacement`,
`context-collapse-snapshot`, `context-collapse-commit`.

Each transcript message includes: `uuid`, `parentUuid`, `logicalParentUuid`,
`isSidechain`, `sessionId`, `cwd`, `userType`, `timestamp`, `version`, `gitBranch`,
`slug`.

#### Write Queue System

Writes are batched via a queue with `FLUSH_INTERVAL_MS = 100` and
`MAX_CHUNK_BYTES = 100 * 1024 * 1024` (100MB). `drainWriteQueue()` appends in chunks
with mode `0o600`. Session file materialized lazily on first user/assistant message
(prevents metadata-only files).

#### Metadata Tail Window

Session metadata (title, tag, agent name, mode, PR link) is **re-appended at EOF**
during compaction and session exit. This keeps metadata within the last 64KB that
`readLiteMetadata` scans for the resume picker, avoiding expensive full-file reads.

### 7.2 Session Resume

> **Source:** `src/commands/resume/`, `src/utils/sessionRestore.ts`

#### Resume Flow

1. **Load transcript**: Read JSONL forward. For files >5MB, skip pre-compaction content
   via fast boundary scan.
2. **Restore agent state**: Read agent definition from `agentSetting` entry, re-apply
   model and agent type.
3. **Build conversation chain**: Walk `parentUuid` chain from leaf to root:
   - Handle compaction boundaries (prune everything before last boundary)
   - Apply preserved segment relinks (`applyPreservedSegmentRelinks()`)
   - Apply snip removals (`applySnipRemovals()`)
   - Recover orphaned parallel tool results from streaming
   - Validate via `checkResumeConsistency(chain)`
4. **Restore cost state**: `setCostStateForRestore()` from session file

#### What IS Restored

Full message history (within compaction boundary), file history snapshots, attribution
state, context collapse state, worktree state, session metadata, cost state.

#### What IS Lost

Progress messages (ephemeral), pre-compaction messages, snipped messages, orphaned
attachments from crash-at-yield, metadata not in the tail window.

#### Cross-Project Resume

Detected via `checkCrossProjectResume()`. Requires explicit
`claude -r <sessionId> /path/to/project`. Reuses resumed session's ID unless
`--fork-session`. Worktree state restored only if directory still exists.

### 7.3 Session Search

> **Source:** `src/utils/sessionStorage.ts`

#### Lite Metadata (Fast Resume Picker)

`getSessionFilesLite()` reads only head (first 1KB) and tail (last 64KB) of each
transcript. Extracts: first prompt, title, tag, summary, modified date. Fast for
hundreds of sessions.

#### Agentic Session Search

`agenticSessionSearch()`:
1. Pre-filter by keyword (title, tag, branch, summary, first prompt)
2. Send top 100 to Claude API for relevance ranking
3. Return ranked indices
4. Priorities: tag > title > branch > summary > transcript

#### Searching Past Context (In-Session)

When `tengu_coral_fern` enabled, memory prompt includes grep commands:
- Memory files: `Grep pattern="<term>" path="<memoryDir>" glob="*.md"`
- Session transcripts: `Grep pattern="<term>" path="<projectDir>/" glob="*.jsonl"`

---

## 8. Cross-Cutting Concerns

### 8.1 Prompt Cache Efficiency

The system is optimized for Anthropic's prompt caching at every level:

- **Static/dynamic boundary**: `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` splits cacheable
  (global scope) from session-specific content
- **Section-level caching**: `systemPromptSection()` caches resolved sections until
  `/clear` or `/compact`; only `mcp_instructions` uses `DANGEROUS_uncachedSystemPromptSection()`
- **Forked agents**: Compaction and extraction share parent's cache via `CacheSafeParams`
- **Cached microcompact**: `cache_edits` API deletes tool results without invalidating cache
- **Partial compaction direction**: `from` preserves cache prefix; `up_to` invalidates
- **Tool pool ordering**: Built-in tools sorted alphabetically as contiguous prefix;
  MCP tools sorted after — stable cache key across turns
- **Prompt cache latches**: `afkModeHeaderLatched`, `fastModeHeaderLatched`,
  `cacheEditingHeaderLatched` — once a beta header is sent, it stays latched to
  prevent cache-busting oscillation
- **1-hour cache TTL awareness**: Time-based microcompact uses 60-minute threshold
  matching server cache TTL — never forces a miss that wouldn't happen anyway

### 8.2 Security Boundaries

#### Memory Path Validation

`validateMemoryPath()` (`src/memdir/paths.ts:109-150`):
- Rejects relative, root/near-root, Windows drive root, UNC, null-byte paths
- Normalizes to NFC Unicode with trailing separator
- `~` expansion only for `~/...` (bare `~` rejected — would match all of `$HOME`)

**Project settings exclusion**: `projectSettings` (.claude/settings.json) excluded
from `autoMemoryDirectory` overrides. Without this, a malicious repo could set
`autoMemoryDirectory: "~/.ssh"` and gain write access via the filesystem.ts carve-out.

**Team memory path security**: Two-pass validation — normalize + containment check,
then symlink resolution via `realpathDeepestExisting()`. Catches escapes that
`path.resolve()` alone cannot detect.

#### Memory Content Security

- `filterInjectedMemoryFiles()`: Gates AutoMem/TeamMem removal when injected via
  attachments instead
- HTML comment stripping from CLAUDE.md content
- `processedPaths` Set prevents circular reference attacks
- `MAX_INCLUDE_DEPTH = 5` caps @include recursion
- Text-file allowlist (~100 extensions) blocks binary includes
- `MAX_MEMORY_CHARACTER_COUNT = 40,000` recommended max per file

### 8.3 Telemetry and Observability

Every context management decision is instrumented:

| Event | Meaning |
|---|---|
| `tengu_memdir_loaded` | Memory dir stats (file/subdir count, content size, truncation) |
| `tengu_memdir_disabled` | Why memory was disabled (env var, setting) |
| `tengu_compact` | Full compaction metrics (tokens pre/post, cache, discovery state) |
| `tengu_partial_compact` | Partial compaction (direction, kept/summarized counts) |
| `tengu_cached_microcompact` | Cache-editing microcompact (tools cleared, tokens saved) |
| `tengu_time_based_microcompact` | Time-based trigger (gap minutes, results cleared) |
| `tengu_auto_mem_tool_denied` | Extraction agent tool denial |
| `tengu_extract_memories_coalesced` | Overlapping extraction stashed |
| `tengu_claude_md_permission_error` | CLAUDE.md file permission denied |
| `tengu_claudemd__initial_load` | Initial CLAUDE.md load statistics |
| `system_context_started/completed` | System context collection timing |
| `git_status_started/completed` | Git status timing and truncation |

### 8.4 Feature Gating

**Compile-time (`bun:bundle` — dead code eliminated):**

| Flag | Feature |
|------|---------|
| `REACTIVE_COMPACT` | Reactive compaction on prompt-too-long |
| `CONTEXT_COLLAPSE` | Alternative compaction strategy |
| `HISTORY_SNIP` | Snip-based compaction with message ID tags |
| `CACHED_MICROCOMPACT` | Cache-editing API microcompact |
| `EXTRACT_MEMORIES` | Background memory extraction |
| `KAIROS` | Assistant mode daily logs |
| `KAIROS_BRIEF` | Brief-mode tool guidance |
| `TEAMMEM` | Team memory synchronization |
| `BREAK_CACHE_COMMAND` | System prompt cache-breaking injection |
| `PROACTIVE` | Autonomous work mode |
| `FORK_AGENT` | Fork-based sub-agents |
| `EXPERIMENTAL_SKILL_SEARCH` | ToolSearch-based skill discovery |
| `VERIFICATION_AGENT` | Post-implementation verification |
| `TOKEN_BUDGET` | User-specified token budgets |

**Runtime (GrowthBook — progressive rollout):**

| Gate | Feature |
|------|---------|
| `tengu_coral_fern` | "Searching past context" in memory prompt |
| `tengu_passport_quail` | Extract mode activation |
| `tengu_slate_thimble` | Extract mode in non-interactive sessions |
| `tengu_moth_copse` | Skip MEMORY.md index in prompt (use attachments) |
| `tengu_herring_clock` | Team memory cohort tracking |
| `tengu_amber_prism` | Memory correction hints on user rejection |
| `tengu_bramble_lintel` | Extraction throttle (every N turns) |
| `tengu_onyx_plover` | Auto-dream timing (minHours, minSessions) |
| `tengu_sm_compact_config` | Session memory compact parameters |
| `tengu_slate_heron` | Time-based microcompact config |
| `tengu_compact_cache_prefix` | Forked agent cache sharing for compaction |
| `tengu_cobalt_raccoon` | Reactive-only compaction mode |
| `tengu_paper_halyard` | Skip project-level CLAUDE.md in prompt |
| `tengu_chair_sermon` | System-reminder smooshing in attachments |
| `tengu_toolref_defer_j8m` | Tool reference sibling relocation |

---

## 9. Design Principles

### 9.1 Invisibility

Users should never think about context limits. The system auto-compacts when the
window fills, auto-extracts learnings into persistent memory, auto-restores critical
context after compaction, and auto-resumes from disk — all without user intervention.

### 9.2 Structured Forgetting

Compaction is not random truncation. It is a 9-section structured summarization that
preserves **intent, decisions, file snapshots, and pending tasks** while discarding
verbatim tool output. An `<analysis>` scratchpad section is generated then stripped —
the model reasons about what matters before committing to the summary.

### 9.3 Strategic Restoration

Post-compact context is **curated, not just smaller**. The 5 most recently accessed
files, active skills, plan state, and background agent status are re-injected because
they are statistically most likely to be referenced next. Skill content is truncated
to heads (where setup instructions live), keeping the 5K most useful tokens per skill.

### 9.4 Defense in Depth for Memory

Memory files pass through multiple independent safety gates:
- Path traversal validation (rejects relative, root, UNC, null-byte)
- Symlink resolution (two-pass: normalize + realpath)
- Settings source exclusion (projectSettings cannot set memory directory)
- Content injection filtering (HTML comments stripped, suspicious patterns detected)
- Truncation caps (200 lines / 25KB for MEMORY.md)
- Include depth limits (max 5 levels)
- Text-file allowlist (~100 extensions, blocks binaries)
- NFC Unicode normalization

### 9.5 Cache-First Architecture

Every design decision considers prompt cache impact. The static/dynamic boundary
maximizes global cache sharing. Forked agents share parent caches. Section-level
caching prevents recomputation. MCP instructions are the only uncached section.
Cache-editing microcompact deletes tool results without invalidating the cache.
Beta header latching prevents oscillation. Tool pool ordering is alphabetically
stable. Partial compaction offers a cache-preserving direction.

### 9.6 Graceful Degradation

- Circuit breaker stops compaction after 3 consecutive failures
- Reactive compaction fires as last resort on prompt-too-long
- Session memory compact provides lightweight fallback (no API call)
- Time-based microcompact clears stale results after idle periods
- Max output tokens escalation (8K → 64K) with 3x recovery messages
- Model fallback on streaming errors (transparent model switch)
- Memory extraction silently skips when main agent already wrote memories
- Feature gates enable progressive rollout and instant rollback

### 9.7 Separation of Ephemeral and Durable State

Session context (messages, tool results) is **ephemeral** — grows, compresses,
disappears at session end. Durable learnings (user preferences, project patterns,
corrections) are **extracted** into persistent memory files surviving across sessions.
KAIROS extends this: raw observations are append-only logs, nightly distillation
produces the durable index. The `/remember` command promotes mature auto-memories
into CLAUDE.md or CLAUDE.local.md for permanent project configuration.

### 9.8 Pair Preservation

Tool-use and tool-result messages are always kept together. Session memory compact
walks the message array to find pair boundaries via
`adjustIndexToPreserveAPIInvariants()`. Streaming-split messages (same `message.id`,
different `uuid`s) are tracked and kept intact. `ensureToolResultPairing()` in the
API normalization pipeline inserts synthetic error results for orphaned tool_use
blocks and strips orphaned tool_result blocks.

### 9.9 Append-Only Durability

Session transcripts are JSONL, append-only, never rewritten. This provides crash
recovery — partial writes produce valid JSONL prefixes. The `parentUuid` chain
enables chain reconstruction from any suffix. Content replacement entries allow
large tool results to be externalized while maintaining cache stability. Metadata
re-append keeps critical fields in the tail window without rewriting the file.
