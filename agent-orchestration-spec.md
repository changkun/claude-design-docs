# Agent Orchestration Architecture Spec

Architecture-level specification of Claude Code's multi-agent orchestration system.
Covers agent lifecycle, orchestration patterns, communication protocols, agent definitions, resource management, and the design decisions behind each subsystem.

## Table of Contents

1. [Overview & Design Principles](#1-overview--design-principles)
2. [Core Abstractions](#2-core-abstractions)
3. [Agent Lifecycle](#3-agent-lifecycle)
4. [Orchestration Patterns](#4-orchestration-patterns)
5. [Communication Protocols](#5-communication-protocols)
6. [Agent Definition System](#6-agent-definition-system)
7. [Resource Management](#7-resource-management)
8. [State Management](#8-state-management)
9. [Resilience & Fault Tolerance](#9-resilience--fault-tolerance)
10. [Key Source File Index](#10-key-source-file-index)

---

## 1. Overview & Design Principles

Claude Code's orchestration system manages multiple concurrent agents with isolated contexts, independent tool pools, and multiple communication channels. The system supports five distinct orchestration patterns ranging from simple parent-child subagents to full multi-agent swarms.

### Design Principles

**Generator-based streaming.** Agent execution uses async generators that yield `Message` objects progressively. This unifies sync and async execution paths under a single consumption model—the same generator serves both a blocking foreground caller and a fire-and-forget background lifecycle.

**Context isolation via AsyncLocalStorage.** When agents are backgrounded (Ctrl+B), multiple agents run concurrently in the same Node.js process. A single global state shared by all agents would cause collisions: analytics events from Agent A would carry Agent B's context, file reads under one agent's ID would appear in another's transcript. AsyncLocalStorage provides per-execution-chain isolation without requiring process-level separation. Each concurrent agent gets its own async context; parent, Agent A, and Agent B never interfere. The trade-off is that every spawn/resume boundary must explicitly set context via `runWithAgentContext()`—miss the boundary and events leak to the parent context.

**Prompt cache optimization.** Fork subagents construct byte-identical API prefixes across all children so they share prompt cache entries. One-shot built-in agents (Explore, Plan) omit non-essential message trailers to save tokens. At scale (34M+ weekly Explore spawns), these optimizations compound into significant token savings.

**Tool pool independence.** Each agent receives its own computed tool pool based on its definition, permission mode, and execution context (sync vs. async, built-in vs. custom). Trust decreases with distance from the parent: built-in agents get wider access, custom agents are cautious, async agents are isolated.

**File-based coordination.** Multi-agent swarms use file-based mailboxes with `proper-lockfile` for concurrent access, enabling cross-process communication without shared memory. File-based mailboxes were chosen over IPC, shared memory, or network sockets because they work uniformly across all three execution backends (in-process, tmux, iTerm2) without requiring daemons, port allocation, or special cleanup.

**Graceful lifecycle transitions.** Foreground agents can transition to background mid-execution via a race between the message stream and a background signal. The agent generator continues without restart—only the communication channel changes, preserving prompt cache stability.

**Isolation over concurrency.** The system accepts that concurrent agents cannot collaborate on the same file in real-time, but gains simplicity, memory safety, and cache-sharing benefits. This reflects a deliberate choice: optimize for isolated background work and sequential tool use, not real-time collaborative editing.

---

## 2. Core Abstractions

### 2.1 AgentDefinition

Union type representing all possible agent sources. Three variants share a common base:

- **BuiltInAgentDefinition** — Programmatic, with dynamic system prompts keyed to runtime context (feature flags, user type, available tools). Built-in agents are trusted, maintained by the platform team, and can react to build-time and runtime state.
- **CustomAgentDefinition** — User/project/policy-defined, with static system prompts loaded from markdown files. Treated as black boxes—their prompt text is fixed at load time for predictability.
- **PluginAgentDefinition** — Registered by plugins, following the same format as custom agents but namespaced to avoid collisions.

Key fields on the base type:

| Field | Purpose |
|-------|---------|
| `agentType` | Unique identifier, used for deduplication and routing |
| `whenToUse` | Description shown to the LLM for tool selection |
| `tools` / `disallowedTools` | Allowlist/denylist defining the agent's tool pool |
| `mcpServers` | MCP servers this agent needs (name refs or inline defs) |
| `model` | Model override (`"inherit"`, `"sonnet"`, `"opus"`, `"haiku"`) |
| `permissionMode` | Permission mode override for the agent's execution |
| `memory` | Persistence scope: `"user"`, `"project"`, or `"local"` |
| `isolation` | Execution isolation: `"worktree"` or `"remote"` |
| `omitClaudeMd` | Skip CLAUDE.md injection (token optimization for read-only agents) |
| `hooks` | Session-scoped lifecycle hooks registered when agent starts |

Defined in: `src/tools/AgentTool/loadAgentsDir.ts`

### 2.2 AgentToolInput / AgentToolResult

The AgentTool (externally named "Task" for backward compatibility) accepts a `prompt`, optional `subagent_type`, and execution parameters (`run_in_background`, `model`, `isolation`, `name`/`team_name` for swarms). It returns one of four statuses: `completed` (sync result), `async_launched` (background), `teammate_spawned` (swarm), or `remote_launched` (CCR).

### 2.3 AgentContext (AsyncLocalStorage)

Two context variants exist:

- **SubagentContext** — For agents spawned via AgentTool. Carries `agentId`, `parentSessionId`, and whether the agent is built-in.
- **TeammateAgentContext** — For swarm members. Adds `agentName`, `teamName`, `isTeamLead`, `planModeRequired`, and team-specific routing metadata.

Both are set via `runWithAgentContext()` at spawn boundaries and accessed via `AsyncLocalStorage.getStore()` anywhere in the async execution chain.

Defined in: `src/utils/agentContext.ts`

### 2.4 Message

The fundamental unit flowing through agent generators. All agent execution produces a stream of `Message` objects consumed by the parent. Key subtypes include `AssistantMessage` (tool use blocks, thinking, text), `UserMessage`, progress events, and system boundary markers.

---

## 3. Agent Lifecycle

All agents follow a four-phase lifecycle: **Spawn → Init → Run → Cleanup**. The init phase branches into sync or async paths, and sync agents can transition to async mid-execution.

### 3.1 Spawn Phase

Entry point: `AgentTool.call()` in `src/tools/AgentTool/AgentTool.tsx`.

1. **Validate constraints.** Check multi-agent feature gates. Reject recursive fork attempts.
2. **Route multi-agent.** If `name` + `team_name` are provided, delegate to `spawnTeammate()` and return early.
3. **Resolve agent type.** If `subagent_type` is omitted and fork feature is enabled, take the fork path. Otherwise resolve from the active agents list.
4. **Validate agent.** Check `allowedAgentTypes` from permission specs. Verify `requiredMcpServers` are available (poll up to 30s).
5. **Assemble tool pool.** Compute the agent's independent tool set based on its definition and execution context.
6. **Create agent ID.** Stable ID for tracking and transcript correlation.
7. **Set up isolation.** Optionally create a git worktree.
8. **Build system prompt.** Fork path: inherit parent's rendered prompt verbatim. Normal path: call the agent's `getSystemPrompt()`.

### 3.2 Init & Branch

#### Async Path (Background)

Register the task in AppState, register the name→agentId mapping for routing, fire-and-forget the async lifecycle, and return `async_launched` immediately.

#### Sync Path (Foreground)

Register as foreground, create the generator iterator, and enter a race loop: each `iterator.next()` is raced against a `backgroundSignal`. After 2 seconds, a background hint is shown to the user. If the user backgrounds the agent mid-run, the generator transparently transitions to the async lifecycle without restart.

**Design decision — auto-background.** The 2-second threshold solves a UX problem: short commands complete before the user notices, while long-running agents don't block the prompt. A configurable auto-background timer (default 2 minutes) can forcibly transition foreground agents that run too long. The transition preserves the generator's state and prompt cache—no work is lost or restarted.

### 3.3 Run Phase

Core execution via the `runAgent()` async generator in `src/tools/AgentTool/runAgent.ts`.

1. Initialize agent MCP servers (connect new or reuse shared).
2. Register session hooks from the agent definition.
3. Load agent memory if configured.
4. Enter the query loop: call the API repeatedly, execute tools, yield each `Message`.
5. On completion or abort, unregister hooks and close agent-specific MCP connections.

### 3.4 Cleanup Phase

**Ordering invariant:** The status transition (mark completed) happens BEFORE post-completion embellishments (handoff classification, worktree cleanup). This ensures `TaskOutputTool` consumers are unblocked even if git operations or the transcript classifier hang. Embellishments are best-effort and non-blocking.

Cleanup covers:
- Stop summarization if active.
- Finalize result (extract text, compute usage metrics).
- Mark completed in AppState.
- Post-completion: handoff classification, worktree cleanup (parallel, non-blocking).
- Enqueue notification with final message, usage stats, and worktree info.
- On error: extract partial results (walks messages backward), notify with partial progress.
- Final: clear invoked skills, dump state, release cloned file state cache.

### Lifecycle Diagram

```
AgentTool.call()
  |
  +-- validate constraints
  +-- resolve agent type + definition
  +-- assemble tool pool
  +-- build system prompt
  |
  +-- async? ----yes----> registerAsyncAgent()
  |                          |
  |                     runAsyncAgentLifecycle()
  |                          |
  |                     runAgent() [generator]
  |                          |
  |                     mark completed (FIRST)
  |                          |
  |                     embellishments + notify (THEN)
  |
  +-- sync -----> registerAgentForeground()
                     |
                race loop: next() vs backgroundSignal
                     |
                +-- backgrounded? --> transition to async lifecycle (no restart)
                |
                +-- completed? --> finalizeAgentTool() -> return result
```

---

## 4. Orchestration Patterns

### 4.1 Sub-Agent Pattern

The basic parent-child relationship. The parent spawns a child agent via `AgentTool`, the child runs to completion, and the parent receives the result.

**Characteristics:**
- Child gets its own system prompt and independently computed tool pool.
- Sync execution blocks the parent's turn; async returns immediately.
- One-shot agents (Explore, Plan) skip the agentId/SendMessage trailer to save ~135 bytes per invocation.

**Design rationale — no recursive spawning by default.** Custom agents cannot spawn sub-agents (AgentTool is in `CUSTOM_AGENT_DISALLOWED_TOOLS`). This prevents a user-defined agent from creating unbounded nesting. Built-in agents can recurse because they are vetted, narrow-scoped code maintained by the platform team. The `Agent(type1,type2)` permission spec allows controlled recursion where needed.

**Built-in sub-agent types:**

| Agent | Tools | Purpose |
|-------|-------|---------|
| `general-purpose` | All (`*`) | Multi-step tasks |
| `Explore` | Read-only (Glob, Grep, Read, WebFetch, WebSearch) | Codebase exploration. One-shot. |
| `Plan` | Read-only | Architecture planning. One-shot. |
| `claude-code-guide` | Read-only + docs | Documentation lookup. Resumable. |
| `verification` | Configurable | Post-implementation checks |
| `statusline-setup` | Read, Edit only | Statusline configuration |

### 4.2 Coordinator Mode

An environment-level mode that restructures the main session into a coordinator that delegates to worker agents.

**Activation:** `CLAUDE_CODE_COORDINATOR_MODE` environment variable + feature gate.

**Why an environment variable, not a tool or agent type?** Coordinator mode is not a capability the main agent invokes; it is a complete restructuring of the session's conversation model. It changes what system prompt the main agent receives, what tools are available, and how the session reasons about its role. An environment variable makes this a bootstrap decision—a session-level invariant that persists across resumes and cannot be toggled mid-conversation. Tools are runtime features; coordinator mode is infrastructure.

**The "synthesis over delegation" principle.** The coordinator system prompt enforces a critical rule: the coordinator must read and understand worker findings before issuing the next instruction. This prevents the "lazy delegation" anti-pattern where a coordinator relays worker output without understanding it, leading to compounding misalignment. Each hand-off that skips synthesis loses context; the coordinator is the intelligence center that sees the full picture across multiple workers.

**Coordinator-available tools:** AgentTool, SendMessageTool, TaskStopTool, TeamCreateTool, TeamDeleteTool. The coordinator deliberately lacks file manipulation tools—it directs workers rather than doing the work itself.

**Summarization.** Coordinators need visibility into long-running parallel workers. A periodic summarization fork (every 30s) generates 3-5 word progress descriptions ("Reading auth.ts", "Running tests") by forking the worker's conversation. The fork shares the worker's prompt cache (identical prefix), making it cheap. Regular sub-agents don't need this because the user watches foreground output directly.

Defined in: `src/coordinator/coordinatorMode.ts`

### 4.3 Team Swarms

A peer-based multi-agent pattern where a team lead manages multiple teammates that communicate directly with each other.

**When to use swarms vs. coordinator vs. sub-agents.** Sub-agents are for delegated sub-tasks with parent control. Coordinator mode is for structured research→synthesis→implementation→verification workflows. Swarms are for collaborative work where agents have overlapping responsibilities and need coordination negotiation. The three levels compose: a coordinator can spawn workers (sub-agents), and a sub-agent can spawn a swarm within its scope.

**Team creation flow:**
1. `TeamCreateTool` creates a team file at `~/.claude/teams/{team_name}/config.json`.
2. Generates deterministic `leadAgentId` via `formatAgentId("team-lead", teamName)`.
3. Sets up a task list directory for the team.
4. Updates AppState with `teamContext`.

#### Execution Backends

Three backends hide different process models behind a common `TeammateExecutor` interface (`spawn`, `sendMessage`, `terminate`, `kill`, `isActive`):

| Backend | Process Model | Visualization | Trade-offs |
|---------|--------------|---------------|------------|
| `in-process` | Same Node.js process, AsyncLocalStorage isolation | None (headless) | Minimal overhead, shared resources, no visual isolation |
| `tmux` | Separate tmux panes | Native pane borders, live scrollback | Shell startup overhead (~200ms), works with existing tmux sessions |
| `iterm2` | iTerm2 split panes via AppleScript | Native macOS integration | macOS-only, requires `it2` CLI |

**Detection priority:** tmux (if inside tmux) → iTerm2 (if available with `it2`) → external tmux session (if tmux installed) → error. This ensures no nested multiplexing (tmux inside iTerm2 is confusing) and provides a clear fallback chain.

**Why file-based mailboxes?** IPC requires server setup per process. Shared memory breaks across process boundaries. Network sockets require port allocation and firewall handling. File-based mailboxes work uniformly across all backends: in-process agents write via Node.js fs, tmux/iTerm2 agents write from their shell. Messages survive temporary crashes (readable after reconnection), provide natural debugging checkpoints (`cat inbox.json`), and require no broker daemon. The trade-off is millisecond-level latency (vs. microsecond for in-memory queues), acceptable because swarm messages are paced by human and agent response times.

#### Deterministic Agent IDs

Format: `agentName@teamName` (e.g., `researcher@my-project`).

Deterministic IDs solve three coordination problems:
1. **Reconnection.** After a crash, restart with the same name → same ID → same mailbox → resume from stored messages.
2. **Leader discovery.** `team-lead@{teamName}` is always the lead. No lookup needed.
3. **Routing.** SendMessage computes the mailbox path directly from the name, without a registry lookup: `~/.claude/teams/{teamName}/inboxes/{agentName}.json`.

#### Leader-Follower Model

The team lead is a soft authority, not a dictator. The lead can redirect at the task level and halt execution (shutdown request), but cannot override implementation decisions. This design prevents coordination deadlocks: the lead's messages have priority over peer messages (no starvation behind peer chatter), shutdown is asymmetric (lead requests, teammate approves/rejects), and abort signals propagate unidirectionally (user → lead → teammates).

Pure peer-to-peer was rejected because without a leader, agents must reach consensus on state (are we done? can we shut down?), introducing deadlock risk and violating the non-negotiable requirement that user intent (via the lead) is never blocked by peer negotiation.

#### Shutdown Protocol

Graceful shutdown uses a request/response pattern rather than force-kill:
1. Lead sends `shutdown_request` to teammate's mailbox.
2. Teammate reads request, finishes current operation, responds with `shutdown_response` (approve or reject with reason).
3. On approval, teammate cleans up resources and exits.
4. Lead can escalate to force-kill if needed.

This prevents orphaned operations, partial file edits, and broken tool result chains. The request is non-blocking—the lead doesn't wait indefinitely and can force-kill a stuck teammate.

#### Plan Mode Requirement

Teammates can be spawned with `planModeRequired: true`, forcing them to show their implementation plan to the team lead before executing. This prevents parallel teammates from making divergent assumptions. Without it, 5 teammates might implement 5 different approaches simultaneously. With it, each teammate shows its plan, the lead coordinates, and only then implementation begins.

Key files: `src/tools/TeamCreateTool/TeamCreateTool.ts`, `src/utils/swarm/teamHelpers.ts`, `src/utils/swarm/backends/`

### 4.4 Fork Sub-Agents (Experimental)

Implicit subagent spawning that inherits the parent's full conversation context for maximum prompt cache sharing.

**The cache-sharing strategy.** The Anthropic API's cache key includes the entire message prefix. Fork children are constructed so that every child sends byte-identical messages up to a divergence point near the end. All children share one cached prefix; only the per-child directive (~100 tokens) is unique. If 5 agents fork in parallel, the cache is computed once and reused 5 times.

**How byte-identity is maintained:**
- The parent's rendered system prompt is passed verbatim (not recomputed, which could diverge due to feature flag state changes mid-session).
- All tool_result blocks use an identical placeholder string rather than real results. Real results would differ per attempt, busting the cache.
- Only the final text block (the per-child directive) diverges—this falls after the cache prefix is already computed.

**Why placeholders instead of real results?** Children inherit the full conversation context above the fork point and can re-read files or re-run commands if needed. The cache savings from identical placeholders outweigh the cost of occasional redundant reads.

**Context isolation for forks:**
- **State cloning.** `readFileState` is cloned so the fork sees everything the parent saw (same cache prefix) but subsequent reads don't pollute the parent's cache.
- **Mutation suppression.** All mutation callbacks (`setAppState`, `setResponseLength`) are no-op by default, preventing concurrent forks from corrupting parent UI state or metrics.
- **Task registration exception.** `setAppStateForTasks` always reaches the root store because background bash tasks spawned by forks must be tracked session-wide to prevent zombie processes (PPID=1 orphans).
- **Linked AbortControllers.** A new child controller is linked to the parent via `WeakRef`. Parent abort propagates to children, but not vice versa. The WeakRef prevents the parent from retaining garbage-collected children.

**Recursion guard.** Fork children detect themselves via a boilerplate tag in their message history and reject fork attempts. The AgentTool remains in the child's tool pool (for cache-identical tool definitions) but is blocked at call time.

Key files: `src/tools/AgentTool/forkSubagent.ts`, `src/utils/forkedAgent.ts`

### 4.5 Async Task Execution

Any agent can be launched as a background task. Additionally, foreground agents can transition to background mid-run.

**Two task systems coexist by design:**
- **Agent tasks** (internal execution tracking) — Track what the assistant is currently doing: background agents, shell commands, in-process teammates. These have lifecycle states (`running`, `completed`, `killed`, `failed`), progress tracking, and eviction timers.
- **User-facing tasks** (project management) — Persistent, user-created work items with `pending`/`in_progress`/`completed` status. These survive session resets and are stored as JSON files with file-level locking for multi-process access.

The separation prevents agent execution from polluting the user's task list. A background bash command is an agent task; "Implement feature X" is a user task.

**Task types in AppState:**

| Task Type | Purpose |
|-----------|---------|
| `local_agent` | Background subagent execution |
| `local_shell` | Background shell commands |
| `remote_agent` | Remote (CCR) execution |
| `in_process_teammate` | In-process swarm member |

**Progress tracking** uses a `ProgressTracker` with tool use counts, cumulative token metrics, and a circular buffer of the 5 most recent activities.

**Output retrieval** via `TaskOutputTool`: polls AppState every 100ms until the task completes or a configurable timeout (default 30s, max 600s). Extracts text from in-memory result or tails the disk output file.

**Disk output** supports three tiers: in-memory buffer (0-8MB), disk spill (8MB-5GB), and truncation (>5GB). Bash commands bypass JavaScript entirely—stdout/stderr go directly to a file descriptor, with progress extracted by tail-polling. This prevents OOM from commands that produce gigabytes of output.

**Numeric task IDs** (not UUIDs) are used for human readability: "Task #3" is clearer than a 36-character hex string. A high-water mark prevents ID reuse after deletion.

**Eviction.** Completed tasks get a 30-second grace period (`evictAfter` timestamp) for the user to inspect them. The `retain` flag blocks eviction while the UI is showing the task. After the grace period, `evictTerminalTask()` removes the task from AppState and cleans up disk output.

---

## 5. Communication Protocols

### 5.1 Parent-Child Results (Generator Stream)

The primary channel between parent and child agents. The `runAgent()` async generator yields `Message` objects that the parent consumes. For sync agents, the parent blocks on each yield. For async agents, messages accumulate in a background loop and the parent is notified on completion.

```
Parent                          Child (runAgent generator)
  |                                |
  |--- spawn ---------------------->|
  |                                |--- query() -> API
  |<-- yield Message --------------|
  |                                |--- tool execution
  |<-- yield Message --------------|
  |<-- return (done) --------------|
  |                                |
  |--- finalizeAgentTool() -------->|
```

### 5.2 Teammate Mailbox (File-Based)

Swarm members communicate through JSON-based inboxes at `~/.claude/teams/{team_name}/inboxes/{agent_name}.json`.

Each message carries: sender name, text content, ISO timestamp, read flag, optional sender color, and a 5-10 word summary for UI display.

**Concurrency model:** `proper-lockfile` creates an mtime-based lock directory for each inbox file. The lock configuration (10 retries, 5-100ms exponential backoff with jitter) handles contention spikes when multiple teammates send messages simultaneously (e.g., 5 teammates reporting "ready" at once). Total worst-case wait is ~700ms, sufficient for teams of ~10 members. Larger teams would need a different coordination primitive.

Stale lock detection uses mtime checks—if a process crashes holding the lock, the lock becomes readable by other processes. No PID-based races across restarts.

Defined in: `src/utils/teammateMailbox.ts`

### 5.3 SendMessage Tool

High-level inter-agent messaging API with five routing targets:

1. **In-process local agents** — Route by name via `agentNameRegistry` in AppState. The name registry is a thin facade (name→agentId map) decoupled from the task system (agentId→state machine). If the target is stopped, auto-resume from its transcript.
2. **In-process/tmux/iTerm2 teammates** — Write to file-based mailbox.
3. **Bridge (inter-Claude)** — Send via `postInterClaudeMessage()` to a remote session.
4. **UDS (local peer)** — Send via Unix domain socket.
5. **Broadcast (`"*"`)** — Iterate all known teammates, write to each mailbox.

**Structured message types** enable coordination protocols:
- `shutdown_request` / `shutdown_response` — Graceful shutdown handshake.
- `plan_approval_response` — Team lead approval/rejection of a teammate's plan.

Defined in: `src/tools/SendMessageTool/SendMessageTool.ts`

### 5.4 Task Notifications

When background agents complete, they communicate results via XML-tagged notifications enqueued to the message queue—not via direct state mutation. This is async-safe: there's no race between agent state and model knowledge. The notification includes task ID, output file path, status, summary, and usage metrics. The main agent sees these inline in its transcript and can decide what to do next.

### 5.5 Signal Primitive

Lightweight stateless pub/sub used internally for event notification (mailbox arrivals, state changes). `subscribe()` returns an unsubscribe function; `emit()` notifies only current subscribers. ~15 usage sites across the codebase.

Defined in: `src/utils/signal.ts`

### 5.6 Generic Mailbox (In-Process)

A message queue with synchronous polling (`poll()` for non-blocking reads) and asynchronous waiting (`receive()` for blocking reads with predicate filtering). Message sources include `"user"`, `"teammate"`, `"system"`, `"tick"`, and `"task"`.

**Message priority for teammates:** Team lead messages are dequeued before peer messages. This prevents the lead's urgent redirections from being starved behind peer-to-peer chatter.

Defined in: `src/utils/mailbox.ts`

---

## 6. Agent Definition System

### 6.1 Definition Sources & Override Hierarchy

Agent definitions are loaded from multiple sources with a last-write-wins override strategy.

**Priority order (lowest to highest):**
1. **Built-in** — Programmatic definitions in `src/tools/AgentTool/built-in/`.
2. **Plugin** — Loaded from the plugin system.
3. **User settings** — `~/.claude/agents/*.md`
4. **Project settings** — `.claude/agents/*.md`
5. **Flag settings** — `--agent` CLI flag.
6. **Policy/Managed settings** — Enterprise/organization level.

**Why this ordering?**
- **Enterprise-last.** Policy settings can override anything, allowing organizations to enforce agents without fighting user customizations.
- **Project over user.** If a team defines a "code-reviewer" agent, it takes precedence over an accidentally-named personal variant.
- **CLI for one-offs.** Flag agents let you test a new definition on a single invocation without modifying files.
- **Plugins as extensions.** Plugins sit above user (extending system capability) but below project (the team's source of truth).
- **Built-ins as foundation.** Always available, with dynamic prompts that adapt to runtime context.

**Deduplication is complete replacement, not merging.** If two sources define the same `agentType`, the higher-priority source completely replaces the lower one—all fields, not a partial merge. This avoids "which source's tool list wins?" ambiguity. The `allAgents` return value preserves shadowed definitions for display/audit.

### 6.2 Markdown Agent Format

**Why markdown with YAML frontmatter?** The body becomes the system prompt—prose that reads naturally, diffs cleanly in git, and self-documents in the filesystem (`ls .claude/agents/` shows readable names). Progressive enrichment means a minimal agent is just `name`, `description`, and a few lines of prose. JSON would bury system prompts in escaped strings with visual noise.

Custom agents are defined as `.md` files with YAML frontmatter specifying metadata (name, tools, model, memory scope, etc.) and a markdown body that becomes the system prompt.

### 6.3 Dynamic vs. Static System Prompts

Built-in agents have dynamic prompts (functions that receive `toolUseContext`) so they can react to runtime state: feature flags, user type, available embedded tools. Custom agents have static prompts (closures that return stored text) for predictability—users expect their agent to behave the same way every time.

Memory injection is the exception: for any agent with `memory` enabled, the prompt is augmented at call time with the agent's MEMORY.md content.

### 6.4 Agent Resolution at Call Time

When `AgentTool` is invoked:
1. Filter by `allowedAgentTypes` from `Agent(type1,type2)` permission specs.
2. Filter by permission deny rules.
3. Look up by `effectiveType`.
4. If found in `allAgents` but filtered out, report "denied by permission rule" (not "not found").

### 6.5 Required MCP Servers

Agents can declare `requiredMcpServers` using case-insensitive substring matching (not exact matching). An agent requiring `"slack"` matches `slack`, `slack-web`, `slack-enterprise`. Unmet requirements silently filter the agent from the available list—no error, no blocking. This handles version variations and plugin namespacing without requiring agents to enumerate every variant.

### 6.6 Built-In Agent Registry

| Agent | Tools | Notes |
|-------|-------|-------|
| `general-purpose` | All (`*`) | Multi-step tasks, full tool access |
| `Explore` | Read-only subset | One-shot, `omitClaudeMd`, skips trailer |
| `Plan` | Read-only subset | One-shot, `omitClaudeMd`, skips trailer |
| `claude-code-guide` | Read-only + docs | Resumable across turns |
| `verification` | Configurable | Post-implementation checks |
| `statusline-setup` | Read, Edit | Narrow scope |

One-shot agent types (Explore, Plan) skip the agentId/SendMessage result trailer. At 34M+ weekly spawns, ~135 bytes/invocation × 34M = significant token savings.

Defined in: `src/tools/AgentTool/builtInAgents.ts`, `src/tools/AgentTool/constants.ts`

---

## 7. Resource Management

### 7.1 Tool Resolution & Security Model

Each agent gets an independently computed tool pool. The system uses a layered security model where trust decreases with distance from the parent:

**Layer 1 — Universal blocklist (`ALL_AGENT_DISALLOWED_TOOLS`).** Applied to all agents. Prevents recursion (AgentTool), permission escalation (ExitPlanMode, AskUserQuestion), and cross-cutting concerns (TaskOutput, TaskStop). These tools are main-thread abstractions that subagents must not access.

**Layer 2 — Custom agent restrictions (`CUSTOM_AGENT_DISALLOWED_TOOLS`).** Applied only to non-built-in agents. Enforces a trust boundary: user-defined agents cannot spawn sub-agents by default. Built-in agents bypass this because they are vetted, narrow-scoped code.

**Layer 3 — Async agent allowlist (`ASYNC_AGENT_ALLOWED_TOOLS`).** Background agents are restricted to a whitelist of safe tools (file ops, search, shell, web). Unrestricted async access would allow detached agents to silently modify state the parent relies on, create coordination chains outside the parent's awareness, or spawn unbounded subagent trees.

**Layer 4 — Swarm teammate extensions (`IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`).** In-process teammates get additional tools (TaskCreate, TaskGet, TaskUpdate, TaskList, SendMessage) to enable team coordination. They can also spawn sync subagents but not background agents.

**MCP tools always pass through all filters.** They represent user-configured external integrations with tight API boundaries. If a user configures Slack on an agent, the agent should be able to use it regardless of filtering rules.

**Wildcard vs. explicit tool lists.** `tools: ["*"]` inherits the parent's full (filtered) pool—used by fork children for cache-identical tool definitions. Explicit lists (`tools: ["Read", "Grep"]`) defend a role boundary. Explore agents should only search and read, not modify files.

**Permission mode interaction.** Plan mode allows `ExitPlanMode` (subagents need an escape hatch to signal "planning done"). The `Agent(type1,type2)` spec constrains which agent types can be spawned, preventing scope creep across roles.

Defined in: `src/tools/AgentTool/agentToolUtils.ts`, `src/constants/tools.ts`

### 7.2 MCP Server Composition

Agents declare MCP servers via two mechanisms:

- **Name reference** (`string`): Looks up existing server config by name. Shared connection—not cleaned up when agent finishes. Multiple agents can reference the same server.
- **Inline definition** (`{ name: config }`): Creates a new connection scoped to the agent. Cleaned up when agent finishes. Useful for agent-specific integrations that shouldn't persist.

Required MCP servers use substring pattern matching with a 30-second polling wait at spawn time. If a required server isn't connected, the agent blocks rather than spawning with incomplete capabilities.

### 7.3 Agent Memory

Three persistence scopes, reflecting three levels of sharing:

| Scope | Path | Sharing | VCS | Use Case |
|-------|------|---------|-----|----------|
| `user` | `~/.claude/agent-memory/<agentType>/MEMORY.md` | Cross-project | No | Personal learnings, portable across codebases |
| `project` | `.claude/agent-memory/<agentType>/MEMORY.md` | Team-wide | Yes | Team conventions, project-specific knowledge |
| `local` | `.claude/agent-memory-local/<agentType>/MEMORY.md` | Machine-only | No | Local paths, API keys, machine-specific context |

**Memory snapshots** prevent accidental loss: project-level snapshots are copied to user-level on first run, and newer snapshots trigger an update prompt. This keeps teams in sync when someone pulls new project memory via VCS.

**Skill isolation:** Invoked skills are keyed by `${agentId}:${skillName}` to prevent cross-agent overwrites in the shared skill map.

Defined in: `src/tools/AgentTool/agentMemory.ts`, `src/tools/AgentTool/agentMemorySnapshot.ts`

### 7.4 CLAUDE.md Injection

By default, agents receive the full CLAUDE.md hierarchy (managed → user → project → local → auto-memory). Read-only agents (Explore, Plan) set `omitClaudeMd: true` to skip this injection. These agents don't commit, create PRs, or modify code—they don't need enforcement of rules they'll never trigger. The main agent has full CLAUDE.md and interprets their output.

**Token impact:** Typical CLAUDE.md is 500-1000 tokens. Across 34M+ weekly Explore spawns, omitting it saves ~5-15 Gtok/week. Feature-gated via `tengu_slim_subagent_claudemd` for A/B testing and rollback.

### 7.5 Lifecycle Cleanup

Every agent runs comprehensive cleanup in a `finally` block, regardless of exit path:

| Resource | Cleanup | Consequence if leaked |
|----------|---------|-----------------------|
| Inline MCP servers | `client.cleanup()` | Dangling connections, resource exhaustion |
| Git worktree | Remove if no changes detected | Orphaned worktrees, disk consumption |
| Invoked skills | `clearInvokedSkillsForAgent()` | Stale skill state across agents |
| Background bash tasks | `killShellTasksForAgent()` | Zombie processes (PPID=1) |
| File state cache | `readFileState.clear()` | Memory accumulation over 100s of agents |
| Perfetto tracing | `unregisterPerfettoAgent()` | Telemetry pollution |
| Summarization | `stopSummarization()` | Dangling fork agents |
| Transcript subdir | `clearAgentTranscriptSubdir()` | Stale mapping |

The explicit per-resource approach prevents subtle leaks that accumulate across sessions with hundreds of agent spawns.

---

## 8. State Management

### 8.1 AppState Agent-Related Fields

The global `AppState` (React-style functional updates via `setState`) holds:

| Field | Purpose |
|-------|---------|
| `tasks` | All active tasks (agents, shells, teammates) |
| `agentColorMap` / `agentColorIndex` | Per-agent UI color assignment |
| `mainThreadAgentType` | From `--agent` CLI flag |
| `agentNameRegistry` | `name → agentId` mapping for SendMessage routing |
| `sessionCreatedTeams` | Teams created this session (for cleanup on shutdown) |
| `teamContext` | Team state (only present when a team is active) |

### 8.2 Team Context

Present in AppState only when a team is active. Contains the team name, file path, deterministic lead agent ID, self-identity fields (for teammates), and a map of all teammates with their metadata (name, type, color, pane ID, cwd, worktree path, spawn time).

**Why file-based team state?** Not AppState (per-session, in-memory—team must outlive sessions for reconnection and multi-process coordination). Not a database (overkill for tiny JSON structures, no daemon dependency). File-based provides persistence, inspectability, and cross-process visibility with minimal complexity.

### 8.3 Task State Lifecycle

```
                 registerTask()
                      |
                      v
  +----------+  in_progress  +------------+
  | pending  |  ---------->  | in_progress|
  +----------+               +-----+------+
                                   |
                     +---------+---+---------+
                     |         |             |
                     v         v             v
              +----------+ +--------+ +----------+
              |completed | | killed | |  failed  |
              +----+-----+ +---+----+ +----+-----+
                   |           |            |
                   v           v            v
              enqueueAgentNotification()
                   |
                   v
              evictTerminalTask() (after 30s grace period)
```

### 8.4 User-Facing Task Tracking

Separate from agent task states, the user-facing task system provides structured project management:

- File-based storage at `~/.claude/tasks/{taskListId}/{taskId}.json` with file-level locking.
- Concurrency control via `proper-lockfile` (30 retries, exponential backoff) for multi-process swarm access.
- Numeric IDs with high-water mark (monotonically increasing, no reuse after deletion).
- Dependency tracking via `blocks`/`blockedBy` arrays for task ordering.

---

## 9. Resilience & Fault Tolerance

### 9.1 Error Taxonomy

| Error Type | Cause | Handling | Retryable |
|------------|-------|----------|-----------|
| `AbortError` | User cancellation, parent abort | Extract partial result, notify | No |
| `APIError` (401/403) | Authentication failure | Force token refresh, retry | Yes |
| `APIError` (429) | Rate limit | Exponential backoff (500ms base, 32s cap, 25% jitter) | Yes |
| `APIError` (529) | Overloaded | Retry for foreground only; background fails immediately | Conditional |
| `APIConnectionError` | Network (ECONNRESET, EPIPE) | Reset keep-alive pool, acquire fresh client | Yes |
| Context overflow (400) | Message too large | Reduce `max_tokens`, retry | Yes |
| Generic `Error` | Unexpected | Log, fail agent task | No |

### 9.2 Abort/Cancellation Model

Abort signals propagate unidirectionally via linked AbortControllers: parent → children, never children → parent. The implementation uses `WeakRef` to prevent parents from retaining garbage-collected children.

When a task is killed:
1. `stopTask()` aborts the agent's AbortController.
2. The query loop detects the abort signal and throws `AbortError`.
3. The catch handler extracts partial results by walking messages backward.
4. Status transitions to `killed` with a partial result notification.

### 9.3 API Retry Strategy

Exponential backoff with configurable max retries (default 10). Key design decisions:

- **529 (overloaded) retries only for foreground.** Background tasks (summaries, classifiers) fail immediately on 529 to prevent retry amplification during cascading failures. If every background agent retries during an overload, it worsens the overload.
- **Persistent retry mode** (internal): 429/529 retry indefinitely with a 5-minute backoff cap and keep-alive heartbeat yields every 30 seconds.
- **Stale connection recovery:** ECONNRESET/EPIPE errors trigger keep-alive pooling reset and fresh client acquisition.

### 9.4 Partial Result Preservation

When an agent is interrupted (abort or error), `extractPartialResult()` walks the message history backward to find the last assistant message with text content. This preserves work-in-progress even when the agent didn't reach a clean completion. The partial result is included in the notification so the parent can use it.

### 9.5 MCP Server Failure Resilience

Individual MCP server failures don't crash the agent. Failed connections are logged as warnings, and tools from that server are simply unavailable. The agent continues with its remaining tools. All agent-specific MCP cleanup runs in a `finally` block regardless of the exit path.

### 9.6 Swarm Resilience

**Teammate reconnection.** Teammates are initialized or resumed from stored team metadata. No active reconnection polling—the teammate simply re-reads the team file and picks up from its mailbox.

**Leader crash.** No leader election or failover. If the lead crashes, teammates can continue accessing the team file for metadata, but there's no auto-recovery. This is a known design gap, acceptable for the current single-leader team model where sessions are short-lived and human-supervised.

**Worktree cleanup on crash.** Worktrees that were modified before a crash are preserved (not auto-deleted). Resumed worktrees bump their mtime to prevent background cleanup from deleting them.

### 9.7 Design Philosophy

The resilience model prioritizes operational resilience (fail gracefully, don't cascade, preserve work) over high availability (no failover, no leader election). This reflects the CLI use case: humans supervise small agent teams in interactive sessions, not unattended distributed systems.

Key principles:
- **Fail-fast for non-retryable errors.** Don't retry authentication failures or invalid requests.
- **Protect background from cascades.** Background retries during overload amplify the problem.
- **One-way abort chains.** Parents can kill children; children can't affect parents.
- **Guaranteed cleanup.** Finally blocks ensure resource release on every exit path.
- **Partial results over total loss.** Interrupted agents extract and report what they accomplished.

---

## 10. Key Source File Index

### Agent Tool Core
| File | Purpose |
|------|---------|
| `src/tools/AgentTool/AgentTool.tsx` | Main tool definition, spawn logic, sync/async branching |
| `src/tools/AgentTool/runAgent.ts` | Agent execution generator, query loop, MCP init |
| `src/tools/AgentTool/agentToolUtils.ts` | Tool filtering, resolution, async lifecycle, finalization |
| `src/tools/AgentTool/forkSubagent.ts` | Fork agent definition, message construction, recursion guard |
| `src/tools/AgentTool/resumeAgent.ts` | Resume existing agent sessions |
| `src/tools/AgentTool/loadAgentsDir.ts` | Agent definition loading, parsing, deduplication |
| `src/tools/AgentTool/agentDisplay.ts` | Override resolution, display formatting |
| `src/tools/AgentTool/agentMemory.ts` | Agent memory scope management |
| `src/tools/AgentTool/agentMemorySnapshot.ts` | Memory snapshot sync |
| `src/tools/AgentTool/agentColorManager.ts` | Color assignment per agent |
| `src/tools/AgentTool/builtInAgents.ts` | Built-in agent registry |
| `src/tools/AgentTool/constants.ts` | Agent tool constants, one-shot types |
| `src/tools/AgentTool/prompt.ts` | Agent listing prompt generation |

### Built-In Agent Definitions
| File | Agent Type |
|------|-----------|
| `src/tools/AgentTool/built-in/generalPurposeAgent.ts` | `general-purpose` |
| `src/tools/AgentTool/built-in/exploreAgent.ts` | `Explore` |
| `src/tools/AgentTool/built-in/planAgent.ts` | `Plan` |
| `src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts` | `claude-code-guide` |
| `src/tools/AgentTool/built-in/verificationAgent.ts` | `verification` |
| `src/tools/AgentTool/built-in/statuslineSetup.ts` | `statusline-setup` |

### Communication & Messaging
| File | Purpose |
|------|---------|
| `src/tools/SendMessageTool/SendMessageTool.ts` | Inter-agent messaging, routing, structured messages |
| `src/utils/teammateMailbox.ts` | File-based team inbox with locking |
| `src/utils/mailbox.ts` | Generic in-process message queue |
| `src/utils/signal.ts` | Lightweight pub/sub signal primitive |

### Team / Swarm
| File | Purpose |
|------|---------|
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | Team creation, file setup |
| `src/tools/TeamDeleteTool/TeamDeleteTool.ts` | Team cleanup |
| `src/coordinator/coordinatorMode.ts` | Coordinator mode detection, system prompt |
| `src/utils/swarm/teamHelpers.ts` | Team file I/O, member management |
| `src/utils/swarm/backends/types.ts` | Backend abstraction (TeammateExecutor) |
| `src/utils/swarm/backends/InProcessBackend.ts` | In-process teammate lifecycle |
| `src/utils/swarm/backends/TmuxBackend.ts` | Tmux pane management |
| `src/utils/swarm/backends/ITermBackend.ts` | iTerm2 pane management |
| `src/utils/swarm/backends/PaneBackendExecutor.ts` | Shared pane backend logic |
| `src/utils/swarm/backends/detection.ts` | Backend type detection |
| `src/utils/swarm/backends/registry.ts` | Backend registry |
| `src/utils/swarm/spawnInProcess.ts` | In-process teammate spawning |
| `src/utils/swarm/spawnUtils.ts` | Spawn utilities |
| `src/utils/swarm/inProcessRunner.ts` | In-process teammate execution loop |
| `src/utils/swarm/leaderPermissionBridge.ts` | Permission bridging for teammates |
| `src/utils/swarm/permissionSync.ts` | Permission synchronization |
| `src/utils/swarm/reconnection.ts` | Teammate reconnection |
| `src/utils/swarm/teammateInit.ts` | Teammate initialization |
| `src/utils/swarm/teammateLayoutManager.ts` | Pane layout management |
| `src/utils/swarm/teammateModel.ts` | Teammate model selection |
| `src/utils/swarm/teammatePromptAddendum.ts` | Teammate prompt modifications |

### Context & Identity
| File | Purpose |
|------|---------|
| `src/utils/agentContext.ts` | AsyncLocalStorage context for agents |
| `src/utils/teammate.ts` | Teammate identity resolution |
| `src/utils/teammateContext.ts` | Teammate-specific context type |
| `src/utils/forkedAgent.ts` | Fork context isolation, execution |

### Task System
| File | Purpose |
|------|---------|
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | Background agent task state, progress tracking |
| `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` | In-process teammate task state |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | Remote agent task state |
| `src/tasks/LocalShellTask/LocalShellTask.tsx` | Background shell task state |
| `src/tasks/types.ts` | Task type definitions |
| `src/tasks/stopTask.ts` | Task termination |
| `src/tools/TaskOutputTool/TaskOutputTool.tsx` | Task output retrieval with blocking wait |
| `src/tools/TaskStopTool/TaskStopTool.ts` | Task stop tool |
| `src/utils/tasks.ts` | User-facing task CRUD with file locking |
| `src/utils/task/framework.ts` | Task state updates, registration, eviction |
| `src/utils/task/diskOutput.ts` | Disk-based task output management |
| `src/utils/task/sdkProgress.ts` | SDK progress event emission |

### Multi-Agent Spawning
| File | Purpose |
|------|---------|
| `src/tools/shared/spawnMultiAgent.ts` | Multi-agent spawn orchestration |

### Interface Boundaries (out of scope — mentioned for reference)
| File | Role |
|------|------|
| `src/utils/permissions/`, `src/hooks/toolPermission/` | Permission pipeline — agents interact via `permissionMode` and `canUseTool` |
| `src/services/mcp/` | MCP protocol — agents interact via `mcpServers` spec and tool namespacing |
| `src/query.ts` | Query loop — agents call this to execute API turns |
