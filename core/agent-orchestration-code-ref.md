# Agent Orchestration — Code Reference

### Agent Tool Core

| File | Purpose |
|------|---------|
| `src/tools/AgentTool/AgentTool.tsx` | Main tool definition, spawn logic, sync/async branching. Entry point: `AgentTool.call()`. |
| `src/tools/AgentTool/runAgent.ts` | Agent execution generator (`runAgent()` async generator), query loop, MCP init. |
| `src/tools/AgentTool/agentToolUtils.ts` | Tool filtering, resolution, async lifecycle (`runAsyncAgentLifecycle()`), finalization (`finalizeAgentTool()`). Constants: `ALL_AGENT_DISALLOWED_TOOLS`, `CUSTOM_AGENT_DISALLOWED_TOOLS`, `ASYNC_AGENT_ALLOWED_TOOLS`, `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`. |
| `src/tools/AgentTool/forkSubagent.ts` | Fork agent definition, message construction with byte-identical placeholders, recursion guard via boilerplate tag detection. |
| `src/tools/AgentTool/resumeAgent.ts` | Resume existing agent sessions. |
| `src/tools/AgentTool/loadAgentsDir.ts` | Agent definition loading from `.md` files, YAML frontmatter parsing, deduplication (complete replacement, not merging). Defines `AgentDefinition` union type (`BuiltInAgentDefinition`, `CustomAgentDefinition`, `PluginAgentDefinition`). Returns `allAgents` with shadowed definitions preserved. |
| `src/tools/AgentTool/agentDisplay.ts` | Override resolution, display formatting. |
| `src/tools/AgentTool/agentMemory.ts` | Agent memory scope management. Paths: `~/.claude/agent-memory/<agentType>/MEMORY.md` (user), `.claude/agent-memory/<agentType>/MEMORY.md` (project), `.claude/agent-memory-local/<agentType>/MEMORY.md` (local). |
| `src/tools/AgentTool/agentMemorySnapshot.ts` | Memory snapshot sync between project and user levels. |
| `src/tools/AgentTool/agentColorManager.ts` | Color assignment per agent (stored in `AppState.agentColorMap` / `agentColorIndex`). |
| `src/tools/AgentTool/builtInAgents.ts` | Built-in agent registry. |
| `src/tools/AgentTool/constants.ts` | Agent tool constants, one-shot types list. |
| `src/tools/AgentTool/prompt.ts` | Agent listing prompt generation. |

### Built-In Agent Definitions

| File | Agent Type |
|------|-----------|
| `src/tools/AgentTool/built-in/generalPurposeAgent.ts` | `general-purpose` -- tools: `["*"]` |
| `src/tools/AgentTool/built-in/exploreAgent.ts` | `Explore` -- tools: `[Glob, Grep, Read, WebFetch, WebSearch]`, `omitClaudeMd: true`, one-shot (skips trailer) |
| `src/tools/AgentTool/built-in/planAgent.ts` | `Plan` -- read-only tools, `omitClaudeMd: true`, one-shot (skips trailer) |
| `src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts` | `claude-code-guide` -- read-only + docs tools, resumable |
| `src/tools/AgentTool/built-in/verificationAgent.ts` | `verification` -- configurable tools |
| `src/tools/AgentTool/built-in/statuslineSetup.ts` | `statusline-setup` -- tools: `[Read, Edit]` |

### Communication and Messaging

| File | Purpose |
|------|---------|
| `src/tools/SendMessageTool/SendMessageTool.ts` | Inter-agent messaging with 5 routing targets. Uses `agentNameRegistry` in AppState for name-to-agentId resolution. Routes: in-process local (name registry), teammate (file mailbox), bridge (`postInterClaudeMessage()`), UDS (Unix domain socket), broadcast (`"*"`). Structured message types: `shutdown_request`, `shutdown_response`, `plan_approval_response`. |
| `src/utils/teammateMailbox.ts` | File-based team inbox at `~/.claude/teams/{team_name}/inboxes/{agent_name}.json`. Uses `proper-lockfile` with config: 10 retries, 5-100ms exponential backoff with jitter. Message fields: sender name, text content, ISO timestamp, read flag, sender color, 5-10 word summary. |
| `src/utils/mailbox.ts` | Generic in-process message queue. Methods: `poll()` (non-blocking), `receive()` (blocking with predicate). Message sources: `"user"`, `"teammate"`, `"system"`, `"tick"`, `"task"`. Team lead message priority over peer messages. |
| `src/utils/signal.ts` | Lightweight pub/sub. `subscribe()` returns unsubscribe function; `emit()` notifies current subscribers. ~15 usage sites. |

### Team / Swarm

| File | Purpose |
|------|---------|
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | Team creation. Creates config at `~/.claude/teams/{team_name}/config.json`. Generates deterministic `leadAgentId` via `formatAgentId("team-lead", teamName)`. |
| `src/tools/TeamDeleteTool/TeamDeleteTool.ts` | Team cleanup. |
| `src/coordinator/coordinatorMode.ts` | Coordinator mode detection via `CLAUDE_CODE_COORDINATOR_MODE` env var. Coordinator system prompt generation. Available tools: AgentTool, SendMessageTool, TaskStopTool, TeamCreateTool, TeamDeleteTool. Summarization: periodic fork every 30s generating 3-5 word progress descriptions. |
| `src/utils/swarm/teamHelpers.ts` | Team file I/O, member management. |
| `src/utils/swarm/backends/types.ts` | `TeammateExecutor` interface: `spawn`, `sendMessage`, `terminate`, `kill`, `isActive`. |
| `src/utils/swarm/backends/InProcessBackend.ts` | In-process teammate lifecycle. |
| `src/utils/swarm/backends/TmuxBackend.ts` | Tmux pane management (~200ms shell startup overhead). |
| `src/utils/swarm/backends/ITermBackend.ts` | iTerm2 pane management via AppleScript. Requires `it2` CLI. macOS-only. |
| `src/utils/swarm/backends/PaneBackendExecutor.ts` | Shared pane backend logic. |
| `src/utils/swarm/backends/detection.ts` | Backend type detection. Priority: tmux (if inside tmux) -> iTerm2 (if `it2` available) -> external tmux -> error. |
| `src/utils/swarm/backends/registry.ts` | Backend registry. |
| `src/utils/swarm/spawnInProcess.ts` | In-process teammate spawning. |
| `src/utils/swarm/spawnUtils.ts` | Spawn utilities. |
| `src/utils/swarm/inProcessRunner.ts` | In-process teammate execution loop. |
| `src/utils/swarm/leaderPermissionBridge.ts` | Permission bridging for teammates. |
| `src/utils/swarm/permissionSync.ts` | Permission synchronization. |
| `src/utils/swarm/reconnection.ts` | Teammate reconnection from stored metadata. |
| `src/utils/swarm/teammateInit.ts` | Teammate initialization. |
| `src/utils/swarm/teammateLayoutManager.ts` | Pane layout management. |
| `src/utils/swarm/teammateModel.ts` | Teammate model selection. |
| `src/utils/swarm/teammatePromptAddendum.ts` | Teammate prompt modifications. |

### Context and Identity

| File | Purpose |
|------|---------|
| `src/utils/agentContext.ts` | AsyncLocalStorage context. Two variants: `SubagentContext` (agentId, parentSessionId, isBuiltIn) and `TeammateAgentContext` (agentName, teamName, isTeamLead, planModeRequired). Set via `runWithAgentContext()`, accessed via `AsyncLocalStorage.getStore()`. |
| `src/utils/teammate.ts` | Teammate identity resolution. |
| `src/utils/teammateContext.ts` | Teammate-specific context type. |
| `src/utils/forkedAgent.ts` | Fork context isolation. `readFileState` cloning. Mutation suppression: `setAppState`, `setResponseLength` no-op. Exception: `setAppStateForTasks` always reaches root store. Linked AbortControllers via `WeakRef`. |

### Task System

| File | Purpose |
|------|---------|
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | Background agent task state, `ProgressTracker` with tool use counts, token metrics, circular buffer (5 recent activities). |
| `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` | In-process teammate task state. |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | Remote agent task state. |
| `src/tasks/LocalShellTask/LocalShellTask.tsx` | Background shell task state. |
| `src/tasks/types.ts` | Task type definitions: `local_agent`, `local_shell`, `remote_agent`, `in_process_teammate`. States: `pending`, `in_progress`, `completed`, `killed`, `failed`. |
| `src/tasks/stopTask.ts` | Task termination via `stopTask()`. Aborts agent's AbortController. |
| `src/tools/TaskOutputTool/TaskOutputTool.tsx` | Task output retrieval. Polls every 100ms, default timeout 30s, max 600s. Disk output tiers: in-memory (0-8MB), disk spill (8MB-5GB), truncation (>5GB). |
| `src/tools/TaskStopTool/TaskStopTool.ts` | Task stop tool. |
| `src/utils/tasks.ts` | User-facing task CRUD at `~/.claude/tasks/{taskListId}/{taskId}.json`. `proper-lockfile` with 30 retries, exponential backoff. Numeric IDs with high-water mark. Dependency tracking: `blocks`/`blockedBy` arrays. |
| `src/utils/task/framework.ts` | Task state updates (`registerTask()`), eviction (`evictTerminalTask()` with 30s grace period, `evictAfter` timestamp, `retain` flag). |
| `src/utils/task/diskOutput.ts` | Disk-based task output management. Stdout/stderr go directly to file descriptor for shell commands. |
| `src/utils/task/sdkProgress.ts` | SDK progress event emission. |

### Multi-Agent Spawning

| File | Purpose |
|------|---------|
| `src/tools/shared/spawnMultiAgent.ts` | Multi-agent spawn orchestration. |

### Specific Constants and Identifiers

- Tool name mapping: AgentTool is externally named "Task" for backward compatibility.
- Agent ID format for swarms: `agentName@teamName` (e.g., `researcher@my-project`).
- Lead agent ID: generated via `formatAgentId("team-lead", teamName)`.
- Mailbox path: `~/.claude/teams/{teamName}/inboxes/{agentName}.json`.
- Team config: `~/.claude/teams/{team_name}/config.json`.
- User memory: `~/.claude/agent-memory/<agentType>/MEMORY.md`.
- Project memory: `.claude/agent-memory/<agentType>/MEMORY.md`.
- Local memory: `.claude/agent-memory-local/<agentType>/MEMORY.md`.
- User tasks: `~/.claude/tasks/{taskListId}/{taskId}.json`.
- Feature gate for CLAUDE.md omission: `tengu_slim_subagent_claudemd`.
- Environment variable for coordinator mode: `CLAUDE_CODE_COORDINATOR_MODE`.
- Skill isolation key format: `${agentId}:${skillName}`.
- One-shot trailer savings: ~135 bytes/invocation.
- Summarization interval: 30 seconds.
- Auto-background hint: 2 seconds.
- Auto-background timer: default 2 minutes.
- Lockfile config (teammate mailbox): 10 retries, 5-100ms exponential backoff with jitter.
- Lockfile config (user tasks): 30 retries, exponential backoff.
- Task output poll interval: 100ms.
- Task output default timeout: 30s, max 600s.
- Disk output tiers: 0-8MB (in-memory), 8MB-5GB (disk), >5GB (truncation).
- Eviction grace period: 30 seconds.
- Retry backoff: 500ms base, 32s cap, 25% jitter, default 10 max retries.
- Persistent retry: 5-minute backoff cap, 30s heartbeat.
- `extractPartialResult()`: walks messages backward for last assistant text content.
- Cleanup functions: `client.cleanup()`, `clearInvokedSkillsForAgent()`, `killShellTasksForAgent()`, `readFileState.clear()`, `unregisterPerfettoAgent()`, `stopSummarization()`, `clearAgentTranscriptSubdir()`.

### Interface Boundaries (out of scope, referenced for completeness)

| File | Role |
|------|------|
| `src/utils/permissions/`, `src/hooks/toolPermission/` | Permission pipeline -- agents interact via `permissionMode` and `canUseTool`. |
| `src/services/mcp/` | MCP protocol -- agents interact via `mcpServers` spec and tool namespacing. |
| `src/query.ts` | Query loop -- agents call this to execute API turns. |
