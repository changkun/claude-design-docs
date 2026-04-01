# Context Management — Design Document

*(Core design, architecture, strategy, data flow, state machines, invariants, trade-offs, and conceptual content independent of source code.)*

---

# Claude Code: Context Management System -- Design Document

## 1. Vision

Claude Code's context management solves a fundamental tension in agentic AI systems: **the model needs rich, growing context to do useful work, but has a finite context window**. The system's vision is to make context limits invisible to the user -- sessions of arbitrary length should "just work" without the user ever thinking about tokens, windows, or memory.

The design achieves this through three interlocking strategies:

1. **Layered context assembly** -- compose the right context from heterogeneous sources at the right time
2. **Automatic context compression** -- summarize old context when the window fills, preserving what matters
3. **Persistent memory extraction** -- distill durable learnings from ephemeral sessions into files that survive across conversations

---

## 2. Architecture Overview

Context flows through five lifecycle phases:

```
+---------------+   +---------------+   +----------------+   +----------------+   +----------------+
|  Assembly     | > |   Growth      | > | Compression    | > |  Extraction    | > | Persistence    |
|               |   |               |   |                |   |                |   |                |
| System        |   | Messages      |   | Auto-compact   |   | Background     |   | JSONL          |
| prompt +      |   | accumulate    |   | summarizes     |   | agent writes   |   | transcripts    |
| memory +      |   | across        |   | old messages   |   | durable        |   | enable         |
| CLAUDE.md +   |   | tool turns    |   | when window    |   | memories       |   | resume         |
| git status    |   |               |   | fills          |   |                |   |                |
+---------------+   +---------------+   +----------------+   +----------------+   +----------------+
```

---

## 3. Phase 1: Context Assembly

### 3.1 System Prompt Construction

The system prompt is returned as a **string array**, then flattened for the API. It is constructed with a critical architectural split: a **dynamic boundary marker** separates static (cacheable) content from session-specific content.

#### Static Region (Before Boundary -- Cacheable)

Seven sections identical across users and organizations, enabling global prompt cache sharing:

| Section | Content |
|---------|---------|
| Intro | "You are Claude Code, Anthropic's official CLI..." |
| System | Permission modes, tool results, compression reminders |
| Tasks | Code style, over-engineering avoidance, security awareness |
| Actions | Reversibility heuristics, destructive operation confirmation |
| Tools | Dedicated tool guidance (Read/Edit/Glob/Grep vs Bash) |
| Tone | Emoji policy, code references, conciseness |
| Efficiency | Output size constraints, communication style |

#### Dynamic Region (After Boundary -- Per-Session)

Thirteen sections resolved asynchronously, each individually cached until `/clear` or `/compact`:

| Section | Cached? | Content |
|---------|---------|---------|
| Session guidance | Yes | Conditional tool guidance (skills, ask-user, explore) |
| Memory | Yes | Auto-memory index + behavioral instructions |
| Environment info | Yes | Git status, model name, knowledge cutoff, platform |
| Language | Yes | User language preference |
| Output style | Yes | Custom output formatting rules |
| MCP instructions | **No** | MCP server instructions (volatile -- servers connect/disconnect) |
| Scratchpad | Yes | Scratchpad directory guidance |
| Function result clearing | Yes | Microcompact instructions |
| Summarize tool results | Yes | Tool result note-taking guidance |
| Token budget | Yes | Token budget instructions (feature-gated) |
| Brief | Yes | KAIROS brief-mode tool guidance |
| Numeric length anchors | Yes | Output length constraints |
| Model override | Yes | Model override suffix |

The only **uncached** section is MCP instructions because MCP servers can connect or disconnect between turns.

#### Proactive/KAIROS Mode Variant

When autonomous mode is active, the system prompt follows an entirely different path -- a minimal autonomous-agent intro replacing all seven static sections, with risk instructions, memory prompt, environment info, and proactive-specific guidance (tick/wake-up loop, sleep tool, bias toward action).

#### Simple (Bare) Mode

Returns a single-line prompt bypassing all section construction.

#### Model-Specific Variations

- Per-model knowledge cutoff dates
- Microcompact support gated on model capabilities
- Model name display (suppressed in undercover mode)
- Frontier model info shown to help users building AI applications

### 3.2 System Context Collection

Two memoized async functions run **once per session**:

**System context**: Collects git branch, default branch, short status, recent 5 commits, user name. All git commands run in parallel. Status truncated at 2000 characters. Skipped in remote sessions or when git instructions disabled.

**User context**: Loads CLAUDE.md files from the 4-level hierarchy. Applies injection filtering. Caches result for the auto-mode classifier. Includes current date string. Respects env vars and `--bare` mode.

Both compute once and cache for the session's lifetime.

### 3.3 Memory File Hierarchy (CLAUDE.md)

Context is assembled from a four-level hierarchy:

```
Priority (lowest -> highest, higher overrides lower):

  /etc/claude-code/CLAUDE.md               <-- Managed (enterprise policy)
  ~/.claude/CLAUDE.md + ~/.claude/rules/    <-- User (personal preferences)
  .claude/CLAUDE.md + .claude/rules/*.md    <-- Project (repo-level instructions)
  CLAUDE.local.md                           <-- Local (gitignored overrides)
```

#### Directory Walk Strategy

Starts at CWD and walks **upward** to the filesystem root, collecting memory files at each level. Files processed in **root-to-CWD order** because the model pays more attention to later content -- closest instructions win.

**Worktree detection**: Nested git worktrees detected by comparing the git root with the canonical git root. When detected, project-type files from the main repo above the worktree are skipped to avoid duplication.

#### @include Directives

CLAUDE.md files support `@path` syntax for composing instructions:

- Relative, home-directory, and absolute paths supported
- Escaped spaces in paths
- Fragment stripping (e.g., `@file.md#section` -> `@file.md`)
- Circular reference prevention via processed-paths tracking
- Non-existent targets silently ignored
- Max include depth: 5
- Text-file allowlist (~100 extensions) restricts includes to safe formats

#### Conditional Rules

Files in `.claude/rules/` can have frontmatter `paths` fields specifying glob patterns. When present, the rule is only loaded if the current target path matches. Two-pass loading: unconditional rules load eagerly, conditional rules load lazily.

#### Injection Prevention

- HTML comments stripped from markdown content
- Processed-paths set prevents both symlink-based and direct circular references
- Feature-gated memory file removal (surfaced via attachments instead when gate active)

#### Assembly into Prompt Text

Files are iterated with per-type descriptions (e.g., "project instructions, checked into the codebase"), joined with a header explaining these instructions override default behavior.

#### Caching

Memory files are memoized. Cache cleared on session start, compact, or include events. A separate clear function exists for correctness-only invalidation without firing hooks.

### 3.4 Auto Memory (Persistent Cross-Session State)

The auto-memory system provides persistent, per-project memory stored in the user's home directory.

#### MEMORY.md Entrypoint

A capped index file (200 lines, 25KB) **always loaded into the system prompt**. Topic files linked from the index are read on-demand by the model.

**Truncation strategy**: Line-truncates first (natural boundary), then byte-truncates at last newline. Appends a warning naming which cap was hit.

#### Path Resolution (Priority Order)

1. Environment variable override (SDK/Cowork integration)
2. Settings-based override (trusted sources only; project settings excluded for security)
3. Default: `<memoryBase>/projects/<sanitized-git-root>/memory/`

Git worktrees sharing the same canonical root share one memory directory.

**Path validation**: Rejects relative paths, root/near-root, Windows drive roots, UNC paths, null bytes. Normalizes to NFC Unicode with trailing separator.

#### Behavioral Instructions

The system prompt includes detailed guidance for the model:

- **Four-type taxonomy**: user, feedback, project, reference
- **What NOT to save**: code patterns, architecture, git history, debugging recipes
- **How to save**: two-step process (write topic file, then add index entry)
- **When to access**: trust recall, proactive consultation, verify before recommending
- **Distinction**: memory = cross-session, plans/tasks = within-session

#### Team Memory

Feature-gated shared memory subdirectory. Security includes path sanitization, symlink resolution, and two-pass write path validation.

#### Enable/Disable Chain

Priority: env var > bare mode > remote without memory dir > settings > default (enabled).

### 3.5 Context Assembly Summary

```
+----------------------------------------------------------+
| System Prompt (static, globally cacheable)                |
|  +-- Intro + system + tasks + actions + tools + tone      |
|  +-- Output efficiency                                    |
+-- -- DYNAMIC BOUNDARY -- -- -- -- -- -- -- -- -- -- -- -- +
| System Prompt (dynamic, per-session, individually cached) |
|  +-- Session guidance                                     |
|  +-- Auto-memory index (<= 200 lines / 25KB)             |
|  +-- Memory behavioral instructions                       |
|  +-- Environment info (git, model, platform, date)        |
|  +-- Language, output style, MCP instructions (UNCACHED)  |
|  +-- Scratchpad, FRC, token budget, brief (conditional)   |
|  +-- CLAUDE.md hierarchy (filtered for injection)         |
+----------------------------------------------------------+
| Conversation Messages                                     |
|  +-- [Compaction boundary + summary, if compacted]        |
|  +-- User messages + attachments (20+ types)              |
|  +-- Assistant messages + tool calls + thinking blocks    |
|  +-- Tool results (paired with tool-use requests)         |
|  +-- System messages (metadata, tombstones, progress)     |
+----------------------------------------------------------+
```

---

## 4. Phase 2: Context Growth

### 4.1 The Query Loop

The query pipeline is an async generator implementing the core agentic loop via a `while(true)` state machine.

#### Loop Iteration Structure

1. **Setup**: Skill discovery prefetch, stream request start, query checkpoint
2. **Pre-processing**: Apply tool result budget, snip, microcompact, context collapse
3. **Proactive auto-compact check**: If tokens exceed threshold, compact before API call
4. **Model selection**: Pick model based on permission mode and token count
5. **API streaming**: Call model, process streamed blocks, collect tool calls
6. **Post-streaming**: Recovery checks, stop hooks, budget checks
7. **Tool execution**: Run tools, collect results, generate summaries
8. **Continue decision**: Inject tool results, advance turn count, loop

#### Continue Transitions

| Reason | Trigger |
|--------|---------|
| `next_turn` | Normal tool execution -> next API call |
| `collapse_drain_retry` | Drained context collapses after prompt-too-long error |
| `reactive_compact_retry` | Emergency compaction after prompt-too-long error |
| `max_output_tokens_escalate` | Escalated output cap 8K -> 64K |
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
| `prompt_too_long` | Recovery exhausted |
| `stop_hook_prevented` | Hook returned preventContinuation |
| `hook_stopped` | Hook set shouldPreventContinuation |
| `max_turns` | Turn count exceeds limit |

#### Error Withholding

Prompt-too-long, max_output_tokens, and media errors are **withheld** during streaming. Recovery is attempted first; the error surfaces only if recovery is exhausted. This prevents intermediate errors from terminating sessions in SDKs that kill on any error.

#### Model Fallback

A fallback handler switches to a fallback model mid-stream. Tombstone messages clean up orphaned partial responses. Thinking signatures are stripped if the fallback model doesn't support them.

### 4.2 The Query Engine Wrapper

The query engine wraps the query loop for SDK/headless mode:

- **Mutable message array**: Session history persisting across turns
- **Read file state**: Recently-accessed files cache, persists across turns for post-compact restoration and deduplication
- **Discovered skill names**: Turn-scoped (cleared each turn)
- **Loaded nested memory paths**: Prevents circular memory includes
- **Total usage**: Accumulated API usage across turns
- **Permission denials**: Tracked per-turn, reported in result

### 4.3 Message Types and Structure

#### Core Message Types

- **UserMessage**: Text + attachments, with metadata flags (isMeta, isVirtual, isCompactSummary, origin)
- **AssistantMessage**: Text + tool_use + thinking blocks, with error and virtual flags
- **SystemMessage**: 14+ subtypes (compact boundary, microcompact boundary, API error, permission retry, bridge status, stop hook summary, turn duration, memory saved, API metrics, local command, etc.)
- **AttachmentMessage**: 20+ types (file, directory, plan, skills, memory, team context, skill discovery, queued commands, teammate mailbox)
- **TombstoneMessage**: Deletion markers preserving conversation structure
- **ProgressMessage**: Streaming tool execution updates
- **ToolUseSummaryMessage**: SDK-only batch summaries

#### API Normalization Pipeline

A **12-step transformation pipeline** before messages reach the API:

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

Defensive validation ensuring API invariants:
- Strips duplicate tool_use IDs (cross-message dedup)
- Strips orphaned server/MCP tool_use blocks
- Inserts synthetic error tool_result blocks for unmatched tool_use blocks
- Strips orphaned tool_result blocks referencing non-existent tool_use blocks
- Creates placeholder user message for role alternation if needed

#### Attachment Normalization

Converts 20+ attachment types into user-message tuples, wrapping results in system-reminder tags when enabled.

### 4.4 Token Estimation

#### Two Parallel Approaches

**Exact counting** (from API responses): Only available AFTER an API call completes. Sums all input and output token categories.

**Rough estimation** (for prospective decisions): Character-to-token ratio. Default 4 bytes per token, except JSON files which use 2 bytes per token (dense token structure).

**Canonical measurement function**: Walks backward to the last API response (exact count), then estimates the delta for new messages added since.

#### Context Window Determination (Priority Order)

1. Environment variable explicit cap
2. `[1m]` suffix in model string -> 1M tokens
3. Model capability metadata
4. Beta header + model support check -> 1M
5. Default: 200,000

Effective window: `contextWindow - min(maxOutputTokens, 20,000)` (reserves space for compaction summary output). Further limited by user override.

#### Image and Document Estimation

Fixed 2000 tokens per image or document. Images auto-resize to max 2000x2000. This prevents massive overestimation from naive base64 byte counting.

#### Token Budget Feature

Users can specify budgets (e.g., "+500k", "use 2M tokens"). Tracked with continuation counting and diminishing-returns detection (3+ continuations with <500 token deltas triggers completion).

### 4.5 Tool Execution and Context Expansion

#### Tool Execution Pipeline (5 Phases)

**Phase 1 -- Lookup**: Find tool by name, check aliases, return error if not found.

**Phase 2 -- Permission & Pre-Hooks**: Input validation, custom validation, observable input backfill, pre-tool hooks (can override permission), speculative bash classifier (parallel), permission decision (allow/deny/ask).

**Phase 3 -- Execution**: Tool call with progress reporting.

**Phase 4 -- Post-Hooks**: Can modify MCP output, prevent continuation, add context.

**Phase 5 -- Result Persistence**: Large results exceeding size limits persisted to disk, replaced with file path preview.

#### Streaming Tool Executor

Enables concurrent tool execution during model streaming:

- **Safe tools**: Execute in parallel
- **Unsafe tools**: Execute alone (exclusive lock), order preserved
- Max concurrency: configurable (default 10)
- Results streamed in completion order
- Bash errors cascade to siblings via abort controller

#### Sub-Agent Spawning

Sub-agents created with rich configuration including MCP server initialization, worktree isolation, async/background launch, and teammate spawning for multi-agent orchestration (flat roster, no nesting).

### 4.6 State Management

#### Two-Layer Architecture

**React State**: Reactive UI updates, 150+ fields tracking UI, permissions, tools, agents, MCP, plugins, costs. Zustand-like store with pub/sub.

**Bootstrap State**: Non-React globals persisting across compaction. Includes:

- **Three CWD concepts**:
  - `originalCwd` -- initial CWD, **never updated** (sets project identity)
  - `projectRoot` -- stable root, set once at startup
  - `cwd` -- current CWD, updated by worktree tool mid-session
- Cost/duration accumulators, session identity, invoked skills, token budget state, telemetry meters

#### State Survival Across Compaction

**Preserved**: Invoked skills, plan slug cache, model usage, cost accumulators, model overrides, session ID.

**Reset**: Turn-level metrics, thinking/speculation state, pending-post-compaction flag.

**Session metadata re-append**: After compaction, metadata (title, tag, agent name, PR link) is re-appended to the transcript tail to stay within the 64KB tail window for fast resume scanning.

---

## 5. Phase 3: Context Compression

### 5.1 Decision: Should We Compact?

Evaluates:

1. **Token threshold**: Estimated tokens >= (contextWindow - 13K)
2. **Recursion guard**: Won't compact during compact, session_memory, or other internal queries
3. **Feature suppression**: Reactive compact or context collapse may suppress
4. **Snip accounting**: Subtract tokens freed by pending snips
5. **Env override**: Percentage-based threshold override

Token warning states:
- **Warning/Error**: threshold - 20K tokens (UI color difference)
- **Blocking limit**: effective window - 3K (hard stop, user must `/compact`)

### 5.2 Strategy 1: Session Memory Compact

A lightweight strategy that avoids an API call entirely:

- Keeps recent messages meeting BOTH minimums: configurable token count (default 10K) AND configurable interactive message count (default 5 text blocks)
- Hard cap: 40K tokens
- Floors at last compact boundary (preserves disk discontinuity invariant)

**Pair preservation**: Collects all tool_result IDs from kept messages, walks backward to find matching tool_use blocks. Handles streaming-split messages. Never splits a tool chain or thinking block sequence.

**Tried first**; falls back to full summarization on failure.

### 5.3 Strategy 2: Full Summarization

When session memory compact is insufficient:

#### Pre-Processing

- Replace image/document blocks with text markers (prevents the compaction API call itself from hitting prompt-too-long)
- Remove re-injectable attachment types (will be re-surfaced post-compact)

#### Summarization Prompt

Structured summary with **9 required sections**:

1. Primary Request and Intent
2. Key Technical Concepts (bullet list)
3. Files and Code Sections (with full snippets)
4. Errors and fixes (with resolution)
5. Problem Solving (solved and ongoing)
6. All user messages (non-tool-use only)
7. Pending Tasks
8. Current Work (precise description from recent messages)
9. Optional Next Step (with direct quotes)

Includes aggressive "TEXT ONLY" instructions because newer models sometimes attempt tool calls during summarization despite explicit prohibitions.

Output capped at 20,000 tokens. If the compact request itself overflows, oldest API-round groups are dropped and retried (up to 3 retries).

#### Cache-Sharing Optimization

Tries a **forked agent path** first to reuse the main conversation's cached prompt prefix. Falls back to regular streaming if the fork fails. Retry with exponential backoff and keep-alive signals to prevent idle timeouts.

#### CompactionResult

Contains: boundary marker, summary messages, attachments, hook results, optional preserved messages, pre/post token counts, compaction usage metrics.

### 5.4 Post-Compact Restoration

After compaction, **strategically re-inject** context lost in summarization:

| Restored Item | Budget | Details |
|---|---|---|
| Recently-accessed files (top 5) | 50K tokens total, 5K each | Most-recent-first, skips plan/memory files |
| Active skills | 25K tokens total, 5K each | Most-recent-first, truncated heads (where instructions live) |
| Plan file | Unbounded | If plan exists for current session |
| Plan mode instructions | Minimal | Ensures model continues in plan mode |
| Background agent status | Minimal | Running agents (prevent duplicate spawn), finished agents (results pending) |
| Deferred tools delta | Minimal | Tools announced since last boundary |
| Agent listing delta | Minimal | Agents discovered since last boundary |
| MCP instructions delta | Minimal | MCP server changes since last boundary |
| Session start hooks | Minimal | Restores CLAUDE.md context |

### 5.5 Microcompact: Per-Turn Tool Result Clearing

A fine-grained, per-turn optimization that clears old tool results without full compaction:

#### Time-Based Microcompact

When the gap since last assistant message exceeds a threshold (default 60 minutes), clears old tool results. Keeps last N results (default 5), content-clears others. Mutates message content directly (cache is cold anyway).

#### Cached Microcompact (Cache-Editing API)

For active sessions with warm cache: identifies compactable tool IDs, registers tool results grouped by user message, determines which to clear, creates cache-edit blocks. Does NOT mutate local message content -- uses cache_reference/cache_edits at API layer. Preserves prompt cache while reducing context.

#### API-Side Context Management

Server-side clearing configs: tool result clearing at 180K input tokens (keep 40K), tool use clearing (same trigger, exclude write-type tools), thinking clearing (keep last 1 turn if >1h idle, otherwise keep all).

### 5.6 Partial Compaction

Targeted compaction around a selected message index:

- **Direction `from`**: Summarizes messages AFTER pivot, keeps earlier. **Preserves prompt cache prefix.**
- **Direction `up_to`**: Summarizes messages BEFORE pivot, keeps later. **Invalidates cache** since summary precedes kept messages.

Strips old compact boundaries from kept messages. Annotates boundary with preserved-segment metadata for resume relinking.

### 5.7 Reactive Compaction

When the API returns a "prompt too long" error, recovery sequence:

1. Drain context-collapse queue (if feature enabled)
2. Emergency compaction (one-shot guard prevents infinite loops)
3. Surface the withheld error if both fail

Guard reset on each new tool-execution turn.

### 5.8 Snip Compaction

Identifies and snips specific segments (large tool results no longer relevant). Uses message ID tags appended during API normalization for the model to reference specific segments.

### 5.9 Circuit Breaker

Stops after 3 consecutive auto-compact failures. Empirical data showed some sessions accumulating 50+ consecutive failures (up to 3,272), wasting significant API calls per day. Counter resets on any successful compaction.

### 5.10 Compaction Flow Summary

```
+-- autoCompactIfNeeded() -----------------------------------------+
|                                                                    |
|  Circuit breaker: 3+ consecutive failures? -> skip                 |
|         |                                                          |
|         v                                                          |
|  Session memory compact (lightweight, no API call)                 |
|  +-- Uses last summarized message boundary                         |
|  +-- Keeps recent messages (10K-40K tokens, 5+ text blocks)        |
|  +-- Preserves tool_use/tool_result pairs                          |
|  +-- Success? -> done                                              |
|         | failure                                                  |
|         v                                                          |
|  Full summarization (API call, forked agent for cache sharing)     |
|  +-- Strip images/documents, re-injectable attachments             |
|  +-- 9-section structured summary (<= 20K output tokens)           |
|  +-- PTL retry: truncate head, retry up to 3x                     |
|  +-- Post-compact restore: files, skills, plan, agents             |
|  +-- Success? -> done                                              |
|         | failure                                                  |
|         v                                                          |
|  Increment failures -> retry next turn or circuit break            |
|                                                                    |
+-- Meanwhile, on API "prompt too long" error: ---------------------+
|  1. Drain context-collapse queue                                   |
|  2. Reactive compact (emergency, one-shot guard)                   |
|  3. Surface error if both fail                                     |
|                                                                    |
+-- Every turn, before API call: -----------------------------------+
|  microcompactMessages()                                            |
|  +-- Time-based trigger (>60min gap -> clear old tool results)     |
|  +-- Cached MC path (cache-editing API, warm cache preserved)      |
|                                                                    |
+-------------------------------------------------------------------+
```

---

## 6. Phase 4: Memory Extraction

### 6.1 Background Extraction Agent

A **forked agent** runs asynchronously after each model response, analyzing the session for durable learnings.

#### Lifecycle

- **Trigger**: Fires when model stops (no pending tool calls)
- **Execution**: Fire-and-forget
- **Drain**: Called before graceful shutdown with 60-second soft timeout
- **State**: Closure-scoped (in-flight extractions set, cursor position, overlap guard, throttle counter, pending context)

#### Fork Configuration

- Shares parent's prompt cache
- Restricted permissions (see below)
- Hard cap of 5 turns (well-behaved: 2-4 turns)
- No transcript writing (prevents race conditions)

#### Restricted Permissions

| Tool | Access |
|---|---|
| Read, Grep, Glob | Unrestricted |
| REPL | When enabled; inner primitives re-gated |
| Bash | Read-only commands only |
| Edit, Write | Only within auto-memory directory |
| MCP, Agent, all others | Denied |

#### Deduplication and Throttling

- **Mutual exclusion**: Checks if main agent wrote memories; skips extraction for that range
- **Overlap guard**: New requests during extraction are stashed as trailing runs (only latest context kept)
- **Throttle**: Configurable N-turn interval
- **Trailing runs bypass throttle**: Process already-committed work

#### Memory Manifest Pre-Injection

All existing memory files with type, timestamp, and description are injected into the extraction prompt so the agent doesn't spend a turn on file listing.

### 6.2 The Four-Type Memory Taxonomy

```
Types: user, feedback, project, reference
```

| Type | Scope | Purpose | Examples |
|------|-------|---------|----------|
| **user** | Always private | Role, goals, knowledge, preferences | "data scientist investigating logging", "10 years Go, new to React" |
| **feedback** | Private (team if project-wide) | Corrections AND confirmations on approach | "don't mock databases" + why + how to apply |
| **project** | Private or team | Ongoing work, goals NOT derivable from code/git | "freeze merges 2026-03-05", "auth rewrite for compliance" |
| **reference** | Usually team | Pointers to external systems | "bugs tracked in Linear INGEST", "Grafana latency board" |

#### Frontmatter Format

Each memory file has YAML frontmatter with name, description, and type. Content includes the rule/fact followed by **Why** and **How to apply** sections.

#### What NOT to Save

Code patterns, conventions, architecture, file paths, git history, debugging solutions, CLAUDE.md content, ephemeral task details.

#### Trust and Verification

Memories are treated as claims, not ground truth. Before recommending based on memory: check file exists, grep for function/flag, verify before user acts. Stale memories should be updated or removed when they conflict with current state.

### 6.3 Auto-Dream Consolidation

A nightly consolidation engine that distills accumulated memory:

#### Three-Stage Gating

1. **Time gate**: Minimum hours since last consolidation (default 24h)
2. **Session gate**: Minimum session count (default 5)
3. **Lock gate**: Per-process consolidation lock

Session scan throttled to 10-minute intervals.

#### Execution

Same forked-agent pattern as extraction. Progress tracked via a task object with UI visibility. Tool use collapsed to counts. Edit/Write paths collected.

#### /remember Command

User-invocable skill that reviews all memory layers and proposes promotions (auto-memory -> CLAUDE.md), cleanup (duplicates, outdated), and presents all proposals before changes.

### 6.4 Assistant Mode Daily Logs (KAIROS)

For perpetual sessions:

```
memory/
+-- MEMORY.md                          <-- distilled index (nightly)
+-- logs/
|   +-- 2026/
|       +-- 03/
|           +-- 2026-03-30.md          <-- timestamped bullets
|           +-- 2026-03-31.md          <-- today's log
+-- topic files (distilled from logs)
```

- Agent appends timestamped bullets to daily log file
- Date path described as pattern in prompt (not literal) -- cached without date-change invalidation
- Nightly `/dream` skill distills logs into topic files + MEMORY.md
- MEMORY.md serves as read-only distilled index

---

## 7. Phase 5: Session Persistence and Resume

### 7.1 Transcript Format

Sessions serialized as **JSONL** (JSON Lines) -- one JSON object per line, append-only, never rewritten in place.

**Entry types**: Transcript messages (user, assistant, system, attachment) plus metadata entries (title, tag, agent name/color/setting, mode, worktree state, PR link, file history snapshot, attribution snapshot, content replacement, context collapse snapshot/commit).

Each transcript message includes: uuid, parentUuid, logicalParentUuid, isSidechain, sessionId, cwd, userType, timestamp, version, gitBranch, slug.

#### Write Queue System

Writes batched via a queue (100ms flush interval, 100MB max chunk). Session file materialized lazily on first user/assistant message (prevents metadata-only files).

#### Metadata Tail Window

Session metadata re-appended at EOF during compaction and session exit. Keeps metadata within the last 64KB scanned by the fast resume picker, avoiding expensive full-file reads.

### 7.2 Session Resume

#### Resume Flow

1. **Load transcript**: Read JSONL forward. For files >5MB, skip pre-compaction content via fast boundary scan.
2. **Restore agent state**: Re-apply model and agent type from agent definition entry.
3. **Build conversation chain**: Walk parentUuid chain from leaf to root, handling compaction boundaries, preserved segment relinks, snip removals, orphaned parallel tool results, and consistency validation.
4. **Restore cost state** from session file.

#### What IS Restored

Full message history (within compaction boundary), file history snapshots, attribution state, context collapse state, worktree state, session metadata, cost state.

#### What IS Lost

Progress messages (ephemeral), pre-compaction messages, snipped messages, orphaned attachments from crash-at-yield, metadata not in the tail window.

#### Cross-Project Resume

Requires explicit path specification. Reuses resumed session's ID unless fork-session requested. Worktree state restored only if directory still exists.

### 7.3 Session Search

#### Lite Metadata (Fast Resume Picker)

Reads only head (first 1KB) and tail (last 64KB) of each transcript. Extracts: first prompt, title, tag, summary, modified date. Fast for hundreds of sessions.

#### Agentic Session Search

1. Pre-filter by keyword (title, tag, branch, summary, first prompt)
2. Send top 100 to Claude API for relevance ranking
3. Return ranked indices
4. Priorities: tag > title > branch > summary > transcript

#### Searching Past Context (In-Session)

When feature-gated, the memory prompt includes grep commands for searching memory files and session transcripts.

---

## 8. Cross-Cutting Concerns

### 8.1 Prompt Cache Efficiency

The system is optimized for prompt caching at every level:

- **Static/dynamic boundary**: Splits globally-cacheable from session-specific content
- **Section-level caching**: All sections cached except MCP instructions
- **Forked agents**: Share parent's cache
- **Cached microcompact**: Deletes tool results without invalidating cache
- **Partial compaction direction**: `from` preserves cache prefix; `up_to` invalidates
- **Tool pool ordering**: Built-in tools sorted alphabetically as contiguous prefix; MCP tools after -- stable cache key
- **Prompt cache latches**: Once a beta header is sent, it stays to prevent oscillation
- **1-hour cache TTL awareness**: Time-based microcompact threshold matches server cache TTL

### 8.2 Security Boundaries

#### Memory Path Validation

- Rejects relative, root/near-root, Windows drive root, UNC, null-byte paths
- Normalizes to NFC Unicode with trailing separator
- Tilde expansion only for `~/...` (bare `~` rejected)
- **Project settings exclusion**: Project settings file cannot override memory directory (prevents malicious repos from gaining write access)
- **Team memory path security**: Two-pass validation (normalize + containment check, then symlink resolution)

#### Memory Content Security

- Injection filtering via feature-gated removal
- HTML comment stripping
- Circular reference prevention
- Include depth limits (max 5)
- Text-file allowlist (~100 extensions)
- Per-file character cap recommendation (40,000)

### 8.3 Telemetry and Observability

Key instrumented events:

| Event | Meaning |
|---|---|
| Memory dir loaded | File/subdir count, content size, truncation |
| Memory disabled | Why (env var, setting) |
| Compact | Full compaction metrics (tokens pre/post, cache, discovery state) |
| Partial compact | Direction, kept/summarized counts |
| Cached microcompact | Tools cleared, tokens saved |
| Time-based microcompact | Gap minutes, results cleared |
| Memory tool denied | Extraction agent tool denial |
| Extract memories coalesced | Overlapping extraction stashed |
| CLAUDE.md permission error | File permission denied |
| CLAUDE.md initial load | Load statistics |
| System context timing | Collection timing and truncation |

### 8.4 Feature Gating

Two categories:

**Compile-time (dead code eliminated)**: Reactive compact, context collapse, history snip, cached microcompact, extract memories, KAIROS, team memory, cache-breaking, proactive mode, fork agent, skill search, verification agent, token budget.

**Runtime (progressive rollout)**: Past-context search, extract mode, team memory cohort, memory correction hints, extraction throttle, auto-dream timing, session memory compact params, time-based microcompact config, cache sharing for compaction, reactive-only mode, CLAUDE.md skipping, attachment smooshing, tool reference relocation.

---

## 9. Design Principles

1. **Invisibility**: Users should never think about context limits. Auto-compact, auto-extract, auto-restore, auto-resume.

2. **Structured Forgetting**: Compaction is a 9-section structured summarization preserving intent, decisions, file snapshots, and pending tasks -- not random truncation.

3. **Strategic Restoration**: Post-compact context is curated. The 5 most recently accessed files, active skills, plan state, and background agent status are re-injected as statistically most likely to be needed next.

4. **Defense in Depth for Memory**: Multiple independent safety gates (path validation, symlink resolution, settings source exclusion, content filtering, truncation caps, include depth limits, text-file allowlist, NFC normalization).

5. **Cache-First Architecture**: Every design decision considers prompt cache impact.

6. **Graceful Degradation**: Circuit breaker, reactive compaction, lightweight fallback, time-based cleanup, output escalation, model fallback, silent skip, feature gates.

7. **Separation of Ephemeral and Durable State**: Session context is ephemeral; extracted learnings are persistent. KAIROS extends this with append-only logs and nightly distillation.

8. **Pair Preservation**: Tool-use and tool-result messages always kept together, with streaming-split tracking and synthetic error insertion for orphans.

9. **Append-Only Durability**: JSONL transcripts are never rewritten. Crash recovery via valid JSONL prefixes. ParentUuid chain enables reconstruction. Content replacement entries externalize large results. Metadata re-append keeps critical fields accessible.
