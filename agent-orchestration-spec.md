# Agent Orchestration Architecture Spec

Architecture-level specification of Claude Code's multi-agent orchestration system.
Covers agent lifecycle, orchestration patterns, communication protocols, agent definitions, and resource management.

## Table of Contents

1. [Overview & Design Principles](#1-overview--design-principles)
2. [Core Abstractions](#2-core-abstractions)
3. [Agent Lifecycle](#3-agent-lifecycle)
4. [Orchestration Patterns](#4-orchestration-patterns)
5. [Communication Protocols](#5-communication-protocols)
6. [Agent Definition System](#6-agent-definition-system)
7. [Resource Management](#7-resource-management)
8. [State Management](#8-state-management)
9. [Key Source File Index](#9-key-source-file-index)

---

## 1. Overview & Design Principles

Claude Code's orchestration system manages multiple concurrent agents with isolated contexts, independent tool pools, and multiple communication channels. The system supports five distinct orchestration patterns ranging from simple parent-child subagents to full multi-agent swarms.

### Design Principles

- **Generator-based streaming.** Agent execution uses async generators (`async function*`) that yield `Message` objects progressively. This unifies sync and async execution paths under a single consumption model.
- **Context isolation via AsyncLocalStorage.** Concurrent agents each get their own execution context through Node.js `AsyncLocalStorage`, preventing state cross-contamination without process-level isolation.
- **Prompt cache optimization.** Fork subagents construct byte-identical API prefixes across all children so they share prompt cache entries. One-shot built-in agents (Explore, Plan) omit non-essential message trailers to save tokens.
- **Tool pool independence.** Each agent receives its own computed tool pool based on its definition, permission mode, and execution context (sync vs. async, built-in vs. custom).
- **File-based coordination.** Multi-agent swarms use file-based mailboxes with `proper-lockfile` for concurrent access, enabling cross-process communication without shared memory.
- **Graceful lifecycle transitions.** Foreground agents can transition to background mid-execution via a race between the message stream and a background signal. The agent generator continues without restart.

---

## 2. Core Abstractions

### 2.1 AgentDefinition

Union type representing all possible agent sources.

```
BaseAgentDefinition
  agentType:          string              -- Unique identifier (e.g., "Explore", "general-purpose")
  whenToUse:          string              -- Description for LLM tool selection
  tools?:             string[]            -- Allowed tool names (* = wildcard)
  disallowedTools?:   string[]            -- Deny list of tool names
  skills?:            string[]            -- Skills to preload
  mcpServers?:        AgentMcpServerSpec[] -- MCP servers (name refs or inline defs)
  hooks?:             HooksSettings       -- Session-scoped hooks
  model?:             string              -- Model override ("inherit", "sonnet", "opus", "haiku")
  effort?:            EffortValue         -- Reasoning effort level
  permissionMode?:    PermissionMode      -- Permission mode override
  maxTurns?:          number              -- Max agentic turns
  memory?:            AgentMemoryScope    -- "user" | "project" | "local"
  isolation?:         "worktree" | "remote"
  background?:        boolean             -- Always run async
  omitClaudeMd?:      boolean             -- Skip CLAUDE.md injection

BuiltInAgentDefinition = BaseAgentDefinition & {
  source: "built-in"
  getSystemPrompt(toolUseContext) -> string   -- Dynamic prompt generation
}

CustomAgentDefinition = BaseAgentDefinition & {
  source: SettingSource   -- "userSettings" | "projectSettings" | "policySettings" | "flagSettings"
  getSystemPrompt() -> string
}

PluginAgentDefinition = BaseAgentDefinition & {
  source: "plugin"
  plugin: string
  getSystemPrompt() -> string
}

AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

Defined in: `src/tools/AgentTool/loadAgentsDir.ts`

### 2.2 AgentToolInput

Parameters accepted by the AgentTool (externally named "Task" for backward compatibility).

```
AgentToolInput
  description:       string              -- 3-5 word summary
  prompt:            string              -- Full task instruction
  subagent_type?:    string              -- Agent type (omit for fork path)
  model?:            "sonnet" | "opus" | "haiku"
  run_in_background?: boolean            -- Fire-and-forget async
  name?:             string              -- Teammate name (multi-agent)
  team_name?:        string              -- Team name (multi-agent)
  mode?:             PermissionMode      -- "plan" | "acceptEdits" | "bubble"
  isolation?:        "worktree" | "remote"
  cwd?:              string              -- Working directory override
```

### 2.3 AgentToolResult

Result returned from a completed agent.

```
AgentToolResult
  agentId:             string
  agentType?:          string
  content:             Array<{ type: "text", text: string }>
  totalToolUseCount:   number
  totalDurationMs:     number
  totalTokens:         number
  usage:               { input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens }
```

### 2.4 AgentContext (AsyncLocalStorage)

Per-agent execution context, set via `runWithAgentContext()`.

```
SubagentContext
  agentId:             string
  parentSessionId?:    string
  agentType:           "subagent"
  subagentName?:       string
  isBuiltIn?:          boolean
  invokingRequestId?:  string

TeammateAgentContext
  agentId:             string
  agentName:           string
  teamName:            string
  agentColor?:         string
  planModeRequired:    boolean
  parentSessionId:     string
  isTeamLead:          boolean
  agentType:           "teammate"
```

Defined in: `src/utils/agentContext.ts`

### 2.5 Message

The fundamental unit flowing through agent generators. All agent execution produces a stream of `Message` objects consumed by the parent. Key subtypes include `AssistantMessage` (tool use blocks, thinking, text), `UserMessage`, progress events, and system boundary markers.

---

## 3. Agent Lifecycle

All agents follow a four-phase lifecycle: **Spawn -> Init -> Run -> Cleanup**. The init phase branches into sync or async paths, and sync agents can transition to async mid-execution.

### 3.1 Spawn Phase

Entry point: `AgentTool.call()` in `src/tools/AgentTool/AgentTool.tsx`.

1. **Validate constraints.** Check multi-agent feature gates. Reject recursive fork attempts (via `isInForkChild()`).
2. **Route multi-agent.** If `name` + `team_name` are provided, delegate to `spawnTeammate()` and return early.
3. **Resolve agent type.** Determine `effectiveType` from `subagent_type` param. If omitted and fork feature is enabled, take the fork path (implicit `FORK_AGENT`). Otherwise default to `GENERAL_PURPOSE_AGENT`.
4. **Validate agent.** Look up agent in the active agents list. Check `allowedAgentTypes` from `Agent(type1,type2)` permission specs. Verify `requiredMcpServers` are available (poll up to 30s).
5. **Assemble tool pool.** Call `resolveAgentTools()` with the agent's `tools`/`disallowedTools` against available tools filtered for agent context.
6. **Create agent ID.** Stable ID via `createAgentId()`.
7. **Set up isolation.** Optionally create a git worktree for `isolation: "worktree"`.
8. **Build system prompt.** Fork path: inherit parent's rendered system prompt for cache-identical prefix. Normal path: call `agent.getSystemPrompt()` and enhance with environment details.

### 3.2 Init & Branch

After spawn, execution branches based on `run_in_background`:

#### Async Path (Background)

1. Register task via `registerAsyncAgent()` in AppState.
2. Register `name -> agentId` mapping in `agentNameRegistry` (for SendMessage routing).
3. Fire-and-forget: launch `runAsyncAgentLifecycle()` inside `runWithAgentContext()`.
4. Return `{ status: "async_launched", agentId, outputFile }` immediately.

#### Sync Path (Foreground)

1. Register as foreground task via `registerAgentForeground()`.
2. Create `runAgent()` async generator iterator.
3. Enter race loop:
   - After 2s, show background hint UI.
   - Race each `iterator.next()` against `backgroundSignal`.
   - If backgrounded mid-run: stop foreground summarization, transition generator to async lifecycle, return `async_launched`.
   - If message received: accumulate, track progress.
   - If done: finalize and return `{ status: "completed", ...result }`.

### 3.3 Run Phase

Core execution in `runAgent()` (`src/tools/AgentTool/runAgent.ts`).

```
async function* runAgent(params): AsyncGenerator<Message, void>
```

1. Initialize agent MCP servers (connect or reuse shared connections).
2. Register session hooks from agent definition.
3. Load agent memory if configured.
4. Enter query loop: call `query()` repeatedly until abort, max turns, or completion.
5. Yield each `Message` from the stream.
6. On completion or abort, unregister hooks and close agent-specific MCP connections.

### 3.4 Cleanup Phase

Handled by `runAsyncAgentLifecycle()` for async agents (`src/tools/AgentTool/agentToolUtils.ts`).

1. **Stop summarization** if active.
2. **Finalize result** via `finalizeAgentTool()` — extract text content, compute usage metrics.
3. **Mark completed** in AppState (unblocks `TaskOutputTool` consumers).
4. **Post-completion embellishments** (non-blocking):
   - Handoff classification (transcript classifier).
   - Git worktree cleanup (remove if no changes detected).
5. **Enqueue notification** with final message, usage stats, and optional worktree info.
6. **Error handling:**
   - `AbortError` (user cancel): mark killed, enqueue partial result notification.
   - Exception: mark failed, enqueue error notification.
7. **Final cleanup:** clear invoked skills, clear dump state.

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
  |                     finalize -> notify
  |
  +-- sync -----> registerAgentForeground()
                     |
                race loop: next() vs backgroundSignal
                     |
                +-- backgrounded? --> transition to async lifecycle
                |
                +-- completed? --> finalizeAgentTool() -> return result
```

---

## 4. Orchestration Patterns

### 4.1 Sub-Agent Pattern

The basic parent-child relationship. The parent spawns a child agent via `AgentTool`, the child runs to completion, and the parent receives the result.

**Characteristics:**
- Child gets its own system prompt from its `AgentDefinition`.
- Child's tool pool is independently computed (filtered, no AgentTool recursion by default).
- Child runs as a generator; parent consumes messages.
- Sync execution blocks the parent's turn; async returns immediately.
- One-shot agents (Explore, Plan) skip the agentId/SendMessage trailer to save tokens.

**Built-in sub-agent types:**
- `general-purpose` — Full tool access, multi-step tasks.
- `Explore` — Read-only codebase exploration (Glob, Grep, Read, WebFetch, WebSearch). One-shot.
- `Plan` — Architecture planning, read-only tools. One-shot.
- `claude-code-guide` — Documentation lookup, resumable.
- `verification` — Post-implementation verification.
- `statusline-setup` — Statusline configuration (Read, Edit only).

### 4.2 Coordinator Mode

An environment-level mode (not a special agent type) that restructures the main session into a coordinator that delegates to worker agents.

**Activation:** `CLAUDE_CODE_COORDINATOR_MODE` environment variable + `feature('COORDINATOR_MODE')` gate.

**Key characteristics:**
- The coordinator receives a specialized system prompt emphasizing synthesis over delegation.
- Workers are spawned via `AgentTool` and communicated with via `SendMessageTool`.
- Coordinator system prompt prescribes patterns: parallel research, serial implementation per file set, independent verification.
- Core rule: "Do not delegate understanding to workers" — coordinator must synthesize findings.

**Coordinator-available tools:** AgentTool, SendMessageTool, TaskStopTool, TeamCreateTool, TeamDeleteTool.

Defined in: `src/coordinator/coordinatorMode.ts`

### 4.3 Team Swarms

A peer-based multi-agent pattern where a team lead manages multiple teammates that can communicate with each other.

**Team creation flow:**
1. `TeamCreateTool` creates a team file at `~/.claude/teams/{team_name}/config.json`.
2. Generates deterministic `leadAgentId` via `formatAgentId(TEAM_LEAD_NAME, teamName)`.
3. Sets up task list directory for the team.
4. Updates AppState with `teamContext`.

**Team file structure** (`TeamFile`):
```
name:             string
description?:     string
createdAt:        number
leadAgentId:      string              -- "team-lead@teamName"
leadSessionId?:   string
members:          Array<TeamMember>
teamAllowedPaths?: TeamAllowedPath[]
```

Each `TeamMember` includes: `agentId`, `name`, `agentType`, `model`, `prompt`, `color`, `tmuxPaneId`, `cwd`, `worktreePath`, `sessionId`, `subscriptions`, `backendType`, `isActive`, `mode`.

**Execution backends:**

| Backend | Process Model | Communication | Termination |
|---------|--------------|---------------|-------------|
| `in-process` | Same Node.js process, AsyncLocalStorage isolation | File-based mailbox | `AbortController.abort()` |
| `tmux` | Separate tmux panes | File-based mailbox | `tmux kill-pane` |
| `iterm2` | iTerm2 split panes | File-based mailbox | Pane kill |

All backends implement the `TeammateExecutor` interface: `spawn()`, `sendMessage()`, `terminate()`, `kill()`, `isActive()`.

**Teammate identity** is resolved in priority order:
1. `AsyncLocalStorage` (in-process teammates via `getTeammateContext()`)
2. Dynamic team context (tmux teammates via CLI args)
3. Environment variables (fallback)

Key files: `src/tools/TeamCreateTool/TeamCreateTool.ts`, `src/utils/swarm/teamHelpers.ts`, `src/utils/swarm/backends/`

### 4.4 Fork Sub-Agents (Experimental)

Implicit subagent spawning that inherits the parent's full conversation context for maximum prompt cache sharing.

**Characteristics:**
- Triggered when `subagent_type` is omitted and fork feature is enabled.
- Child inherits parent's rendered system prompt verbatim (byte-identical API prefix).
- Child receives parent's full assistant message (all tool_use blocks intact) with identical placeholder tool_results.
- Only the final directive text block differs per child, maximizing cache hits across parallel forks.
- `permissionMode: "bubble"` surfaces permission prompts to parent terminal.
- `model: "inherit"` keeps parent's model for context length parity.
- Recursive forking is blocked via `isInForkChild()` (detects `FORK_BOILERPLATE_TAG` in history).

**Child message format:**
```
[...parent_history, assistant(all_tool_uses), user(placeholder_results..., <fork-boilerplate>directive</fork-boilerplate>)]
```

Child output format is structured: `Scope:`, `Result:`, `Key files:`, `Files changed:`, `Issues:` (max 500 words).

**Context isolation for forks** (`createSubagentContext()` in `src/utils/forkedAgent.ts`):
- `readFileState`: cloned from parent (prevents cache contamination).
- `contentReplacementState`: cloned for cache-sharing.
- `AbortController`: new child controller linked to parent (parent abort propagates).
- All mutation callbacks (`setAppState`, `setResponseLength`) are no-op by default.
- Exception: `setAppStateForTasks` always reaches root for task registration.

Key files: `src/tools/AgentTool/forkSubagent.ts`, `src/utils/forkedAgent.ts`

### 4.5 Async Task Execution

Any agent can be launched as a background task. Additionally, foreground agents can transition to background mid-run.

**Task types in AppState:**

| Task Type | File | Purpose |
|-----------|------|---------|
| `local_agent` | `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | Background subagent execution |
| `local_shell` | `src/tasks/LocalShellTask/LocalShellTask.tsx` | Background shell commands |
| `remote_agent` | `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | Remote (CCR) execution |
| `in_process_teammate` | `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` | In-process swarm member |

**Async agent task state** (`LocalAgentTaskState`):
```
type:              "local_agent"
agentId:           string
prompt:            string
selectedAgent:     AgentDefinition
agentType:         string
model:             string
abortController:   AbortController
error?:            string
result?:           AgentToolResult
progress:          AgentProgress        -- { toolUseCount, tokenCount, lastActivity, recentActivities[] }
isBackgrounded:    boolean
retain:            boolean              -- UI holding task, blocks eviction
messages?:         Message[]            -- In-memory transcript
```

**Progress tracking** uses a `ProgressTracker` that accumulates tool use counts, token metrics, and a circular buffer of the 5 most recent activities (tool name, input, description).

**Output retrieval** via `TaskOutputTool`:
- Input: `task_id`, `block` (wait for completion, default true), `timeout` (default 30s, max 600s).
- Polls AppState every 100ms until task completes or timeout.
- Extracts text content from in-memory result or disk file.

**Disk output** (`src/utils/task/diskOutput.ts`):
- Path: `{projectTempDir}/{sessionId}/tasks/{taskId}.output`
- `DiskTaskOutput` class: async write queue with single drain loop.
- Retrieval: `getTaskOutput(taskId, maxBytes=8MB)` tails file; `getTaskOutputDelta()` reads from offset.
- Symlink support: task output can link to agent transcript file.

---

## 5. Communication Protocols

### 5.1 Parent-Child Results (Generator Stream)

The primary communication channel between parent and child agents. The `runAgent()` async generator yields `Message` objects that the parent consumes:

```
Parent                          Child (runAgent generator)
  |                                |
  |--- spawn ---------------------->|
  |                                |--- query() -> API
  |                                |<-- stream events
  |<-- yield Message --------------|
  |<-- yield Message --------------|
  |                                |--- tool execution
  |<-- yield Message --------------|
  |                                |--- query() -> API (next turn)
  |<-- yield Message --------------|
  |<-- return (done) --------------|
  |                                |
  |--- finalizeAgentTool() -------->|
```

For sync agents, the parent blocks on each `iterator.next()`. For async agents, messages accumulate in a background loop.

### 5.2 Teammate Mailbox (File-Based)

Swarm members communicate through JSON-based inboxes at `~/.claude/teams/{team_name}/inboxes/{agent_name}.json`.

**Message format:**
```
TeammateMessage
  from:        string       -- Sender agent name
  text:        string       -- Message content
  timestamp:   string       -- ISO 8601
  read:        boolean      -- Unread flag
  color?:      string       -- Sender's UI color
  summary?:    string       -- 5-10 word preview
```

**Operations:**
- `writeToMailbox(recipient, message, teamName)` — Write with file lock.
- `readMailbox(agentName, teamName)` — Read all messages.
- `readUnreadMessages(agentName, teamName)` — Filter unread only.
- `markInboxMessageAsRead(agentName, teamName, index)` — Mark read.

**Concurrency:** `proper-lockfile` with 10 retries, 5-100ms exponential backoff. Handles ~10-way concurrent writers.

Defined in: `src/utils/teammateMailbox.ts`

### 5.3 SendMessage Tool

High-level inter-agent messaging API with multiple routing targets and structured message types.

**Input:**
```
to:       string                      -- Teammate name, "*" (broadcast), "uds:<socket>", "bridge:<session-id>"
summary?: string                      -- 5-10 word preview
message:  string | StructuredMessage
```

**Structured message types:**
- `shutdown_request` — Graceful shutdown with optional reason.
- `shutdown_response` — Approve/reject shutdown.
- `plan_approval_response` — Team lead approval/rejection of teammate plan with feedback.

**Routing logic:**
1. **In-process local agents:** Route by name or agentId via `agentNameRegistry` in AppState.
2. **In-process teammates:** Write to file-based mailbox.
3. **Tmux/iTerm2 teammates:** Write to file-based mailbox.
4. **Bridge (inter-Claude):** Send via `postInterClaudeMessage()` to remote session.
5. **UDS (local peer):** Send via `sendToUdsSocket()`.
6. **Auto-resume:** If target agent is stopped, automatically resume with message.

**Broadcast** (`"*"`): Iterates all known teammates and writes to each mailbox.

Defined in: `src/tools/SendMessageTool/SendMessageTool.ts`

### 5.4 Task Notifications

When background agents complete, they enqueue notifications for the main session.

**Notification content:**
```
taskId:        string
description:   string
status:        "completed" | "killed" | "failed"
finalMessage:  string
usage:         { totalTokens, toolUses, durationMs }
worktreePath?: string
worktreeBranch?: string
error?:        string
```

Notifications are enqueued via `enqueueAgentNotification()` and attached to the next user message via `TASK_NOTIFICATION_TAG` XML markers.

### 5.5 Signal Primitive

Lightweight pub/sub used internally for event notification (mailbox arrivals, state changes).

```
Signal<Args>
  subscribe(listener) -> unsubscribe()
  emit(...args) -> void
  clear() -> void
```

Stateless — only notifies current subscribers. ~15 usage sites across the codebase.

Defined in: `src/utils/signal.ts`

### 5.6 Generic Mailbox (In-Process)

A message queue with synchronous and asynchronous consumption.

```
Mailbox
  send(msg: Message) -> void
  poll(predicate?) -> Message | undefined    -- Non-blocking
  receive(predicate?) -> Promise<Message>    -- Blocking wait
  subscribe -> Signal.subscribe
```

Message sources: `"user"`, `"teammate"`, `"system"`, `"tick"`, `"task"`.

Defined in: `src/utils/mailbox.ts`

---

## 6. Agent Definition System

### 6.1 Definition Sources

Agent definitions are loaded from multiple sources with a last-write-wins override strategy.

**Priority order (lowest to highest):**
1. **Built-in** — Programmatic definitions in `src/tools/AgentTool/built-in/`.
2. **Plugin** — Loaded from the plugin system.
3. **User settings** — `~/.claude/agents/*.md`
4. **Project settings** — `.claude/agents/*.md`
5. **Flag settings** — `--agent` CLI flag.
6. **Policy/Managed settings** — Enterprise/organization level.

**Loading function:** `getAgentDefinitionsWithOverrides(cwd)` in `src/tools/AgentTool/loadAgentsDir.ts`.

Returns `{ activeAgents, allAgents, failedFiles }` where `activeAgents` is the deduplicated set and `allAgents` preserves all definitions for override display.

### 6.2 Markdown Agent Format

Custom agents are defined as `.md` files with YAML frontmatter:

```yaml
---
name: my-agent           # Required: agent type identifier
description: When to use  # Required: selection guidance for LLM
tools:                    # Optional: allowed tools (* = all)
  - Read
  - Grep
  - Glob
disallowedTools:          # Optional: deny list
  - Bash
model: sonnet             # Optional: model override
permissionMode: plan      # Optional: permission mode
mcpServers:               # Optional: MCP server specs
  - slack                 # Name reference
  - myserver:             # Inline definition
      command: npx
      args: ["-y", "my-mcp-server"]
maxTurns: 50              # Optional: turn limit
memory: project           # Optional: memory scope
isolation: worktree       # Optional: execution isolation
background: true          # Optional: always async
---

System prompt content goes here.
This becomes the agent's system prompt via getSystemPrompt().
```

### 6.3 Deduplication

When multiple sources define agents with the same `agentType`, later sources override earlier ones:

```
getActiveAgentsFromList(allAgents):
  for each group in [builtIn, plugin, user, project, flag, managed]:
    for each agent in group:
      agentMap.set(agent.agentType, agent)   // last write wins
  return Array.from(agentMap.values())
```

For display purposes, `resolveAgentOverrides()` deduplicates by `(agentType, source)` composite key to handle git worktree duplicates, and annotates overridden agents with the overriding source.

### 6.4 Agent Resolution at Call Time

When `AgentTool` is invoked:

1. Filter `allAgents` by `allowedAgentTypes` (from `Agent(type1,type2)` permission rule content).
2. Filter by `getDenyRuleForAgent()` permission checks.
3. Look up by `effectiveType` match.
4. If not found in filtered set but exists in `allAgents`, report "denied by permission rule".
5. If not found at all, report "agent not found".

### 6.5 Built-In Agent Registry

```
BUILT_IN_AGENTS = [
  GENERAL_PURPOSE_AGENT   -- "general-purpose", tools: [*]
  EXPLORE_AGENT           -- "Explore", tools: [Glob, Grep, Read, WebFetch, WebSearch, ...] (read-only)
  PLAN_AGENT              -- "Plan", tools: [Glob, Grep, Read, WebFetch, WebSearch, ...] (read-only)
  CLAUDE_CODE_GUIDE_AGENT -- "claude-code-guide", tools: [Glob, Grep, Read, WebFetch, WebSearch]
  VERIFICATION_AGENT      -- "verification"
  STATUSLINE_SETUP_AGENT  -- "statusline-setup", tools: [Read, Edit]
]
```

One-shot agent types (Explore, Plan) are tracked in `ONE_SHOT_BUILTIN_AGENT_TYPES` — these skip the agentId/SendMessage result trailer to save ~135 bytes per invocation.

Defined in: `src/tools/AgentTool/builtInAgents.ts`, `src/tools/AgentTool/constants.ts`

---

## 7. Resource Management

### 7.1 Tool Resolution

Each agent gets an independently computed tool pool via `resolveAgentTools()`.

**Resolution algorithm:**

1. **Filter available tools** based on agent context:
   - MCP tools (`mcp__*`) always pass.
   - `ALL_AGENT_DISALLOWED_TOOLS` are removed for all agents (TaskOutputTool, ExitPlanModeTool, EnterPlanModeTool, AskUserQuestionTool, TaskStopTool).
   - `CUSTOM_AGENT_DISALLOWED_TOOLS` are additionally removed for non-built-in agents.
   - Async agents are restricted to `ASYNC_AGENT_ALLOWED_TOOLS` (file ops, search, shell, web).
   - In-process teammates get additional tools (`IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`: task CRUD, SendMessage).

2. **Apply agent's disallowedTools** deny list.

3. **Resolve agent's tools spec:**
   - `undefined` or `["*"]`: wildcard — all filtered tools allowed.
   - Explicit list: resolve each name against filtered pool, track invalid specs.
   - `Agent(type1,type2)` specs: extract `allowedAgentTypes` metadata for recursive agent filtering.

**Key constant sets:**

| Set | Applies To | Examples |
|-----|-----------|----------|
| `ALL_AGENT_DISALLOWED_TOOLS` | All agents | TaskOutputTool, EnterPlanModeTool, AskUserQuestionTool |
| `CUSTOM_AGENT_DISALLOWED_TOOLS` | Non-built-in agents | AgentTool (prevents custom agents spawning sub-agents) |
| `ASYNC_AGENT_ALLOWED_TOOLS` | Background agents | Read, Edit, Write, Bash, Glob, Grep, WebSearch, WebFetch, Skill |
| `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` | Swarm teammates | TaskCreate, TaskGet, TaskList, TaskUpdate, SendMessage |

Defined in: `src/tools/AgentTool/agentToolUtils.ts`, `src/constants/tools.ts`

### 7.2 MCP Server Composition

Agents can declare MCP servers via two mechanisms:

1. **Name reference** (`string`): Looks up existing server config by name. Shared connection — not cleaned up when agent finishes.
2. **Inline definition** (`{ name: config }`): Creates a new connection scoped to the agent. Cleaned up when agent finishes.

**Initialization flow** (in `runAgent()`):
1. For each spec in `agentDefinition.mcpServers`:
   - String ref: look up config via `getMcpConfigByName()`, connect (memoized).
   - Inline def: create scoped config, connect (new).
2. Fetch tools from each connected server.
3. Merge agent-specific MCP tools into the tool pool.

**Cleanup:** Only newly created (inline) clients are disconnected when the agent completes. Shared (name-ref) clients persist for the session.

**Required MCP servers:** Agents can declare `requiredMcpServers` — pattern-matched (case-insensitive substring) against available servers. The agent spawn blocks (polls up to 30s) until all required servers are connected.

### 7.3 Agent Memory

Agents can persist knowledge across sessions via scoped MEMORY.md files.

**Scopes:**

| Scope | Path | Shared | VCS |
|-------|------|--------|-----|
| `user` | `~/.claude/agent-memory/<agentType>/MEMORY.md` | Cross-project | No |
| `project` | `.claude/agent-memory/<agentType>/MEMORY.md` | Team-wide | Yes |
| `local` | `.claude/agent-memory-local/<agentType>/MEMORY.md` | Machine-only | No |

**Memory snapshots** (feature-gated: `AGENT_MEMORY_SNAPSHOT`): Copy project-level snapshots to user-level on first load. Detect newer snapshots for user notification.

**Skill isolation:** Invoked skills are keyed by `${agentId}:${skillName}` to prevent cross-agent overwrites. `getInvokedSkillsForAgent(agentId)` filters the global map.

Defined in: `src/tools/AgentTool/agentMemory.ts`, `src/tools/AgentTool/agentMemorySnapshot.ts`

### 7.4 CLAUDE.md Injection

By default, agents receive the full CLAUDE.md hierarchy (managed -> user -> project -> local -> auto-memory). Read-only agents (Explore, Plan) set `omitClaudeMd: true` to skip this injection, saving significant token overhead across high-volume spawns.

### 7.5 Lifecycle Cleanup

| Resource | Cleanup Trigger | Mechanism |
|----------|----------------|-----------|
| Inline MCP servers | Agent completion | `client.cleanup()` |
| Git worktree | Agent completion | Remove if no changes detected |
| Invoked skills | Agent completion | `clearInvokedSkillsForAgent(agentId)` |
| Dump state | Agent completion | `clearDumpState(agentId)` |
| Summarization | Agent completion/error | `stopSummarization()` |
| AbortController | User cancel / error | `abort()` propagates to child |

---

## 8. State Management

### 8.1 AppState Agent-Related Fields

The global `AppState` (React-style setState with functional updates) holds:

```
tasks:                  Record<string, TaskState>      -- All active tasks (agents, shells, teammates)
agentColorMap:          Map<string, AgentColorName>    -- Per-agent color assignment
agentColorIndex:        number                         -- Color rotation counter
mainThreadAgentType?:   string                         -- From --agent flag
agentNameRegistry:      Map<string, string>            -- name -> agentId mapping for SendMessage
sessionCreatedTeams:    Set<string>                    -- Teams created this session (for cleanup)
```

### 8.2 Team Context

Present in AppState only when a team is active:

```
teamContext
  teamName:        string
  teamFilePath:    string
  leadAgentId:     string              -- Deterministic "team-lead@teamName"
  selfAgentId?:    string              -- This session's agent ID (if teammate)
  selfAgentName?:  string
  isLeader?:       boolean
  selfAgentColor?: string
  teammates:       Record<string, TeammateInfo>
```

Each `TeammateInfo`: `name`, `agentType`, `color`, `tmuxSessionName`, `tmuxPaneId`, `cwd`, `worktreePath`, `spawnedAt`.

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
              evictTerminalTask()  (after notification consumed)
```

### 8.4 Task Tracking (User-Facing Tasks)

Separate from agent task states, the user-facing task system (`src/utils/tasks.ts`) provides structured task lists for project management:

```
Task
  id:          string
  subject:     string
  description: string
  activeForm?: string              -- Present-continuous form for UI spinner
  owner?:      string
  status:      "pending" | "in_progress" | "completed"
  blocks:      string[]            -- Task IDs this task blocks
  blockedBy:   string[]            -- Task IDs blocking this task
  metadata?:   Record<string, unknown>
```

File-based storage at `~/.claude/tasks/{taskListId}/{taskId}.json` with file-level locking (30 retries, exponential backoff). Task IDs are numeric strings with high-water-mark to prevent reuse after deletion.

---

## 9. Key Source File Index

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
