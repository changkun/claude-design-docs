# Specification Index

19 design specs reverse-engineering the architecture of Claude Code,
organized into four tiers by architectural significance.

**Codebase scale:** ~1,900 files, 512k+ lines of TypeScript.
**Runtime:** Bun | **UI:** React + Ink | **Protocols:** MCP, LSP

---

## Reading Orders

### Newcomer Track (start here)

Read these 5 docs to understand the core execution model:

1. **[Query Loop](core/query-loop.md)** — The central `while(true)` state machine that drives every conversation turn.
2. **[Tool System](core/tool-system.md)** — The ~40 tools the model can invoke, how they are registered, validated, and executed.
3. **[Oversight / Safety](core/oversight.md)** — The 14-layer Swiss cheese safety model.
4. **[Context Management](core/context-management.md)** — How context is assembled, grows, compresses, and persists.
5. **[Command System](systems/command-system.md)** — Slash commands, skills, and the user-facing extension surface.

### Security Track

1. [Oversight / Safety](core/oversight.md)
2. [Permission Modes](systems/permission-modes.md)
3. [Bash Scanner](infra/bash-scanner.md)
4. [CLAUDE.md Hierarchy](infra/claudemd-hierarchy.md)
5. [Auto-Memory](infra/auto-memory.md)

### Extensibility Track

1. [Tool System](core/tool-system.md)
2. [Plugin & Skill System](supporting/plugin-skill.md)
3. [MCP Integration](supporting/mcp-integration.md)
4. [Command System](systems/command-system.md)
5. [Feature Flags](infra/feature-flags.md)

### Infrastructure Track

1. [Service Layer](systems/service-layer.md)
2. [Bridge System](systems/bridge-system.md)
3. [Analytics & Telemetry](supporting/analytics-telemetry.md)
4. [Prompt Cache](infra/prompt-cache.md)
5. [Streaming Executor](supporting/streaming-executor.md)

### Agent & Autonomy Track

1. [Agent Orchestration](core/agent-orchestration.md)
2. [KAIROS Autonomous Mode](systems/kairos.md)
3. [Auto-Memory](infra/auto-memory.md)
4. [Context Management](core/context-management.md)

---

## Full Spec List

### Tier 1 — Core Architecture

| # | Spec | Scope |
|---|------|-------|
| 1 | [query-loop](core/query-loop.md) | Agentic query loop, state machine, 7 error recovery paths |
| 2 | [tool-system](core/tool-system.md) | Tool interface, registry, 45 tools, MCP composition |
| 3 | [oversight](core/oversight.md) | 14-layer Swiss cheese safety model |
| 4 | [context-management](core/context-management.md) | Context assembly, compression, memory extraction |
| 5 | [agent-orchestration](core/agent-orchestration.md) | Multi-agent orchestration, 5 patterns, swarms |

### Tier 2 — Major Systems

| # | Spec | Scope |
|---|------|-------|
| 6 | [kairos](systems/kairos.md) | Autonomous assistant mode, perpetual sessions |
| 7 | [bridge-system](systems/bridge-system.md) | IDE/remote integration, 2 architectures, transports |
| 8 | [command-system](systems/command-system.md) | 100+ slash commands, 3 execution models, skills |
| 9 | [service-layer](systems/service-layer.md) | 21 backend services (API, MCP, OAuth, LSP, analytics) |
| 10 | [permission-modes](systems/permission-modes.md) | 6 permission modes, AI classifier, denial limits |

### Tier 3 — Infrastructure

| # | Spec | Scope |
|---|------|-------|
| 11 | [auto-memory](infra/auto-memory.md) | Persistent memory, team sync, KAIROS logs, extraction |
| 12 | [prompt-cache](infra/prompt-cache.md) | Static/dynamic boundary, fork sharing, cache detection |
| 13 | [bash-scanner](infra/bash-scanner.md) | 23 injection checks, tree-sitter AST, Zsh attacks |
| 14 | [claudemd-hierarchy](infra/claudemd-hierarchy.md) | 4-level instruction hierarchy, @path syntax, rules |
| 15 | [feature-flags](infra/feature-flags.md) | Compile-time + runtime flags, GrowthBook, kill switches |

### Tier 4 — Supporting Design

| # | Spec | Scope |
|---|------|-------|
| 16 | [plugin-skill](supporting/plugin-skill.md) | Plugin lifecycle, skill frontmatter, SkillTool |
| 17 | [mcp-integration](supporting/mcp-integration.md) | 5 transports, OAuth/XAA auth, tool wrapping |
| 18 | [streaming-executor](supporting/streaming-executor.md) | Parallel execution, 26 hooks, concurrency, result budget |
| 19 | [analytics-telemetry](supporting/analytics-telemetry.md) | 3 sinks, Datadog, GrowthBook, PII handling, cost tracking |

---

## Tier Summaries

### 1. Query Loop State Machine
`src/query.ts` · `src/QueryEngine.ts` · `src/context.ts`

Single `while(true)` loop with 7 continuation sites implementing the
full agentic cycle: context assembly, LLM streaming, tool execution,
attachment collection. All error recovery (prompt-too-long, max output
tokens, model fallback) is handled via state mutation + `continue`,
eliminating recursive calls. Generator-based (`AsyncGenerator<Message>`)
for unified sync/async consumption.

### 2. 14-Layer Oversight System (Swiss Cheese Model)
`src/utils/permissions/` · `src/tools/BashTool/`

Fourteen independent safety layers where no two adjacent layers share
the same weakness: system prompt instructions, Zod input validation,
rule-based permission hierarchy (8 sources), tool-specific checks,
bash security scanner (23 injection categories via tree-sitter AST),
dangerous pattern stripping, OS-level sandbox (seatbelt/bubblewrap),
mode transformation, hooks (26 types), 2-stage AI classifier, denial
limit tracking, human-in-the-loop 5-way race, decision audit log
(3 sinks), and query pipeline safety.

### 3. Context Management Lifecycle
`src/services/compact/`

Four-phase system: assembly (static/dynamic prompt boundary for cache
efficiency), growth (message accumulation with token tracking),
compression (session-memory compact + LLM summarization + reactive
emergency compact + snip), and memory extraction (background agent
distills durable learnings). Post-compact restoration strategically
re-injects top-5 recent files (50k tokens), active skills (25k), and
plan state. Circuit breaker stops after 3 consecutive failures.

### 4. Multi-Agent Orchestration
`src/tools/AgentTool/`

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

### 6. KAIROS Autonomous Mode
`src/assistant/`

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

### 11. Auto-Memory System
`src/services/extractMemories/` · `src/memdir/`

Background extraction agent (forked, shares parent prompt cache) distills
session context into persistent `MEMORY.md` (200-line, 25KB cap) plus
topic files. Four-type taxonomy (user, feedback, project, reference).
Path traversal validation, prompt injection filtering, NFC Unicode
normalization. KAIROS extends with append-only daily logs + nightly
distillation.

### 12. Prompt Cache Architecture
`src/constants/prompts.ts` · `src/context.ts`

System prompt split at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`: static region
(tool descriptions, safety instructions) cached across API calls,
dynamic region (git status, CLAUDE.md, memory, MCP instructions) changes
per-session. Fork agents construct byte-identical prefixes. One-shot
agents omit trailers. Parallel prefetch of MDM settings and keychain
on startup (~65ms savings).

### 13. Bash Security Scanner
`src/tools/BashTool/`

23 injection check categories using tree-sitter WASM AST analysis with
regex fallback. Covers: command substitution, shell metacharacter
injection, obfuscated flags, parser differentials, variable injection,
IFS injection, brace expansion, Unicode whitespace, Zsh-specific attacks
(zmodload, emulate, sysopen), comment-quote desynchronization, quoted
newline hiding, /proc/environ access, and control characters.

### 14. CLAUDE.md Hierarchy
`src/utils/claudeMd.ts`

Four-level priority: managed (enterprise `/etc/claude-code/`) < user
(`~/.claude/`) < project (`.claude/` + `.claude/rules/`) < local
(`CLAUDE.local.md`, gitignored). `@path` syntax for composing from
multiple sources. Circular reference prevention. Injection filtering.

### 15. Feature Flag System
`src/services/analytics/growthbook.ts`

Bun's `feature()` compile-time dead-code elimination. Notable gates:
PROACTIVE, KAIROS, BRIDGE_MODE, DAEMON, VOICE_MODE, AGENT_TRIGGERS,
COORDINATOR_MODE, WORKFLOW_SCRIPTS. GrowthBook runtime gates for
progressive rollout with instant server-side kill switch.

### 16. Plugin & Skill System
`src/services/plugins/` · `src/skills/`

Plugin loader with manifest validation and repository integration.
Skill directory (`~/.claude/skills/`) with auto-discovery. Dynamic
skill detection during file operations. Skills can specify `paths`
globs for applicability and constrain `allowedTools`.

### 17. MCP Protocol Integration
`src/services/mcp/` (25 files, 944KB)

Full Model Context Protocol client: 5 transports, authentication (XAA),
connection lifecycle management, resource enumeration, tool wrapping.
Scoped cleanup (inline clients cleaned up; name-ref clients persist).
Dynamic tool pool composition with built-in tool deduplication.

### 18. Streaming Tool Executor
`src/services/tools/`

Parallel tool execution with streaming output. Permission checking,
progress tracking, error recovery. Hook system with 26 hook types and
6 implementation types (shell, LLM, agent, HTTP, callback, function).

### 19. Analytics & Telemetry
`src/services/analytics/`

Datadog metrics, first-party event logging with PII tagging, OpenTelemetry
structured spans. AI classifier logs ~40 telemetry fields per decision.
9 distinct audit event types for permission decisions.
