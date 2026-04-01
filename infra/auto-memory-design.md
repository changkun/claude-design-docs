# Auto-Memory — Design Document

#### 1. Overview

Claude Code sessions are ephemeral by default. When a conversation ends, its context window evaporates. The auto-memory system maintains a persistent, file-based memory layer that bridges sessions: learnings extracted from one conversation become available context in every subsequent conversation for the same project.

The system answers a precise question: **what from this session would be valuable in a future session that cannot be derived from reading the current project state?** Code patterns, architecture, and git history are rederivable from the filesystem; user preferences, behavioral corrections, project context, and external-system pointers are not. The memory system captures the latter and discards the former.

Three complementary subsystems cooperate:

1. **Auto-memory (persistent cross-session)** -- a file-based memory directory with a MEMORY.md index always loaded into the system prompt, topic files read on demand, and a background extraction agent that distills durable learnings after each model response.

2. **Session memory** -- a structured markdown file summarizing the current session's state, periodically updated by a forked subagent, used to improve context compaction quality.

3. **Team memory sync** -- a shared `team/` subdirectory synchronized to a server API, enabling memories to propagate across team members working on the same repository.

These layers are additive. Auto-memory is the foundation; session memory improves within-session continuity after compaction; team memory extends the persistence boundary from individual to organizational.

---

#### 2. Memory Directory Structure

##### 2.1 Directory Layout

Each project gets a memory directory under the user's configuration home:

```
~/.claude/projects/<sanitized-project-root>/memory/
├── MEMORY.md                        <-- index file (always in system prompt)
├── user_role.md                     <-- topic file (user type)
├── feedback_testing.md              <-- topic file (feedback type)
├── project_auth_rewrite.md          <-- topic file (project type)
├── reference_dashboards.md          <-- topic file (reference type)
├── logs/                            <-- KAIROS daily logs (when active)
│   └── 2026/
│       └── 03/
│           ├── 2026-03-30.md
│           └── 2026-03-31.md
└── team/                            <-- shared team memory (when enabled)
    ├── MEMORY.md                    <-- team index
    ├── feedback_db_tests.md
    └── reference_linear.md
```

##### 2.2 Path Resolution

The memory directory is resolved through a three-level priority chain:

| Priority | Source | Notes |
|---|---|---|
| 1 | Environment variable override | Full-path override for SDK/Cowork integrations |
| 2 | Settings configuration | Trusted sources only (policy, flag, local, user) |
| 3 | Default computation | `<memoryBase>/projects/<sanitized-git-root>/memory/` |

The default path uses a memory base directory (configurable via environment, falling back to `~/.claude`) joined with a sanitized project root.

**Worktree sharing:** Git worktrees are resolved to their canonical root, so all worktrees of the same repository share one memory directory. This prevents memory fragmentation.

**Memoization:** The path resolution is memoized by project root because the computation involves multiple settings lookups with filesystem operations, which is too expensive for render-path callers.

##### 2.3 Directory Existence Guarantee

The directory tree is created idempotently before the prompt is built. The system prompt includes guidance that the directory already exists, eliminating a common failure mode where the model wastes a turn checking for or creating the directory.

---

#### 3. The MEMORY.md Contract

MEMORY.md is the entrypoint file -- the only memory artifact guaranteed to be in the system prompt for every API call.

##### 3.1 Role

MEMORY.md is an **index, not a memory**. Each entry is a one-line pointer to a topic file:

```markdown
- [User role](user_role.md) -- data scientist, focused on observability
- [Testing policy](feedback_testing.md) -- integration tests must use real DB
- [Auth rewrite context](project_auth_rewrite.md) -- legal-driven, compliance priority
```

The model reads MEMORY.md to orient itself at the start of a session, then reads individual topic files on demand when they seem relevant. This two-level indirection keeps the always-loaded index small while supporting an unbounded number of memories.

##### 3.2 Size Caps

Two caps prevent MEMORY.md from bloating the system prompt:

| Cap | Value |
|---|---|
| Line cap | 200 lines |
| Byte cap | 25,000 bytes |

**Truncation logic:**

1. Line-truncate first -- a natural boundary that preserves complete index entries
2. If the result still exceeds the byte cap, byte-truncate at the last newline before the limit (never cuts mid-line)
3. Append a warning naming which cap fired

The byte cap targets a specific failure mode observed in production: indexes with fewer than 200 lines but extremely long lines (worst case observed: 197KB under 200 lines). The 25KB limit (~125 chars/line at 200 lines) catches this at the 97th percentile.

##### 3.3 Index Skip Mode

An experimental mode (feature-gated) where MEMORY.md is not maintained as an index. The extraction agent and system prompt omit the two-step save process (write topic file, then add index entry) and instead instruct the model to write standalone topic files with no index pointer. This simplifies the write protocol at the cost of losing the always-loaded orientation index.

---

#### 4. Background Extraction Agent

The extraction agent is the most novel component: a **background subagent** that runs asynchronously after each model response, analyzing the session for durable learnings worth persisting to disk.

##### 4.1 Execution Model

**Trigger:** The extractor fires when the model stops (no pending tool calls), invoked as a post-sampling hook. It runs fire-and-forget -- it does not block the response to the user.

**Forked agent pattern:** The extraction agent is a **perfect fork** of the main conversation. This is architecturally significant: the fork shares the parent's prompt cache (system prompt + message prefix), so the API call's input tokens are almost entirely cache reads. The extraction is cheap because it reuses cached context rather than recomputing it.

##### 4.2 Turn Throttling

The extractor does not run on every turn. A configurable gate (default 1) controls how many eligible turns must pass between extractions. This reduces API costs during rapid tool-call sequences where no durable learning is likely.

##### 4.3 Overlap Guard and Trailing Runs

Only one extraction runs at a time. If a new hook fires during an in-progress extraction:

1. The new context is stashed
2. When the current extraction finishes, it checks for a stashed context
3. If present, a **trailing run** executes with the latest stashed context
4. The trailing run's message count is computed relative to the just-advanced cursor, so it only processes messages added between the two calls

This coalescing ensures no messages are missed while preventing unbounded concurrency.

##### 4.4 Main Agent Mutual Exclusion

The main agent's system prompt always includes full memory-save instructions. When the user says "remember this" or the model spontaneously decides to save, the main agent writes directly to the memory directory. The extraction agent detects this by scanning for Write/Edit tool uses targeting memory paths.

When detected, the extractor **skips** and advances the cursor past the main agent's writes. This makes the two agents mutually exclusive per turn range.

##### 4.5 Tool Permissions

The extraction agent has a strict permission model:

| Tool | Permission |
|---|---|
| Read, Grep, Glob | Unrestricted (inherently read-only) |
| Bash | Read-only commands only |
| Edit, Write | **Only within the auto-memory directory** |
| REPL | Allowed (inner tool calls are re-gated) |
| All other tools | Denied |

Bash `rm` is explicitly denied -- the extraction agent cannot delete memory files. MCP tools, Agent tools, and write-capable Bash are all blocked.

The REPL allowance is architecturally motivated: in certain tool configuration modes, primitive tools are hidden from the tool list and the model calls REPL instead. REPL's VM context re-invokes the permission check for each inner primitive. Giving the fork a different tool list would **break prompt cache sharing** -- tools are part of the cache key.

##### 4.6 Pre-injected Memory Manifest

Before invoking the forked agent, the extractor scans the memory directory and formats a manifest including file names, types, timestamps, and frontmatter descriptions. This is injected into the extraction prompt so the agent does not waste a turn listing directory contents.

##### 4.7 Turn Budget

The forked agent has a hard cap of 5 turns. The prompt instructs an efficient strategy: turn 1 -- issue all Read calls in parallel for files to update; turn 2 -- issue all Write/Edit calls in parallel. Well-behaved extractions complete in 2-4 turns. The hard cap prevents verification rabbit-holes from burning excessive API calls.

##### 4.8 Drain on Shutdown

Pending extractions are awaited with a soft timeout (default 60s) after the response is flushed but before process shutdown, ensuring the forked agent completes before the shutdown failsafe kills the process.

##### 4.9 System Message Notification

When the extractor writes memory files, it injects a notification into the main conversation so the user sees feedback (e.g., "Saved 2 memories") without the extraction blocking the primary response.

---

#### 5. Memory Taxonomy

##### 5.1 The Four Types

Memories are constrained to a **closed four-type taxonomy** capturing context that is NOT derivable from the current project state:

| Type | Description | Scope (team mode) |
|---|---|---|
| `user` | User's role, goals, preferences, knowledge | Always private |
| `feedback` | Behavioral corrections and confirmed approaches | Default private; team for project-wide conventions |
| `project` | Work context, deadlines, decisions, incidents | Bias toward team |
| `reference` | Pointers to external systems (dashboards, trackers) | Usually team |

##### 5.2 What NOT to Save

The taxonomy's negative space is explicitly defined:

- Code patterns, conventions, architecture, file paths, project structure -- derivable from reading the current project state
- Git history, recent changes, who-changed-what -- git log / git blame are authoritative
- Debugging solutions or fix recipes -- the fix is in the code; the commit message has the context
- Anything already documented in CLAUDE.md files
- Ephemeral task details: in-progress work, temporary state, current conversation context

These exclusions apply **even when the user explicitly asks**. If a user asks to save a PR list or activity summary, the model is instructed to ask what was surprising or non-obvious -- that is the part worth keeping.

##### 5.3 Frontmatter-Based Structured Storage

Each topic file uses YAML frontmatter with name, description, and type fields. The `description` field is critical for recall: the relevance selector uses descriptions to decide which memories to surface. The `type` field enables filtering and scoping logic.

For `feedback` and `project` types, the body is structured as: rule/fact, then **Why:** and **How to apply:** lines. Knowing why lets the model judge edge cases instead of blindly following the rule.

##### 5.4 Individual vs. Combined Mode

Two variants exist:

- **Individual mode** -- single directory, no scope tags, no team/private qualifiers
- **Combined mode** -- private + team directories, per-type scope guidance embedded in XML-style type blocks

The combined variant includes explicit scope routing baked into the type blocks.

---

#### 6. Session Memory

Session memory is complementary to auto-memory: where auto-memory captures durable cross-session learnings, session memory captures the **current session's state** in a structured markdown file. Its primary purpose is improving the quality of context compaction.

##### 6.1 The Session Memory File

A structured markdown file with predefined sections:

- Session Title
- Current State
- Task specification
- Files and Functions
- Workflow
- Errors & Corrections
- Codebase and System Documentation
- Learnings
- Key results
- Worklog

The template and update prompt are customizable via configuration files.

##### 6.2 Extraction Triggers

Session memory extraction is threshold-gated:

| Threshold | Default |
|---|---|
| Initialization threshold | 10,000 tokens |
| Update interval | 5,000 tokens growth |
| Tool call minimum | 3 tool calls |

Extraction triggers when:
- Both the token threshold AND tool call threshold are met, OR
- The token threshold is met AND the last assistant turn has no tool calls (natural conversation break)

The token threshold is **always required**.

##### 6.3 Extraction Mechanics

Session memory extraction uses the same forked agent pattern as auto-memory extraction. The forked agent is restricted to file edit operations on exactly the session memory file path -- no other tools are permitted.

##### 6.4 Section Size Management

Each section is capped at approximately 2,000 tokens. The total file is capped at 12,000 tokens. When the total budget is exceeded, the update prompt includes a critical instruction to aggressively condense.

##### 6.5 Compaction Integration

Session memory's primary consumer is the compaction system. When auto-compact fires, the session memory content provides structured notes that help the summarization model produce a higher-quality summary. Oversized sections are truncated before insertion.

##### 6.6 Sequential Execution

The extraction hook uses a generic sequential wrapper ensuring at most one invocation runs at a time.

---

#### 7. Memory Recall

##### 7.1 On-Demand Relevance Selection

When a user query arrives, the system selects up to 5 relevant memory files to surface. This is a two-stage process:

**Stage 1: Scan:**
- Read all `.md` files in the memory directory (excluding MEMORY.md)
- Parse the first 30 lines of each for frontmatter (description, type)
- Sort by modification time (newest first)
- Cap at 200 files

**Stage 2: Select:**
- Send the query + memory manifest to a lightweight model side-query
- Use structured JSON output for selected memory paths
- Filter recently-used tools to suppress API documentation noise
- Filter already-surfaced paths to avoid redundant selections
- Return absolute file paths + modification time

##### 7.2 Memory Age and Staleness

The system provides freshness annotations:
- Human-readable age: "today", "yesterday", "47 days ago"
- Staleness caveat for memories older than 1 day
- System reminder wrapper for the caveat

##### 7.3 Recall-Side Guidance

The system prompt includes two sections governing how recalled memories are used:

**"When to access memories":**
- Access when memories seem relevant or user references prior work
- MUST access when the user explicitly asks
- If the user says to ignore memory: proceed as if MEMORY.md were empty

**"Before recommending from memory":**
- A memory naming a file path: check the file exists
- A memory naming a function or flag: grep for it
- A memory summarizing repo state: prefer git log or reading code for current state
- "The memory says X exists" is not the same as "X exists now"

The drift caveat instructs the model to verify stale memories against current state and to update or remove stale memories rather than acting on them.

---

#### 8. Team Memory Sync

##### 8.1 Architecture

Team memory enables shared memories across team members working on the same repository. It uses a `team/` subdirectory under the auto-memory path, synchronized to a server-side API scoped by GitHub repository slug.

```
API contract:
  GET  ?repo={owner/repo}             -> full content + checksums
  GET  ?repo={owner/repo}&view=hashes -> metadata + checksums only
  PUT  ?repo={owner/repo}             -> upsert entries
```

Prerequisites: first-party OAuth with inference and profile scopes, and a GitHub remote.

##### 8.2 Sync Semantics

| Direction | Semantics |
|---|---|
| Pull | Server wins per-key -- server content overwrites local files |
| Push | Local wins per-key -- local edits overwrite server content for that key |
| Deletion | Does NOT propagate -- deleting locally does not delete from server; next pull restores |

This asymmetry is intentional: pull-first ensures the user starts with the team's latest state; local-wins-on-push ensures the actively editing user's work is never silently discarded.

##### 8.3 Delta Upload

Push uses **delta upload**: only keys whose local content hash differs from known server checksums are included. Server checksums are populated from the server's response on each pull and updated from local hashes on successful push.

##### 8.4 ETag-Based Conditional Requests

Pull sends `If-None-Match` for conditional GET (304 = not modified). Push sends `If-Match` for optimistic concurrency control (412 = conflict).

##### 8.5 Conflict Resolution

On 412 Precondition Failed:

1. Probe the lightweight hashes-only endpoint
2. Refresh server checksums from the probe response
3. Recompute delta -- keys where a teammate's push matches our content are naturally excluded
4. Retry PUT with the tighter delta
5. Up to 2 retries

##### 8.6 Batch Splitting

PUT bodies are capped at 200,000 bytes to stay under the API gateway's body-size limit. Entries are greedily bin-packed into batches, sorted by key for deterministic ordering. Each batch is an independent PUT with upsert semantics -- if batch N fails, batches 1..N-1 are already committed.

##### 8.7 File Watcher

The file watcher:

1. Performs an initial pull on startup (before the watcher starts, so its disk writes do not trigger a push)
2. Starts a recursive filesystem watch on the team memory directory
3. On file change: debounces (2s), then pushes

The watcher uses `fs.watch` rather than chokidar because chokidar 4+ dropped fsevents, and Bun's fallback uses kqueue which requires one open fd per file. With `recursive: true`, macOS uses FSEvents (O(1) fds); Linux uses inotify (O(subdirs) fds).

**Push suppression:** After a permanent failure, the watcher suppresses further pushes until a file deletion is detected (recovery action for too-many-entries) or the session restarts.

##### 8.8 Secret Scanning

Before upload, every team memory file is scanned for credentials using a curated subset of high-confidence gitleaks patterns. Files containing detected secrets are **skipped** (excluded from the upload set), never leaving the user's machine.

The scanner covers approximately 30 rules across cloud providers, AI APIs, version control tokens, communication platforms, dev tooling, observability, payment, and cryptographic keys. Rule patterns use distinctive prefixes with near-zero false-positive rates. Generic keyword-context rules are omitted. The actual matched secret value is never logged -- only the rule ID.

A synchronous write-side guard prevents the model from writing secrets into team memory files in the first place. This is a write-side guard complementing the read-side scanner.

---

#### 9. KAIROS Memory Model

##### 9.1 Daily-Log Pattern

KAIROS (autonomous assistant mode) sessions are effectively perpetual. The standard MEMORY.md-as-live-index pattern generates too much memory traffic for these sessions. KAIROS replaces it with an **append-only daily log** pattern:

```
memory/
├── MEMORY.md                     <-- distilled index (maintained nightly)
├── logs/
│   └── 2026/
│       └── 03/
│           ├── 2026-03-30.md     <-- timestamped bullets
│           └── 2026-03-31.md     <-- today's log
└── topic files (distilled from logs)
```

##### 9.2 Write Protocol

The agent appends timestamped bullets to a daily log file. The file and parent directories are created on first write. The log is append-only -- the agent does not rewrite or reorganize it.

##### 9.3 Nightly Distillation

A separate nightly process distills daily logs into topic files (organized semantically) and an updated MEMORY.md index. MEMORY.md serves as the distilled read-only index, loaded into context automatically. The agent reads it for orientation but records new information in today's log.

##### 9.4 Date Rollover

The daily log path is described in the prompt as a **pattern** rather than a literal path for today's date. This is because the prompt is cached and not invalidated on date change. The model derives the current date from a date-change attachment rather than the cached user-context message -- the latter is intentionally left stale to preserve the prompt cache prefix across midnight.

##### 9.5 Interaction with Team Memory

KAIROS daily-log mode takes precedence over team memory sync. The append-only log paradigm does not compose with team sync (which expects a shared MEMORY.md that both sides read and write). When KAIROS is active, the memory prompt returns the daily-log prompt and skips the team memory path entirely.

---

#### 10. Security

##### 10.1 Path Traversal Validation

Memory paths are validated against multiple attack vectors:

| Check | Attack Vector |
|---|---|
| Non-absolute path rejection | `../foo` -- interpreted relative to CWD |
| Short path rejection | `/` -- root directory access |
| Drive root rejection | `C:\` -- Windows drive root |
| UNC path rejection | Network paths -- opaque trust boundary |
| Null byte rejection | Null bytes survive normalize(), can truncate in syscalls |
| NFC normalization | Applied on the final path |

For team memory keys, additional checks include:
- URL-encoded traversals (`%2e%2e%2f`)
- NFKC-normalized Unicode traversals (fullwidth periods/slashes)
- Backslashes (Windows separator as traversal vector)
- Absolute path keys
- **Symlink resolution**: resolves symlinks on the deepest existing ancestor, detecting symlink-based escapes that string-level resolution alone cannot catch

##### 10.2 Prompt Injection Filtering

Memory files are scanned for suspicious patterns before loading them into the prompt. This defends against repositories shipping adversarial configuration files attempting prompt injection.

##### 10.3 Settings Source Restrictions

Project-level settings (committed to the repo) are **deliberately excluded** from memory directory overrides. Without this restriction, a malicious repository could set the memory directory to a sensitive path (e.g., `~/.ssh`) and gain silent write access via the filesystem write carve-out.

Only trusted settings sources are honored: policy, flag, local, and user settings.

##### 10.4 NFC Unicode Normalization

All memory paths are normalized to NFC form to prevent normalization-dependent comparison mismatches.

##### 10.5 Team Memory Secret Guard

Two layers of secret protection:

1. **Write-side guard**: blocks the model from writing secrets into team memory files before they hit disk
2. **Upload-side scanner**: excludes files containing secrets from the upload payload

Both use the same curated rule set. The secret value is never logged.

##### 10.6 Symlink-Based Escape Prevention

Team memory write paths undergo two-pass validation:

1. **String-level**: resolve() eliminates `..` segments and checks the result starts with the team directory prefix
2. **Filesystem-level**: resolves symlinks on the deepest existing ancestor, then checks the real path is within the real team directory

This catches dangling symlinks, symlink loops, and symlinks pointing outside the team directory.

---

#### 11. Activation and Gating

##### 11.1 Auto-Memory Enable/Disable

The enable/disable check follows a priority chain:

| Priority | Check | Result |
|---|---|---|
| 1 | Explicit disable env var = true | OFF |
| 2 | Explicit disable env var = false | ON |
| 3 | Bare/simple mode | OFF |
| 4 | Remote execution without remote memory dir | OFF |
| 5 | Settings configuration | Per setting |
| 6 | Default | ON |

##### 11.2 Extract Mode Gating

Extraction requires all of:
- A compile-time feature flag (tree-shakes the code)
- A runtime gate
- Either an interactive session or a non-interactive gate override

Callers must also guard on the compile-time flag directly, because the flag only tree-shakes when used in an `if` condition (not inside a helper function).

##### 11.3 Compile-Time Feature Flags

| Flag | Feature |
|---|---|
| Extract memories | Background memory extraction agent |
| KAIROS | Assistant mode daily logs and nightly distillation |
| Team memory | Team memory sync, combined prompts, secret scanning |
| Memory shape telemetry | Memory recall shape analytics |

##### 11.4 Runtime Gates

| Purpose |
|---|
| Extract mode activation |
| Extract mode in non-interactive sessions |
| Skip MEMORY.md index in prompt |
| Team memory cohort tracking |
| "Searching past context" instructions |
| Memory correction hints on user rejection |
| Turn throttle for extraction |
| Session memory feature gate |
| Session memory remote configuration |

---

#### 12. Integration with Context Management

##### 12.1 Context Assembly

Memory content enters the system prompt during context assembly (Phase 1 of the context lifecycle):

```
System Prompt (dynamic, per-session)
  ├── Git status snapshot
  ├── Current date
  ├── CLAUDE.md hierarchy
  ├── Auto-memory MEMORY.md index (<=200 lines, <=25KB)
  ├── Memory behavioral instructions (taxonomy, save/recall guidance)
  ├── Team memory MEMORY.md index (when enabled)
  ├── MCP server instructions
  └── Skill listings
```

The memory prompt dispatches based on enabled systems:
1. KAIROS active -> daily-log prompt
2. Team memory enabled -> combined memory prompt
3. Auto-memory only -> memory lines joined into a string

##### 12.2 Post-Compact Restoration

When context compaction fires, memories are not directly re-injected because they are part of the system prompt, which survives compaction. However, the session memory file is used to improve compaction quality by providing structured notes to the summarization model.

The auto-memory MEMORY.md index remains in the system prompt before and after compaction -- it is not part of the message array that gets summarized.

##### 12.3 Memory Recall as Context Growth

When relevant memories are surfaced for a user query, those files are read and injected as user-message attachments. This is context growth -- the memory content becomes part of the conversation messages, subject to summarization during future compaction.

Staleness annotations help the model treat recalled memories appropriately: fresh memories are trusted directly; older memories get a caveat instructing verification.

##### 12.4 Memory vs. Plans vs. Tasks

The system prompt explicitly distinguishes memory from other persistence mechanisms:

- **Memory**: cross-session, recalled in future conversations
- **Plans**: within-session, alignment tool for non-trivial implementation tasks
- **Tasks**: within-session, discrete steps or progress tracking for current work

---

#### 13. Telemetry and Observability

The system emits structured telemetry events across four categories:

- **Memory directory events**: file count, content size, disabled status
- **Extraction events**: token usage, message count, turn count, files written, duration, skip/coalesce/error conditions
- **Session memory events**: initialization, gate status, extraction metrics, file reads
- **Team memory events**: sync start/pull/push, secret scan results, push suppression, entry capping

---

#### 14. Design Principles

1. **Derivability Test**: If it can be derived from the current project state, do not save it. User preferences and behavioral corrections cannot be derived and remain valuable.

2. **Background, Not Blocking**: Memory extraction runs fire-and-forget. The forked agent pattern makes this cheap via prompt cache sharing.

3. **Write-Once Correctness**: The two-step save protocol and mutual exclusion between the main agent and extraction agent ensure concurrent memory operations do not corrupt the index.

4. **Structured Forgetting**: Not everything from a session should be persisted. The taxonomy acts as a filter. The negative space is as important as the positive space.

5. **Defense in Depth for Memory Paths**: Multiple validation layers ensure no single failure compromises the security boundary.

6. **Server Wins on Pull, Local Wins on Push**: Reflects team collaboration reality. Content-level merge is not attempted -- last-writer-wins is sufficient for small, topic-scoped files.

7. **Graceful Degradation**: Every subsystem degrades gracefully. Failures in one layer do not cascade.

8. **Cache-First Architecture**: Every design decision considers prompt cache impact. The extraction agent shares the parent's cache. Tool lists are preserved to avoid breaking cache keys. Path patterns (not literals) are used in cached prompts.

9. **Secrets Never Leave the Machine**: Secret scanning is a one-way gate. Secret values are never logged, reported, or transmitted. Only rule IDs appear in analytics.
