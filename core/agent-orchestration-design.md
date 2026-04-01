# Agent Orchestration — Design Document

Architecture-level specification of Claude Code's multi-agent orchestration system. Covers agent lifecycle, orchestration patterns, communication protocols, agent definitions, resource management, and the design decisions behind each subsystem.

---

#### 1. Overview and Design Principles

The orchestration system manages multiple concurrent agents with isolated contexts, independent tool pools, and multiple communication channels. It supports five distinct orchestration patterns ranging from simple parent-child subagents to full multi-agent swarms.

**Generator-based streaming.** Agent execution uses async generators that yield Message objects progressively. This unifies sync and async execution paths under a single consumption model -- the same generator serves both a blocking foreground caller and a fire-and-forget background lifecycle.

**Context isolation via AsyncLocalStorage.** When agents are backgrounded, multiple agents run concurrently in the same process. A single global state shared by all agents would cause collisions: analytics events from Agent A would carry Agent B's context, file reads under one agent's ID would appear in another's transcript. AsyncLocalStorage provides per-execution-chain isolation without requiring process-level separation. Each concurrent agent gets its own async context; parent, Agent A, and Agent B never interfere. The trade-off is that every spawn/resume boundary must explicitly set context -- miss the boundary and events leak to the parent context.

**Prompt cache optimization.** Fork subagents construct byte-identical API prefixes across all children so they share prompt cache entries. One-shot built-in agents (Explore, Plan) omit non-essential message trailers to save tokens. At scale (34M+ weekly Explore spawns), these optimizations compound into significant token savings.

**Tool pool independence.** Each agent receives its own computed tool pool based on its definition, permission mode, and execution context (sync vs. async, built-in vs. custom). Trust decreases with distance from the parent: built-in agents get wider access, custom agents are cautious, async agents are isolated.

**File-based coordination.** Multi-agent swarms use file-based mailboxes with file-level locking for concurrent access, enabling cross-process communication without shared memory. File-based mailboxes were chosen over IPC, shared memory, or network sockets because they work uniformly across all three execution backends (in-process, tmux, iTerm2) without requiring daemons, port allocation, or special cleanup.

**Graceful lifecycle transitions.** Foreground agents can transition to background mid-execution via a race between the message stream and a background signal. The agent generator continues without restart -- only the communication channel changes, preserving prompt cache stability.

**Isolation over concurrency.** The system accepts that concurrent agents cannot collaborate on the same file in real-time, but gains simplicity, memory safety, and cache-sharing benefits. This reflects a deliberate choice: optimize for isolated background work and sequential tool use, not real-time collaborative editing.

---

#### 2. Core Abstractions

##### 2.1 AgentDefinition

Union type representing all possible agent sources. Three variants share a common base:

- **BuiltInAgentDefinition** -- Programmatic, with dynamic system prompts keyed to runtime context (feature flags, user type, available tools). Built-in agents are trusted, maintained by the platform team, and can react to build-time and runtime state.
- **CustomAgentDefinition** -- User/project/policy-defined, with static system prompts loaded from markdown files. Treated as black boxes -- their prompt text is fixed at load time for predictability.
- **PluginAgentDefinition** -- Registered by plugins, following the same format as custom agents but namespaced to avoid collisions.

Key fields on the base type:

| Field | Purpose |
|-------|---------|
| `agentType` | Unique identifier, used for deduplication and routing |
| `whenToUse` | Description shown to the LLM for tool selection |
| `tools` / `disallowedTools` | Allowlist/denylist defining the agent's tool pool |
| `mcpServers` | MCP servers this agent needs (name refs or inline defs) |
| `model` | Model override (inherit, sonnet, opus, haiku) |
| `permissionMode` | Permission mode override for the agent's execution |
| `memory` | Persistence scope: user, project, or local |
| `isolation` | Execution isolation: worktree or remote |
| `omitClaudeMd` | Skip CLAUDE.md injection (token optimization for read-only agents) |
| `hooks` | Session-scoped lifecycle hooks registered when agent starts |

##### 2.2 AgentToolInput / AgentToolResult

The AgentTool (externally named "Task" for backward compatibility) accepts a prompt, optional subagent type, and execution parameters (run_in_background, model, isolation, name/team_name for swarms). It returns one of four statuses: completed (sync result), async_launched (background), teammate_spawned (swarm), or remote_launched (CCR).

##### 2.3 AgentContext (AsyncLocalStorage)

Two context variants exist:

- **SubagentContext** -- For agents spawned via AgentTool. Carries agentId, parentSessionId, and whether the agent is built-in.
- **TeammateAgentContext** -- For swarm members. Adds agentName, teamName, isTeamLead, planModeRequired, and team-specific routing metadata.

Both are set at spawn boundaries and accessed anywhere in the async execution chain.

##### 2.4 Message

The fundamental unit flowing through agent generators. All agent execution produces a stream of Message objects consumed by the parent. Key subtypes include AssistantMessage (tool use blocks, thinking, text), UserMessage, progress events, and system boundary markers.

---

#### 3. Agent Lifecycle

All agents follow a four-phase lifecycle: **Spawn -> Init -> Run -> Cleanup**. The init phase branches into sync or async paths, and sync agents can transition to async mid-execution.

##### 3.1 Spawn Phase

1. **Validate constraints.** Check multi-agent feature gates. Reject recursive fork attempts.
2. **Route multi-agent.** If name + team_name are provided, delegate to teammate spawning and return early.
3. **Resolve agent type.** If subagent_type is omitted and fork feature is enabled, take the fork path. Otherwise resolve from the active agents list.
4. **Validate agent.** Check allowedAgentTypes from permission specs. Verify requiredMcpServers are available (poll up to 30s).
5. **Assemble tool pool.** Compute the agent's independent tool set based on its definition and execution context.
6. **Create agent ID.** Stable ID for tracking and transcript correlation.
7. **Set up isolation.** Optionally create a git worktree.
8. **Build system prompt.** Fork path: inherit parent's rendered prompt verbatim. Normal path: call the agent's getSystemPrompt().

##### 3.2 Init and Branch

**Async Path (Background):** Register the task, register the name-to-agentId mapping for routing, fire-and-forget the async lifecycle, and return async_launched immediately.

**Sync Path (Foreground):** Register as foreground, create the generator iterator, and enter a race loop: each generator step is raced against a background signal. After 2 seconds, a background hint is shown to the user. If the user backgrounds the agent mid-run, the generator transparently transitions to the async lifecycle without restart.

**Design decision -- auto-background.** The 2-second threshold solves a UX problem: short commands complete before the user notices, while long-running agents do not block the prompt. A configurable auto-background timer (default 2 minutes) can forcibly transition foreground agents that run too long. The transition preserves the generator's state and prompt cache -- no work is lost or restarted.

##### 3.3 Run Phase

1. Initialize agent MCP servers (connect new or reuse shared).
2. Register session hooks from the agent definition.
3. Load agent memory if configured.
4. Enter the query loop: call the API repeatedly, execute tools, yield each Message.
5. On completion or abort, unregister hooks and close agent-specific MCP connections.

##### 3.4 Cleanup Phase

**Ordering invariant:** The status transition (mark completed) happens BEFORE post-completion embellishments (handoff classification, worktree cleanup). This ensures consumers are unblocked even if git operations or the transcript classifier hang. Embellishments are best-effort and non-blocking.

Cleanup covers:
- Stop summarization if active.
- Finalize result (extract text, compute usage metrics).
- Mark completed in state store.
- Post-completion: handoff classification, worktree cleanup (parallel, non-blocking).
- Enqueue notification with final message, usage stats, and worktree info.
- On error: extract partial results (walks messages backward), notify with partial progress.
- Final: clear invoked skills, dump state, release cloned file state cache.

##### Lifecycle Diagram

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

#### 4. Orchestration Patterns

##### 4.1 Sub-Agent Pattern

The basic parent-child relationship. The parent spawns a child agent via AgentTool, the child runs to completion, and the parent receives the result.

**Characteristics:**
- Child gets its own system prompt and independently computed tool pool.
- Sync execution blocks the parent's turn; async returns immediately.
- One-shot agents skip the agentId/SendMessage trailer to save approximately 135 bytes per invocation.

**Design rationale -- no recursive spawning by default.** Custom agents cannot spawn sub-agents. This prevents a user-defined agent from creating unbounded nesting. Built-in agents can recurse because they are vetted, narrow-scoped code maintained by the platform team. Permission specs allow controlled recursion where needed.

**Built-in sub-agent types:**

| Agent | Tools | Purpose |
|-------|-------|---------|
| general-purpose | All | Multi-step tasks |
| Explore | Read-only (Glob, Grep, Read, WebFetch, WebSearch) | Codebase exploration. One-shot. |
| Plan | Read-only | Architecture planning. One-shot. |
| claude-code-guide | Read-only + docs | Documentation lookup. Resumable. |
| verification | Configurable | Post-implementation checks |
| statusline-setup | Read, Edit only | Statusline configuration |

##### 4.2 Coordinator Mode

An environment-level mode that restructures the main session into a coordinator that delegates to worker agents.

**Activation:** Environment variable + feature gate.

**Why an environment variable, not a tool or agent type?** Coordinator mode is not a capability the main agent invokes; it is a complete restructuring of the session's conversation model. It changes what system prompt the main agent receives, what tools are available, and how the session reasons about its role. An environment variable makes this a bootstrap decision -- a session-level invariant that persists across resumes and cannot be toggled mid-conversation. Tools are runtime features; coordinator mode is infrastructure.

**The "synthesis over delegation" principle.** The coordinator system prompt enforces a critical rule: the coordinator must read and understand worker findings before issuing the next instruction. This prevents the "lazy delegation" anti-pattern where a coordinator relays worker output without understanding it, leading to compounding misalignment. Each hand-off that skips synthesis loses context; the coordinator is the intelligence center that sees the full picture across multiple workers.

**Coordinator-available tools:** AgentTool, SendMessageTool, TaskStopTool, TeamCreateTool, TeamDeleteTool. The coordinator deliberately lacks file manipulation tools -- it directs workers rather than doing the work itself.

**Summarization.** Coordinators need visibility into long-running parallel workers. A periodic summarization fork (every 30s) generates 3-5 word progress descriptions by forking the worker's conversation. The fork shares the worker's prompt cache (identical prefix), making it cheap. Regular sub-agents do not need this because the user watches foreground output directly.

##### 4.3 Team Swarms

A peer-based multi-agent pattern where a team lead manages multiple teammates that communicate directly with each other.

**When to use swarms vs. coordinator vs. sub-agents.** Sub-agents are for delegated sub-tasks with parent control. Coordinator mode is for structured research-synthesis-implementation-verification workflows. Swarms are for collaborative work where agents have overlapping responsibilities and need coordination negotiation. The three levels compose: a coordinator can spawn workers (sub-agents), and a sub-agent can spawn a swarm within its scope.

**Team creation flow:**
1. Create a team config file.
2. Generate a deterministic lead agent ID.
3. Set up a task list directory for the team.
4. Update application state with team context.

**Execution Backends:**

Three backends hide different process models behind a common executor interface (spawn, sendMessage, terminate, kill, isActive):

| Backend | Process Model | Visualization | Trade-offs |
|---------|--------------|---------------|------------|
| in-process | Same process, AsyncLocalStorage isolation | None (headless) | Minimal overhead, shared resources, no visual isolation |
| tmux | Separate tmux panes | Native pane borders, live scrollback | Shell startup overhead (~200ms), works with existing tmux sessions |
| iterm2 | iTerm2 split panes via AppleScript | Native macOS integration | macOS-only, requires it2 CLI |

**Detection priority:** tmux (if inside tmux) -> iTerm2 (if available) -> external tmux session (if tmux installed) -> error. This ensures no nested multiplexing and provides a clear fallback chain.

**Why file-based mailboxes?** IPC requires server setup per process. Shared memory breaks across process boundaries. Network sockets require port allocation and firewall handling. File-based mailboxes work uniformly across all backends. Messages survive temporary crashes, provide natural debugging checkpoints, and require no broker daemon. The trade-off is millisecond-level latency (vs. microsecond for in-memory queues), acceptable because swarm messages are paced by human and agent response times.

**Deterministic Agent IDs:**

Format: `agentName@teamName` (e.g., `researcher@my-project`).

Deterministic IDs solve three coordination problems:
1. **Reconnection.** After a crash, restart with the same name -> same ID -> same mailbox -> resume from stored messages.
2. **Leader discovery.** `team-lead@{teamName}` is always the lead. No lookup needed.
3. **Routing.** SendMessage computes the mailbox path directly from the name, without a registry lookup.

**Leader-Follower Model:**

The team lead is a soft authority, not a dictator. The lead can redirect at the task level and halt execution (shutdown request), but cannot override implementation decisions. This design prevents coordination deadlocks: the lead's messages have priority over peer messages (no starvation behind peer chatter), shutdown is asymmetric (lead requests, teammate approves/rejects), and abort signals propagate unidirectionally (user -> lead -> teammates).

Pure peer-to-peer was rejected because without a leader, agents must reach consensus on state (are we done? can we shut down?), introducing deadlock risk and violating the non-negotiable requirement that user intent (via the lead) is never blocked by peer negotiation.

**Shutdown Protocol:**

Graceful shutdown uses a request/response pattern rather than force-kill:
1. Lead sends shutdown_request to teammate's mailbox.
2. Teammate reads request, finishes current operation, responds with shutdown_response (approve or reject with reason).
3. On approval, teammate cleans up resources and exits.
4. Lead can escalate to force-kill if needed.

This prevents orphaned operations, partial file edits, and broken tool result chains.

**Plan Mode Requirement:**

Teammates can be spawned with planModeRequired, forcing them to show their implementation plan to the team lead before executing. This prevents parallel teammates from making divergent assumptions.

##### 4.4 Fork Sub-Agents (Experimental)

Implicit subagent spawning that inherits the parent's full conversation context for maximum prompt cache sharing.

**The cache-sharing strategy.** The API's cache key includes the entire message prefix. Fork children are constructed so that every child sends byte-identical messages up to a divergence point near the end. All children share one cached prefix; only the per-child directive (~100 tokens) is unique. If 5 agents fork in parallel, the cache is computed once and reused 5 times.

**How byte-identity is maintained:**
- The parent's rendered system prompt is passed verbatim (not recomputed, which could diverge due to feature flag state changes mid-session).
- All tool_result blocks use an identical placeholder string rather than real results. Real results would differ per attempt, busting the cache.
- Only the final text block (the per-child directive) diverges -- this falls after the cache prefix is already computed.

**Why placeholders instead of real results?** Children inherit the full conversation context above the fork point and can re-read files or re-run commands if needed. The cache savings from identical placeholders outweigh the cost of occasional redundant reads.

**Context isolation for forks:**
- **State cloning.** Read file state is cloned so the fork sees everything the parent saw (same cache prefix) but subsequent reads do not pollute the parent's cache.
- **Mutation suppression.** All mutation callbacks are no-op by default, preventing concurrent forks from corrupting parent UI state or metrics.
- **Task registration exception.** Task state updates always reach the root store because background bash tasks spawned by forks must be tracked session-wide to prevent zombie processes.
- **Linked AbortControllers.** A new child controller is linked to the parent via WeakRef. Parent abort propagates to children, but not vice versa. The WeakRef prevents the parent from retaining garbage-collected children.

**Recursion guard.** Fork children detect themselves via a boilerplate tag in their message history and reject fork attempts. The AgentTool remains in the child's tool pool (for cache-identical tool definitions) but is blocked at call time. This is a special case that bypasses Layer 1 filtering for tool _definitions_ but not tool _execution_ (see Section 7.1 for the layer model).

##### 4.5 Async Task Execution

Any agent can be launched as a background task. Additionally, foreground agents can transition to background mid-run.

**Two task systems coexist by design:**
- **Agent tasks** (internal execution tracking) -- Track what the assistant is currently doing: background agents, shell commands, in-process teammates. These have lifecycle states, progress tracking, and eviction timers.
- **User-facing tasks** (project management) -- Persistent, user-created work items. These survive session resets and are stored as JSON files with file-level locking for multi-process access.

The separation prevents agent execution from polluting the user's task list.

**Agent task types:**

| Task Type | Purpose |
|-----------|---------|
| local_agent | Background subagent execution |
| local_shell | Background shell commands |
| remote_agent | Remote execution |
| in_process_teammate | In-process swarm member |

**Progress tracking** uses a progress tracker with tool use counts, cumulative token metrics, and a circular buffer of the 5 most recent activities.

**Output retrieval:** Polls state every 100ms until the task completes or a configurable timeout (default 30s, max 600s). Extracts text from in-memory result or tails the disk output file.

**Disk output** supports three tiers: in-memory buffer (0-8MB), disk spill (8MB-5GB), and truncation (>5GB). Shell commands bypass the runtime's memory entirely -- stdout/stderr go directly to a file descriptor, with progress extracted by tail-polling. This prevents OOM from commands that produce gigabytes of output.

**Numeric task IDs** (not UUIDs) are used for human readability. A high-water mark prevents ID reuse after deletion.

**Eviction.** Completed tasks get a 30-second grace period for the user to inspect them. A retain flag blocks eviction while the UI is showing the task.

---

#### 5. Communication Protocols

##### 5.1 Parent-Child Results (Generator Stream)

The primary channel between parent and child agents. The runAgent async generator yields Message objects that the parent consumes. For sync agents, the parent blocks on each yield. For async agents, messages accumulate in a background loop and the parent is notified on completion.

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

##### 5.2 Teammate Mailbox (File-Based)

Swarm members communicate through JSON-based inboxes. Each message carries: sender name, text content, ISO timestamp, read flag, optional sender color, and a 5-10 word summary for UI display.

**Concurrency model:** An mtime-based lock directory is created for each inbox file. The lock configuration (10 retries, 5-100ms exponential backoff with jitter) handles contention spikes when multiple teammates send messages simultaneously. Total worst-case wait is ~655-820ms (655ms base sum without jitter, up to ~820ms with 25% jitter), sufficient for teams of ~10 members. Larger teams would need a different coordination primitive.

Stale lock detection uses mtime checks -- if a process crashes holding the lock, the lock becomes readable by other processes. No PID-based races across restarts.

##### 5.3 SendMessage Tool

High-level inter-agent messaging API with five routing targets:

1. **In-process local agents** -- Route by name via a name registry. If the target is stopped, auto-resume from its transcript.
2. **In-process/tmux/iTerm2 teammates** -- Write to file-based mailbox.
3. **Bridge (inter-Claude)** -- Send to a remote session.
4. **UDS (local peer)** -- Send via Unix domain socket.
5. **Broadcast ("*")** -- Iterate all known teammates, write to each mailbox.

**Structured message types** enable coordination protocols:
- shutdown_request / shutdown_response -- Graceful shutdown handshake.
- plan_approval_response -- Team lead approval/rejection of a teammate's plan.

##### 5.4 Task Notifications

When background agents complete, they communicate results via XML-tagged notifications enqueued to the message queue -- not via direct state mutation. This is async-safe: there is no race between agent state and model knowledge. The notification includes task ID, output file path, status, summary, and usage metrics.

##### 5.5 Signal Primitive

Lightweight stateless pub/sub used internally for event notification (mailbox arrivals, state changes). subscribe() returns an unsubscribe function; emit() notifies only current subscribers. Approximately 15 usage sites across the codebase.

##### 5.6 Generic Mailbox (In-Process)

A message queue with synchronous polling (non-blocking reads) and asynchronous waiting (blocking reads with predicate filtering). Message sources include user, teammate, system, tick, and task.

**Message priority for teammates:** Team lead messages are dequeued before peer messages. This prevents the lead's urgent redirections from being starved behind peer-to-peer chatter.

---

#### 6. Agent Definition System

##### 6.1 Definition Sources and Override Hierarchy

Agent definitions are loaded from multiple sources with a last-write-wins override strategy.

**Priority order (lowest to highest):**
1. Built-in -- Programmatic definitions.
2. Plugin -- Loaded from the plugin system.
3. User settings -- `~/.claude/agents/*.md`
4. Project settings -- `.claude/agents/*.md`
5. Flag settings -- CLI flag.
6. Policy/Managed settings -- Enterprise/organization level.

**Why this ordering?**
- Enterprise-last: Policy settings can override anything, allowing organizations to enforce agents without fighting user customizations.
- Project over user: If a team defines a "code-reviewer" agent, it takes precedence over an accidentally-named personal variant.
- CLI for one-offs: Flag agents let you test a new definition on a single invocation without modifying files.
- Plugins as extensions: Plugins sit above user (extending system capability) but below project (the team's source of truth).
- Built-ins as foundation: Always available, with dynamic prompts that adapt to runtime context.

**Deduplication is complete replacement, not merging.** If two sources define the same agentType, the higher-priority source completely replaces the lower one -- all fields, not a partial merge. This avoids "which source's tool list wins?" ambiguity. The return value preserves shadowed definitions for display/audit.

##### 6.2 Markdown Agent Format

Custom agents are defined as `.md` files with YAML frontmatter specifying metadata (name, tools, model, memory scope, etc.) and a markdown body that becomes the system prompt.

**Why markdown with YAML frontmatter?** The body becomes the system prompt -- prose that reads naturally, diffs cleanly in git, and self-documents in the filesystem. Progressive enrichment means a minimal agent is just name, description, and a few lines of prose. JSON would bury system prompts in escaped strings with visual noise.

##### 6.3 Dynamic vs. Static System Prompts

Built-in agents have dynamic prompts (functions that receive tool-use context) so they can react to runtime state. Custom agents have static prompts (closures that return stored text) for predictability.

Memory injection is the exception: for any agent with memory enabled, the prompt is augmented at call time with the agent's memory content.

##### 6.4 Agent Resolution at Call Time

When AgentTool is invoked:
1. Filter by allowedAgentTypes from permission specs.
2. Filter by permission deny rules.
3. Look up by effectiveType.
4. If found in allAgents but filtered out, report "denied by permission rule" (not "not found").

##### 6.5 Required MCP Servers

Agents can declare requiredMcpServers using case-insensitive substring matching (not exact matching). An agent requiring "slack" matches slack, slack-web, slack-enterprise. Unmet requirements silently filter the agent from the available list -- no error, no blocking.

##### 6.6 Built-In Agent Registry

See the built-in sub-agent types table in Section 4.1 for the full registry.

One-shot agent types (Explore, Plan) skip the agentId/SendMessage result trailer. At 34M+ weekly spawns, approximately 135 bytes/invocation x 34M = significant token savings.

---

#### 7. Resource Management

##### 7.1 Tool Resolution and Security Model

Each agent gets an independently computed tool pool. The system uses a layered security model where trust decreases with distance from the parent:

**Layer 1 -- Universal blocklist.** Applied to all agents. Prevents recursion (AgentTool), permission escalation, and cross-cutting concerns. These tools are main-thread abstractions that subagents must not access.

**Layer 2 -- Custom agent restrictions.** Applied only to non-built-in agents. Enforces a trust boundary: user-defined agents cannot spawn sub-agents by default.

**Layer 3 -- Async agent allowlist.** Background agents are restricted to a whitelist of safe tools (file ops, search, shell, web). Unrestricted async access would allow detached agents to silently modify state the parent relies on.

**Layer 4 -- Swarm teammate extensions.** In-process teammates get additional tools to enable team coordination. They can also spawn sync subagents but not background agents.

**MCP tools always pass through all filters.** They represent user-configured external integrations with tight API boundaries.

**Wildcard vs. explicit tool lists.** Wildcard inherits the parent's full (filtered) pool -- used by fork children for cache-identical tool definitions. Explicit lists defend a role boundary.

**Permission mode interaction.** Plan mode allows ExitPlanMode (subagents need an escape hatch to signal "planning done"). Permission specs constrain which agent types can be spawned.

##### 7.2 MCP Server Composition

Agents declare MCP servers via two mechanisms:

- **Name reference (string):** Looks up existing server config by name. Shared connection -- not cleaned up when agent finishes.
- **Inline definition (object):** Creates a new connection scoped to the agent. Cleaned up when agent finishes.

Required MCP servers use substring pattern matching with a 30-second polling wait at spawn time.

##### 7.3 Agent Memory

Three persistence scopes:

| Scope | Path Pattern | Sharing | VCS | Use Case |
|-------|-------------|---------|-----|----------|
| user | User home directory | Cross-project | No | Personal learnings, portable across codebases |
| project | Project directory | Team-wide | Yes | Team conventions, project-specific knowledge |
| local | Project-local directory | Machine-only | No | Local paths, API keys, machine-specific context |

**Memory snapshots** prevent accidental loss: project-level snapshots are copied to user-level on first run, and newer snapshots trigger an update prompt.

**Skill isolation:** Invoked skills are keyed by agentId and skillName to prevent cross-agent overwrites in the shared skill map.

##### 7.4 CLAUDE.md Injection

By default, agents receive the full CLAUDE.md hierarchy (managed -> user -> project -> local -> auto-memory). Read-only agents (Explore, Plan) set omitClaudeMd to skip this injection. These agents do not commit, create PRs, or modify code -- they do not need enforcement of rules they will never trigger.

**Token impact:** Typical CLAUDE.md is 500-1000 tokens. Across 34M+ weekly Explore spawns, omitting it saves approximately 5-15 billion tokens per week. Feature-gated for A/B testing and rollback.

##### 7.5 Lifecycle Cleanup

Every agent runs comprehensive cleanup in a finally block, regardless of exit path:

| Resource | Cleanup Action | Consequence if Leaked |
|----------|---------------|----------------------|
| Inline MCP servers | Close connection | Dangling connections, resource exhaustion |
| Git worktree | Remove if no changes detected | Orphaned worktrees, disk consumption |
| Invoked skills | Clear for agent | Stale skill state across agents |
| Background bash tasks | Kill tasks for agent | Zombie processes |
| File state cache | Clear | Memory accumulation over hundreds of agents |
| Tracing | Unregister agent | Telemetry pollution |
| Summarization | Stop | Dangling fork agents |
| Transcript subdir | Clear mapping | Stale mapping |

---

#### 8. State Management

##### 8.1 Application State Agent-Related Fields

The global application state holds:

| Field | Purpose |
|-------|---------|
| tasks | All active tasks (agents, shells, teammates) |
| agentColorMap / agentColorIndex | Per-agent UI color assignment |
| mainThreadAgentType | From CLI flag |
| agentNameRegistry | name-to-agentId mapping for SendMessage routing |
| sessionCreatedTeams | Teams created this session (for cleanup on shutdown) |
| teamContext | Team state (only present when a team is active) |

##### 8.2 Team Context

Present in state only when a team is active. Contains the team name, file path, deterministic lead agent ID, self-identity fields (for teammates), and a map of all teammates with their metadata.

**Why file-based team state?** Not in-memory state (per-session -- team must outlive sessions for reconnection and multi-process coordination). Not a database (overkill for tiny JSON structures, no daemon dependency). File-based provides persistence, inspectability, and cross-process visibility with minimal complexity.

##### 8.3 Task State Lifecycle

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

##### 8.4 User-Facing Task Tracking

Separate from agent task states, the user-facing task system provides structured project management:

- File-based storage with file-level locking.
- Concurrency control via lockfiles (30 retries, exponential backoff) for multi-process swarm access.
- Numeric IDs with high-water mark (monotonically increasing, no reuse after deletion).
- Dependency tracking via blocks/blockedBy arrays for task ordering.

---

#### 9. Resilience and Fault Tolerance

##### 9.1 Error Taxonomy

| Error Type | Cause | Handling | Retryable |
|------------|-------|----------|-----------|
| AbortError | User cancellation, parent abort | Extract partial result, notify | No |
| APIError (401/403) | Authentication failure | Force token refresh, retry | Yes |
| APIError (429) | Rate limit | Exponential backoff (500ms base, 32s cap, 25% jitter) | Yes |
| APIError (529) | Overloaded | Retry for foreground only; background fails immediately | Conditional |
| APIConnectionError | Network (ECONNRESET, EPIPE) | Reset keep-alive pool, acquire fresh client | Yes |
| Context overflow (400) | Message too large | Reduce max_tokens, retry | Yes |
| Generic Error | Unexpected | Log, fail agent task | No |

##### 9.2 Abort/Cancellation Model

Abort signals propagate unidirectionally via linked AbortControllers: parent -> children, never children -> parent. The implementation uses WeakRef to prevent parents from retaining garbage-collected children.

When a task is killed:
1. Abort the agent's AbortController.
2. The query loop detects the abort signal and throws AbortError.
3. The catch handler extracts partial results by walking messages backward.
4. Status transitions to killed with a partial result notification.

##### 9.3 API Retry Strategy

Exponential backoff with configurable max retries (default 10). Key design decisions:

- **529 (overloaded) retries only for foreground.** Background tasks fail immediately on 529 to prevent retry amplification during cascading failures.
- **Persistent retry mode** (internal): 429/529 retry indefinitely with a 5-minute backoff cap and keep-alive heartbeat yields every 30 seconds.
- **Stale connection recovery:** ECONNRESET/EPIPE errors trigger keep-alive pooling reset and fresh client acquisition.

##### 9.4 Partial Result Preservation

When an agent is interrupted, the system walks the message history backward to find the last assistant message with text content. This preserves work-in-progress even when the agent did not reach a clean completion.

##### 9.5 MCP Server Failure Resilience

Individual MCP server failures do not crash the agent. Failed connections are logged as warnings, and tools from that server are simply unavailable. The agent continues with its remaining tools.

##### 9.6 Swarm Resilience

**Teammate reconnection.** Teammates are initialized or resumed from stored team metadata. No active reconnection polling -- the teammate simply re-reads the team file and picks up from its mailbox.

**Leader crash.** No leader election or failover. If the lead crashes, teammates can continue accessing the team file for metadata, but there is no auto-recovery. This is a known design gap, acceptable for the current single-leader team model where sessions are short-lived and human-supervised.

**Worktree cleanup on crash.** Worktrees that were modified before a crash are preserved (not auto-deleted). Resumed worktrees bump their mtime to prevent background cleanup from deleting them.

##### 9.7 Design Philosophy

The resilience model prioritizes operational resilience (fail gracefully, do not cascade, preserve work) over high availability (no failover, no leader election). This reflects the CLI use case: humans supervise small agent teams in interactive sessions, not unattended distributed systems.

Key principles:
- Fail-fast for non-retryable errors.
- Protect background from cascades.
- One-way abort chains.
- Guaranteed cleanup via finally blocks.
- Partial results over total loss.
