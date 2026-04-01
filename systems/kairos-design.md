# KAIROS — Design Document

### 1. Overview: What KAIROS Is

KAIROS is Claude Code's **autonomous assistant mode** -- a specialized operating mode that transforms the CLI from a traditional interactive tool into a long-lived, self-directed agent. Where normal Claude Code sessions are user-driven (the user types a prompt, the agent responds, the user types another prompt), KAIROS-mode sessions are **agent-driven**: the agent works autonomously, communicates results asynchronously via a structured messaging tool (`SendUserMessage`), and persists across CLI restarts.

The key architectural differences from standard mode:

| Property | Standard Mode | KAIROS Mode |
|----------|--------------|-------------|
| Session lifetime | Single CLI invocation | Perpetual (survives restarts) |
| User output | Direct text streaming | `SendUserMessage` tool calls |
| Memory model | `MEMORY.md` (rewritable index) | Append-only daily logs + nightly distillation |
| Agent spawning | Explicit `TeamCreate` required | Pre-seeded team, `Agent(name:)` spawns directly |
| Bridge behavior | New session per invocation | Perpetual session continuity via bridge pointer |
| Background consolidation | autoDream (forked subagent) | Replaced by disk-skill dream |
| System prompt | Standard prompts only | Standard + assistant addendum appended |

KAIROS mode does **not** change the permission system. The user's chosen permission mode (`default`, `auto`, `plan`, etc.) applies identically -- KAIROS only forces brief mode on and adds its system prompt addendum.

### 2. Naming and Etymology

**KAIROS** (kairos) is the ancient Greek word for "the right, critical, or opportune moment" -- as opposed to *chronos* (chronos), which denotes sequential, measurable time. The name captures the system's design intent: an agent that recognizes **when** to take action and communicate, working asynchronously rather than in lockstep with user input.

Identifier conventions in the codebase:
- `feature('KAIROS')` -- the build-time feature flag
- `kairosActive` -- the runtime session state boolean
- `kairosEnabled` -- computed during startup, stored in AppState
- `tengu_kairos` -- the GrowthBook entitlement gate name
- All sub-feature gates prefixed with `tengu_kairos_*`

### 3. Activation Lifecycle

KAIROS activation follows a multi-gate pipeline. Every gate must pass for the mode to activate. The gates are evaluated in order from cheapest to most expensive.

#### 3.1 Gate Pipeline

```
feature('KAIROS')                          <-- Build-time dead-code gate
    | false -> all KAIROS code stripped, no runtime cost
    v true
isAssistantMode()                          <-- Settings check (.claude/settings.json)
    | false -> normal mode
    v true (assistant: true OR --assistant flag)
checkHasTrustDialogAccepted()              <-- Security trust gate
    | false -> console.warn, refuse activation
    v true (directory explicitly trusted)
!options.agentId                           <-- Not a spawned teammate (compound conditional)
    | false -> teammate mode (inherits leader's state)
isAssistantForced() || isKairosEnabled()   <-- Entitlement gate (GrowthBook or daemon)
    | false -> normal mode (not entitled)
    v true
setKairosActive(true)                      <-- Latch on
opts.brief = true                          <-- Force brief mode
initializeAssistantTeam()                  <-- Pre-seed team context
```

#### 3.2 Activation Paths

There are three distinct paths to KAIROS activation:

**Path 1: Settings-based (interactive user)**
User places `"assistant": true` in `.claude/settings.json` and launches `claude` normally. The settings flag triggers `isAssistantMode()`, then the GrowthBook gate `tengu_kairos` is checked. If the gate is open and the directory is trusted, KAIROS activates.

**Path 2: Daemon-forced (`--assistant` flag)**
Used by the Agent SDK daemon mode. The `--assistant` flag marks the session as forced, bypassing the GrowthBook gate entirely -- the daemon has already verified entitlement before spawning the child process.

**Path 3: Viewer client (`claude assistant [sessionId]`)**
This path does **not** activate KAIROS on the local process. Instead it opens a **read-only REPL viewer** that streams events from a remote assistant session. The agentic loop runs remotely; the local process only renders output and posts user messages.

#### 3.3 Teammate Exclusion

When KAIROS spawns teammates via `Agent(name: "foo")`, those child processes share the leader's `cwd` and `.claude/settings.json`. This means `isAssistantMode()` would return `true` for them too. The activation pipeline explicitly excludes spawned teammates by checking `!options.agentId` -- if an agent ID is set, the process is a spawned teammate and skips KAIROS re-initialization.

### 4. The SendUserMessage Tool

#### 4.1 Purpose

In standard Claude Code, the model's text output streams directly to the user's terminal. In KAIROS mode, the agent's primary communication channel is the `SendUserMessage` tool (internally called "Brief"). This structured tool call replaces free-form text streaming, enabling:

- Asynchronous message delivery (agent works while user is away)
- Structured metadata (status, attachments)
- Channel relay (messages forwarded to Slack, Discord, etc.)
- IDE bridge rendering

#### 4.2 Entitlement Chain

```
feature('KAIROS') || feature('KAIROS_BRIEF')     <-- Build-time gate
    v
getKairosActive()                                 <-- Session state
  || env var bypass                               <-- Dev bypass
  || GrowthBook gate (tengu_kairos_brief)         <-- Runtime gate
    v
isBriefEntitled() = true
    v
(kairosActive || userMsgOptIn) && isBriefEntitled()
    v
isBriefEnabled() = true -> SendUserMessage tool available
```

#### 4.3 Opt-In Paths

SendUserMessage can be activated via six distinct paths:

1. **KAIROS mode** -- automatic (`kairosActive -> true`)
2. **`--brief` CLI flag** -- explicit opt-in
3. **`defaultView: 'chat'` in settings** -- persisted preference
4. **`/brief` slash command** -- mid-session toggle
5. **`--tools SendUserMessage`** -- SDK explicit tool listing
6. **`CLAUDE_CODE_BRIEF=1` env var** -- development bypass

#### 4.4 System Prompt Integration

When brief mode is active within a proactive context, the system prompt dynamically adjusts its visibility instruction to tell the model to use SendUserMessage at checkpoints, versus informing it that text output is directly visible to the user.

#### 4.5 Three-Mode UI Rendering

- **Transcript mode**: Black circle gutter + markdown + attachments
- **Brief-only (chat) mode**: "Claude" label, 2-column indent, optional timestamp
- **Default mode**: No gutter mark, spacer, plain markdown. Redundant assistant text in turns that called Brief is dropped.

#### 4.6 Message Filtering Pipeline

1. **Brief-only filter**: In brief-only mode, drops all assistant messages except Brief tool_use blocks. Keeps system messages (except `api_metrics`), user input, and matching tool_results.
2. **Brief-turn text drop**: In default mode, finds turns containing a Brief tool_use and drops assistant text blocks from those turns (the Brief output replaces the text).

#### 4.7 `/brief` Slash Command

The `/brief` command toggles brief-only mode on AppState and opt-in on bootstrap state. When KAIROS is active, the meta-message injection is skipped (KAIROS system prompt already mandates SendUserMessage). Entitlement check only gates the on-transition -- off is always allowed.

### 5. Memory System: Daily-Log Mode

#### 5.1 Design Rationale

Standard Claude Code uses `MEMORY.md` as a rewritable index -- the agent reads and rewrites it each session. KAIROS sessions are **effectively perpetual**, so rewriting MEMORY.md on every interaction would cause churn and conflicts. Instead, KAIROS uses an **append-only daily log** with periodic distillation.

#### 5.2 Architecture

```
.claude/auto_mem/
+-- MEMORY.md              <-- Distilled index (read-only for agent, nightly-written)
+-- logs/
|   +-- YYYY/
|       +-- MM/
|           +-- YYYY-MM-DD.md   <-- Append-only daily log (agent writes here)
+-- topics/
    +-- *.md               <-- Topic files (nightly distillation output)
```

The agent appends timestamped bullets to today's log file. A separate nightly `/dream` skill reads accumulated logs and distills them into `MEMORY.md` and topic files.

#### 5.3 Prompt Construction

The daily-log prompt replaces the standard memory prompt when KAIROS is active. The prompt instructs the agent to:
- Append to `logs/YYYY/MM/YYYY-MM-DD.md` (path pattern, not literal -- prompt is cached)
- Use timestamped bullets
- Create directories on first write
- Never rewrite or reorganize the log
- Log user corrections, preferences, project context, external system pointers

#### 5.4 Date Rollover

The log path is specified as a **pattern** rather than today's literal date because the system prompt is cached and is **not invalidated on date change**. The model derives the current date from a `date_change` attachment appended at midnight rollover, rather than from the user-context message (which is intentionally left stale to preserve the prompt cache prefix).

#### 5.5 Incompatibility with Team Memory Sync

KAIROS daily-log mode explicitly takes precedence over team memory sync. The append-only log paradigm does not compose with team sync, which expects a shared `MEMORY.md` that both sides read and write.

#### 5.6 Past-Context Search

When enabled via a feature gate, the memory prompt includes instructions for searching past context:
1. **Memory directory search:** grep through `.claude/auto_mem/` markdown files
2. **Transcript logs (last resort):** grep through project JSONL transcript files

#### 5.7 What NOT to Save (Exclusion Rules)

Five categories of information are excluded from memory, even when the user explicitly asks:
- Code patterns, architecture, file paths (derivable from code)
- Git history (authoritative via `git log` / `git blame`)
- Debugging solutions (fix is in the code, context in commit message)
- Anything in CLAUDE.md files
- Ephemeral task details

The agent should ask what was surprising or non-obvious instead.

### 6. Team Spawning and Multi-Agent Coordination

#### 6.1 Pre-Seeded Team Context

KAIROS pre-seeds an in-process team during activation so that `Agent(name: "foo")` can spawn teammates without an explicit `TeamCreate` call. This must happen BEFORE the teammate mode snapshot is captured.

#### 6.2 Team Context Precedence

KAIROS team context takes priority over the default computed context.

#### 6.3 Session Types

| Type | Description | Agent ID | KAIROS Init |
|------|-------------|-----------|-------------|
| **Leader** | Main assistant daemon | undefined | Full init |
| **Teammate** | Spawned via `Agent(name:)` | Set by leader | Skipped (inherits) |
| **Viewer** | `claude assistant [id]` | N/A | No agentic loop |

#### 6.4 Teammate Mode Types

`'auto' | 'tmux' | 'in-process'`

The mode is captured at startup as an immutable snapshot for the session. CLI override takes precedence.

#### 6.5 Spawn Backend Selection

The spawn handler dispatches based on mode:
- If in-process enabled -> spawn in-process
- If backend available -> split-pane or separate-window
- If auto mode and no backend -> fallback to in-process
- If explicit tmux mode and no backend -> error with install instructions

#### 6.6 In-Process Spawning Lifecycle

1. Generate unique name and deterministic agent ID
2. Assign teammate color
3. Create independent AbortController
4. Create teammate context for AsyncLocalStorage
5. Register task
6. Start execution loop (fire-and-forget)
7. Update AppState teammates map
8. Auto-register leader if no prior spawnTeam call

#### 6.7 Teammate State Inheritance

- **tmux**: CLI args propagation (agent-id, agent-name, team-name, agent-color, parent-session-id)
- **in-process**: AsyncLocalStorage context isolation
- Both: Excluded from KAIROS re-initialization by agent ID check

### 7. Perpetual Bridge Sessions

#### 7.1 Problem

In standard Claude Code, each CLI invocation creates a new bridge session. When the process exits, the bridge session is archived and deregistered. This means claude.ai shows separate conversation threads for each CLI run.

#### 7.2 KAIROS Solution

KAIROS enables **perpetual bridge sessions** -- a single continuous conversation thread that survives CLI restarts.

#### 7.3 Bridge Pointer Mechanism

Perpetual mode uses a bridge pointer file to persist session identity across restarts:

```
.claude/projects/<sanitized-dir>/bridge-pointer.json
-> { sessionId, environmentId, source: 'standalone' | 'repl' }
```

**Standard mode:** Writes pointer after session create; clears on teardown (crash-recovery only).

**Perpetual mode:** Reads pointer at init to resume prior session; **skips** archive/deregister/pointer-clear at teardown. The pointer survives clean exits, not just crashes.

**TTL:** 4 hours. Stale pointers are cleared on read. An hourly refresh timer prevents TTL expiry on long-running sessions.

#### 7.4 Teardown Differences

```
Standard mode teardown:        KAIROS perpetual teardown:
1. Archive session             1. Leave session alive on server
2. Deregister environment      2. Leave environment alive
3. Clear bridge-pointer.json   3. Keep bridge-pointer.json (refresh mtime)
4. Done                        4. Log: "leaving env=X session=Y alive"
```

#### 7.5 Reconnect Strategy

Two-strategy recovery when environment is lost:
- **Strategy 1**: Idempotent re-register with prior environment ID
- **Strategy 2**: Archive old session, create fresh session on new environment

Maximum reconnection attempts are capped with counter reset on success. Multiple abort-signal checks prevent zombie operations during teardown.

#### 7.6 Worktree-Aware Pointer Fanout

When `--continue` is invoked from a different git worktree, the system scans worktree siblings for bridge pointers. Fanout is capped to bound I/O.

### 8. Channel Notifications

#### 8.1 Concept

Channel notifications allow KAIROS to relay messages through external communication platforms (Slack, Discord, SMS, Telegram, etc.) via MCP servers. A "channel" is an MCP server that:
1. Exposes tools for **outbound** messages (e.g., `send_message`) -- standard MCP
2. Sends `notifications/claude/channel` notifications for **inbound** messages

#### 8.2 Notification Protocol

| Method | Direction | Purpose |
|--------|-----------|---------|
| `notifications/claude/channel` | Inbound | External messages arriving for the agent |
| `notifications/claude/channel/permission` | Inbound | Permission decisions relayed from channels |

#### 8.3 Gating (Seven-Gate Access Control)

Channel servers must pass all seven gates:
1. **Capability**: Server declares `claude/channel` experimental capability
2. **Runtime**: GrowthBook gate enabled
3. **OAuth**: Access token present (API key users blocked)
4. **Org policy**: Teams/Enterprise admin approval
5. **Session list**: Server name in `--channels` list
6. **Plugin marketplace**: Source marketplace matches requested
7. **Allowlist**: Plugin or server on approved list (or dev flag)

Additional runtime gates control permission relay and plugin allowlists separately.

#### 8.4 Message Format

Inbound channel messages are wrapped in XML tags with sanitized metadata attributes for the system prompt.

#### 8.5 Security: XML Injection Prevention

Meta keys are validated against a strict identifier regex. Values are XML-attribute-escaped. Non-matching keys are silently dropped.

#### 8.6 Permission Relay Protocol

**Outbound request**: Contains request_id, tool_name, description, input_preview (truncated to 200 chars). Request IDs are 5-letter strings excluding 'l' (phone readability), with profanity blocklist.

**Inbound response**: Contains request_id and behavior (allow/deny).

#### 8.7 CLI Flags

```bash
claude --channels server1,server2                              # Production channels
claude --dangerously-load-development-channels dev1,dev2       # Dev channel bypass
```

Dev channel flag is tracked per-entry (not session-wide), preventing dev bypass leaking to production channels.

### 9. Scheduled Tasks (Cron)

#### 9.1 Purpose

KAIROS can create scheduled tasks (cron jobs) that fire at specified intervals, enabling autonomous periodic work without user interaction.

#### 9.2 Gating

| Gate | Purpose |
|------|---------|
| `tengu_kairos_cron` | Task creation availability |
| `tengu_kairos_cron_durable` | Persistent (disk-backed) tasks |
| `tengu_kairos_cron_config` | Scheduling parameters (minHours, minSessions) |

#### 9.3 Persistence Model

- **Durable**: `.claude/scheduled_tasks.json` -- survives restarts
- **Session-only**: In-memory bootstrap state -- dies with process
- **Teammate crons**: Always session-only (teammates don't persist)

#### 9.4 Scheduler Lock

Lock file: `.claude/scheduled_tasks.lock` -- contains session ID, PID, acquired timestamp. Only the lock owner fires file-backed tasks. Other sessions probe every 5 seconds and take over if the owner's PID is dead. Lock is exclusive via create-then-verify pattern.

#### 9.5 Jitter Strategy

**Recurring tasks**: Forward delay based on deterministic hash of task ID, capped at a configurable maximum (default 15 min). Prevents thundering herd on identical schedules.

**One-shot tasks**: Backward lead on round-time marks (`:00`/`:30`) -- fire up to 90 seconds early.

Jitter fraction is **deterministic per task ID** (hash-based), stable across restarts.

#### 9.6 Auto-Expiry

Recurring tasks auto-delete after a configurable maximum age (default 7 days) unless marked `permanent`. Aged-out tasks fire one final time, then delete.

#### 9.7 Teammate Routing

When a teammate creates a cron, the agent ID is stored on the task. The scheduler routes fires to the teammate's pending message queue. If the teammate is gone, the orphaned cron is cleaned up.

#### 9.8 `/loop` Skill

The `/loop` skill parses interval + prompt, converts to cron, and calls the cron create tool. Default interval: 10 minutes. Enabled only when cron is enabled.

### 10. Session History and Remote Viewer

#### 10.1 Viewer Mode

The `claude assistant [sessionId]` command provides a **read-only REPL** that streams events from a remote assistant session. The agentic loop runs remotely; the local process only renders output and accepts user message input.

#### 10.2 Session Discovery

When invoked without a session ID (`claude assistant`), the viewer enters **discovery mode**:
1. If zero sessions: launch install wizard
2. If one session: auto-select
3. If multiple: show chooser dialog

#### 10.3 Viewer Initialization

The viewer sets kairosActive, enables brief-only mode, sets remote mode, and launches the REPL with no local tools and viewerOnly configuration.

#### 10.4 History Lazy-Loading

- **Initial**: Fetch latest page, chain fill-viewport loads (max 10 pages)
- **Scroll-up**: Trigger at a threshold number of rows from top
- **Scroll anchor**: Snapshot height before prepend, compensate after render
- **Sentinels**: "loading older messages..." / "start of session" indicators

#### 10.5 WebSocket Event Streaming

Live events are streamed via WebSocket with auto-reconnect and backoff, plus ping interval for keepalive.

#### 10.6 Message Posting

Messages are POSTed to the session API with a 30-second timeout (cold-start margin). Echo filtering via a bounded UUID set (UUID recorded pre-POST) prevents duplicate rendering.

#### 10.7 Permission Handling

The viewer can respond to permission requests via WebSocket with allow/deny behavior.

### 11. AutoDream Integration

#### 11.1 AutoDream Disabled in KAIROS

The standard `autoDream` background consolidation (which fires `/dream` as a forked subagent to distill memories) is **explicitly disabled** when KAIROS is active. The gate check is the cheapest (first) in the sequence.

#### 11.2 Replacement: Disk-Skill Dream

Instead of the process-forked autoDream, KAIROS uses a **disk-skill-based** dream mechanism. The nightly distillation is triggered as a skill invocation rather than a background subprocess. This ensures the dream runs within the assistant's own context and has access to its full session state.

#### 11.3 Dream Consolidation Workflow (Four Phases)

1. **Orient**: List memory dir, read MEMORY.md, skim topic files
2. **Gather**: Read daily logs, check for drifted memories, grep transcripts narrowly
3. **Consolidate**: Write/update topic files, convert relative dates to absolute, delete contradicted facts
4. **Prune**: Update MEMORY.md index to stay under 200 lines / 25KB

#### 11.4 AutoDream Gate Sequence (Non-KAIROS, for reference)

1. **KAIROS check**: If active, return false immediately
2. **Remote mode check**: If remote, return false
3. **Auto memory check**: Must be enabled
4. **AutoDream enabled check**: Must be enabled
5. **Time gate**: Hours since last consolidation >= minHours (default: 24)
6. **Scan throttle**: 10 min between session scans
7. **Sessions gate**: Transcript count >= minSessions (default: 5)
8. **Lock acquisition**: No other process mid-consolidation

#### 11.5 Consolidation Lock

Lock file: `.claude/auto_mem/.consolidate-lock`. PID written as body. Stale threshold: 1 hour (guards against PID reuse). Write-then-verify pattern handles concurrent reclaimers. Lock mtime is rolled back on failure.

#### 11.6 Dream Agent Tool Restrictions

The forked dream agent is restricted to: Read/Grep/Glob (unrestricted), Bash (read-only only), Edit/Write (only within memory directory).

### 12. Security and Trust Model

#### 12.1 Trust Dialog Requirement

The single most important security gate in KAIROS activation is the **trust dialog**. `.claude/settings.json` is attacker-controllable in an untrusted repository clone -- a malicious repo could include `"assistant": true` to force KAIROS activation on an unsuspecting user.

The trust dialog checks:
1. Session-level trust flag (for home directory case)
2. Global config trust for the project path
3. All parent directories up to filesystem root

Trust only transitions `false -> true` (never reversed). The `true` result is latched; `false` is re-checked every call.

#### 12.2 Trust Persistence Pathways

**Project directories**: Trust saved to global config under the git root or cwd. Persists across sessions.

**Home directory (`~`)**: Trust stored in session-only memory. Forces re-trust on each invocation -- prevents malicious settings from permanently forcing KAIROS.

#### 12.3 Entitlement Isolation

- **GrowthBook gate** (`tengu_kairos`): server-side kill switch for the entire feature
- **OAuth requirement** for channels: prevents API-key sessions from relay access
- **Managed settings** (`channelsEnabled`): Teams/Enterprise admin control
- **Teammate exclusion**: spawned agents cannot independently activate KAIROS

#### 12.4 Permission System Unchanged

KAIROS does **not** bypass or weaken the permission system. The full oversight pipeline (rules, tool checks, bash security scanner, classifier, human-in-the-loop) applies identically.

#### 12.5 Dev Channel Isolation

The `--dangerously-load-development-channels` flag is tracked per-entry (not session-wide). Passing both `--channels` and the dev flag does not leak the dev dialog's acceptance bypass to production channel entries.

### 13. Feature Gating Strategy

KAIROS uses a **two-tier gating strategy**: build-time dead-code elimination and runtime GrowthBook feature flags.

#### 13.1 Build-Time Gates

| Flag | Controls |
|------|----------|
| `feature('KAIROS')` | Core assistant module, system prompt, team init, memory, bridge, cron, channels |
| `feature('KAIROS_BRIEF')` | SendUserMessage tool only (subset for non-KAIROS builds) |
| `feature('KAIROS_CHANNELS')` | Channel notification subsystem only |
| `feature('KAIROS_CRON')` | Scheduled task subsystem only |

The bundler evaluates `feature()` at compile time. When a flag is `false`, all code inside the conditional is stripped -- zero runtime cost, zero binary bloat.

#### 13.2 Runtime Gates (GrowthBook)

| Gate Name | Purpose | Default |
|-----------|---------|---------|
| `tengu_kairos` | Main KAIROS activation entitlement | `false` |
| `tengu_kairos_brief` | Brief tool availability | `false` |
| `tengu_kairos_cron` | Cron task creation | `false` |
| `tengu_kairos_cron_durable` | Persistent cron tasks | `false` |
| `tengu_kairos_cron_config` | Cron scheduling parameters | defaults |
| `tengu_harbor` | Channel notification support | `false` |
| `tengu_harbor_permissions` | Channel permission relay | `false` |
| `tengu_harbor_ledger` | Channel plugin allowlist | -- |
| `tengu_moth_copse` | Skip MEMORY.md index in assistant prompt | `false` |
| `tengu_coral_fern` | Memory search feature | `false` |
| `tengu_onyx_plover` | AutoDream scheduling config | `{minHours:24, minSessions:5}` |

#### 13.3 Kill Switch Behavior

The `tengu_kairos` gate acts as a **server-side kill switch**. When disabled:
- New sessions cannot activate KAIROS
- Running sessions continue until restart (gate is checked at startup, not per-turn)
- The `--assistant` daemon flag bypasses the gate (daemon pre-verifies entitlement)

### 14. Permission System Interactions

#### 14.1 No Permission Mode Override

KAIROS does not change the user's permission mode. The settings `defaultMode` or `--permission-mode` apply as normal.

#### 14.2 SendUserMessage Tool Permissions

The Brief tool bypasses normal tool deferral when enabled, ensuring the agent can send messages between tool calls without queuing.

#### 14.3 Channel Permission Relay

Channel notifications can relay permission decisions from external platforms, enabling remote approval of tool invocations. This participates in the interactive permission race alongside terminal input, IDE bridge, and hooks.

#### 14.4 Four-Way Permission Race Architecture

Interactive permission handling races four sources:
1. **Bridge** (CCR/claude.ai)
2. **Channels** (KAIROS)
3. **Permission hooks**
4. **Async bash classifier**

All racers use an atomic claim guard ensuring exactly one winner. Every win point cleans up all other racers. KAIROS channels **only participate in the interactive handler** -- never in coordinator or swarm worker handlers.

### 15. Analytics and Observability

When KAIROS is active, analytics metadata includes:
- `kairosActive: true`
- `is_assistant_mode: true`
- `assistantActivationPath: ...`
- Datadog tag for dashboard filtering

Key analytics events:
| Event | When |
|-------|------|
| Trust dialog shown | Trust dialog displayed |
| Brief mode toggled | `/brief` command used |
| Brief mode enabled | Brief activated at startup |
| Brief send | SendUserMessage tool executed |
| Agent memory loaded | Agent memory initialized |

### 16. Data Flow Diagrams

#### 16.1 Activation Flow

```
.claude/settings.json { "assistant": true }
    |
    v
feature('KAIROS') check (build-time dead-code gate)
    | true
    v
isAssistantMode() (settings OR --assistant flag)
    | true
    v
checkHasTrustDialogAccepted() (directory trust verification)
    | true
    v
Entitlement gate (GrowthBook OR daemon-forced)
    | true
    v
KAIROS ACTIVATED
  - setKairosActive(true)
  - Force brief mode
  - initializeAssistantTeam()
  - System prompt addendum appended
  - Daily-log memory enabled
  - Perpetual bridge enabled
  - AutoDream disabled
```

#### 16.2 Runtime Message Flow

```
User Input  -->  Claude Code REPL (KAIROS active)  -->  Anthropic API
(terminal /      SendUserMessage tool call               (streaming)
 bridge /        { message, status, attachments }
 channel)
                          |
              +-----------+-----------+
              v           v           v
          Terminal     IDE Bridge   Channel Relay
          Output
```

#### 16.3 Memory Lifecycle

```
Agent working (KAIROS active)
    | append timestamped bullets
    v
Daily log: .claude/auto_mem/logs/YYYY/MM/YYYY-MM-DD.md  (append-only)
    | [midnight rollover -> new date file]
    v
Next day's log
    | [nightly /dream skill]
    v
Distilled: .claude/auto_mem/MEMORY.md (rewritten)
Topic files: .claude/auto_mem/topics/*.md (rewritten)
```

### 17. Configuration Reference

#### 17.1 Settings

```json
{
  "assistant": true,           // Enable KAIROS mode
  "defaultView": "chat"        // Enable Brief view by default (optional)
}
```

#### 17.2 CLI Flags

| Flag | Effect |
|------|--------|
| `--assistant` | Force KAIROS (daemon mode, skips gate) |
| `--brief` | Enable SendUserMessage tool |
| `--channels <servers>` | MCP servers for channel notifications |
| `--dangerously-load-development-channels <servers>` | Dev channel bypass |
| `--permission-mode <mode>` | Permission mode (unchanged by KAIROS) |

#### 17.3 Environment Variables

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_BRIEF` | Force brief entitlement (`1`/`true`) |
| `CLAUDE_CODE_PROACTIVE` | Force proactive mode |
| `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` | Extra memory policy text |

#### 17.4 CLI Subcommands

```bash
claude                           # Normal launch (KAIROS if settings.json says so)
claude --assistant               # Daemon mode (forced KAIROS)
claude assistant <sessionId>     # Viewer: connect to remote session
claude assistant                 # Viewer: discover available sessions
```

### 18. Proactive Mode Composition

Both `PROACTIVE` and `KAIROS` feature flags enable proactive module loading.

#### 18.1 Tick Mechanism

Ticks are XML-wrapped timestamps injected periodically. The SleepTool controls inter-tick wait time. Multiple ticks may batch into a single message.

#### 18.2 System Prompt Composition Order

1. **Proactive prompt**: Appended if proactive and not coordinator. Includes dynamic brief visibility string.
2. **KAIROS addendum**: Appended if KAIROS enabled. Contains assistant-specific instructions.

The proactive prompt includes: tick handling, pacing (Sleep usage), first wake-up behavior, bias toward action, conciseness rules, terminal focus sensitivity, and (conditionally) the Brief proactive section.

#### 18.3 Agent Composition

In proactive mode, custom agent instructions **append** to the default prompt rather than replacing it.

### 19. Fast Mode and Model Selection

#### 19.1 KAIROS Fast Mode Exemption

KAIROS sessions are exempt from the SDK fast-mode block -- they're first-party orchestration. The kairosActive flag is set before the check runs.

#### 19.2 Model Resolution Priority

1. Model override during session (`/model` command)
2. Model override at startup (`--model` flag)
3. `ANTHROPIC_MODEL` environment variable
4. Settings (`model` in user settings)
5. Built-in default

#### 19.3 Fast Mode State Machine

Fast mode cycles: `active` -> `cooldown` (rate limit / overloaded) -> `active` (on timer expiry). Supported model set is defined in code and may change with model roster updates.

### 20. GrowthBook Caching Architecture

- **In-memory**: Map -- authoritative after remote eval payload processing
- **Disk**: Global config file -- synced on every successful payload
- **Exposure dedup**: Set prevents duplicate exposure events in hot paths

Refresh intervals:
- **Production**: 6 hours
- **Internal**: 20 minutes

### 21. Failure Modes

#### 21.1 Daily Log Write Failures

When the daily log directory does not yet exist, the agent creates it on first write. If directory creation or file write fails (e.g., permissions issue), the failure is graceful -- the agent continues operating without persisting the log entry. No crash or abort occurs; the entry is simply lost.

#### 21.2 Bridge Pointer Corruption

If `bridge-pointer.json` fails schema validation on read (e.g., corrupted JSON, missing required fields, invalid `source` enum value), the pointer is cleared and a fresh session is created. This is the same recovery path as a stale TTL-expired pointer -- the system treats any unreadable pointer as absent.

#### 21.3 Missing Daily Log File

The daily log file (`YYYY-MM-DD.md`) is created on first append. If the file does not exist when the agent attempts to write, the agent creates both the directory tree and the file. There is no pre-creation step during KAIROS initialization -- the file materializes lazily.
