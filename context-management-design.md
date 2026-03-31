# Claude Code: Context Management System — Design Specification

This document analyzes the context management architecture of Claude Code — Anthropic's
agentic CLI tool — focusing on how it assembles, grows, compresses, and persists context
across the lifecycle of a session and across sessions.

## Table of Contents

- [1. Vision](#1-vision)
- [2. Architecture Overview](#2-architecture-overview)
- [3. Phase 1: Context Assembly](#3-phase-1-context-assembly)
  - [3.1 System Prompt Construction](#31-system-prompt-construction)
  - [3.2 The Cache Boundary](#32-the-cache-boundary)
  - [3.3 System Prompt Sections and Caching](#33-system-prompt-sections-and-caching)
  - [3.4 System Context Collection](#34-system-context-collection)
  - [3.5 Memory File Hierarchy (CLAUDE.md)](#35-memory-file-hierarchy-claudemd)
  - [3.6 @include Directives](#36-include-directives)
  - [3.7 Auto Memory (Persistent Cross-Session State)](#37-auto-memory-persistent-cross-session-state)
  - [3.8 Memory Type Taxonomy](#38-memory-type-taxonomy)
  - [3.9 Context Assembly Summary](#39-context-assembly-summary)
- [4. Phase 2: Context Growth](#4-phase-2-context-growth)
  - [4.1 The Query Loop State Machine](#41-the-query-loop-state-machine)
  - [4.2 The Query Engine Wrapper](#42-the-query-engine-wrapper)
  - [4.3 Message Types and Structure](#43-message-types-and-structure)
  - [4.4 Attachment System](#44-attachment-system)
  - [4.5 Tool Execution and Streaming](#45-tool-execution-and-streaming)
  - [4.6 Token Estimation](#46-token-estimation)
  - [4.7 Transition and Recovery Mechanisms](#47-transition-and-recovery-mechanisms)
- [5. Phase 3: Context Compression](#5-phase-3-context-compression)
  - [5.1 Compaction Decision](#51-compaction-decision)
  - [5.2 Session Memory Compact (Lightweight)](#52-session-memory-compact-lightweight)
  - [5.3 Full Summarization (Heavyweight)](#53-full-summarization-heavyweight)
  - [5.4 Summarization Prompt Design](#54-summarization-prompt-design)
  - [5.5 Post-Compact Restoration](#55-post-compact-restoration)
  - [5.6 Microcompaction](#56-microcompaction)
  - [5.7 Partial Compaction](#57-partial-compaction)
  - [5.8 Reactive Compaction](#58-reactive-compaction)
  - [5.9 Snip Compaction](#59-snip-compaction)
  - [5.10 Circuit Breaker](#510-circuit-breaker)
  - [5.11 Post-Compact State Cleanup](#511-post-compact-state-cleanup)
  - [5.12 Compaction Flow Summary](#512-compaction-flow-summary)
- [6. Phase 4: Memory Extraction](#6-phase-4-memory-extraction)
  - [6.1 Background Extraction Agent](#61-background-extraction-agent)
  - [6.2 Extraction Prompt and Manifest](#62-extraction-prompt-and-manifest)
  - [6.3 Deduplication and Throttling](#63-deduplication-and-throttling)
  - [6.4 Auto-Dream Consolidation](#64-auto-dream-consolidation)
  - [6.5 Assistant Mode Daily Logs (KAIROS)](#65-assistant-mode-daily-logs-kairos)
- [7. Session Persistence and Resume](#7-session-persistence-and-resume)
  - [7.1 Transcript Format](#71-transcript-format)
  - [7.2 Session Storage Layout](#72-session-storage-layout)
  - [7.3 Write Queue System](#73-write-queue-system)
  - [7.4 Resume Flow](#74-resume-flow)
  - [7.5 Compaction and Resume Interaction](#75-compaction-and-resume-interaction)
- [8. State Management](#8-state-management)
  - [8.1 Bootstrap State (Global)](#81-bootstrap-state-global)
  - [8.2 AppState (Reactive UI)](#82-appstate-reactive-ui)
  - [8.3 State Survival Across Compaction](#83-state-survival-across-compaction)
  - [8.4 Session and CWD Identity](#84-session-and-cwd-identity)
- [9. Cross-Cutting Concerns](#9-cross-cutting-concerns)
  - [9.1 Prompt Cache Efficiency](#91-prompt-cache-efficiency)
  - [9.2 Security Boundaries](#92-security-boundaries)
  - [9.3 Telemetry and Observability](#93-telemetry-and-observability)
  - [9.4 Feature Gating](#94-feature-gating)
- [10. Design Principles](#10-design-principles)

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

Context flows through four lifecycle phases:

```
┌─────────────┐    ┌─────────────┐    ┌──────────────┐    ┌──────────────┐
│  Assembly    │ →  │   Growth    │ →  │ Compression  │ →  │  Extraction  │
│             │    │             │    │              │    │              │
│ System      │    │ Messages    │    │ Auto-compact │    │ Background   │
│ prompt +    │    │ accumulate  │    │ summarizes   │    │ agent writes │
│ memory +    │    │ across      │    │ old messages  │    │ durable      │
│ CLAUDE.md + │    │ tool turns  │    │ when window  │    │ memories     │
│ git status  │    │             │    │ fills        │    │              │
└─────────────┘    └─────────────┘    └──────────────┘    └──────────────┘
                                             │
                                    ┌────────┴────────┐
                                    │  Persistence     │
                                    │  JSONL transcript │
                                    │  + session resume │
                                    └─────────────────┘
```

The system implements **effectively unlimited context** through automatic summarization
while maintaining **prompt cache efficiency** and **recovery reliability**. Sessions are
persisted as JSONL transcripts, enabling resume across process restarts.

---

## 3. Phase 1: Context Assembly

### 3.1 System Prompt Construction

> **Source:** `src/constants/prompts.ts`

The system prompt is returned as a **string array** from `getSystemPrompt()`. It has
three distinct paths depending on the execution mode:

**Normal (interactive) mode** — 7 static sections + 13 dynamic sections:

```
Static (cacheable):
  1. getSimpleIntroSection()          — "You are Claude Code..." + cyber risk
  2. getSimpleSystemSection()         — Permission modes, tool results, hooks
  3. getSimpleDoingTasksSection()     — Code style, over-engineering avoidance
  4. getActionsSection()              — Reversibility heuristics, destructive ops
  5. getUsingYourToolsSection()       — Dedicated tools vs Bash, parallel calls
  6. getSimpleToneAndStyleSection()   — Emoji policy, code references, phrasing
  7. getOutputEfficiencySection()     — Communication style guidance

  ── SYSTEM_PROMPT_DYNAMIC_BOUNDARY ──

Dynamic (per-session, resolved async):
  1. session_guidance       — Tool-specific guidance (skills, verification, etc.)
  2. memory                 — Auto-memory MEMORY.md + behavioral instructions
  3. ant_model_override     — Internal model suffix
  4. env_info_simple        — Git status, model name, platform, shell
  5. language               — User language preference
  6. output_style           — Custom formatting rules
  7. mcp_instructions       — MCP server capabilities (DANGEROUS_uncached)
  8. scratchpad             — Scratchpad directory (replaces /tmp)
  9. frc                    — Function result clearing (microcompact)
  10. summarize_tool_results — Tool result note-taking
  11. numeric_length_anchors — Output length guidance (internal only)
  12. token_budget           — Token budget instructions
  13. brief                  — KAIROS brief tool section
```

**Proactive/KAIROS mode** — entirely different prompt:

```
"You are an autonomous agent. Use the available tools to do useful work."
+ CYBER_RISK_INSTRUCTION
+ System reminders
+ Memory prompt
+ Environment info
+ Language, MCP, scratchpad, FRC, tool result notes
+ Proactive work instructions (tick/wake-up loop, SleepTool pacing)
```

**Simple mode** (`--bare`) — single-line prompt, all sections skipped.

### 3.2 The Cache Boundary

> **Source:** `src/constants/prompts.ts:114-115`

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

This marker separates **cross-org cacheable** content from **session-specific** content.
It is consumed by `src/utils/api.ts` (`splitSysPromptPrefix`) and
`src/services/api/claude.ts` (`buildSystemPromptBlocks`) to set cache scope headers.

Everything before the boundary can use `scope: 'global'` (shared cache across
organizations); everything after is unique per user/session.

The boundary is inserted **only if** `shouldUseGlobalCacheScope()` returns true.

### 3.3 System Prompt Sections and Caching

> **Source:** `src/constants/systemPromptSections.ts`

Dynamic sections use a two-tier caching mechanism:

```typescript
systemPromptSection(name, compute)             // Cached until /clear or /compact
DANGEROUS_uncachedSystemPromptSection(name, compute, reason)  // Recomputed every turn
```

Cached sections (`memory`, `env_info_simple`, `language`, etc.) are resolved once via
`resolveSystemPromptSections()` and stored in a module-level cache map. The cache is
invalidated by `/clear` and `/compact` commands via `clearSystemPromptSections()`.

Only `mcp_instructions` is `DANGEROUS_uncached` — MCP servers can connect/disconnect
between turns, so their instructions must be recomputed. A delta optimization
(`isMcpInstructionsDeltaEnabled()`) can move MCP instructions to persistent attachments
instead, avoiding prompt cache busts on late MCP connect.

### 3.4 System Context Collection

> **Source:** `src/context.ts`

Two memoized async functions run **once per session**:

**`getSystemContext()`** (line 116):
- Collects git branch, default branch, short status, recent 5 commits, user name
- All git commands run in parallel via `Promise.all()`
- Status truncated at `MAX_STATUS_CHARS = 2000` characters
- Skipped in remote sessions (`CLAUDE_CODE_REMOTE`) or when git instructions disabled
- Supports `systemPromptInjection` for cache-breaking (debugging only)

**`getUserContext()`** (line 155):
- Loads CLAUDE.md files from the 4-level hierarchy (see §3.5)
- Applies injection filtering via `filterInjectedMemoryFiles()`
- Caches result for the auto-mode classifier via `setCachedClaudeMdContent()` (avoids
  import cycle: permissions → filesystem → permissions → yoloClassifier)
- Includes current date string
- Respects `CLAUDE_CODE_DISABLE_CLAUDE_MDS` env var and `--bare` mode

Both use lodash `memoize` — cache can be explicitly cleared when `systemPromptInjection`
changes or on compaction via `resetGetMemoryFilesCache('compact')`.

### 3.5 Memory File Hierarchy (CLAUDE.md)

> **Source:** `src/utils/claudemd.ts`

Context is assembled from a four-level hierarchy of "memory files", loaded by
`getMemoryFiles()` at line 790. The walk traverses upward from CWD to filesystem root,
then processes directories in **reverse** order (root-to-CWD) so that closer files are
higher priority (the model pays more attention to later content).

```
Priority (lowest → highest):

  /etc/claude-code/CLAUDE.md               ← Managed (enterprise policy)
  ~/.claude/CLAUDE.md + ~/.claude/rules/   ← User (personal preferences)
  .claude/CLAUDE.md + .claude/rules/*.md   ← Project (repo-level, checked in)
  CLAUDE.local.md                          ← Local (gitignored overrides)
```

Additionally loaded:
- **AutoMem entrypoint**: `~/.claude/projects/<slug>/memory/MEMORY.md`
- **TeamMem entrypoint**: `~/.claude/projects/<slug>/memory/team/MEMORY.md` (feature-gated)

**Nested worktree detection**: When the CWD is inside a git worktree, the system
detects the canonical root via `findCanonicalGitRoot()` and skips Project-type files
from the main repo above the worktree to avoid duplicate loading.

**Rules directory**: `.claude/rules/*.md` files are recursively discovered. Each file
can have frontmatter with `paths` glob patterns for conditional inclusion (rules that
apply only to certain files).

**Assembly into prompt text** happens in `getClaudeMds()` (line 1153), which:
- Applies optional type filter
- Generates per-file description strings based on type
- Wraps TeamMem content in `<team-memory-content>` XML tags
- Prefixes with instruction: "These instructions OVERRIDE any default behavior"

**Injection prevention**: `filterInjectedMemoryFiles()` (line 1142) removes AutoMem
and TeamMem entries when they're being injected via attachments instead (gated by
`tengu_moth_copse`).

**Error handling**: Missing files (`ENOENT`) silently ignored. Permission errors
(`EACCES`) logged to analytics without PII. Non-text files silently skipped via a
100+ extension allowlist.

### 3.6 @include Directives

> **Source:** `src/utils/claudemd.ts:448-535`

CLAUDE.md files support `@path` syntax for composing instructions from multiple sources.
The regex pattern is `(?:^|\s)@((?:[^\s\\]|\\ )+)`.

**Supported path forms:**
- `@path` — treated as `@./path` (relative to file's directory)
- `@./relative/path` — explicit relative
- `@~/home/path` — home directory expansion
- `@/absolute/path` — absolute path
- Escaped spaces: `@path\ with\ spaces.md`

**Safety constraints:**
- Only works in **leaf text nodes** (not code blocks, not codespans)
- Circular references prevented via `processedPaths` Set (tracks both path and resolved symlink)
- Max include depth: `MAX_INCLUDE_DEPTH = 5`
- Text-file allowlist (~100 extensions) restricts to safe formats
- Binary files (images, PDFs) explicitly excluded
- Fragment stripping: `@file.md#section` → `@file.md`
- HTML comments stripped before processing

### 3.7 Auto Memory (Persistent Cross-Session State)

> **Source:** `src/memdir/memdir.ts`, `src/memdir/paths.ts`

The auto-memory system provides persistent, per-project memory stored at
`~/.claude/projects/<sanitized-project-root>/memory/`.

**The entrypoint is `MEMORY.md`** — a capped index file always loaded into the
system prompt:
- Line cap: `MAX_ENTRYPOINT_LINES = 200`
- Byte cap: `MAX_ENTRYPOINT_BYTES = 25,000`
- Line-truncates first, then byte-truncates at last newline before the cap
- Appends warning naming which cap fired

**Path resolution** (`getAutoMemPath()`):
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` — full-path override for SDK/Cowork
2. `autoMemoryDirectory` in settings.json — trusted sources only (policy, local, user;
   **not** projectSettings — see §9.2)
3. `<memoryBase>/projects/<sanitized-git-root>/memory/` — default

Git worktrees sharing the same canonical root share one memory directory.

**Behavioral instructions** (`buildMemoryLines()`): The system prompt includes:
- Closed four-type taxonomy (see §3.8)
- What NOT to save (derivable-from-code patterns)
- How to save (frontmatter format, two-step: write file → add index entry)
- When to access (trust recall, verify before recommending)
- Distinction from plans/tasks (memory = cross-session)
- Searching past context (grep memory dir + session transcripts)

**Directory existence guarantee**: `ensureMemoryDirExists()` runs once per session via
the `systemPromptSection` cache. The prompt tells the model the directory "already
exists" so it doesn't waste turns on `mkdir`.

**Team memory** (feature-gated `TEAMMEM`): A shared `team/` subdirectory under the
auto-memory path. Path traversal defense uses two-pass validation: normalize +
containment check, then symlink resolution via `realpathDeepestExisting()`.

### 3.8 Memory Type Taxonomy

> **Source:** `src/memdir/memoryTypes.ts`

Memories are classified into four types:

| Type | Scope | Purpose | Examples |
|------|-------|---------|---------|
| **user** | Always private | User's role, goals, knowledge, preferences | "data scientist", "10 years Go, new to React" |
| **feedback** | Default private | Corrections AND confirmations on how to work | "don't mock databases" (with why + how to apply) |
| **project** | Private or team | Ongoing work, goals, deadlines NOT in code/git | "freeze merges for mobile release" |
| **reference** | Usually team | Pointers to external systems for up-to-date info | "bugs tracked in Linear INGEST" |

**Explicit exclusions** (WHAT_NOT_TO_SAVE_SECTION):
- Code patterns, conventions, architecture, file paths (derive from reading current state)
- Git history, recent changes, blame info
- Debugging solutions or fix recipes (the code is authoritative)
- Anything in CLAUDE.md files
- Ephemeral task details, current conversation context

**Frontmatter format:**
```markdown
---
name: {{memory name}}
description: {{one-line description — decides relevance in future conversations}}
type: {{user, feedback, project, reference}}
---
{{content — for feedback/project: rule/fact, then Why: and How to apply: lines}}
```

**Memory drift caveat** (WHEN_TO_ACCESS_SECTION): "Memory records can become stale.
Before answering based on memory, verify against current state. If a recalled memory
conflicts with current information, trust what you observe now — and update or remove
the stale memory."

**Verification heuristic** (TRUSTING_RECALL_SECTION): If memory names a file path,
check it exists. If memory names a function or flag, grep for it. "The memory says X
exists" is not the same as "X exists now."

### 3.9 Context Assembly Summary

The complete context handed to the API for each turn:

```
┌─────────────────────────────────────────────────────┐
│ System Prompt (static, globally cacheable)            │
│  ├── Intro + cyber risk instruction                  │
│  ├── System behavior (permissions, hooks, tags)      │
│  ├── Doing tasks (code style, over-engineering)      │
│  ├── Executing actions (reversibility, risk)         │
│  ├── Using tools (dedicated tools vs Bash)           │
│  ├── Tone and style (emoji, references)              │
│  └── Output efficiency                               │
├─ ─ ─ DYNAMIC BOUNDARY ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
│ System Prompt (dynamic, per-session, cached once)    │
│  ├── Session-specific tool guidance                  │
│  ├── Auto-memory MEMORY.md index (≤200 lines)       │
│  ├── Memory behavioral instructions                  │
│  ├── Environment info (git, model, platform, date)   │
│  ├── Language preference                             │
│  ├── Output style config                             │
│  ├── MCP server instructions (uncached)              │
│  ├── Scratchpad directory                            │
│  ├── Microcompact FRC instructions                   │
│  ├── Tool result note-taking                         │
│  └── Token budget / brief tool / length anchors      │
├──────────────────────────────────────────────────────┤
│ CLAUDE.md hierarchy (filtered for injection)         │
│  ├── Managed (/etc/claude-code/)                     │
│  ├── User (~/.claude/)                               │
│  ├── Project (.claude/ + .claude/rules/)             │
│  └── Local (CLAUDE.local.md)                         │
├──────────────────────────────────────────────────────┤
│ Conversation Messages                                │
│  ├── [Compaction boundary + summary, if compacted]   │
│  ├── User messages + attachments (20+ types)         │
│  ├── Assistant messages + tool calls + thinking      │
│  ├── Tool results (paired with tool-use requests)    │
│  └── System messages (metadata, tombstones)          │
└──────────────────────────────────────────────────────┘
```

---

## 4. Phase 2: Context Growth

### 4.1 The Query Loop State Machine

> **Source:** `src/query.ts`

The query pipeline is an async generator (`queryLoop()`) implementing the core agentic
loop inside a `while(true)` with **7 continue sites** and **11 terminal return sites**.

**Per-iteration setup:**
1. Destructure state
2. Prefetch skill discovery (async)
3. Apply tool result budget truncation
4. Apply snip (if `HISTORY_SNIP` enabled)
5. Apply microcompact (if `CACHED_MICROCOMPACT` enabled)
6. Apply context collapse (if `CONTEXT_COLLAPSE` enabled)
7. Proactive auto-compact check → may yield summary and reset messages
8. Model selection (based on permission mode, 200K threshold)
9. Blocking limit check → may return `{ reason: 'blocking_limit' }`

**API streaming loop:**
- Fallback model retry wrapper (catches `FallbackTriggeredError`, switches model)
- For each streamed message: backfill tool inputs, withhold recoverable errors
  (prompt-too-long, max_output_tokens, media size), yield to SDK, collect tool_use
  blocks, execute streaming tools

**Post-streaming decision tree** (when `needsFollowUp` is false — no tool calls):
1. Prompt-too-long recovery: context-collapse drain → reactive compact → surface error
2. Max output tokens recovery: escalate to 64K → inject recovery messages (3x) → surface
3. API error: return completed
4. Stop hooks: may prevent continuation or inject blocking errors
5. Token budget check: may inject nudge message and continue
6. Normal completion: return completed

**Tool execution path** (when `needsFollowUp` is true):
1. Execute remaining tools (streaming executor or `runTools()`)
2. Generate tool-use summary (non-blocking haiku call)
3. Abort check (mid-tool)
4. Hook prevention check
5. Collect queued commands, attachment messages, memory prefetch, skill discovery
6. Max turns check
7. Continue to next iteration with accumulated messages

### 4.2 The Query Engine Wrapper

> **Source:** `src/QueryEngine.ts`

`QueryEngine` wraps the query loop for SDK/headless mode. Key state it manages:

- `mutableMessages: Message[]` — session history, persists across `submitMessage()` calls
- `readFileState` — recently-accessed files cache (for post-compact restoration)
- `discoveredSkillNames` — turn-scoped (cleared each `submitMessage()`)
- `loadedNestedMemoryPaths` — prevents circular memory includes
- `totalUsage` — accumulated API usage across all turns
- `permissionDenials` — tracked per turn, reported in result

`submitMessage()` is the per-turn entry: clears turn-scoped state, fetches model-specific
system prompt parts, validates thinking config, processes user input, records to
transcript, and invokes the query loop. For each message from the loop, it records to
`mutableMessages`, writes transcript, and yields to the SDK caller.

### 4.3 Message Types and Structure

> **Source:** `src/utils/messages.ts` (5,512 lines), `src/types/message.ts`

**Core message types:**
- **UserMessage**: Text + attachments + metadata (uuid, isMeta, isVirtual, origin, imagePasteIds)
- **AssistantMessage**: Text + tool_use + thinking/redacted_thinking blocks + usage
- **SystemMessage**: 14 subtypes including compact_boundary, microcompact_boundary,
  api_error, permission_retry, bridge_status, stop_hook_summary, turn_duration,
  memory_saved, agents_killed, api_metrics, local_command, away_summary, scheduled_task
- **TombstoneMessage**: Deletion markers preserving conversation structure
- **AttachmentMessage**: 20+ attachment types (see §4.4)
- **ProgressMessage**: Tool execution progress with toolUseID and data
- **ToolUseSummaryMessage**: SDK-only batch summaries

**Message normalization for API** (`normalizeMessagesForAPI()`) is a 12-step pipeline:
1. Reorder attachments (bubble up until tool_result or assistant)
2. Filter virtual messages
3. Strip problematic document/image blocks (based on preceding API errors)
4. Merge consecutive user messages (API requirement)
5. Normalize tool inputs (strip non-API fields)
6. Strip tool_reference blocks (when tool search disabled)
7. Inject turn boundary text after tool_reference messages
8. Relocate tool_reference siblings (feature-gated)
9. Filter orphaned thinking-only assistant messages
10. Strip trailing thinking from last assistant
11. Ensure non-empty assistant content
12. Smoosh system-reminder siblings into tool_result blocks

**Tool pairing**: `ensureToolResultPairing()` defensively validates all tool_use/tool_result
pairs. It strips duplicate IDs, inserts synthetic error results for unmatched tool_use
blocks, and strips orphaned tool_result blocks. In strict mode (HFI training data), it
throws on mismatch rather than repairing.

### 4.4 Attachment System

Attachments are converted to API-compatible UserMessage tuples via
`normalizeAttachmentForAPI()`. Each type has dedicated handling:

| Attachment Type | Conversion |
|---|---|
| `file` | FileReadTool use + result with content/images/PDFs/notebooks |
| `directory` | Bash `ls` use + result |
| `edited_text_file` | User message with line-diff snippet |
| `compact_file_reference` | User message noting file was read before compaction |
| `pdf_reference` | User message with page count/size + pages parameter instruction |
| `selected_lines_in_ide` | User message with line range and content (max 2000 chars) |
| `plan_file_reference` | User message with plan file path and contents |
| `invoked_skills` | User message listing invoked skills with content |
| `nested_memory` | User message with memory file contents |
| `relevant_memories` | Multiple user messages, one per memory with header + content |
| `skill_listing` | User message with available skills |
| `queued_command` | User message with queued command input |
| `teammate_mailbox` | Formatted teammate messages (agent swarms) |
| `team_context` | System-reminder wrapped team coordination info |
| `skill_discovery` | System-reminder wrapped skill suggestions |
| `task_reminder` | User message with task list |

All attachments are wrapped in `<system-reminder>` tags (feature-gated via
`tengu_chair_sermon`).

### 4.5 Tool Execution and Streaming

> **Source:** `src/services/tools/StreamingToolExecutor.ts`, `src/services/tools/toolExecution.ts`

**StreamingToolExecutor** enables concurrent tool execution during model streaming:

- Tools added to queue as they stream in from the API
- **Safe tools** (`isConcurrencySafe(input) === true`) execute in parallel
- **Unsafe tools** execute alone (exclusive lock)
- Order preserved for non-safe tools
- Each tool has a child AbortController (sibling Bash errors cascade)
- Results yielded in order via `getCompletedResults()` generator

**Tool execution pipeline** (`runToolUse()`):
1. Tool lookup (with deprecated-alias fallback)
2. Input validation (Zod schema)
3. Custom validation (`tool.validateInput()`)
4. Observable input backfill (`tool.backfillObservableInput()`)
5. Pre-tool hooks (can override permission decision)
6. Speculative Bash classifier (parallel with permission check)
7. Permission decision (rule-based → hook → interactive → classifier)
8. Tool execution (`tool.call()`) with progress reporting
9. Post-tool hooks (can modify MCP output, prevent continuation)
10. Result persistence (large results → disk, replaced with preview)

**Concurrency control** (`runTools()`): Tool calls are partitioned into batches —
consecutive safe tools form parallel batches, non-safe tools form exclusive batches.
Max concurrency: `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (default 10).

### 4.6 Token Estimation

> **Source:** `src/services/tokenEstimation.ts`, `src/utils/tokens.ts`

**Two approaches:**

1. **Exact** (post-API): `getTokenUsage()` extracts `usage` from assistant messages.
   Sums `input_tokens + cache_creation_input_tokens + cache_read_input_tokens + output_tokens`.

2. **Estimated** (prospective): `tokenCountWithEstimation()` walks backward to the last
   API response, takes its exact token count, then estimates delta for new messages.

**Estimation formula:**
```
tokens = Math.round(content.length / bytesPerToken)
```
- Default: `bytesPerToken = 4` (4 chars/token)
- JSON files: `bytesPerToken = 2` (dense single-char tokens)
- Images: Fixed `2000` tokens per image (API charges ~2000 despite high max)
- Documents/PDFs: Fixed `2000` tokens (prevents massive overestimation from base64)
- Conservative padding: `4/3` multiplier for microcompact estimation

**Context window resolution** (hierarchical, first-match wins):
1. `CLAUDE_CODE_MAX_CONTEXT_TOKENS` env var
2. `[1m]` suffix in model string → 1M tokens
3. Model capability metadata (`max_input_tokens`)
4. `CONTEXT_1M_BETA_HEADER` in betas → check `modelSupports1M(model)`
5. Default: `MODEL_CONTEXT_WINDOW_DEFAULT = 200,000`

**Effective context window:**
```
effectiveWindow = contextWindow - min(maxOutputTokens, 20,000)
```

**Warning thresholds:**
- Auto-compact: `effectiveWindow - 13,000`
- Warning UI: `effectiveWindow - 20,000`
- Blocking limit: `effectiveWindow - 3,000`

**Cost tracking**: Per-model `ModelUsage` accumulates `inputTokens`, `outputTokens`,
`cacheReadInputTokens`, `cacheCreationInputTokens`, `webSearchRequests`, `costUSD`.
Persisted to disk and restored on session resume.

### 4.7 Transition and Recovery Mechanisms

The query loop continues for 7 distinct reasons, each tracked in `state.transition`:

| Transition | Trigger | Reset On |
|---|---|---|
| `next_turn` | Tool execution completed | (normal flow) |
| `collapse_drain_retry` | Drained context collapses after PTL | New turn |
| `reactive_compact_retry` | Emergency compact after PTL | New turn (`hasAttempted`) |
| `max_output_tokens_escalate` | Cap escalated to 64K | New turn |
| `max_output_tokens_recovery` | Recovery message injected (3x max) | New turn |
| `stop_hook_blocking` | Stop hook injected blocker | (preserved) |
| `token_budget_continuation` | Budget allows continuation | (budget tracker) |

Key recovery guards:
- `hasAttemptedReactiveCompact: boolean` — prevents infinite loops on repeated PTL
- `maxOutputTokensRecoveryCount < 3` — caps recovery attempts
- All recovery counters **reset on tool execution** (fresh quota per agentic turn)

---

## 5. Phase 3: Context Compression

### 5.1 Compaction Decision

> **Source:** `src/services/compact/autoCompact.ts`

`shouldAutoCompact()` evaluates these conditions:
1. Token count against threshold (`effectiveContextWindow - 13K`)
2. Recursion guard (won't compact during compact, session_memory, or marble_origami)
3. Feature gates (`REACTIVE_COMPACT` or `CONTEXT_COLLAPSE` may suppress)
4. Reactive-only mode (`tengu_cobalt_raccoon` gate)
5. Snip token accounting (subtract tokens freed by pending snips)

`autoCompactIfNeeded()` orchestrates with a **circuit breaker** (max 3 consecutive
failures — data showed 1,279 sessions with 50+ failures wasting ~250K API calls/day).

### 5.2 Session Memory Compact (Lightweight)

> **Source:** `src/services/compact/sessionMemoryCompact.ts`

A cheap strategy that avoids an API call by using existing session memory content:

**Configuration** (remote-configurable via GrowthBook):
- `minTokens: 10,000` — minimum tokens to preserve
- `minTextBlockMessages: 5` — minimum interactive messages to keep
- `maxTokens: 40,000` — hard cap for preserved messages

**Message preservation rules** (`calculateMessagesToKeepIndex()`):
- Starts from `lastSummarizedMessageId + 1`
- Expands backwards to meet BOTH minimums
- Floors at last compact boundary (disk discontinuity invariant)
- `adjustIndexToPreserveAPIInvariants()` ensures tool_use/tool_result pairs and
  thinking block chains are never split

**Activation**: Requires both `tengu_session_memory` AND `tengu_sm_compact` feature gates.

### 5.3 Full Summarization (Heavyweight)

> **Source:** `src/services/compact/compact.ts`

`compactConversation()` runs when session memory compact is insufficient:

1. Execute pre-compact hooks with trigger (`'auto'` or `'manual'`) and custom instructions
2. **Pre-process messages:**
   - `stripImagesFromMessages()` — replace images/documents with `[image]`/`[document]`
     markers (prevents PTL on the compaction request itself)
   - `stripReinjectedAttachments()` — remove skill_discovery/skill_listing (re-surfaced
     post-compact anyway)
3. Call `streamCompactSummary()` — uses forked agent sharing parent's prompt cache
   - Tries fork path first (for cache prefix reuse)
   - Falls back to regular streaming
   - Retries with exponential backoff (max 2 retries)
   - 30-second keepalive signals prevent WebSocket idle timeouts
   - Output capped at `COMPACT_MAX_OUTPUT_TOKENS = 20,000`
4. If compact request itself overflows: `truncateHeadForPTLRetry()` drops oldest
   API-round groups (up to 3 retries)
5. Clear read file cache and nested memory paths
6. Create post-compact attachments in parallel (see §5.5)
7. Re-append session metadata (stay within 16KB tail window)
8. Execute post-compact hooks
9. Return `CompactionResult`

**CompactionResult** contains:
- `boundaryMarker` — system message with compact metadata
- `summaryMessages` — AI-generated summary as UserMessage
- `attachments` — files, skills, plan, agents, tools delta, MCP delta
- `hookResults` — session start hook messages
- `messagesToKeep` — preserved segment for partial compaction
- `preCompactTokenCount`, `postCompactTokenCount`, `compactionUsage`

### 5.4 Summarization Prompt Design

> **Source:** `src/services/compact/prompt.ts`

The summarization prompt enforces a strict no-tools constraint:

```
CRITICAL: Respond with TEXT ONLY.
Do NOT call Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
Tool calls will be REJECTED and waste the only turn.
```

The prompt requires two parts:

1. **`<analysis>` section** (scratchpad, stripped from final output):
   Chronological analysis of each message covering user intent, assistant approach,
   key decisions, file names, code snippets, errors and fixes, user feedback.

2. **`<summary>` section** (9 required subsections):
   1. Primary Request and Intent
   2. Key Technical Concepts
   3. Files and Code Sections (with full snippets)
   4. Errors and Fixes (with resolution)
   5. Problem Solving (solved and ongoing)
   6. All User Messages (non-tool-use only)
   7. Pending Tasks
   8. Current Work (from recent messages)
   9. Optional Next Step (with direct quotes)

For auto-compaction, the summary message includes a directive to continue without
acknowledgment or recap. In proactive mode, it adds: "continue your work loop."

### 5.5 Post-Compact Restoration

After compaction, strategically re-inject frequently-needed context:

| Restored Item | Budget | Source |
|---|---|---|
| Recently-accessed files (top 5) | 50,000 tokens total, 5,000 per file | `readFileState` recency |
| Active skills (truncated heads) | 25,000 tokens total, 5,000 per skill | `invokedSkills` registry |
| Plan file | Unbounded | Plan mode state |
| Plan mode instructions | Minimal | Mode detection |
| Background agent status | Minimal | Running/finished agents |
| Deferred tools delta | Minimal | New tools since last boundary |
| Agent listing delta | Minimal | New agents since last boundary |
| MCP instructions delta | Minimal | MCP server instructions |
| Session start hook results | Minimal | Hook execution cache |

Files exclude plan files and memory files (`shouldExcludeFromPostCompactRestore()`).
Skills are sorted most-recent-first with per-skill truncation keeping the head (where
setup/usage instructions live).

### 5.6 Microcompaction

> **Source:** `src/services/compact/microCompact.ts`

Microcompaction clears old tool results **without summarization** — a lighter approach
that operates at the individual tool-result level rather than the conversation level.

**Two strategies:**

1. **Time-based microcompact**: If the gap since the last assistant message exceeds a
   threshold (default 60 minutes, via `tengu_slate_heron`), clear old tool results
   in-place (mutate content to `'[Old tool result content cleared]'`). Fires when
   the prompt cache is cold anyway.

2. **Cached microcompact (cache-editing API)**: Uses Anthropic's `cache_edits` API to
   delete tool results from the cached prompt prefix without invalidating the cache.
   Does NOT mutate local message content — the `cache_reference`/`cache_edits` blocks
   are added at API time.

**Compactable tools**: Read, Bash/Zsh, Grep, Glob, WebSearch, WebFetch, Edit, Write.

**Integration**: `microcompactMessages()` runs every turn before the API call as part
of the query loop setup phase.

### 5.7 Partial Compaction

> **Source:** `src/services/compact/compact.ts`, `partialCompactConversation()`

Targeted compaction around a user-selected message index:

- **Direction `from`**: Summarizes messages AFTER pivot, keeps earlier (preserves
  prompt cache for kept messages)
- **Direction `up_to`**: Summarizes messages BEFORE pivot, keeps later (invalidates
  cache since summary precedes kept messages)

Boundary is annotated with `preservedSegment` metadata (`headUuid`, `tailUuid`,
`anchorUuid`) so the transcript loader can patch the parentUuid chain across the
compaction boundary on resume.

### 5.8 Reactive Compaction

> **Source:** `src/services/compact/reactiveCompact.ts`

Emergency compaction when the API returns "prompt too long." Fires only when the
proactive auto-compact threshold miscalculated or token estimation was inaccurate.

Recovery order in the query loop:
1. Try context-collapse drain (if not already attempted)
2. Try reactive compact (with `hasAttempted` guard)
3. Surface error if both fail

Gated behind `REACTIVE_COMPACT` feature flag.

### 5.9 Snip Compaction

> **Source:** `src/services/compact/snipCompact.ts`

Fine-grained approach: identifies and removes specific message segments (e.g., large
tool results no longer relevant). Messages are tagged with short IDs
(`deriveShortMessageId()`) for the model to reference. Gated behind `HISTORY_SNIP`.

### 5.10 Circuit Breaker

```
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

After 3 consecutive failures, auto-compact stops attempting. Counter resets on any
successful compaction. This prevents the system from hammering the API when compaction
is failing due to irrecoverably oversized context.

### 5.11 Post-Compact State Cleanup

> **Source:** `src/services/compact/postCompactCleanup.ts`

`runPostCompactCleanup()` invalidates caches and tracking state:
- Reset microcompact state
- Reset context collapse (main-thread only)
- Clear user context cache
- Clear memory files cache
- Clear system prompt section cache
- Clear classifier approvals and speculative checks
- Sweep file content cache
- Clear session messages cache
- **Does NOT** clear invoked skill content (must survive across multiple compactions)

### 5.12 Compaction Flow Summary

```
┌─────────────────────────────────────────────────────────────────┐
│ Every turn (before API call):                                    │
│  ├── Time-based microcompact (if >60min gap, cache cold)        │
│  └── Cached microcompact (cache-editing, warm cache)            │
├─────────────────────────────────────────────────────────────────┤
│ Token count exceeds threshold?                                   │
│  ├── Circuit breaker (3+ failures) → skip                       │
│  ├── Session memory compact (lightweight, no API) → done        │
│  └── Full summarization (API call) → post-compact restore       │
├─────────────────────────────────────────────────────────────────┤
│ API returns "prompt too long"?                                   │
│  ├── Context-collapse drain → retry                             │
│  └── Reactive compact (emergency) → retry                       │
├─────────────────────────────────────────────────────────────────┤
│ User invokes /compact?                                           │
│  └── Full or partial compaction with optional user instructions │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Phase 4: Memory Extraction

### 6.1 Background Extraction Agent

> **Source:** `src/services/extractMemories/extractMemories.ts`

A **background forked agent** runs asynchronously after each model response, analyzing
the session for durable learnings worth persisting.

**Lifecycle:**
- State is **closure-scoped** inside `initExtractMemories()` (not module-level)
- Triggered by `handleStopHooks` in `stopHooks.ts` when model stops (no tool calls)
- Fire-and-forget: doesn't block the response to the user
- For `-p`/SDK mode: `drainPendingExtraction()` called by `print.ts` before shutdown

**Forked agent configuration:**
- Shares parent's prompt cache via `CacheSafeParams`
- `maxTurns: 5` — hard cap to prevent rabbit-holes
- `skipTranscript: true` — prevents transcript race conditions
- `querySource: 'extract_memories'` — identifies as extraction agent

**Tool permissions** (`createAutoMemCanUseTool()`):
- **Unrestricted**: Read, Grep, Glob
- **Read-only**: Bash (validates via `tool.isReadOnly(parsedData)`)
- **Memory-only write**: Edit, Write (only paths where `isAutoMemPath(filePath)`)
- **Denied**: Everything else (MCP tools, Agent tools, etc.)

**Overlap guard**: Only one extraction runs at a time. If a new request arrives during
extraction, only the latest context is stashed (overwrites previous). Trailing run
processes the most recent stashed context after current extraction finishes.

### 6.2 Extraction Prompt and Manifest

The extraction prompt:
- Instructs: "Analyze the most recent ~N messages above"
- Provides a **pre-computed manifest** of existing memory files with type, timestamp,
  and description — so the agent doesn't spend a turn on `ls`
- Strategy guidance: "Turn 1 — Read all files you might update; Turn 2 — Write in parallel"
- Critical constraint: "Only use content from the last ~N messages. Do NOT investigate
  further or verify content."

### 6.3 Deduplication and Throttling

**Main agent → extraction mutual exclusion:**
`hasMemoryWritesSince(messages, lastMemoryMessageUuid)` scans for Write/Edit tool_use
blocks targeting auto-memory paths. If the main agent already wrote memories, extraction
is skipped for that range. Cursor (`lastMemoryMessageUuid`) advances on success only —
failed runs are reconsidered.

**Turn throttling:** `tengu_bramble_lintel` gate controls extraction frequency (default:
every 1 eligible turn). Trailing runs bypass the throttle.

**Index skipping:** When `tengu_moth_copse` is ON, extraction skips the MEMORY.md
indexing step (just writes topic files). A separate process rebuilds the index.

### 6.4 Auto-Dream Consolidation

> **Source:** `src/services/extractMemories/autoDream.ts`

Long-form memory consolidation across multiple sessions:

**Three-stage gating (cheapest first):**
1. **Time gate**: hours since last dream ≥ minHours (default 24h)
2. **Session gate**: session count ≥ minSessions (default 5) — throttled to 10-minute scans
3. **Lock gate**: Acquire per-process consolidation lock

**Execution**: Forked agent with same restricted tool permissions. Progress tracked via
`DreamTask` for UI visibility. Collects Edit/Write file_paths for completion message.

**Difference from extraction**: Extraction catches recent turns. Dream distills
accumulated daily logs or performs cross-session consolidation.

### 6.5 Assistant Mode Daily Logs (KAIROS)

> **Source:** `src/memdir/memdir.ts`, `buildAssistantDailyLogPrompt()`

For perpetual sessions (feature `KAIROS`), the **append-only daily log** pattern:

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

- Agent appends timestamped bullets to daily log
- A separate nightly `/dream` skill distills logs into topic files + MEMORY.md
- MEMORY.md serves as read-only distilled index
- Log path described as a pattern in the prompt (not a literal) because the prompt
  cache is NOT invalidated on date change — the model derives the date from a
  `date_change` attachment on midnight rollover

---

## 7. Session Persistence and Resume

### 7.1 Transcript Format

> **Source:** `src/utils/sessionStorage.ts`

Sessions are persisted as **JSONL** (JSON Lines) — one JSON object per line, append-only.

Each line has a `type` field. Message entries (`TranscriptMessage`) carry:
`uuid`, `parentUuid`, `logicalParentUuid`, `isSidechain`, `sessionId`, `cwd`,
`userType`, `timestamp`, `version`, `gitBranch`, `slug`.

**Entry types** beyond messages: `custom-title`, `ai-title`, `last-prompt`, `tag`,
`agent-name`, `agent-color`, `agent-setting`, `pr-link`, `mode`, `worktree-state`,
`file-history-snapshot`, `attribution-snapshot`, `content-replacement`,
`context-collapse-snapshot`, `context-collapse-commit`, `queue-operation`.

### 7.2 Session Storage Layout

```
~/.claude/
├── projects/
│   └── <sanitized-cwd>/
│       ├── <sessionId>.jsonl           # Main transcript
│       ├── <sessionId>/
│       │   └── subagents/
│       │       └── agent-<agentId>.jsonl  # Subagent transcripts
│       └── memory/
│           ├── MEMORY.md               # Auto-memory index
│           └── *.md                    # Topic files
└── history.jsonl                       # Command-line input history
```

The session file is **NOT created** until the first user/assistant message — buffered
entries (hooks, attachments, system) wait in memory. This prevents metadata-only files.

### 7.3 Write Queue System

Writes are batched for efficiency:

- `enqueueWrite(filePath, entry)` adds to per-file queue
- `scheduleDrain()` sets 100ms timer calling `drainWriteQueue()`
- `drainWriteQueue()` batches entries, appends in 100MB chunks
- File created with mode `0o600` (user-only read/write)
- `flushResolvers` handle sync coordination for cleanup/exit

**Metadata re-append**: On session exit or compaction, metadata entries (title, tag,
agent, mode, worktree, PR link) are re-appended to EOF to keep them within the 64KB
tail window that `readLiteMetadata()` scans for the fast resume picker.

### 7.4 Resume Flow

1. **Load transcript** via `loadTranscriptFile()`:
   - For files >5MB, skip pre-compaction content via fast boundary scan
   - Parse JSONL sequentially, populate `Map<uuid, TranscriptMessage>`
   - Extract metadata (title, tag, agent, mode, worktree state)
   - Recover metadata before compaction boundary via byte-level scan

2. **Restore agent state**: Re-apply agent definition, model override

3. **Build conversation chain**: Walk backward from leaf via `parentUuid`,
   apply compaction relinks, snip removals, chain consistency checks

4. **Restore cost state**: `setCostStateForRestore()` with persisted metrics

**What IS restored**: Full message history (within boundary), file history, attribution,
worktree state, session metadata, cost state.

**What IS lost**: Progress messages (ephemeral), pre-compaction messages, snipped messages.

### 7.5 Compaction and Resume Interaction

**Standard compaction**: Boundary marks truncation point. Everything before is deleted.
On resume, `loadTranscriptFile()` skips pre-boundary bytes entirely.

**Preserved segment compaction**: `preservedSegment` records UUIDs of kept messages.
On load, `applyPreservedSegmentRelinks()` splices them back and fixes parentUuid chains.
Token usage zeroed for preserved messages (prevents re-compaction loop).

**Snip removals**: `snipMetadata.removedUuids` filters out deleted messages. ParentUuid
re-linked to skip gaps.

---

## 8. State Management

### 8.1 Bootstrap State (Global)

> **Source:** `src/bootstrap/state.ts` (1,758 lines)

A single module-level `STATE` object is the **source of truth** for non-React global
state. It persists across compaction, session switches, and the full app lifetime.

**Key categories:**

**Working directory** (3 distinct concepts):
- `originalCwd` — initial CWD at process start, **never updated**, sets project identity
- `projectRoot` — stable root set once via `--worktree`, never updated mid-session
- `cwd` — current working directory, updated by EnterWorktreeTool

**Cost and duration tracking**: `totalCostUSD`, `totalAPIDuration`,
`totalToolDuration`, `modelUsage` (per-model breakdown), turn-level counters.

**Session identity**: `sessionId` (UUID), `parentSessionId` (lineage tracking),
`sessionProjectDir`.

**Model selection**: `mainLoopModelOverride` (from `--model`), `initialMainLoopModel`,
`modelStrings` (cached ID mappings).

**Skills**: `invokedSkills: Map<string, InvokedSkillInfo>` — keyed by
`${agentId}:${skillName}`, **preserved across compaction**.

**Telemetry**: OpenTelemetry meters, counters, loggers, tracers.

**Prompt cache latches**: `promptCache1hEligible`, `afkModeHeaderLatched`,
`fastModeHeaderLatched`, `cacheEditingHeaderLatched` — latched after first eval,
never reset within session.

### 8.2 AppState (Reactive UI)

> **Source:** `src/state/AppState.tsx`, `src/state/AppStateStore.tsx`

AppState is a `DeepImmutable` type with 150+ fields for UI state. It uses a custom
Zustand-like store with `useSyncExternalStore` for React integration:

```typescript
const verbose = useAppState(s => s.verbose)     // Subscribe to one field
const setAppState = useSetAppState()            // Stable updater, no re-renders
```

Key fields: `toolPermissionContext`, `mainLoopModel`, `tasks`, `mcp` (clients + tools),
`plugins`, `notifications`, `inbox`, `remoteConnectionStatus`, `speculation`,
`denialTracking`, `companionReaction`.

The store uses `Object.is` comparison on `setState` — no-ops when state hasn't changed.

### 8.3 State Survival Across Compaction

**Preserved:**
- `invokedSkills` — skills used remain available for re-injection
- `modelUsage` / cost tracking — cumulative metrics
- `mainLoopModelOverride` — model selection
- `sessionId` — same session continues
- `planSlugCache` — session ID → slug mapping

**Cleared:**
- Turn-level metrics (tool/hook/classifier duration and count)
- Microcompact state
- Context collapse state
- User context cache
- Memory files cache
- System prompt section cache
- Classifier approvals and speculative checks

**Flagged:** `pendingPostCompaction` is set after compaction and consumed after the
next successful API call — distinguishes compaction-induced cache misses from TTL expiry.

### 8.4 Session and CWD Identity

**Session lifecycle:**
- `getSessionId()` — returns current UUID
- `regenerateSessionId({ setCurrentAsParent })` — new session, optionally links parent
- `switchSession(sessionId, projectDir?)` — atomic update, emits `sessionSwitched` signal
- `onSessionSwitch.subscribe(callback)` — keeps PID file in sync

**CWD tracking** (3 independent concepts):
- `getOriginalCwd()` — project identity (sessions, skills, history). Never changes.
- `getProjectRoot()` — set once at startup via `--worktree`. Never changes mid-session.
- `getCwdState()` — current working directory for file operations. Updated by
  EnterWorktreeTool.

---

## 9. Cross-Cutting Concerns

### 9.1 Prompt Cache Efficiency

The system is heavily optimized for Anthropic's prompt caching:

- **Static/dynamic boundary**: All cross-org cacheable content precedes the marker
- **Section-level caching**: `systemPromptSection()` cached until `/clear` or `/compact`;
  only `mcp_instructions` is `DANGEROUS_uncached`
- **Forked agents**: Compaction and extraction agents share parent's prompt cache via
  `CacheSafeParams`, using identical tool lists to maintain cache key compatibility
- **Cache-editing microcompact**: Deletes tool results from cached prefix without
  invalidating the cache
- **MCP instructions delta**: Can move MCP instructions to attachments to avoid
  cache bust on late MCP connect
- **Memoized context**: `getSystemContext()` and `getUserContext()` compute once per session
- **Partial compaction direction**: `from` preserves cache prefix; `up_to` invalidates it
- **Prompt cache latches**: 1h TTL eligibility, AFK mode, fast mode, cache editing
  headers latched after first eval to prevent header flip-flopping

### 9.2 Security Boundaries

**Memory path validation** (`src/memdir/paths.ts:109-150`, `validateMemoryPath()`):
- Rejects relative paths, root/near-root paths, Windows drive roots, UNC paths, null bytes
- Normalizes to NFC Unicode with exactly one trailing separator
- Tilde expansion only for `~/path` (not bare `~`, `~/`, `~/.` which would match `$HOME`)

**Project settings exclusion**: `projectSettings` (`.claude/settings.json` committed
to the repo) deliberately excluded from `autoMemoryDirectory` overrides — a malicious
repo could set `autoMemoryDirectory: "~/.ssh"` and gain silent write access via the
filesystem.ts write carve-out.

**Team memory path traversal**: Two-pass validation — normalize + containment check,
then `realpathDeepestExisting()` resolves symlinks on the deepest existing ancestor to
detect escape attempts that `path.resolve()` alone cannot catch.

**CLAUDE.md injection filtering**: `filterInjectedMemoryFiles()` removes AutoMem/TeamMem
from the prompt when they're being surfaced via attachments instead. HTML comments
stripped before @include processing.

**Memory file error handling**: Missing files (`ENOENT`) silently ignored. Permission
errors (`EACCES`) logged to analytics without PII. Non-text files silently skipped.

### 9.3 Telemetry and Observability

Every context management decision is instrumented:

| Event | Meaning |
|---|---|
| `tengu_memdir_loaded` | Memory directory stats (file/subdir count, content size, truncation) |
| `tengu_memdir_disabled` | Why memory was disabled (env var, setting) |
| `tengu_compact` | Full compaction metrics (tokens pre/post, cache, discovery state) |
| `tengu_partial_compact` | Partial compaction (direction, messages summarized/kept) |
| `tengu_cached_microcompact` | Cache-editing microcompact (tools deleted, tokens saved) |
| `tengu_time_based_microcompact` | Time-based clearing (gap minutes, results cleared) |
| `tengu_sm_compact_flag_check` | Session memory compact gate check |
| `tengu_extract_memories_coalesced` | Extraction overlap (new request during extraction) |
| `tengu_auto_mem_tool_denied` | Extraction agent tool access denied |
| `tengu_claudemd__initial_load` | CLAUDE.md initial load statistics |
| `system_context_started/completed` | System context collection timing |
| `user_context_started/completed` | User context collection timing |

### 9.4 Feature Gating

**Compile-time (`bun:bundle` — dead code elimination):**
- `REACTIVE_COMPACT`, `CONTEXT_COLLAPSE`, `HISTORY_SNIP` — compaction strategies
- `EXTRACT_MEMORIES` — background memory extraction
- `KAIROS`, `KAIROS_BRIEF` — assistant mode
- `TEAMMEM` — team memory synchronization
- `CACHED_MICROCOMPACT` — cache-editing microcompact
- `BREAK_CACHE_COMMAND` — system prompt cache-breaking
- `EXPERIMENTAL_SKILL_SEARCH` — deferred skill discovery
- `PROACTIVE` — autonomous work mode
- `TOKEN_BUDGET` — user-specified token budget
- `FORK_AGENT` — fork-based subagent spawning
- `VERIFICATION_AGENT` — verification contract

**Runtime (GrowthBook):**
- `tengu_coral_fern` — "searching past context" instructions
- `tengu_passport_quail` — extraction mode activation
- `tengu_slate_thimble` — extraction in non-interactive sessions
- `tengu_moth_copse` — skip MEMORY.md index in prompt
- `tengu_herring_clock` — team memory cohort tracking
- `tengu_amber_prism` — memory correction hints on user rejection
- `tengu_bramble_lintel` — extraction turn throttle
- `tengu_cobalt_raccoon` — reactive-only compaction mode
- `tengu_chair_sermon` — system-reminder smooshing
- `tengu_paper_halyard` — skip project-level CLAUDE.md in prompt
- `tengu_slate_heron` — time-based microcompact config
- `tengu_onyx_plover` — auto-dream time/session thresholds
- `tengu_sm_compact` / `tengu_session_memory` — session memory compact gates
- `tengu_compact_cache_prefix` — forked compact agent for cache sharing
- `tengu_pewter_ledger` — plan mode phase 4 variant (A/B test)

---

## 10. Design Principles

### 10.1 Invisibility

Users should never think about context limits. The system auto-compacts when the window
fills, auto-extracts learnings into persistent memory, and auto-restores critical
context after compaction — all without user intervention. Sessions resume seamlessly
across process restarts.

### 10.2 Structured Forgetting

Compaction is not random truncation. It is AI-generated summarization with a 9-section
structured format that preserves **intent, key decisions, file names, code snippets,
errors, and next steps** while discarding verbatim tool output. The `<analysis>` scratchpad
ensures thorough processing before the summary is written.

### 10.3 Strategic Restoration

Post-compact context is **curated, not just smaller**. The 5 most recently accessed
files and active skills are re-injected with token budgets. Skills are truncated to
preserve heads (where instructions live). This makes compacted sessions feel continuous.

### 10.4 Multi-Strategy Compression

No single compaction strategy suffices. The system layers:
1. **Microcompact** — per-turn tool result clearing (cheap, preserves cache)
2. **Session memory compact** — message trimming without API call
3. **Full summarization** — AI-generated summary (expensive, thorough)
4. **Partial compaction** — user-directed with cache-preserving direction choice
5. **Reactive compaction** — emergency fallback on prompt-too-long
6. **Snip compaction** — targeted segment removal

### 10.5 Defense in Depth for Memory

Memory files are validated for path traversal attacks, filtered for prompt injection
patterns, truncated at line and byte caps, security-gated at the settings source level
(project settings excluded from directory overrides), and normalized to NFC Unicode.
Team memory uses two-pass validation with symlink resolution. No single failure can
weaponize the memory system.

### 10.6 Cache-First Architecture

Every design decision considers prompt cache impact. The static/dynamic boundary
placement maximizes cross-org cache hits. Forked agents share parent caches using
identical tool lists. System context is memoized once per session. MCP instructions
can be moved to attachments to avoid cache busts. Microcompact uses cache-editing
instead of content mutation. Prompt cache latches prevent header flip-flopping.

### 10.7 Graceful Degradation

Circuit breakers stop compaction after 3 consecutive failures. Reactive compaction
fires as a last resort. Session memory compact provides a lightweight fallback. Feature
gates allow progressive rollout and instant rollback. Memory extraction silently skips
when the main agent already wrote memories or when it's running overlapped. The time-based
microcompact fires only when the cache is cold anyway (60-minute threshold matches server
TTL).

### 10.8 Separation of Ephemeral and Durable State

Session context (messages, tool results) is **ephemeral** — it grows, gets compressed,
and disappears when the session ends. Durable learnings (user preferences, project
patterns, corrections) are **extracted** into persistent memory files. The KAIROS
daily-log pattern extends this: raw observations are append-only logs, while nightly
distillation produces the durable index.

Bootstrap state separates **session-global** (cost, model, session ID — survive
compaction) from **turn-scoped** (metrics, caches — cleared on compaction). React
AppState tracks **UI-reactive** fields separately. Transcript JSONL provides
**crash-durable** persistence.

### 10.9 Pair Preservation

Tool-use and tool-result messages are always kept together. Compaction never splits a
tool chain. Session memory compact explicitly walks the message array via
`adjustIndexToPreserveAPIInvariants()` to find pair boundaries.
`ensureToolResultPairing()` defensively validates all pairs before API submission,
inserting synthetic error results for any orphaned tool_use blocks.

### 10.10 Append-Only Durability

Transcripts use append-only JSONL — no in-place rewrites. This ensures crash durability
(partial writes don't corrupt previous entries). Metadata is re-appended at EOF to stay
within the fast-scan tail window. The write queue batches at 100ms intervals with 100MB
chunk limits. Content replacement stubs enable prompt cache stability without rewriting
history.
