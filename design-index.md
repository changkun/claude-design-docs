# Design Index

An index of the most valuable components, systems, and designs in this
Claude Code archive, sorted by impact and architectural significance.

**Codebase scale:** ~1,900 files, 512k+ lines of TypeScript.
**Runtime:** Bun | **UI:** React + Ink | **Protocols:** MCP, LSP

---

## Tier 1 — Core Architecture

### 1. Query Loop State Machine
`src/query.ts` · `src/QueryEngine.ts` · `src/context.ts`

Single `while(true)` loop with 7 continuation sites implementing the
full agentic cycle: context assembly, LLM streaming, tool execution,
attachment collection. All error recovery (prompt-too-long, max output
tokens, model fallback) is handled via state mutation + `continue`,
eliminating recursive calls. Generator-based (`AsyncGenerator<Message>`)
for unified sync/async consumption.

### 2. 14-Layer Oversight System (Swiss Cheese Model)
`oversight-design.md` · `src/utils/permissions/` · `src/tools/BashTool/`

Fourteen independent safety layers where no two adjacent layers share
the same weakness: system prompt instructions, Zod input validation,
rule-based permission hierarchy (8 sources), tool-specific checks,
bash security scanner (23 injection categories via tree-sitter AST),
dangerous pattern stripping, OS-level sandbox (seatbelt/bubblewrap),
mode transformation, hooks (26 types), 2-stage AI classifier, denial
limit tracking, human-in-the-loop 5-way race, decision audit log
(3 sinks), and query pipeline safety.

### 3. Context Management Lifecycle
`context-management-design.md` · `src/services/compact/`

Four-phase system: assembly (static/dynamic prompt boundary for cache
efficiency), growth (message accumulation with token tracking),
compression (session-memory compact + LLM summarization + reactive
emergency compact + snip), and memory extraction (background agent
distills durable learnings). Post-compact restoration strategically
re-injects top-5 recent files (50k tokens), active skills (25k), and
plan state. Circuit breaker stops after 3 consecutive failures.

### 4. Multi-Agent Orchestration
`agent-orchestration-spec.md` · `src/tools/AgentTool/`

Five patterns: sub-agent, coordinator mode, team swarms, fork
sub-agents, and one-shot agents. AsyncLocalStorage for context
isolation without process overhead. File-based mailboxes with
`proper-lockfile` for swarm communication. Fork agents share
byte-identical API prefixes for prompt cache maximization. Mid-execution
foreground-to-background transitions without restart.

### 5. Tool System
`src/tools.ts` · `src/Tool.ts` · `src/tools/` (45 subdirectories)

~40 built-in tools across 8 categories: file system (read/edit/write/
glob/grep/notebook/LSP), execution (bash/powershell), agent & task
management, planning & navigation, MCP/skill integration, web fetch/
search, communication, and configuration. Each tool implements a
contract with Zod schema validation, permission checks, React UI
rendering, and concurrency safety declarations. Deferred loading via
`ToolSearchTool` for large tool sets.

---

## Tier 2 — Major Systems

### 6. KAIROS Autonomous Mode
`kairos-design.md` · `src/assistant/`

Transforms CLI from interactive to self-directed. Perpetual sessions
surviving restarts via `bridge-pointer.json`. `SendUserMessage` tool
for async output. Append-only daily log memory with nightly `/dream`
distillation. Pre-seeded team spawning. Multi-gate activation pipeline
(build-time, settings, trust dialog, entitlement). Channel notifications
via MCP (Slack, Discord). Scheduled tasks via cron.

### 7. Bridge System (IDE + Remote)
`src/bridge/` (33 files)

Multi-protocol framework connecting CLI to IDEs and web sessions.
Two cores: environment-based (polling + work-dispatch) and
environment-less (direct session-ingress). v1 WebSocket and v2
SSE+CCR transport abstraction. UUID ring-buffer deduplication.
Multi-session management with capacity waking, worktree isolation,
and adaptive backoff with sleep detection. JWT token refresh scheduling.

### 8. Command System
`src/commands.ts` · `src/commands/` (103 subdirectories)

100+ commands in 3 execution models: `prompt` (sends content to LLM
with constrained tool sets), `local` (executes without model), and
`local-jsx` (interactive React/Ink UI). Multi-layer loading: bundled
skills, plugins, user skills directory, workflows, built-in commands.
Feature-gated, lazy-loaded, with remote/bridge safety classification.

### 9. Service Layer
`src/services/` (21 modules, 130 files)

API client (streaming, retry, prompt cache break detection), MCP
management (5 transports: SSE, HTTP, WebSocket, STDIO, in-process),
OAuth 2.0, LSP integration, analytics (Datadog + GrowthBook feature
flags), plugin system, context compaction, policy limits, remote
managed settings, memory extraction, token estimation, team memory
sync with secret scanning, and prompt suggestions.

### 10. Permission Mode System
`src/utils/permissions/`

Six modes: `default` (prompt for sensitive ops), `plan` (show intent),
`acceptEdits` (auto-allow edits in CWD), `dontAsk` (convert asks to
denies), `auto` (2-stage AI classifier replacing human), and
`bypassPermissions` (kill-switch gated). Auto mode strips dangerous
allow rules on entry and restores on exit. Denial limits (3 consecutive
/ 20 total) force fallback to human review.

---

## Tier 3 — Infrastructure & UI

### 11. Terminal UI Component Library
`src/components/` (346 TSX files, 31 categories)

React 19 + Ink framework. Design system with themed primitives
(`ThemedBox`, `ThemedText`, `Pane`, `Dialog`, `Tabs`, `FuzzyPicker`).
Permission request UIs for 12+ tool types. Message rendering for 31+
message variants (assistant text/tool-use/thinking, user prompt/bash/
image/channel, system errors). Custom multi-select, agent editor, MCP
settings, diff viewer, and syntax highlighting.

### 12. Auto-Memory System
`src/services/extractMemories/` · `src/memdir/`

Background extraction agent (forked, shares parent prompt cache) distills
session context into persistent `MEMORY.md` (200-line, 25KB cap) plus
topic files. Four-type taxonomy (user, feedback, project, reference).
Path traversal validation, prompt injection filtering, NFC Unicode
normalization. KAIROS extends with append-only daily logs + nightly
distillation.

### 13. Prompt Cache Architecture
`src/constants/prompts.ts` · `src/context.ts`

System prompt split at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`: static region
(tool descriptions, safety instructions) cached across API calls,
dynamic region (git status, CLAUDE.md, memory, MCP instructions) changes
per-session. Fork agents construct byte-identical prefixes. One-shot
agents omit trailers. Parallel prefetch of MDM settings and keychain
on startup (~65ms savings).

### 14. Bash Security Scanner
`src/tools/BashTool/`

23 injection check categories using tree-sitter WASM AST analysis with
regex fallback. Covers: command substitution, shell metacharacter
injection, obfuscated flags, parser differentials, variable injection,
IFS injection, brace expansion, Unicode whitespace, Zsh-specific attacks
(zmodload, emulate, sysopen), comment-quote desynchronization, quoted
newline hiding, /proc/environ access, and control characters.

### 15. CLAUDE.md Hierarchy
`src/utils/claudeMd.ts`

Four-level priority: managed (enterprise `/etc/claude-code/`) < user
(`~/.claude/`) < project (`.claude/` + `.claude/rules/`) < local
(`CLAUDE.local.md`, gitignored). `@path` syntax for composing from
multiple sources. Circular reference prevention. Injection filtering.

### 16. Feature Flag System
`src/services/analytics/growthbook.ts`

Bun's `feature()` compile-time dead-code elimination. Notable gates:
PROACTIVE, KAIROS, BRIDGE_MODE, DAEMON, VOICE_MODE, AGENT_TRIGGERS,
COORDINATOR_MODE, WORKFLOW_SCRIPTS. GrowthBook runtime gates for
progressive rollout with instant server-side kill switch.

---

## Tier 4 — Supporting Design

### 17. Plugin & Skill System
`src/services/plugins/` · `src/skills/`

Plugin loader with manifest validation and repository integration.
Skill directory (`~/.claude/skills/`) with auto-discovery. Dynamic
skill detection during file operations. Skills can specify `paths`
globs for applicability and constrain `allowedTools`.

### 18. MCP Protocol Integration
`src/services/mcp/` (25 files, 944KB)

Full Model Context Protocol client: 5 transports, authentication (XAA),
connection lifecycle management, resource enumeration, tool wrapping.
Scoped cleanup (inline clients cleaned up; name-ref clients persist).
Dynamic tool pool composition with built-in tool deduplication.

### 19. Streaming Tool Executor
`src/services/tools/`

Parallel tool execution with streaming output. Permission checking,
progress tracking, error recovery. Hook system with 26 hook types and
6 implementation types (shell, LLM, agent, HTTP, callback, function).

### 20. Analytics & Telemetry
`src/services/analytics/`

Datadog metrics, first-party event logging with PII tagging, OpenTelemetry
structured spans. AI classifier logs ~40 telemetry fields per decision.
9 distinct audit event types for permission decisions.

---

## Companion Materials

### Copyleft Essay
`2026-03-09-is-legal-the-same-as-legitimate-ai-reimplementation-and-the-erosion-of-copyleft.md`

Analysis of chardet relicensing dispute. Core argument: legal
permissibility and social legitimacy are distinct registers. Proposes
"specification copyleft" covering test suites and API specs as AI
makes source-level reimplementation frictionless.

### OmX Workflow Assets
`assets/omx/`

Screenshots documenting the oh-my-codex multi-agent workflow used to
produce this archive's README and documentation.
