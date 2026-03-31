# Specification Index

19 design specs reverse-engineering the architecture of Claude Code.

## Reading Orders

### Newcomer Track (start here)

Read these 5 docs to understand the core execution model:

1. **[Query Loop](query-loop-design.md)** — The central `while(true)` state machine that drives every conversation turn. Start here to understand how everything connects.
2. **[Tool System](tool-system-design.md)** — The ~40 tools the model can invoke, how they are registered, validated, and executed.
3. **[Oversight / Safety](oversight-design.md)** — The 14-layer Swiss cheese safety model. Essential context for understanding why every other system has permission checks.
4. **[Context Management](context-management-design.md)** — How context is assembled, grows, compresses, and persists across sessions.
5. **[Command System](command-system-design.md)** — Slash commands, skills, and the user-facing extension surface.

### Security Track

For understanding the safety and permission architecture:

1. [Oversight / Safety](oversight-design.md)
2. [Permission Modes](permission-modes-design.md)
3. [Bash Scanner](bash-scanner-design.md)
4. [CLAUDE.md Hierarchy](claudemd-hierarchy-design.md) (section 11: Security)
5. [Auto-Memory](auto-memory-design.md) (section 10: Security)

### Extensibility Track

For understanding how Claude Code is extended:

1. [Tool System](tool-system-design.md)
2. [Plugin & Skill System](plugin-skill-design.md)
3. [MCP Integration](mcp-integration-design.md)
4. [Command System](command-system-design.md)
5. [Feature Flags](feature-flags-design.md)

### Infrastructure Track

For understanding the runtime and deployment machinery:

1. [Service Layer](service-layer-design.md)
2. [Bridge System](bridge-system-design.md)
3. [Analytics & Telemetry](analytics-telemetry-design.md)
4. [Prompt Cache](prompt-cache-design.md)
5. [Streaming Executor](streaming-executor-design.md)

### Agent & Autonomy Track

For understanding multi-agent and autonomous operation:

1. [Agent Orchestration](agent-orchestration-spec.md)
2. [KAIROS Autonomous Mode](kairos-design.md)
3. [Auto-Memory](auto-memory-design.md)
4. [Context Management](context-management-design.md)

---

## Full Spec List

### Tier 1 — Core Architecture

| # | Spec | Scope |
|---|------|-------|
| 1 | [query-loop-design.md](query-loop-design.md) | Agentic query loop, state machine, 7 error recovery paths |
| 2 | [oversight-design.md](oversight-design.md) | 14-layer Swiss cheese safety model |
| 3 | [context-management-design.md](context-management-design.md) | Context assembly, compression, memory extraction |
| 4 | [agent-orchestration-spec.md](agent-orchestration-spec.md) | Multi-agent orchestration, 5 patterns, swarms |
| 5 | [tool-system-design.md](tool-system-design.md) | Tool interface, registry, 45 tools, MCP composition |

### Tier 2 — Major Systems

| # | Spec | Scope |
|---|------|-------|
| 6 | [kairos-design.md](kairos-design.md) | Autonomous assistant mode, perpetual sessions |
| 7 | [bridge-system-design.md](bridge-system-design.md) | IDE/remote integration, 2 architectures, transports |
| 8 | [command-system-design.md](command-system-design.md) | 100+ slash commands, 3 execution models, skills |
| 9 | [service-layer-design.md](service-layer-design.md) | 21 backend services (API, MCP, OAuth, LSP, analytics) |
| 10 | [permission-modes-design.md](permission-modes-design.md) | 6 permission modes, AI classifier, denial limits |

### Tier 3 — Infrastructure

| # | Spec | Scope |
|---|------|-------|
| 11 | [auto-memory-design.md](auto-memory-design.md) | Persistent memory, team sync, KAIROS logs, extraction |
| 12 | [prompt-cache-design.md](prompt-cache-design.md) | Static/dynamic boundary, fork sharing, cache detection |
| 13 | [bash-scanner-design.md](bash-scanner-design.md) | 23 injection checks, tree-sitter AST, Zsh attacks |
| 14 | [claudemd-hierarchy-design.md](claudemd-hierarchy-design.md) | 4-level instruction hierarchy, @path syntax, rules |
| 15 | [feature-flags-design.md](feature-flags-design.md) | Compile-time + runtime flags, GrowthBook, kill switches |

### Tier 4 — Supporting Design

| # | Spec | Scope |
|---|------|-------|
| 16 | [plugin-skill-design.md](plugin-skill-design.md) | Plugin lifecycle, skill frontmatter, SkillTool |
| 17 | [mcp-integration-design.md](mcp-integration-design.md) | 5 transports, OAuth/XAA auth, tool wrapping |
| 18 | [streaming-executor-design.md](streaming-executor-design.md) | Parallel execution, 26 hooks, concurrency, result budget |
| 19 | [analytics-telemetry-design.md](analytics-telemetry-design.md) | 3 sinks, Datadog, GrowthBook, PII handling, cost tracking |
