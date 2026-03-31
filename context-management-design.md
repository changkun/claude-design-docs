# Claude Code: Context Management System — Design Specification

This document analyzes the context management architecture of Claude Code — Anthropic's
agentic CLI tool — focusing on how it assembles, grows, compresses, and persists context
across the lifecycle of a session and across sessions.

## Table of Contents

- [1. Vision](#1-vision)
- [2. Architecture Overview](#2-architecture-overview)
- [3. Phase 1: Context Assembly](#3-phase-1-context-assembly)
- [4. Phase 2: Context Growth](#4-phase-2-context-growth)
- [5. Phase 3: Context Compression](#5-phase-3-context-compression)
- [6. Phase 4: Memory Extraction](#6-phase-4-memory-extraction)
- [7. Cross-Cutting Concerns](#7-cross-cutting-concerns)
- [8. Design Principles](#8-design-principles)

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
```

The system implements **effectively unlimited context** through automatic summarization
while maintaining **prompt cache efficiency** and **recovery reliability**.

---

## 3. Phase 1: Context Assembly

### 3.1 The System Prompt and the Cache Boundary

> **Source:** `src/constants/prompts.ts`

The system prompt is constructed with a critical architectural split: a **dynamic
boundary marker** (`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`) separates static (cacheable)
content from session-specific content. Everything before the boundary can be cached
across API calls via Anthropic's prompt caching; everything after changes per-session.

**Static region** (before boundary, cacheable):
- Tool descriptions and usage guidance
- Safety instructions (OWASP top 10, reversibility heuristics)
- Coding guidelines (over-engineering avoidance, security awareness)
- Permission mode documentation
- Prompt injection awareness reminders

**Dynamic region** (after boundary, per-session):
- Git status snapshot (branch, recent commits, working tree)
- Current date
- CLAUDE.md content (project/user instructions)
- MCP server instructions
- Memory prompt (auto-memory index + behavioral instructions)
- Skill listings

The boundary is documented at `src/constants/prompts.ts:114-115`. Its position
determines the prompt cache hit rate — everything before it is shared across calls.

### 3.2 System Context Collection

> **Source:** `src/context.ts`

Two memoized async functions run **once per session** to collect environmental context:

**`getSystemContext()`** (line 116):
- Collects git branch, default branch, short status, recent 5 commits, user name
- Status truncated at `MAX_STATUS_CHARS = 2000` characters
- All git commands run in parallel via `Promise.all()`
- Skipped entirely in remote sessions (`CLAUDE_CODE_REMOTE`) or when git instructions
  are disabled
- Supports an internal `systemPromptInjection` for cache-breaking (ant-only debugging)

**`getUserContext()`** (line 155):
- Loads CLAUDE.md files from the 4-level hierarchy (see §3.3)
- Applies injection filtering via `filterInjectedMemoryFiles()`
- Caches result for the auto-mode classifier (avoids import cycle through
  permissions → filesystem → permissions → yoloClassifier)
- Includes current date string
- Respects `CLAUDE_CODE_DISABLE_CLAUDE_MDS` env var and `--bare` mode

Both functions use lodash `memoize` — they compute once and return the cached result
for the session's lifetime. Cache can be explicitly cleared when
`systemPromptInjection` changes.

### 3.3 Memory File Hierarchy (CLAUDE.md)

> **Source:** `src/utils/claudemd.ts`

Context is assembled from a four-level hierarchy of "memory files", loaded by
`getMemoryFiles()` at `src/utils/claudemd.ts:790`:

```
Priority (lowest → highest, higher overrides lower):

  /etc/claude-code/CLAUDE.md               ← managed (enterprise policy)
  ~/.claude/CLAUDE.md                      ← user (personal preferences)
  .claude/CLAUDE.md + .claude/rules/*.md   ← project (repo-level instructions)
  CLAUDE.local.md                          ← local (gitignored overrides)
```

The file walk traverses the directory hierarchy from the working directory upward,
collecting CLAUDE.md files at each level. The assembly into prompt text happens in
`getClaudeMds()` at `src/utils/claudemd.ts:1153`.

**@include directives**: CLAUDE.md files support `@path` syntax for composing
instructions from multiple sources. These work in leaf text nodes only, support
relative (`@./`), home-relative (`@~/`), and absolute (`@/`) paths. Circular
references are prevented. Non-existent files are silently ignored. A text-file
allowlist (~40 extensions) restricts includes to safe formats — binary files
(images, PDFs) are explicitly excluded.

**Injection prevention**: `filterInjectedMemoryFiles()` at
`src/utils/claudemd.ts:1142` scans for suspicious patterns in memory files before
loading them into the prompt. This defends against repositories that ship adversarial
`.claude/CLAUDE.md` files attempting prompt injection.

### 3.4 Auto Memory (Persistent Cross-Session State)

> **Source:** `src/memdir/memdir.ts`, `src/memdir/paths.ts`

The auto-memory system provides persistent, per-project memory stored at
`~/.claude/projects/<sanitized-project-root>/memory/`.

**The entrypoint is `MEMORY.md`** — a 200-line, 25KB-capped index file that is
**always loaded into the system prompt**. Topic files linked from the index are
read on-demand by the model.

**Truncation** (`src/memdir/memdir.ts:57`, `truncateEntrypointContent()`):
- Line cap: `MAX_ENTRYPOINT_LINES = 200`
- Byte cap: `MAX_ENTRYPOINT_BYTES = 25,000`
- Line-truncates first (natural boundary), then byte-truncates at last newline
- Appends warning naming which cap fired

**Path resolution** (`src/memdir/paths.ts`, `getAutoMemPath()`):
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` env var — full-path override for SDK/Cowork
2. `autoMemoryDirectory` in settings.json — trusted sources only (policy, local, user;
   **not** projectSettings for security — see §7.2)
3. `<memoryBase>/projects/<sanitized-git-root>/memory/` — default

Git worktrees sharing the same canonical root (`findCanonicalGitRoot()`) share one
memory directory — `git worktree add` does not fragment memories.

**Behavioral instructions** (`src/memdir/memdir.ts`, `buildMemoryLines()`):
The system prompt includes detailed guidance on:
- A closed four-type taxonomy (user, feedback, project, reference)
- What NOT to save (derivable-from-code patterns, architecture, git history)
- How to save (frontmatter format, two-step process: write file → add index entry)
- When to access (trust recall, proactive consultation)
- Distinction from plans and tasks (memory = cross-session, plans/tasks = within-session)

**Team memory** (feature-gated behind `TEAMMEM`): When enabled, a shared `team/`
subdirectory under the auto-memory path allows synchronized memory across team agents.
Combined prompts are built by `teamMemPrompts.buildCombinedMemoryPrompt()`.

### 3.5 Context Assembly Summary

The complete context handed to the API for each turn:

```
┌─────────────────────────────────────────────────────┐
│ System Prompt (static, cached)                       │
│  ├── Tool descriptions                               │
│  ├── Safety instructions                             │
│  ├── Coding guidelines                               │
│  └── Permission mode docs                            │
├─ ─ ─ DYNAMIC BOUNDARY ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
│ System Prompt (dynamic, per-session)                 │
│  ├── Git status snapshot                             │
│  ├── Current date                                    │
│  ├── CLAUDE.md hierarchy (filtered for injection)    │
│  ├── Auto-memory MEMORY.md index (≤200 lines)       │
│  ├── Memory behavioral instructions                  │
│  ├── MCP server instructions                         │
│  └── Skill listings                                  │
├──────────────────────────────────────────────────────┤
│ Conversation Messages                                │
│  ├── [Compaction boundary + summary, if compacted]   │
│  ├── User messages + attachments                     │
│  ├── Assistant messages + tool calls + thinking      │
│  ├── Tool results (paired with tool-use requests)    │
│  └── System messages (metadata, tombstones)          │
└──────────────────────────────────────────────────────┘
```

---

## 4. Phase 2: Context Growth

### 4.1 The Query Loop

> **Source:** `src/query.ts`, `src/QueryEngine.ts`

The query pipeline (`src/query.ts`) is an async generator implementing the core
agentic loop:

```
User message → API call → [Tool calls → Tool results]* → Assistant response
```

Each iteration appends messages to the session's `messages[]` array. The pipeline
yields `StreamEvent`, `RequestStartEvent`, `Message`, and `TombstoneMessage` objects,
and returns a `Terminal` status with usage data.

The internal state machine tracks:

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
  transition: Continue | undefined   // Why previous iteration continued
}
```

**Mutable cross-iteration state** survives compaction:
- `taskBudgetRemaining` — cumulative task budget spend
- `config` — immutable query-entry snapshot (statsig, env, session state)
- `budgetTracker` — TOKEN_BUDGET feature tracking

### 4.2 The Query Engine Wrapper

> **Source:** `src/QueryEngine.ts`

`QueryEngine` wraps the query loop for SDK/headless mode:

- Maintains `mutableMessages: Message[]` as the session history
- Creates abort controllers and permission denial tracking per turn
- Manages `readFileState` — a cache of recently-accessed files for post-compact restoration
- Tracks `discoveredSkillNames` (turn-scoped, cleared each `submitMessage()`)
- Tracks `loadedNestedMemoryPaths` — prevents circular memory includes
- Accumulates `totalUsage` across turns

The `submitMessage()` method is the per-turn entry point: it clears turn-scoped state,
fetches model-specific system prompt parts, validates thinking config, injects
custom/append system prompts, and invokes the query loop.

### 4.3 Message Types and Structure

> **Source:** `src/utils/messages.ts`, `src/types/message.ts`

Messages carry rich metadata beyond text:

- **User messages**: text + attachments (file contents, memory references, MCP resources,
  skill listings). Created via `createUserMessage()` with UUID, isMeta flags, and origin
  tracking.
- **Assistant messages**: text + tool calls + thinking blocks (extended reasoning with
  token budgets)
- **System messages**: compaction boundaries (via `createCompactBoundaryMessage()`),
  tool results, session metadata
- **Tombstone messages**: deleted content markers preserving conversation structure

Tool results are **paired** with their tool-use requests. The compaction system
carefully preserves these pairs to avoid orphaned references that would confuse the
model.

### 4.4 Token Tracking

Token counts are estimated progressively via `tokenCountWithEstimation()`. The system
maintains running totals and checks against model-specific thresholds:

- **Auto-compact threshold**: `contextWindow - AUTOCOMPACT_BUFFER_TOKENS (13,000)`
- **Override**: `CLAUDE_CODE_AUTO_COMPACT_WINDOW` env var
- **Model-specific windows**: fetched from API specs per model

---

## 5. Phase 3: Context Compression

This is the most architecturally sophisticated phase. When context grows beyond the
auto-compact threshold, a multi-strategy compression pipeline activates.

### 5.1 Decision: Should We Compact?

> **Source:** `src/services/compact/autoCompact.ts`

`shouldAutoCompact()` evaluates:

1. Token count against threshold (`contextWindow - 13K`)
2. Recursive compaction guard — won't compact during a compact, session_memory, or
   marble_origami query
3. Feature gates — `REACTIVE_COMPACT` or `CONTEXT_COLLAPSE` may suppress auto-compact
4. Snip token accounting — subtract tokens that will be freed by pending snips

### 5.2 Strategy Selection: Two-Stage Approach

`autoCompactIfNeeded()` tries **two strategies in sequence**:

#### Strategy 1: Session Memory Compact (Lightweight)

> **Source:** `src/services/compact/sessionMemoryCompact.ts`

A cheap alternative that avoids an API call:

- Keeps the most recent N messages (configurable via remote config)
- Defaults: `minTokens: 10,000`, `minTextBlockMessages: 5`, `maxTokens: 40,000`
- **Preserves tool_use/tool_result pairs** — never splits a chain
- Adjusts start index to keep thinking blocks intact
- Detects messages with actual text content via `hasTextBlocks()`

#### Strategy 2: Full Summarization (Heavyweight)

> **Source:** `src/services/compact/compact.ts`

When session memory compact is insufficient:

- Sends old messages to the LLM via `streamCompactSummary()`
- **Pre-processing**:
  - Strips images/documents via `stripImagesFromMessages()` (replaces with `[image]`
    markers) — prevents "prompt too long" on the compaction request itself
  - Strips re-injectable attachments via `stripReinjectedAttachments()` (skill_discovery,
    skill_listing) — these will be restored post-compact anyway
- Output capped at `COMPACT_MAX_OUTPUT_TOKENS = 20,000`
- If the compact request itself overflows, truncates head and retries (up to 3x)

**Result**: A `CompactionResult` containing:
1. Boundary marker (system message marking the compaction point)
2. Summary messages (AI-generated conversation summary)
3. Attachments (file references, plan, skills, deferred tools, agents)
4. Hook results (session start hook messages)
5. Message-to-keep (preserved segment for partial compaction)

### 5.3 Post-Compact Restoration

After compaction, the system **strategically re-injects** frequently-needed context
that was lost in summarization:

| Restored Item | Budget | Function |
|---|---|---|
| Recently-accessed files (top 5) | 50,000 tokens | `createPostCompactFileAttachments()` |
| Active skills (truncated heads) | 25,000 tokens (5K each) | `createSkillAttachmentIfNeeded()` |
| Plan file (if in plan mode) | Unbounded | `createPlanAttachmentIfNeeded()` |
| Plan mode instructions | Minimal | `createPlanModeAttachmentIfNeeded()` |
| Background agent status | Minimal | `createAsyncAgentAttachmentsIfNeeded()` |
| Session start hook results | Minimal | Hook execution cache |

This is a key insight: **compaction doesn't just shrink — it restructures**. The
post-compact context is a curated summary + the specific files and context most likely
to be needed next.

### 5.4 Partial Compaction

> **Source:** `src/services/compact/compact.ts`, `partialCompactConversation()`

Targeted compaction around a specific message index, with two directions:

- **`from`**: summarizes after the pivot, keeps earlier messages (preserves cache prefix)
- **`up_to`**: summarizes before the pivot, keeps later messages (invalidates cache)

### 5.5 Circuit Breaker

A circuit breaker in `autoCompactIfNeeded()` stops attempts after **3 consecutive
failures** (`consecutiveFailures` counter). This prevents hammering the API when
compaction itself is failing. The counter resets on any successful compaction.

```typescript
type AutoCompactTrackingState = {
  compacted: boolean
  turnCounter: number
  turnId: string        // Unique per turn
  consecutiveFailures?: number
}
```

### 5.6 Reactive Compaction

> **Source:** `src/services/compact/reactiveCompact.ts`

When the API returns a "prompt too long" error, reactive compaction attempts an
**emergency compaction before retrying** the request. This is the last-resort strategy —
it fires only when auto-compact's proactive threshold miscalculated or token estimation
was inaccurate.

Gated behind the `REACTIVE_COMPACT` feature flag.

### 5.7 Snip Compaction

> **Source:** `src/services/compact/snipCompact.ts`

A finer-grained approach: instead of summarizing the entire old context, it identifies
and **snips specific segments** (e.g., large tool results that are no longer relevant).
Used as a complementary strategy alongside full compaction. Gated behind the
`HISTORY_SNIP` feature flag.

### 5.8 Compaction Flow Summary

```
Token count exceeds threshold?
        │
        ▼
┌───────────────────────┐
│ Circuit breaker check  │──→ 3+ consecutive failures → skip
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ Session memory compact │──→ success → done
│ (lightweight, no API)  │
└───────────┬───────────┘
            │ insufficient
            ▼
┌───────────────────────┐
│ Full summarization     │──→ success → post-compact restore → done
│ (API call)             │
└───────────┬───────────┘
            │ failure
            ▼
┌───────────────────────┐
│ Increment failure count│──→ retry next turn (or circuit break)
└───────────────────────┘

            [Meanwhile, on API "prompt too long" error:]

┌───────────────────────┐
│ Reactive compaction    │──→ emergency compact → retry request
│ (last resort)          │
└───────────────────────┘
```

---

## 6. Phase 4: Memory Extraction

### 6.1 Background Extraction Agent

> **Source:** `src/services/extractMemories/extractMemories.ts`

The most novel aspect of the context management system: a **background extraction
agent** runs asynchronously after each model response, analyzing the session for
durable learnings worth persisting.

**Execution model**:
- Triggers when the model stops (no pending tool calls)
- Runs via `handleStopHooks` — async, doesn't block the response to the user
- Implemented as a **forked agent** sharing the parent's prompt cache (cost-efficient)
- Uses closure-scoped state to track the extractor instance and drain function

**Restricted tool permissions** (via `createAutoMemCanUseTool()`):

| Tool | Access |
|---|---|
| Read, Grep, Glob | Unrestricted (read-only) |
| Bash | Read-only commands only |
| Edit, Write | **Only within auto-memory directory** |
| All other tools | Denied |

**Deduplication**: The extractor checks whether the main agent already wrote memories
in the current turn range via `hasMemoryWritesSince()`. If so, it skips that range.
Written paths are collected by `extractWrittenPaths()` from tool_use blocks.

**Activation gating**:
- Feature flag: `feature('EXTRACT_MEMORIES')` (compile-time)
- GrowthBook: `tengu_passport_quail` (runtime)
- Session type: non-interactive sessions gated by `tengu_slate_thimble`
- `isExtractModeActive()` at `src/memdir/paths.ts:69` evaluates all conditions

### 6.2 Assistant Mode Daily Logs (KAIROS)

> **Source:** `src/memdir/memdir.ts`, `buildAssistantDailyLogPrompt()`

For long-lived assistant sessions (gated behind `feature('KAIROS')`), memories follow
an **append-only daily log** pattern instead of maintaining MEMORY.md as a live index:

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

- Agent appends timestamped bullets to `memory/logs/YYYY/MM/YYYY-MM-DD.md`
- A separate nightly `/dream` skill distills logs into topic files + MEMORY.md
- MEMORY.md serves as the distilled read-only index, loaded into context automatically
- The date-named log path is described as a pattern in the prompt (not a literal),
  because the prompt is cached by `systemPromptSection('memory', ...)` and NOT
  invalidated on date change

This design acknowledges that perpetual sessions generate too much memory traffic for
a single index file — the log/distill pattern amortizes the maintenance cost.

---

## 7. Cross-Cutting Concerns

### 7.1 Prompt Cache Efficiency

The system is heavily optimized for Anthropic's prompt caching:

- **Static/dynamic boundary**: All cacheable content precedes `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`
- **Forked agents**: Compaction and extraction agents share the parent's prompt cache
  via `CacheSafeParams` threading
- **Classifier cache**: `getUserContext()` caches CLAUDE.md content for the auto-mode
  classifier (via `setCachedClaudeMdContent()`) to avoid import cycles
- **Memoized context**: `getSystemContext()` and `getUserContext()` compute once per
  session
- **Partial compaction direction**: `from` direction preserves the cache prefix;
  `up_to` invalidates it

### 7.2 Security Boundaries

> **Source:** `src/memdir/paths.ts:109-150`

Memory paths are validated against traversal attacks by `validateMemoryPath()`:

- Rejects relative paths (`!isAbsolute`)
- Rejects root/near-root paths (`length < 3`)
- Rejects Windows drive roots (`/^[A-Za-z]:$/`)
- Rejects UNC paths (`\\server\share`, `//server/share`)
- Rejects null bytes (survive `normalize()`, can truncate in syscalls)
- Normalizes to NFC Unicode form with exactly one trailing separator

**Project settings exclusion**: `projectSettings` (`.claude/settings.json` committed
to the repo) is deliberately excluded from `autoMemoryDirectory` overrides
(`src/memdir/paths.ts:173-178`). Without this, a malicious repo could set
`autoMemoryDirectory: "~/.ssh"` and gain silent write access via the filesystem.ts
write carve-out.

**Injection filtering**: `filterInjectedMemoryFiles()` at `src/utils/claudemd.ts:1142`
detects and strips suspicious patterns from memory files before they enter the prompt.

### 7.3 Telemetry and Observability

Every context management decision is instrumented for post-hoc analysis:

| Event | Meaning |
|---|---|
| `tengu_memdir_loaded` | Memory directory stats (file count, subdir count, content size) |
| `tengu_memdir_disabled` | Why memory was disabled (env var, setting, etc.) |
| `tengu_team_memdir_disabled` | Team memory specifically disabled |
| `system_context_started/completed` | System context collection timing |
| `user_context_started/completed` | User context collection timing |
| `git_status_started/completed` | Git status timing and truncation |

Compaction metrics include tokens pre/post, cache usage, compaction input/output
tokens, and duration.

### 7.4 Feature Gating

Context management features are progressively rolled out via compile-time Bun feature
flags and runtime GrowthBook gates:

**Compile-time (`bun:bundle`):**
- `REACTIVE_COMPACT` — reactive compaction on prompt-too-long
- `CONTEXT_COLLAPSE` — alternative compaction strategy
- `HISTORY_SNIP` — snip-based compaction
- `EXTRACT_MEMORIES` — background memory extraction
- `KAIROS` — assistant mode daily logs
- `TEAMMEM` — team memory synchronization
- `BREAK_CACHE_COMMAND` — system prompt cache-breaking injection

**Runtime (GrowthBook):**
- `tengu_coral_fern` — "searching past context" instructions in memory prompt
- `tengu_passport_quail` — extract mode activation
- `tengu_slate_thimble` — extract mode in non-interactive sessions
- `tengu_moth_copse` — skip MEMORY.md index in prompt
- `tengu_herring_clock` — team memory cohort tracking
- `tengu_amber_prism` — memory correction hints on user rejection

---

## 8. Design Principles

### 8.1 Invisibility

Users should never think about context limits. The system auto-compacts when the window
fills, auto-extracts learnings into persistent memory, and auto-restores critical
context after compaction — all without user intervention.

### 8.2 Structured Forgetting

Compaction is not random truncation. It is AI-generated summarization that preserves
**intent and key decisions** while discarding verbatim tool output. The model's summary
captures what happened and why, not the raw bytes of every file read.

### 8.3 Strategic Restoration

Post-compact context is **curated, not just smaller**. The 5 most recently accessed
files and active skills are re-injected because they are statistically most likely to
be referenced in subsequent turns. This makes compacted sessions feel continuous rather
than amnesiac.

### 8.4 Defense in Depth for Memory

Memory files are:
- Validated for path traversal attacks
- Filtered for prompt injection patterns
- Truncated at line and byte caps
- Security-gated at the settings source level (project settings excluded)
- Normalized to NFC Unicode

No single failure can weaponize the memory system.

### 8.5 Cache-First Architecture

Every design decision considers prompt cache impact:
- The static/dynamic boundary placement maximizes cache hits
- Forked agents share the parent's cache
- System context is memoized once per session
- Partial compaction offers a cache-preserving direction
- CLAUDE.md content is pre-cached for the classifier

### 8.6 Graceful Degradation

- Circuit breakers stop compaction after 3 consecutive failures
- Reactive compaction fires as a last resort on prompt-too-long
- Session memory compact provides a lightweight fallback when full summarization
  is too expensive
- Feature gates allow progressive rollout and instant rollback
- Memory extraction silently skips when the main agent already wrote memories

### 8.7 Separation of Ephemeral and Durable State

Session context (messages, tool results) is **ephemeral** — it grows, gets compressed,
and disappears when the session ends. Durable learnings (user preferences, project
patterns, corrections) are **extracted** into persistent memory files that survive
across sessions and inform future conversations.

The KAIROS daily-log pattern extends this further: within a perpetual session, raw
observations are append-only logs (ephemeral-ish), while the nightly distillation
produces the durable index.

### 8.8 Pair Preservation

Tool-use and tool-result messages are always kept together. The compaction system
never splits a tool chain — orphaned tool results without their request (or vice
versa) would confuse the model and break the conversation's logical structure.
Session memory compact explicitly walks the message array to find pair boundaries.
