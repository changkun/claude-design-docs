# Claude Code: Auto-Memory System — Design Specification

This document analyzes the persistent cross-session memory architecture of Claude Code —
Anthropic's agentic CLI tool — covering how it stores, extracts, recalls, synchronizes,
and secures durable learnings that survive across conversations.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Memory Directory Structure](#2-memory-directory-structure)
- [3. The MEMORY.md Contract](#3-the-memorymd-contract)
- [4. Background Extraction Agent](#4-background-extraction-agent)
- [5. Memory Taxonomy](#5-memory-taxonomy)
- [6. Session Memory](#6-session-memory)
- [7. Memory Recall](#7-memory-recall)
- [8. Team Memory Sync](#8-team-memory-sync)
- [9. KAIROS Memory Model](#9-kairos-memory-model)
- [10. Security](#10-security)
- [11. Activation and Gating](#11-activation-and-gating)
- [12. Integration with Context Management](#12-integration-with-context-management)
- [13. Telemetry and Observability](#13-telemetry-and-observability)
- [14. Design Principles](#14-design-principles)

---

## 1. Overview

Claude Code sessions are ephemeral by default. When a conversation ends, its context
window — tool results, user instructions, model reasoning — evaporates. The
**auto-memory system** solves this by maintaining a persistent, file-based memory
layer that bridges sessions: learnings extracted from one conversation become available
context in every subsequent conversation for the same project.

The system answers a precise question: **what from this session would be valuable in a
future session that cannot be derived from reading the current project state?** Code
patterns, architecture, and git history are rederivable from the filesystem; user
preferences, behavioral corrections, project context, and external-system pointers are
not. The memory system captures the latter and discards the former.

Three complementary subsystems cooperate:

1. **Auto-memory (persistent cross-session)** — a file-based memory directory
   (`~/.claude/projects/<path>/memory/`) with a MEMORY.md index always loaded into the
   system prompt, topic files read on demand, and a background extraction agent that
   distills durable learnings after each model response.

2. **Session memory** — a structured markdown file summarizing the current session's
   state, periodically updated by a forked subagent, used to improve context
   compaction quality.

3. **Team memory sync** — a shared `team/` subdirectory synchronized to a server API,
   enabling memories to propagate across team members working on the same repository.

These layers are additive. Auto-memory is the foundation; session memory improves
within-session continuity after compaction; team memory extends the persistence
boundary from individual to organizational.

---

## 2. Memory Directory Structure

> **Source:** `src/memdir/paths.ts`, `src/memdir/memdir.ts`

### 2.1 Directory Layout

Each project gets a memory directory under the user's Claude configuration home:

```
~/.claude/projects/<sanitized-project-root>/memory/
├── MEMORY.md                        ← index file (always in system prompt)
├── user_role.md                     ← topic file (user type)
├── feedback_testing.md              ← topic file (feedback type)
├── project_auth_rewrite.md          ← topic file (project type)
├── reference_dashboards.md          ← topic file (reference type)
├── logs/                            ← KAIROS daily logs (when active)
│   └── 2026/
│       └── 03/
│           ├── 2026-03-30.md
│           └── 2026-03-31.md
└── team/                            ← shared team memory (when enabled)
    ├── MEMORY.md                    ← team index
    ├── feedback_db_tests.md
    └── reference_linear.md
```

### 2.2 Path Resolution

`getAutoMemPath()` at `src/memdir/paths.ts:223` resolves the memory directory through
a three-level priority chain:

| Priority | Source | Notes |
|---|---|---|
| 1 | `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` env var | Full-path override for SDK/Cowork |
| 2 | `autoMemoryDirectory` in settings.json | Trusted sources only (policy, flag, local, user) |
| 3 | `<memoryBase>/projects/<sanitized-git-root>/memory/` | Default computation |

The default path uses `getMemoryBaseDir()` (which checks `CLAUDE_CODE_REMOTE_MEMORY_DIR`
before falling back to `~/.claude`) joined with a sanitized project root. The sanitization
converts the project's absolute path into a safe directory name via `sanitizePath()`.

**Worktree sharing:** `findCanonicalGitRoot()` resolves git worktrees to their canonical
root, so all worktrees of the same repository share one memory directory. This prevents
memory fragmentation across `git worktree add` invocations.

**Memoization:** `getAutoMemPath()` is memoized by `getProjectRoot()` — the
computation involves 4 `getSettingsForSource()` calls, each performing `realpathSync` +
`readFileSync`, which is too expensive for render-path callers like
`collapseReadSearchGroups`.

### 2.3 Directory Existence Guarantee

`ensureMemoryDirExists()` at `src/memdir/memdir.ts:129` creates the directory tree
idempotently before the prompt is built. The system prompt includes
`DIR_EXISTS_GUIDANCE`: "This directory already exists -- write to it directly with the
Write tool (do not run mkdir or check for its existence)." This eliminates a common
failure mode where the model wastes a turn on `ls` or `mkdir -p` before writing.

---

## 3. The MEMORY.md Contract

> **Source:** `src/memdir/memdir.ts:34-103`

MEMORY.md is the entrypoint file — the only memory artifact guaranteed to be in the
system prompt for every API call.

### 3.1 Role

MEMORY.md is an **index, not a memory**. Each entry is a one-line pointer to a topic
file:

```markdown
- [User role](user_role.md) — data scientist, focused on observability
- [Testing policy](feedback_testing.md) — integration tests must use real DB
- [Auth rewrite context](project_auth_rewrite.md) — legal-driven, compliance priority
```

The model reads MEMORY.md to orient itself at the start of a session, then reads
individual topic files on demand when they seem relevant. This two-level indirection
keeps the always-loaded index small while supporting an unbounded number of memories.

### 3.2 Size Caps

Two caps prevent MEMORY.md from bloating the system prompt:

| Cap | Value | Constant |
|---|---|---|
| Line cap | 200 lines | `MAX_ENTRYPOINT_LINES` |
| Byte cap | 25,000 bytes | `MAX_ENTRYPOINT_BYTES` |

**Truncation logic** (`truncateEntrypointContent()` at `src/memdir/memdir.ts:57`):

1. Line-truncate first — a natural boundary that preserves complete index entries
2. If the result still exceeds the byte cap, byte-truncate at the last newline before
   the limit (never cuts mid-line)
3. Append a warning naming which cap fired:
   - Byte only: `"MEMORY.md is 30.2 KB (limit: 25.0 KB) — index entries are too long"`
   - Line only: `"MEMORY.md is 243 lines (limit: 200)"`
   - Both: `"MEMORY.md is 243 lines and 30.2 KB"`

The byte cap targets a specific failure mode observed in production: indexes with fewer
than 200 lines but extremely long lines (p100 observed: 197KB under 200 lines). The
25KB limit (~125 chars/line at 200 lines) catches this at p97.

### 3.3 Index Skip Mode

The `tengu_moth_copse` GrowthBook gate controls an experimental mode where MEMORY.md
is not maintained as an index. When `skipIndex` is true, the extraction agent and
system prompt omit the two-step save process (write topic file, then add index entry)
and instead instruct the model to write standalone topic files with no index pointer.
This simplifies the write protocol at the cost of losing the always-loaded orientation
index.

---

## 4. Background Extraction Agent

> **Source:** `src/services/extractMemories/extractMemories.ts`,
> `src/services/extractMemories/prompts.ts`

The extraction agent is the most novel component: a **background subagent** that runs
asynchronously after each model response, analyzing the session for durable learnings
worth persisting to disk.

### 4.1 Execution Model

**Trigger:** The extractor fires when the model stops (no pending tool calls),
invoked as a post-sampling hook via `handleStopHooks`. It runs fire-and-forget —
it does not block the response to the user.

**Forked agent pattern:** The extraction agent is a **perfect fork** of the main
conversation, created via `runForkedAgent()`. This is architecturally significant:
the fork shares the parent's prompt cache (system prompt + message prefix), so the
API call's input tokens are almost entirely cache reads. The extraction is cheap
because it reuses cached context rather than recomputing it.

**Closure-scoped state:** All mutable state lives inside `initExtractMemories()`,
following the same pattern as `confidenceRating.ts`:

```typescript
let lastMemoryMessageUuid: string | undefined   // cursor position
let inProgress: boolean                          // overlap guard
let turnsSinceLastExtraction: number             // throttle counter
let pendingContext: { context, appendSystemMessage } | undefined  // stash
const inFlightExtractions = new Set<Promise<void>>()  // drain set
```

### 4.2 Turn Throttling

The extractor does not run on every turn. `tengu_bramble_lintel` (default 1) controls
how many eligible turns must pass between extractions. This is useful for reducing API
costs during rapid tool-call sequences where no durable learning is likely.

### 4.3 Overlap Guard and Trailing Runs

Only one extraction runs at a time. If a new hook fires during an in-progress
extraction:

1. The new context is stashed in `pendingContext`
2. When the current extraction finishes (in the `finally` block), it checks for a
   stashed context
3. If present, a **trailing run** executes with the latest stashed context
4. The trailing run's `newMessageCount` is computed relative to the just-advanced
   cursor, so it only processes messages added between the two calls

This coalescing ensures no messages are missed while preventing unbounded concurrency.

### 4.4 Main Agent Mutual Exclusion

The main agent's system prompt always includes full memory-save instructions. When the
user says "remember this" or the model spontaneously decides to save, the main agent
writes directly to the memory directory. The extraction agent detects this via
`hasMemoryWritesSince()`:

```typescript
function hasMemoryWritesSince(messages, sinceUuid): boolean {
  // Scan assistant messages after the cursor for Write/Edit tool_use
  // blocks targeting paths within isAutoMemPath()
}
```

When detected, the extractor **skips** and advances the cursor past the main agent's
writes. This makes the two agents mutually exclusive per turn range — the main agent
and the background agent never both write memories for the same set of messages.

### 4.5 Tool Permissions

`createAutoMemCanUseTool()` at `src/services/extractMemories/extractMemories.ts:171`
implements a strict permission model:

| Tool | Permission |
|---|---|
| Read, Grep, Glob | Unrestricted (inherently read-only) |
| Bash | Read-only commands only (ls, find, grep, cat, stat, wc, head, tail) |
| Edit, Write | **Only within the auto-memory directory** (`isAutoMemPath()`) |
| REPL | Allowed (REPL's inner tool calls are re-gated by canUseTool) |
| All other tools | Denied |

Bash `rm` is explicitly denied — the extraction agent cannot delete memory files. MCP
tools, Agent tools, and write-capable Bash are all blocked.

The REPL allowance deserves explanation: in ant-default REPL mode, primitive tools
(Read/Write/Edit/Bash) are hidden from the tool list so the model calls REPL instead.
REPL's VM context re-invokes `canUseTool` for each inner primitive, so the actual
filesystem operations are still gated. Importantly, giving the fork a different tool
list would **break prompt cache sharing** — tools are part of the cache key.

### 4.6 Pre-injected Memory Manifest

Before invoking the forked agent, the extractor scans the memory directory via
`scanMemoryFiles()` and formats the result as a manifest:

```
- [user] user_role.md (2026-03-31T10:00:00.000Z): data scientist focused on observability
- [feedback] feedback_testing.md (2026-03-30T15:00:00.000Z): real DB required for tests
```

This is injected into the extraction prompt so the agent does not waste a turn on `ls`.
The manifest includes frontmatter descriptions and types, enabling the agent to decide
whether to update an existing file or create a new one without reading every file.

### 4.7 Turn Budget

The forked agent has a hard cap of `maxTurns: 5`. The prompt instructs the efficient
strategy: turn 1 — issue all Read calls in parallel for files to update; turn 2 —
issue all Write/Edit calls in parallel. Well-behaved extractions complete in 2-4 turns.
The hard cap prevents verification rabbit-holes (grepping source files to confirm a
pattern exists) from burning excessive API calls.

### 4.8 Drain on Shutdown

`drainPendingExtraction()` awaits all in-flight extractions with a soft timeout
(default 60s). It is called by `print.ts` after the response is flushed but before
`gracefulShutdownSync`, ensuring the forked agent completes before the 5s shutdown
failsafe kills the process.

### 4.9 System Message Notification

When the extractor writes memory files, it calls `appendSystemMessage()` with a
`createMemorySavedMessage()`. This injects a notification into the main conversation
so the user sees feedback (e.g., "Saved 2 memories") without the extraction blocking
the primary response.

---

## 5. Memory Taxonomy

> **Source:** `src/memdir/memoryTypes.ts`

### 5.1 The Four Types

Memories are constrained to a **closed four-type taxonomy** capturing context that is
NOT derivable from the current project state:

| Type | Description | Scope (team mode) |
|---|---|---|
| `user` | User's role, goals, preferences, knowledge | Always private |
| `feedback` | Behavioral corrections and confirmed approaches | Default private; team for project-wide conventions |
| `project` | Work context, deadlines, decisions, incidents | Bias toward team |
| `reference` | Pointers to external systems (dashboards, trackers) | Usually team |

### 5.2 What NOT to Save

The taxonomy's negative space is explicitly defined in `WHAT_NOT_TO_SAVE_SECTION`:

- Code patterns, conventions, architecture, file paths, project structure — derivable
  from reading the current project state
- Git history, recent changes, who-changed-what — `git log` / `git blame` are
  authoritative
- Debugging solutions or fix recipes — the fix is in the code; the commit message has
  the context
- Anything already documented in CLAUDE.md files
- Ephemeral task details: in-progress work, temporary state, current conversation context

These exclusions apply **even when the user explicitly asks**. If a user asks to save a
PR list or activity summary, the model is instructed to ask what was surprising or
non-obvious — that is the part worth keeping.

### 5.3 Frontmatter-Based Structured Storage

Each topic file uses YAML frontmatter:

```markdown
---
name: {{memory name}}
description: {{one-line description — used to decide relevance}}
type: {{user, feedback, project, reference}}
---

{{memory content}}
```

The `description` field is critical for recall: the relevance selector (section 7) uses
descriptions to decide which memories to surface for a given query. The `type` field
enables filtering and scoping logic, particularly the private/team routing in combined
mode.

For `feedback` and `project` types, the body is structured as: rule/fact, then
`**Why:**` and `**How to apply:**` lines. Knowing why lets the model judge edge cases
instead of blindly following the rule.

### 5.4 Individual vs. Combined Mode

Two variants of the type section exist:

- **`TYPES_SECTION_INDIVIDUAL`** — single directory, no `<scope>` tags, no team/private
  qualifiers
- **`TYPES_SECTION_COMBINED`** — private + team directories, per-type `<scope>` guidance
  embedded in XML-style `<type>` blocks

The combined variant includes explicit scope routing: user memories are always private;
feedback defaults to private but goes team for project-wide conventions; project biases
toward team; reference is usually team. This routing is baked into the type blocks, not
a separate section.

---

## 6. Session Memory

> **Source:** `src/services/SessionMemory/sessionMemory.ts`,
> `src/services/SessionMemory/prompts.ts`,
> `src/services/SessionMemory/sessionMemoryUtils.ts`

Session memory is a complementary system to auto-memory: where auto-memory captures
durable cross-session learnings, session memory captures the **current session's state**
in a structured markdown file. Its primary purpose is improving the quality of context
compaction.

### 6.1 The Session Memory File

A structured markdown file at `~/.claude/session-memory/<session-id>.md` with
predefined sections:

```markdown
# Session Title
_A short and distinctive 5-10 word descriptive title for the session_

# Current State
_What is actively being worked on right now?_

# Task specification
_What did the user ask to build?_

# Files and Functions
_What are the important files?_

# Workflow
_What bash commands are usually run and in what order?_

# Errors & Corrections
_Errors encountered and how they were fixed_

# Codebase and System Documentation
_What are the important system components?_

# Learnings
_What has worked well? What has not?_

# Key results
_If the user asked a specific output, repeat the exact result here_

# Worklog
_Step by step, what was attempted, done?_
```

The template is customizable via `~/.claude/session-memory/config/template.md`. The
update prompt is customizable via `~/.claude/session-memory/config/prompt.md`.

### 6.2 Extraction Triggers

Session memory extraction is threshold-gated to prevent excessive updates:

| Threshold | Default | Config Key |
|---|---|---|
| Initialization threshold | 10,000 tokens | `minimumMessageTokensToInit` |
| Update interval | 5,000 tokens growth | `minimumTokensBetweenUpdate` |
| Tool call minimum | 3 tool calls | `toolCallsBetweenUpdates` |

Extraction triggers when:
- Both the token threshold AND tool call threshold are met, OR
- The token threshold is met AND the last assistant turn has no tool calls (natural
  conversation break)

The token threshold is **always required** — even if the tool call threshold is met,
extraction will not happen until the token threshold is also satisfied.

### 6.3 Extraction Mechanics

Session memory extraction uses the same forked agent pattern as auto-memory extraction:

1. The `extractSessionMemory` hook is registered as a post-sampling hook via
   `registerPostSamplingHook`
2. It creates an isolated `ToolUseContext` via `createSubagentContext` to avoid
   polluting the parent's `readFileState` cache
3. It reads the current memory file content via `FileReadTool`
4. It runs `runForkedAgent` with a prompt that instructs the model to update the
   structured sections using Edit tool calls
5. The forked agent is restricted to `FILE_EDIT_TOOL_NAME` on exactly the session
   memory file path — no other tools are permitted

### 6.4 Section Size Management

Each section is capped at ~2,000 tokens. The total file is capped at 12,000 tokens.
`analyzeSectionSizes()` and `generateSectionReminders()` compute per-section token
counts and generate warnings when limits are approached. When the total budget is
exceeded, the update prompt includes a CRITICAL instruction to aggressively condense.

### 6.5 Compaction Integration

Session memory's primary consumer is the compaction system. When auto-compact fires,
the session memory content provides structured notes that help the summarization model
produce a higher-quality summary. `truncateSessionMemoryForCompact()` truncates
oversized sections before insertion to prevent session memory from consuming the entire
post-compact token budget.

### 6.6 Sequential Execution

The extraction hook is wrapped in `sequential()` — a utility that ensures at most one
invocation runs at a time. Unlike the auto-memory extractor's manual overlap guard,
session memory uses a generic sequential wrapper.

---

## 7. Memory Recall

> **Source:** `src/memdir/findRelevantMemories.ts`, `src/memdir/memoryScan.ts`,
> `src/memdir/memoryAge.ts`

### 7.1 On-Demand Relevance Selection

When a user query arrives, the system selects up to 5 relevant memory files to surface.
This is a two-stage process:

**Stage 1: Scan** (`scanMemoryFiles()` at `src/memdir/memoryScan.ts:35`):
- Reads all `.md` files in the memory directory (excluding MEMORY.md)
- Parses the first 30 lines of each for frontmatter (description, type)
- Sorts by `mtimeMs` (newest first)
- Caps at 200 files (`MAX_MEMORY_FILES`)

**Stage 2: Select** (`selectRelevantMemories()` at
`src/memdir/findRelevantMemories.ts:77`):
- Sends the query + memory manifest to a Sonnet side-query
- Uses structured JSON output (`selected_memories: string[]`)
- Filters recently-used tools to suppress API documentation noise
- Filters already-surfaced paths to avoid redundant selections
- Returns absolute file paths + mtime

### 7.2 Memory Age and Staleness

`memoryAge.ts` provides freshness annotations:

- `memoryAge(mtimeMs)` — human-readable: "today", "yesterday", "47 days ago"
- `memoryFreshnessText(mtimeMs)` — staleness caveat for memories >1 day old:
  "This memory is 47 days old. Memories are point-in-time observations, not live
  state — claims about code behavior or file:line citations may be outdated."
- `memoryFreshnessNote(mtimeMs)` — wraps the caveat in `<system-reminder>` tags

### 7.3 Recall-Side Guidance

The system prompt includes two sections governing how recalled memories are used:

**"When to access memories"** (`WHEN_TO_ACCESS_SECTION`):
- Access when memories seem relevant or user references prior work
- MUST access when the user explicitly asks
- If the user says to **ignore** memory: proceed as if MEMORY.md were empty

**"Before recommending from memory"** (`TRUSTING_RECALL_SECTION`):
- A memory naming a file path: check the file exists
- A memory naming a function or flag: grep for it
- A memory summarizing repo state: prefer `git log` or reading code for current state
- "The memory says X exists" is not the same as "X exists now"

The drift caveat (`MEMORY_DRIFT_CAVEAT`) explicitly instructs the model to verify
stale memories against current state before asserting them as fact — and to update
or remove stale memories rather than acting on them.

---

## 8. Team Memory Sync

> **Source:** `src/services/teamMemorySync/index.ts`,
> `src/services/teamMemorySync/watcher.ts`,
> `src/services/teamMemorySync/secretScanner.ts`,
> `src/services/teamMemorySync/teamMemSecretGuard.ts`,
> `src/services/teamMemorySync/types.ts`,
> `src/memdir/teamMemPaths.ts`, `src/memdir/teamMemPrompts.ts`

### 8.1 Architecture

Team memory enables shared memories across team members working on the same repository.
It uses a `team/` subdirectory under the auto-memory path, synchronized to a
server-side API scoped by GitHub repository slug.

```
API contract:
  GET  /api/claude_code/team_memory?repo={owner/repo}             → full content + checksums
  GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes → metadata + checksums only
  PUT  /api/claude_code/team_memory?repo={owner/repo}             → upsert entries
```

Prerequisites: first-party OAuth with `claude_ai_inference` and `claude_ai_profile`
scopes, and a GitHub remote.

### 8.2 Sync Semantics

| Direction | Semantics |
|---|---|
| Pull | Server wins per-key — server content overwrites local files |
| Push | Local wins per-key — local edits overwrite server content for that key |
| Deletion | Does NOT propagate — deleting locally does not delete from server; next pull restores |

This asymmetry is intentional: pull-first ensures the user starts with the team's latest
state; local-wins-on-push ensures the actively editing user's work is never silently
discarded.

### 8.3 Delta Upload

Push uses **delta upload**: only keys whose local content hash (`sha256:<hex>`) differs
from `serverChecksums` are included in the PUT. The `serverChecksums` map is populated
from the server's `entryChecksums` on each pull and updated from local hashes on
successful push.

### 8.4 ETag-Based Conditional Requests

The `SyncState` object tracks `lastKnownChecksum` (the server's ETag). Pull sends
`If-None-Match` for conditional GET (304 = not modified). Push sends `If-Match` for
optimistic concurrency control (412 = conflict).

### 8.5 Conflict Resolution

On 412 Precondition Failed:

1. Probe `GET ?view=hashes` — lightweight endpoint returning only per-key checksums
   (no entry bodies)
2. Refresh `serverChecksums` from the probe response
3. Recompute delta — keys where a teammate's push matches our content are naturally
   excluded
4. Retry PUT with the tighter delta
5. Up to `MAX_CONFLICT_RETRIES = 2` retries

### 8.6 Batch Splitting

PUT bodies are capped at `MAX_PUT_BODY_BYTES = 200,000` to stay under the API gateway's
body-size limit. `batchDeltaByBytes()` greedily bin-packs entries into batches, sorted
by key for deterministic ordering. Each batch is an independent PUT with upsert
semantics — if batch N fails, batches 1..N-1 are already committed.

### 8.7 File Watcher

`startTeamMemoryWatcher()` at `src/services/teamMemorySync/watcher.ts`:

1. Performs an initial pull on startup (before the watcher starts, so its disk writes
   do not trigger a push)
2. Starts `fs.watch` with `recursive: true` on the team memory directory
3. On file change: debounces (2s), then pushes

The watcher uses `fs.watch` rather than chokidar because chokidar 4+ dropped fsevents,
and Bun's fallback uses kqueue which requires one open fd per file. With
`recursive: true`, macOS uses FSEvents (O(1) fds); Linux uses inotify (O(subdirs) fds).

**Push suppression:** After a permanent failure (no OAuth, 4xx except 409/429), the
watcher suppresses further pushes until a file deletion is detected (recovery action
for too-many-entries) or the session restarts. This prevents infinite retry loops from
watch events triggered by other sessions' writes to the shared team directory.

### 8.8 Secret Scanning

> **Source:** `src/services/teamMemorySync/secretScanner.ts`

Before upload, every team memory file is scanned for credentials using a curated subset
of high-confidence gitleaks patterns. Files containing detected secrets are **skipped**
(excluded from the upload set), never leaving the user's machine.

The scanner includes ~30 rules covering:
- Cloud providers (AWS, GCP, Azure, DigitalOcean)
- AI APIs (Anthropic, OpenAI, HuggingFace)
- Version control (GitHub PATs, GitLab tokens)
- Communication (Slack, Twilio, SendGrid)
- Dev tooling (npm, PyPI, Databricks, HashiCorp)
- Observability (Grafana, Sentry)
- Payment (Stripe, Shopify)
- Crypto (PEM private keys)

Rule patterns use distinctive prefixes with near-zero false-positive rates. Generic
keyword-context rules are omitted. The actual matched secret value is never logged —
only the rule ID.

`checkTeamMemSecrets()` in `teamMemSecretGuard.ts` provides a synchronous check called
from `FileWriteTool` and `FileEditTool` `validateInput` to prevent the model from
writing secrets into team memory files in the first place. This is a write-side guard
complementing the read-side scanner.

---

## 9. KAIROS Memory Model

> **Source:** `src/memdir/memdir.ts:327-370`

### 9.1 Daily-Log Pattern

KAIROS (autonomous assistant mode) sessions are effectively perpetual — they can run
for hours or days without restarting. The standard MEMORY.md-as-live-index pattern
generates too much memory traffic for these sessions. KAIROS replaces it with an
**append-only daily log** pattern:

```
memory/
├── MEMORY.md                     ← distilled index (maintained nightly)
├── logs/
│   └── 2026/
│       └── 03/
│           ├── 2026-03-30.md     ← timestamped bullets
│           └── 2026-03-31.md     ← today's log
└── topic files (distilled from logs)
```

### 9.2 Write Protocol

The agent appends timestamped bullets to `memory/logs/YYYY/MM/YYYY-MM-DD.md`. The
file and parent directories are created on first write if they do not exist. The log
is append-only — the agent does not rewrite or reorganize it.

### 9.3 Nightly Distillation

A separate nightly `/dream` skill distills daily logs into:
- Topic files (organized semantically by subject)
- An updated MEMORY.md index

MEMORY.md serves as the distilled read-only index, loaded into context automatically.
The agent reads it for orientation but records new information in today's log instead.

### 9.4 Date Rollover

The daily log path is described in the prompt as a **pattern** (`YYYY/MM/YYYY-MM-DD.md`)
rather than a literal path for today's date. This is because the prompt is cached by
`systemPromptSection('memory', ...)` and is NOT invalidated on date change. The model
derives the current date from the `date_change` attachment (appended at the tail on
midnight rollover) rather than the user-context message — the latter is intentionally
left stale to preserve the prompt cache prefix across midnight.

### 9.5 Interaction with Team Memory

KAIROS daily-log mode takes precedence over team memory sync. The append-only log
paradigm does not compose with team sync (which expects a shared MEMORY.md that both
sides read and write). When `getKairosActive()` is true and auto-memory is enabled,
`loadMemoryPrompt()` returns the daily-log prompt and skips the team memory path
entirely.

---

## 10. Security

### 10.1 Path Traversal Validation

> **Source:** `src/memdir/paths.ts:109-150`, `src/memdir/teamMemPaths.ts`

`validateMemoryPath()` rejects paths that would be dangerous as a read-allowlist root:

| Check | Attack Vector |
|---|---|
| `!isAbsolute` | `../foo` — interpreted relative to CWD |
| `length < 3` | `/` — root directory access |
| `/^[A-Za-z]:$/` | `C:\` — Windows drive root |
| `startsWith('\\\\')` or `startsWith('//')` | UNC paths — opaque network trust boundary |
| `includes('\0')` | Null bytes survive `normalize()`, can truncate in syscalls |
| NFC normalization | Applied via `.normalize('NFC')` on the final path |

For team memory, `validateTeamMemKey()` adds:
- Rejects URL-encoded traversals (`%2e%2e%2f` = `../`)
- Rejects NFKC-normalized Unicode traversals (fullwidth periods/slashes)
- Rejects backslashes (Windows separator as traversal vector)
- Rejects absolute path keys (`/etc/passwd`)
- **Symlink resolution**: `realpathDeepestExisting()` resolves symlinks on the deepest
  existing ancestor, detecting symlink-based escapes that `path.resolve()` alone cannot
  catch (PSR M22186)

### 10.2 Prompt Injection Filtering

`filterInjectedMemoryFiles()` at `src/utils/claudemd.ts:1142` scans memory files for
suspicious patterns before loading them into the prompt. This defends against
repositories that ship adversarial `.claude/CLAUDE.md` files attempting prompt injection.

### 10.3 Settings Source Restrictions

> **Source:** `src/memdir/paths.ts:173-178`

`projectSettings` (`.claude/settings.json` committed to the repo) is **deliberately
excluded** from `autoMemoryDirectory` overrides. Without this restriction, a malicious
repository could set `autoMemoryDirectory: "~/.ssh"` and gain silent write access to
sensitive directories via the filesystem.ts write carve-out (which fires when
`isAutoMemPath()` matches).

Only trusted settings sources are honored: `policySettings`, `flagSettings`,
`localSettings`, and `userSettings`.

### 10.4 NFC Unicode Normalization

All memory paths are normalized to NFC form (`.normalize('NFC')`) to prevent
normalization-dependent comparison mismatches. Different Unicode representations of
the same string (NFC vs NFD) would otherwise allow a path to pass the containment
check in one form while being written in another.

### 10.5 Team Memory Secret Guard

The team memory system has two layers of secret protection:

1. **Write-side guard** (`checkTeamMemSecrets`): called from `FileWriteTool` and
   `FileEditTool` `validateInput`, blocks the model from writing secrets into team
   memory files before they hit disk
2. **Upload-side scanner** (`scanForSecrets`): called during `readLocalTeamMemory`
   before upload, excludes files containing secrets from the upload payload

Both use the same curated gitleaks rule set. The secret value is never logged — only
the rule ID and a human-readable label.

### 10.6 Symlink-Based Escape Prevention

Team memory write paths undergo two-pass validation:

1. **String-level**: `resolve()` eliminates `..` segments and checks the result starts
   with the team directory prefix
2. **Filesystem-level**: `realpathDeepestExisting()` resolves symlinks on the deepest
   existing ancestor, then checks the real path is within the real team directory

This catches:
- Dangling symlinks (target does not exist — `lstat` distinguishes from truly
  non-existent paths)
- Symlink loops (`ELOOP` → `PathTraversalError`)
- Symlinks pointing outside the team directory

---

## 11. Activation and Gating

### 11.1 Auto-Memory Enable/Disable

> **Source:** `src/memdir/paths.ts:30-55`

`isAutoMemoryEnabled()` follows a priority chain:

| Priority | Check | Result |
|---|---|---|
| 1 | `CLAUDE_CODE_DISABLE_AUTO_MEMORY` = 1/true | OFF |
| 2 | `CLAUDE_CODE_DISABLE_AUTO_MEMORY` = 0/false | ON |
| 3 | `CLAUDE_CODE_SIMPLE` (--bare) | OFF |
| 4 | CCR without `CLAUDE_CODE_REMOTE_MEMORY_DIR` | OFF |
| 5 | `autoMemoryEnabled` in settings.json | Per setting |
| 6 | Default | ON |

### 11.2 Extract Mode Gating

`isExtractModeActive()` at `src/memdir/paths.ts:69`:

```
feature('EXTRACT_MEMORIES')     ← compile-time Bun flag (tree-shakes the code)
  AND tengu_passport_quail      ← GrowthBook runtime gate
  AND (interactive session OR tengu_slate_thimble)  ← non-interactive gate
```

Callers must also guard on `feature('EXTRACT_MEMORIES')` directly, because `feature()`
only tree-shakes when used in an `if` condition (not inside a helper function).

### 11.3 Compile-Time Feature Flags

| Flag | Feature |
|---|---|
| `EXTRACT_MEMORIES` | Background memory extraction agent |
| `KAIROS` | Assistant mode daily logs and /dream |
| `TEAMMEM` | Team memory sync, combined prompts, secret scanning |
| `MEMORY_SHAPE_TELEMETRY` | Memory recall shape analytics |

### 11.4 Runtime GrowthBook Gates

| Gate | Purpose |
|---|---|
| `tengu_passport_quail` | Extract mode activation |
| `tengu_slate_thimble` | Extract mode in non-interactive sessions |
| `tengu_moth_copse` | Skip MEMORY.md index in prompt |
| `tengu_herring_clock` | Team memory cohort tracking |
| `tengu_coral_fern` | "Searching past context" instructions |
| `tengu_amber_prism` | Memory correction hints on user rejection |
| `tengu_bramble_lintel` | Turn throttle for extraction (N turns between runs) |
| `tengu_session_memory` | Session memory feature gate |
| `tengu_sm_config` | Session memory remote configuration |

---

## 12. Integration with Context Management

### 12.1 Context Assembly

Memory content enters the system prompt during context assembly
(Phase 1 of the context lifecycle):

```
System Prompt (dynamic, per-session)
  ├── Git status snapshot
  ├── Current date
  ├── CLAUDE.md hierarchy
  ├── ► Auto-memory MEMORY.md index (<=200 lines, <=25KB)
  ├── ► Memory behavioral instructions (taxonomy, save/recall guidance)
  ├── ► Team memory MEMORY.md index (when enabled)
  ├── MCP server instructions
  └── Skill listings
```

`loadMemoryPrompt()` at `src/memdir/memdir.ts:419` dispatches based on enabled systems:
1. KAIROS active → `buildAssistantDailyLogPrompt()`
2. Team memory enabled → `teamMemPrompts.buildCombinedMemoryPrompt()`
3. Auto-memory only → `buildMemoryLines()` joined into a string

### 12.2 Post-Compact Restoration

When context compaction fires (Phase 3), memories are not directly re-injected because
they are part of the system prompt, which survives compaction. However, the session
memory file is used to improve compaction quality:

1. Session memory content is read via `getSessionMemoryContent()`
2. It is truncated per-section via `truncateSessionMemoryForCompact()`
3. It is injected into the compaction summary to preserve session state across
   the compaction boundary

The auto-memory MEMORY.md index remains in the system prompt before and after
compaction — it is not part of the message array that gets summarized.

### 12.3 Memory Recall as Context Growth

When `findRelevantMemories()` surfaces topic files for a user query, those files are
read and injected as user-message attachments. This is Phase 2 (context growth) — the
memory content becomes part of the conversation messages, subject to summarization
during future compaction.

The staleness annotations from `memoryAge.ts` help the model treat recalled memories
appropriately: fresh memories (today/yesterday) are trusted directly; older memories
get a caveat instructing verification against current state.

### 12.4 Memory vs. Plans vs. Tasks

The system prompt explicitly distinguishes memory from other persistence mechanisms:

- **Memory**: cross-session, recalled in future conversations
- **Plans**: within-session, alignment tool for non-trivial implementation tasks
- **Tasks**: within-session, discrete steps or progress tracking for current work

The prompt instructs: if you are about to start a non-trivial implementation task, use
a Plan. If you need to break work into discrete steps, use Tasks. Memory should be
reserved for information useful in future conversations.

---

## 13. Telemetry and Observability

### 13.1 Memory Directory Events

| Event | Data |
|---|---|
| `tengu_memdir_loaded` | File count, subdir count, content size, memory type |
| `tengu_memdir_disabled` | Disabled by env var or setting |
| `tengu_team_memdir_disabled` | Team memory specifically disabled |

### 13.2 Extraction Events

| Event | Data |
|---|---|
| `tengu_extract_memories_extraction` | Token usage, message count, turn count, files written, duration |
| `tengu_extract_memories_skipped_direct_write` | Main agent already wrote memories |
| `tengu_extract_memories_coalesced` | Stashed context during in-progress extraction |
| `tengu_extract_memories_gate_disabled` | Gate check failed (logged once per session) |
| `tengu_extract_memories_error` | Extraction failed with error |
| `tengu_auto_mem_tool_denied` | Extraction agent attempted a denied tool |

### 13.3 Session Memory Events

| Event | Data |
|---|---|
| `tengu_session_memory_init` | Auto-compact enabled status |
| `tengu_session_memory_gate_disabled` | Gate check failed |
| `tengu_session_memory_extraction` | Token usage, config values |
| `tengu_session_memory_file_read` | Content length |
| `tengu_session_memory_loaded` | Content length (on read) |

### 13.4 Team Memory Events

| Event | Data |
|---|---|
| `tengu_team_mem_sync_started` | Initial pull success, files pulled, server content status |
| `tengu_team_mem_sync_pull` | Files written, not-modified, duration, error type |
| `tengu_team_mem_sync_push` | Files uploaded, conflict status, retries, batches, duration |
| `tengu_team_mem_secret_skipped` | File count, rule IDs |
| `tengu_team_mem_push_suppressed` | Suppression reason, HTTP status |
| `tengu_team_mem_entries_capped` | Total entries, dropped count, max entries |

---

## 14. Design Principles

### 14.1 Derivability Test

The central heuristic: **if it can be derived from reading the current project state,
do not save it as a memory**. Code patterns, architecture, and git history change with
each commit — a memory of them becomes stale immediately. User preferences, behavioral
corrections, and project context cannot be derived from any file and remain valuable
across sessions.

### 14.2 Background, Not Blocking

Memory extraction runs as a fire-and-forget background agent. The user never waits for
memories to be written. The forked agent pattern enables this cheaply — prompt cache
sharing means the extraction API call is mostly cache reads, not new input tokens.

### 14.3 Write-Once Correctness

The two-step save protocol (write topic file, then add index entry) and the mutual
exclusion between the main agent and extraction agent ensure that concurrent memory
operations do not corrupt the index. The main agent writing directly and the extraction
agent writing in the background are never both active for the same turn range.

### 14.4 Structured Forgetting Across Sessions

Not everything from a session should be persisted. The extraction agent applies the
taxonomy (section 5) as a filter: only information matching one of the four types
passes through. The negative space (what NOT to save) is as important as the positive
space — without it, the memory directory would fill with ephemeral code state that
becomes stale within hours.

### 14.5 Defense in Depth for Memory Paths

Memory file paths are validated at multiple layers:
- Path traversal checks (`validateMemoryPath`, `validateTeamMemKey`)
- Symlink resolution (`realpathDeepestExisting`)
- Settings source restriction (project settings excluded)
- NFC Unicode normalization
- Null byte rejection
- UNC path rejection

No single validation failure can compromise the memory system's security boundary.

### 14.6 Server Wins on Pull, Local Wins on Push

Team memory sync's asymmetric conflict resolution reflects the reality of team
collaboration: when you start a session, you want the latest team state (server wins on
pull). When you are actively editing, your local work should not be silently discarded
just because a teammate pushed in the meantime (local wins on push). Content-level merge
of the same key is not attempted — the simpler last-writer-wins model is sufficient for
the memory use case, where files are small and topic-scoped.

### 14.7 Graceful Degradation

Every memory subsystem degrades gracefully:
- Auto-memory disabled → main agent still works, just without persistent context
- Extraction agent fails → the main agent can still write memories directly
- Team sync fails → local memory still works, push retries on next edit
- Session memory fails → compaction falls back to pure summarization
- MEMORY.md exceeds caps → truncated with a warning, not silently dropped
- Memory scan fails → empty array returned, recall skipped silently

### 14.8 Cache-First Architecture

Every design decision considers prompt cache impact:
- The extraction agent shares the parent's cache via `CacheSafeParams`
- REPL is allowed in the extraction agent specifically to preserve cache sharing
  (different tool lists break the cache key)
- KAIROS daily-log paths are described as patterns (not literals) because the prompt
  is cached by `systemPromptSection` and not invalidated on date change
- `getAutoMemPath()` is memoized to avoid expensive settings lookups on render paths

### 14.9 Secrets Never Leave the Machine

Team memory's secret scanning is a one-way gate: secrets are detected and blocked before
upload, but the secret values themselves are never logged, reported, or transmitted.
Only the gitleaks rule ID (e.g., `github-pat`) appears in analytics — never the matched
text, file path, or surrounding context.
