# KAIROS Mode: Autonomous Assistant Architecture Design Specification

This document provides a comprehensive analysis of KAIROS mode — the autonomous assistant
subsystem within Claude Code — covering its activation lifecycle, component architecture,
data flow, security model, and integration with the broader Claude Code platform.

## Table of Contents

- [1. Overview: What KAIROS Is](#1-overview-what-kairos-is)
- [2. Naming and Etymology](#2-naming-and-etymology)
- [3. Activation Lifecycle](#3-activation-lifecycle)
- [4. Component Architecture](#4-component-architecture)
- [5. The SendUserMessage Tool (Brief)](#5-the-sendusermessage-tool-brief)
- [6. Memory System: Daily-Log Mode](#6-memory-system-daily-log-mode)
- [7. Team Spawning and Multi-Agent Coordination](#7-team-spawning-and-multi-agent-coordination)
- [8. Perpetual Bridge Sessions](#8-perpetual-bridge-sessions)
- [9. Channel Notifications](#9-channel-notifications)
- [10. Scheduled Tasks (Cron)](#10-scheduled-tasks-cron)
- [11. Session History and Remote Viewer](#11-session-history-and-remote-viewer)
- [12. AutoDream Integration](#12-autodream-integration)
- [13. Security and Trust Model](#13-security-and-trust-model)
- [14. Feature Gating Strategy](#14-feature-gating-strategy)
- [15. Analytics and Observability](#15-analytics-and-observability)
- [16. Permission System Interactions](#16-permission-system-interactions)
- [17. Data Flow Diagram](#17-data-flow-diagram)
- [18. Configuration Reference](#18-configuration-reference)
- [Appendix A: SendUserMessage Tool — Full Implementation Detail](#appendix-a-sendusermessage-tool--full-implementation-detail)
- [Appendix B: Channel Notification System — Full Implementation Detail](#appendix-b-channel-notification-system--full-implementation-detail)
- [Appendix C: Perpetual Bridge — Full Implementation Detail](#appendix-c-perpetual-bridge--full-implementation-detail)
- [Appendix D: Cron/Scheduled Tasks — Full Implementation Detail](#appendix-d-cronscheduled-tasks--full-implementation-detail)
- [Appendix E: AutoDream and Consolidation — Full Implementation Detail](#appendix-e-autodream-and-consolidation--full-implementation-detail)
- [Appendix F: Trust Dialog — Full Implementation Detail](#appendix-f-trust-dialog--full-implementation-detail)
- [Appendix G: Permission Handler Race — Full Implementation Detail](#appendix-g-permission-handler-race--full-implementation-detail)
- [Appendix H: Team Initialization and Agent Spawning — Full Implementation Detail](#appendix-h-team-initialization-and-agent-spawning--full-implementation-detail)
- [Appendix I: Proactive Mode Composition — Full Implementation Detail](#appendix-i-proactive-mode-composition--full-implementation-detail)
- [Appendix J: Session Viewer and History — Full Implementation Detail](#appendix-j-session-viewer-and-history--full-implementation-detail)
- [Appendix K: UI Rendering Pipeline — Full Implementation Detail](#appendix-k-ui-rendering-pipeline--full-implementation-detail)
- [Appendix L: Fast Mode and Model Selection — Full Implementation Detail](#appendix-l-fast-mode-and-model-selection--full-implementation-detail)
- [Appendix M: Memory Path System — Full Implementation Detail](#appendix-m-memory-path-system--full-implementation-detail)
- [Appendix N: GrowthBook Analytics — Full Implementation Detail](#appendix-n-growthbook-analytics--full-implementation-detail)

---

## 1. Overview: What KAIROS Is

KAIROS is Claude Code's **autonomous assistant mode** — a specialized operating mode that
transforms the CLI from a traditional interactive tool into a long-lived, self-directed
agent. Where normal Claude Code sessions are user-driven (the user types a prompt, the
agent responds, the user types another prompt), KAIROS-mode sessions are **agent-driven**:
the agent works autonomously, communicates results asynchronously via a structured
messaging tool (`SendUserMessage`), and persists across CLI restarts.

The key architectural differences from standard mode:

| Property | Standard Mode | KAIROS Mode |
|----------|--------------|-------------|
| Session lifetime | Single CLI invocation | Perpetual (survives restarts) |
| User output | Direct text streaming | `SendUserMessage` tool calls |
| Memory model | `MEMORY.md` (rewritable index) | Append-only daily logs + nightly distillation |
| Agent spawning | Explicit `TeamCreate` required | Pre-seeded team, `Agent(name:)` spawns directly |
| Bridge behavior | New session per invocation | Perpetual session continuity via `bridge-pointer.json` |
| Background consolidation | `autoDream` (forked `/dream` subagent) | Disabled; uses disk-skill dream instead |
| System prompt | Standard prompts only | Standard + assistant addendum appended |

KAIROS mode does **not** change the permission system. The user's chosen permission mode
(`default`, `auto`, `plan`, etc.) applies identically — KAIROS only forces brief mode on
and adds its system prompt addendum.

---

## 2. Naming and Etymology

**KAIROS** (καιρός) is the ancient Greek word for "the right, critical, or opportune
moment" — as opposed to *chronos* (χρόνος), which denotes sequential, measurable time.
The name captures the system's design intent: an agent that recognizes **when** to take
action and communicate, working asynchronously rather than in lockstep with user input.

In the codebase, "KAIROS" appears as:
- `feature('KAIROS')` — the build-time feature flag
- `kairosActive` — the runtime session state boolean
- `kairosEnabled` — the local variable in `main.tsx` computed during startup
- `tengu_kairos` — the GrowthBook entitlement gate name
- All prefixed with `tengu_kairos_*` for sub-feature gates

---

## 3. Activation Lifecycle

KAIROS activation follows a multi-gate pipeline. Every gate must pass for the mode to
activate. The gates are evaluated in order from cheapest to most expensive.

### 3.1 Gate Pipeline

```
feature('KAIROS')                          ← Build-time dead-code gate
    │ false → all KAIROS code stripped, no runtime cost
    ▼ true
isAssistantMode()                          ← Settings check (.claude/settings.json)
    │ false → normal mode
    ▼ true (assistant: true OR --assistant flag)
!options.agentId                           ← Not a spawned teammate
    │ false → teammate mode (inherits leader's state)
    ▼ true (session leader)
checkHasTrustDialogAccepted()              ← Security trust gate
    │ false → console.warn, refuse activation
    ▼ true (directory explicitly trusted)
isAssistantForced() || kairosGate.isKairosEnabled()  ← Entitlement gate
    │ false → normal mode (not entitled)
    ▼ true (GrowthBook gate or --assistant daemon)
setKairosActive(true)                      ← Latch on
opts.brief = true                          ← Force brief mode
initializeAssistantTeam()                  ← Pre-seed team context
```

> **Source:** `src/main.tsx:1048-1089`

### 3.2 Activation Paths

There are three distinct paths to KAIROS activation:

**Path 1: Settings-based (interactive user)**
```json
// .claude/settings.json
{ "assistant": true }
```
User launches `claude` normally. The settings flag triggers `isAssistantMode() → true`,
then the GrowthBook gate `tengu_kairos` is checked. If the gate is open and the directory
is trusted, KAIROS activates.

**Path 2: Daemon-forced (`--assistant` flag)**
```bash
claude --assistant
```
Used by the Agent SDK daemon mode. Calls `markAssistantForced()` which bypasses the
GrowthBook gate entirely — the daemon has already verified entitlement before spawning
the child process.

> **Source:** `src/main.tsx:1050-1056`

**Path 3: Viewer client (`claude assistant [sessionId]`)**
```bash
claude assistant abc123      # Connect to specific session
claude assistant             # Discover and list sessions
```
This path does **not** activate KAIROS on the local process. Instead it opens a
**read-only REPL viewer** that streams events from a remote assistant session. The agentic
loop runs remotely; the local process only renders output and posts user messages.

> **Source:** `src/main.tsx:685-700, 3259-3274`

### 3.3 Teammate Exclusion

When KAIROS spawns teammates via `Agent(name: "foo")`, those child processes share the
leader's `cwd` and `.claude/settings.json`. This means `isAssistantMode()` would return
`true` for them too. The activation pipeline explicitly excludes spawned teammates by
checking `!options.agentId`:

```typescript
// Spawned teammates share the leader's cwd + settings.json, so
// isAssistantMode() is true for them too. --agent-id being set
// means we ARE a spawned teammate — don't re-init the team or
// override teammateMode/proactive/brief.
!(options as { agentId?: unknown }).agentId && kairosGate
```

> **Source:** `src/main.tsx:1059-1066`

---

## 4. Component Architecture

### 4.1 File Map

| Component | File(s) | Role |
|-----------|---------|------|
| **State flag** | `src/bootstrap/state.ts:72,1085-1090` | `kairosActive` boolean, get/set functions |
| **Assistant module** | `src/assistant/index.js` (not in repo — compiled) | `isAssistantMode()`, `markAssistantForced()`, `initializeAssistantTeam()`, `getAssistantSystemPromptAddendum()`, `getAssistantActivationPath()` |
| **Entitlement gate** | `src/assistant/gate.js` (not in repo — compiled) | `isKairosEnabled()` — GrowthBook `tengu_kairos` check |
| **Session history** | `src/assistant/sessionHistory.ts` | `createHistoryAuthCtx()`, `fetchLatestEvents()`, `fetchOlderEvents()` |
| **Brief tool** | `src/tools/BriefTool/BriefTool.ts` | `isBriefEntitled()`, `isBriefEnabled()`, tool implementation |
| **Brief prompt** | `src/tools/BriefTool/prompt.ts` | `BRIEF_TOOL_NAME`, `LEGACY_BRIEF_TOOL_NAME` |
| **Daily-log memory** | `src/memdir/memdir.ts:319-370` | `buildAssistantDailyLogPrompt()` |
| **Memory paths** | `src/memdir/paths.ts:238` | Daily log file path helper |
| **Bridge integration** | `src/hooks/useReplBridge.tsx:155-170` | Perpetual session flag |
| **Bridge pointer** | `src/bridge/bridgePointer.ts` | `bridge-pointer.json` read/write/clear |
| **Bridge core** | `src/bridge/replBridge.ts:211-212,294-315` | Perpetual mode logic |
| **Channel notifications** | `src/services/mcp/channelNotification.ts` | MCP channel message relay |
| **Channel permissions** | `src/services/mcp/channelPermissions.ts` | Permission relay via channels |
| **AutoDream** | `src/services/autoDream/autoDream.ts:95-96` | KAIROS exclusion gate |
| **Analytics metadata** | `src/services/analytics/metadata.ts:493,736,967` | `kairosActive` tracking |
| **Fast mode** | `src/utils/fastMode.ts:98` | KAIROS timing note |
| **Main orchestrator** | `src/main.tsx` (30+ references) | Activation, tool registration, prompt assembly |

### 4.2 Module Loading Strategy

KAIROS modules are loaded via **conditional `require()`** gated behind `feature('KAIROS')`:

```typescript
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') as typeof import('./assistant/index.js')
  : null;
const kairosGate = feature('KAIROS')
  ? require('./assistant/gate.js') as typeof import('./assistant/gate.js')
  : null;
```

> **Source:** `src/main.tsx:80-81`

Bun's bundler performs dead-code elimination on `feature()` checks at build time. In
non-KAIROS builds, the entire assistant module tree is stripped from the output, resulting
in zero runtime overhead.

---

## 5. The SendUserMessage Tool (Brief)

### 5.1 Purpose

In standard Claude Code, the model's text output streams directly to the user's terminal.
In KAIROS mode, the agent's primary communication channel is the `SendUserMessage` tool
(internally called "Brief"). This structured tool call replaces free-form text streaming,
enabling:

- Asynchronous message delivery (agent works while user is away)
- Structured metadata (status, attachments)
- Channel relay (messages forwarded to Slack, Discord, etc.)
- IDE bridge rendering

### 5.2 Entitlement Chain

```
feature('KAIROS') || feature('KAIROS_BRIEF')     ← Build-time gate
    ▼
getKairosActive()                                 ← Session state
  || isEnvTruthy(CLAUDE_CODE_BRIEF)               ← Dev bypass
  || getFeatureValue('tengu_kairos_brief', false)  ← GrowthBook gate (5-min refresh)
    ▼
isBriefEntitled() = true
    ▼
(getKairosActive() || getUserMsgOptIn()) && isBriefEntitled()
    ▼
isBriefEnabled() = true → SendUserMessage tool available
```

> **Source:** `src/tools/BriefTool/BriefTool.ts:88-134`

### 5.3 Opt-In Paths

SendUserMessage can be activated via six distinct paths:

1. **KAIROS mode** — automatic (`kairosActive → true`)
2. **`--brief` CLI flag** — explicit opt-in
3. **`defaultView: 'chat'` in settings** — persisted preference
4. **`/brief` slash command** — mid-session toggle
5. **`--tools SendUserMessage`** — SDK explicit tool listing
6. **`CLAUDE_CODE_BRIEF=1` env var** — development bypass

> **Source:** `src/main.tsx:1728-1742, 2184-2192, 4622-4652`

### 5.4 System Prompt Integration

When brief mode is active within a proactive context, the system prompt dynamically
adjusts its visibility instruction:

```typescript
const briefVisibility = isBriefEnabled()
  ? 'Call SendUserMessage at checkpoints to mark where things stand.'
  : 'The user will see any text you output.';
```

> **Source:** `src/main.tsx:2201`

---

## 6. Memory System: Daily-Log Mode

### 6.1 Design Rationale

Standard Claude Code uses `MEMORY.md` as a rewritable index — the agent reads and rewrites
it each session. KAIROS sessions are **effectively perpetual**, so rewriting MEMORY.md on
every interaction would cause churn and conflicts. Instead, KAIROS uses an **append-only
daily log** with periodic distillation.

### 6.2 Architecture

```
.claude/auto_mem/
├── MEMORY.md              ← Distilled index (read-only for agent, nightly-written)
├── logs/
│   └── YYYY/
│       └── MM/
│           └── YYYY-MM-DD.md   ← Append-only daily log (agent writes here)
└── topics/
    └── *.md               ← Topic files (nightly distillation output)
```

The agent appends timestamped bullets to today's log file. A separate nightly `/dream`
skill reads accumulated logs and distills them into `MEMORY.md` and topic files.

### 6.3 Prompt Construction

The daily-log prompt replaces the standard memory prompt when KAIROS is active:

```typescript
if (feature('KAIROS') && autoEnabled && getKairosActive()) {
  return buildAssistantDailyLogPrompt(skipIndex)
}
```

> **Source:** `src/memdir/memdir.ts:427-437`

The prompt instructs the agent to:
- Append to `logs/YYYY/MM/YYYY-MM-DD.md` (path pattern, not literal — prompt is cached)
- Use timestamped bullets
- Create directories on first write
- Never rewrite or reorganize the log
- Log user corrections, preferences, project context, external system pointers

### 6.4 Date Rollover

The log path is specified as a **pattern** (`YYYY/MM/YYYY-MM-DD.md`) rather than today's
literal date because the system prompt is cached by `systemPromptSection('memory', ...)`
and is **not invalidated on date change**. The model derives the current date from a
`date_change` attachment appended at midnight rollover, rather than from the user-context
message (which is intentionally left stale to preserve the prompt cache prefix).

> **Source:** `src/memdir/memdir.ts:329-334`

### 6.5 Incompatibility with Team Memory Sync

KAIROS daily-log mode explicitly takes precedence over team memory sync (`TEAMMEM`).
The append-only log paradigm does not compose with team sync, which expects a shared
`MEMORY.md` that both sides read and write.

> **Source:** `src/memdir/memdir.ts:427-431`

### 6.6 Past-Context Search

When the `tengu_coral_fern` GrowthBook gate is enabled, the memory prompt includes
instructions for searching past context:

1. **Memory directory search:** `grep -rn "<term>" .claude/auto_mem/ --include="*.md"`
2. **Transcript logs (last resort):** `grep -rn "<term>" <projectDir>/ --include="*.jsonl"`

> **Source:** `src/memdir/memdir.ts:375-407`

---

## 7. Team Spawning and Multi-Agent Coordination

### 7.1 Pre-Seeded Team Context

KAIROS pre-seeds an in-process team during activation so that `Agent(name: "foo")` can
spawn teammates without an explicit `TeamCreate` call:

```typescript
if (kairosEnabled) {
  // Pre-seed an in-process team so Agent(name: "foo") spawns
  // teammates without TeamCreate. Must run BEFORE setup() captures
  // the teammateMode snapshot.
  assistantTeamContext = await assistantModule.initializeAssistantTeam();
}
```

> **Source:** `src/main.tsx:1082-1086`

### 7.2 Team Context Precedence

The team context follows a precedence chain:

```typescript
teamContext: feature('KAIROS')
  ? assistantTeamContext ?? computeInitialTeamContext?.()
  : computeInitialTeamContext?.()
```

KAIROS team context takes priority over the default computed context.

> **Source:** `src/main.tsx:3031-3035`

### 7.3 Session Types

| Type | Description | `agentId` | KAIROS init |
|------|-------------|-----------|-------------|
| **Leader** | Main assistant daemon | `undefined` | Full init |
| **Teammate** | Spawned via `Agent(name:)` | Set by leader | Skipped (inherits) |
| **Viewer** | `claude assistant [id]` | N/A | No agentic loop |

---

## 8. Perpetual Bridge Sessions

### 8.1 Problem

In standard Claude Code, each CLI invocation creates a new bridge session. When the
process exits, the bridge session is archived and deregistered. This means claude.ai shows
separate conversation threads for each CLI run.

### 8.2 KAIROS Solution

KAIROS enables **perpetual bridge sessions** — a single continuous conversation thread
that survives CLI restarts:

```typescript
// Assistant mode: perpetual bridge session — claude.ai shows one
// continuous conversation across CLI restarts instead of a new
// session per invocation.
let perpetual = false;
if (feature('KAIROS')) {
  const { isAssistantMode } = await import('../assistant/index.js');
  perpetual = isAssistantMode();
}
```

> **Source:** `src/hooks/useReplBridge.tsx:155-169`

### 8.3 Bridge Pointer Mechanism

Perpetual mode uses `bridge-pointer.json` to persist session identity across restarts:

```
.claude/projects/<sanitized-dir>/bridge-pointer.json
→ { environmentId: "env_xxx", sessionId: "session_yyy" }
```

**Standard mode:** Writes pointer after session create; clears on teardown (crash-recovery
only).

**Perpetual mode:** Reads pointer at init to resume prior session; **skips**
archive/deregister/pointer-clear at teardown. The pointer survives clean exits, not just
crashes.

> **Source:** `src/bridge/replBridge.ts:294-315, 1510-1613`

### 8.4 Teardown Differences

```
Standard mode teardown:        KAIROS perpetual teardown:
1. Archive session             1. Leave session alive on server
2. Deregister environment      2. Leave environment alive
3. Clear bridge-pointer.json   3. Keep bridge-pointer.json
4. Done                        4. Log: "leaving env=X session=Y alive"
```

> **Source:** `src/bridge/replBridge.ts:1595-1613`

---

## 9. Channel Notifications

### 9.1 Concept

Channel notifications allow KAIROS to relay messages through external communication
platforms (Slack, Discord, SMS, Telegram, etc.) via MCP servers. A "channel" is an MCP
server that:

1. Exposes tools for **outbound** messages (e.g., `send_message`) — standard MCP
2. Sends `notifications/claude/channel` notifications for **inbound** messages

> **Source:** `src/services/mcp/channelNotification.ts:1-17`

### 9.2 Notification Protocol

| Method | Direction | Purpose |
|--------|-----------|---------|
| `notifications/claude/channel` | Inbound | External messages arriving for the agent |
| `notifications/claude/channel/permission` | Inbound | Permission decisions relayed from channels |

### 9.3 Gating

Channel notifications require:
- `feature('KAIROS') || feature('KAIROS_CHANNELS')` — build-time gate
- `tengu_harbor` — GrowthBook runtime gate
- OAuth authentication (API key users blocked)
- Teams/Enterprise: `channelsEnabled: true` in managed settings

### 9.4 Message Format

Inbound channel messages are wrapped in XML tags for the system prompt:

```xml
<channel meta="...">message content</channel>
```

### 9.5 CLI Flags

```bash
claude --channels server1,server2                              # Production channels
claude --dangerously-load-development-channels dev1,dev2       # Dev channel bypass
```

> **Source:** `src/main.tsx:1642, 3844`

---

## 10. Scheduled Tasks (Cron)

### 10.1 Purpose

KAIROS can create scheduled tasks (cron jobs) that fire at specified intervals, enabling
autonomous periodic work without user interaction.

### 10.2 GrowthBook Gates

| Gate | Purpose | Refresh |
|------|---------|---------|
| `tengu_kairos_cron` | Task creation availability | 5 min |
| `tengu_kairos_cron_durable` | Persistent (disk-backed) tasks | 5 min |
| `tengu_kairos_cron_config` | Scheduling parameters (minHours, minSessions) | 1 min |

### 10.3 Tool

The `CronCreateTool` (`ScheduleCronTool`) allows the agent to define recurring triggers
that inject prompts into the conversation at specified intervals.

> **Source:** `src/tools/ScheduleCronTool/prompt.ts`

---

## 11. Session History and Remote Viewer

### 11.1 `claude assistant [sessionId]` — Viewer Mode

The viewer mode provides a **read-only REPL** that streams events from a remote assistant
session. The agentic loop runs remotely; the local process only renders output and accepts
user message input.

```typescript
// `claude assistant [sessionId]` — REPL as a pure viewer client
// of a remote assistant session. The agentic loop runs remotely; this
// process streams live events and POSTs messages.
```

> **Source:** `src/main.tsx:3259-3263`

### 11.2 Session Discovery

When invoked without a session ID (`claude assistant`), the viewer enters **discovery
mode**, listing available assistant sessions:

```typescript
if (!targetSessionId) {
  sessions = await discoverAssistantSessions();
}
```

> **Source:** `src/main.tsx:3270-3273`

### 11.3 History API

Session history is fetched from Anthropic's API with pagination support:

| Function | Purpose | API Pattern |
|----------|---------|-------------|
| `fetchLatestEvents()` | Newest page via `anchor_to_latest` | `GET /v1/sessions/{id}/events?limit=100&anchor_to_latest=true` |
| `fetchOlderEvents()` | Older page via cursor | `GET /v1/sessions/{id}/events?limit=100&before_id={cursor}` |

**Authentication:**
- OAuth access token (`getOAuthHeaders`)
- Organization UUID (`x-organization-uuid`)
- Beta header: `anthropic-beta: ccr-byoc-2025-07-29`

> **Source:** `src/assistant/sessionHistory.ts`

---

## 12. AutoDream Integration

### 12.1 AutoDream Disabled in KAIROS

The standard `autoDream` background consolidation (which fires `/dream` as a forked
subagent to distill memories) is **explicitly disabled** when KAIROS is active:

```typescript
function isGateOpen(): boolean {
  if (getKairosActive()) return false  // KAIROS mode uses disk-skill dream
  if (getIsRemoteMode()) return false
  if (!isAutoMemoryEnabled()) return false
  return isAutoDreamEnabled()
}
```

> **Source:** `src/services/autoDream/autoDream.ts:95-100`

### 12.2 Replacement: Disk-Skill Dream

Instead of the process-forked autoDream, KAIROS uses a **disk-skill-based** dream
mechanism. The nightly distillation is triggered as a skill invocation rather than a
background subprocess. This ensures the dream runs within the assistant's own context
and has access to its full session state.

### 12.3 AutoDream Gate Sequence (Non-KAIROS Reference)

For comparison, the standard autoDream gates (cheapest first):
1. **Time:** hours since `lastConsolidatedAt` >= `minHours` (default: 24)
2. **Sessions:** transcript count since last consolidation >= `minSessions` (default: 5)
3. **Lock:** no other process mid-consolidation

Configuration from GrowthBook gate `tengu_onyx_plover`.

> **Source:** `src/services/autoDream/autoDream.ts:58-93`

---

## 13. Security and Trust Model

### 13.1 Trust Dialog Requirement

The single most important security gate in KAIROS activation is the **trust dialog**.
`.claude/settings.json` is attacker-controllable in an untrusted repository clone — a
malicious repo could include `"assistant": true` to force KAIROS activation on an
unsuspecting user. The trust dialog prevents this:

```typescript
// Trust gate: .claude/settings.json is attacker-controllable in an
// untrusted clone. We run ~1000 lines before showSetupScreens() shows
// the trust dialog, and by then we've already appended
// .claude/agents/assistant.md to the system prompt. Refuse to activate
// until the directory has been explicitly trusted.
if (!checkHasTrustDialogAccepted()) {
  console.warn('Assistant mode disabled: directory is not trusted...');
}
```

> **Source:** `src/main.tsx:1043-1069`

### 13.2 Entitlement Isolation

- **GrowthBook gate** (`tengu_kairos`): server-side kill switch for the entire feature
- **OAuth requirement** for channels: prevents API-key sessions from relay access
- **Managed settings** (`channelsEnabled`): Teams/Enterprise admin control
- **Teammate exclusion**: spawned agents cannot independently activate KAIROS

### 13.3 Permission System Unchanged

KAIROS does **not** bypass or weaken the permission system. The full oversight pipeline
(rules, tool checks, bash security scanner, classifier, human-in-the-loop) applies
identically. The user's chosen permission mode governs all tool invocations.

### 13.4 Dev Channel Isolation

The `--dangerously-load-development-channels` flag is tracked per-entry (not session-wide).
Passing both `--channels` and `--dangerously-load-development-channels` does not leak
the dev dialog's acceptance bypass to production channel entries.

> **Source:** `src/bootstrap/state.ts:33-39`

---

## 14. Feature Gating Strategy

KAIROS uses a **two-tier gating strategy**: build-time dead-code elimination and runtime
GrowthBook feature flags.

### 14.1 Build-Time Gates

| Flag | Controls |
|------|----------|
| `feature('KAIROS')` | Core assistant module, system prompt, team init, memory, bridge, cron, channels |
| `feature('KAIROS_BRIEF')` | SendUserMessage tool only (subset for non-KAIROS builds) |
| `feature('KAIROS_CHANNELS')` | Channel notification subsystem only |
| `feature('KAIROS_CRON')` | Scheduled task subsystem only |

Bun's bundler evaluates `feature()` at compile time. When a flag is `false`, all code
inside the conditional is stripped — zero runtime cost, zero binary bloat.

### 14.2 Runtime Gates (GrowthBook)

| Gate Name | Purpose | Refresh Interval | Default |
|-----------|---------|------------------|---------|
| `tengu_kairos` | Main KAIROS activation entitlement | Blocking (cached to disk) | `false` |
| `tengu_kairos_brief` | Brief tool availability | 5 min | `false` |
| `tengu_kairos_cron` | Cron task creation | 5 min | `false` |
| `tengu_kairos_cron_durable` | Persistent cron tasks | 5 min | `false` |
| `tengu_kairos_cron_config` | Cron scheduling parameters | 1 min | defaults |
| `tengu_harbor` | Channel notification support | — | `false` |
| `tengu_moth_copse` | Skip MEMORY.md index in assistant prompt | — | `false` |
| `tengu_coral_fern` | Memory search feature | — | `false` |
| `tengu_onyx_plover` | AutoDream scheduling config | — | `{minHours:24, minSessions:5}` |

### 14.3 Kill Switch Behavior

The `tengu_kairos` gate acts as a **server-side kill switch**. When disabled:
- New sessions cannot activate KAIROS
- Running sessions continue until restart (gate is checked at startup, not per-turn)
- The `--assistant` daemon flag bypasses the gate (daemon pre-verifies entitlement)

---

## 15. Analytics and Observability

### 15.1 Session Metadata

When KAIROS is active, analytics metadata includes:

```typescript
{ kairosActive: true }           // src/services/analytics/metadata.ts:736
{ is_assistant_mode: true }      // src/services/analytics/metadata.ts:967
{ assistantActivationPath: ... } // src/main.tsx:2518
```

### 15.2 Datadog Tag

`kairosActive` is included in the Datadog analytics tag set for dashboard filtering:

> **Source:** `src/services/analytics/datadog.ts:72`

### 15.3 Key Analytics Events

| Event | When | Source |
|-------|------|--------|
| `tengu_trust_dialog_shown` | Trust dialog displayed | Trust component |
| `tengu_brief_mode_toggled` | `/brief` command used | Brief command |
| `tengu_brief_mode_enabled` | Brief activated at startup | `src/main.tsx:4623-4652` |
| `tengu_brief_send` | SendUserMessage tool executed | Brief tool |
| `tengu_agent_memory_loaded` | Agent memory initialized | Memory system |

---

## 16. Permission System Interactions

### 16.1 No Permission Mode Override

KAIROS does not change the user's permission mode:

```typescript
// Assistant mode: when .claude/settings.json has assistant: true AND
// the tengu_kairos GrowthBook gate is on, force brief on. Permission
// mode is left to the user — settings defaultMode or --permission-mode
// apply as normal.
```

> **Source:** `src/main.tsx:1034-1039`

### 16.2 SendUserMessage Tool Permissions

The Brief tool bypasses normal tool deferral when `isBriefEnabled() == true`, ensuring
the agent can send messages between tool calls without queuing.

### 16.3 Channel Permission Relay

Channel notifications can relay permission decisions from external platforms
(`notifications/claude/channel/permission`), enabling remote approval of tool invocations
through Slack, Discord, or other messaging platforms. This participates in the interactive
permission race alongside terminal input, IDE bridge, and hooks.

> **Source:** `src/hooks/toolPermission/handlers/interactiveHandler.ts:300`

---

## 17. Data Flow Diagram

### 17.1 Activation Flow

```
                    .claude/settings.json
                    { "assistant": true }
                            │
                            ▼
    ┌──────────────────────────────────────┐
    │         feature('KAIROS') check       │
    │    (build-time dead-code gate)        │
    └──────────────┬───────────────────────┘
                   │ true
                   ▼
    ┌──────────────────────────────────────┐
    │      isAssistantMode() → true        │
    │    (settings OR --assistant flag)     │
    └──────────────┬───────────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────────┐
    │   checkHasTrustDialogAccepted()      │
    │    (directory trust verification)    │
    └──────────────┬───────────────────────┘
                   │ true
                   ▼
    ┌──────────────────────────────────────┐
    │   kairosGate.isKairosEnabled()       │
    │  (GrowthBook tengu_kairos gate)      │
    │  OR isAssistantForced() (daemon)     │
    └──────────────┬───────────────────────┘
                   │ true
                   ▼
    ┌──────────────────────────────────────┐
    │        KAIROS ACTIVATED              │
    │  • setKairosActive(true)             │
    │  • opts.brief = true                 │
    │  • initializeAssistantTeam()         │
    │  • System prompt addendum appended   │
    │  • Daily-log memory enabled          │
    │  • Perpetual bridge enabled          │
    │  • AutoDream disabled                │
    └──────────────────────────────────────┘
```

### 17.2 Runtime Message Flow

```
┌─────────────┐         ┌──────────────────┐        ┌──────────────┐
│  User Input  │────────▶│  Claude Code REPL │───────▶│ Anthropic API │
│ (terminal /  │         │  (KAIROS active)  │        │  (streaming)  │
│  bridge /    │◀────────│                   │◀───────│              │
│  channel)    │         │  SendUserMessage  │        │              │
└─────────────┘         │  ──────────────── │        └──────────────┘
       ▲                 │  • message: "..."  │
       │                 │  • status: normal  │
       │                 │  • attachments: [] │
       │                 └────────┬───────────┘
       │                          │
       │              ┌───────────┼───────────┐
       │              ▼           ▼           ▼
       │     ┌──────────┐ ┌──────────┐ ┌──────────┐
       └─────│ Terminal  │ │ IDE      │ │ Channel  │
             │ Output   │ │ Bridge   │ │ Relay    │
             └──────────┘ └──────────┘ └──────────┘
```

### 17.3 Memory Lifecycle

```
Agent working (KAIROS active)
    │
    │ append timestamped bullets
    ▼
.claude/auto_mem/logs/2026/03/2026-03-31.md    ← Daily log (append-only)
    │
    │ [midnight rollover → new date file]
    ▼
.claude/auto_mem/logs/2026/04/2026-04-01.md    ← Next day's log
    │
    │ [nightly /dream skill]
    ▼
.claude/auto_mem/MEMORY.md                      ← Distilled index (rewritten)
.claude/auto_mem/topics/*.md                    ← Topic files (rewritten)
```

---

## 18. Configuration Reference

### 18.1 Settings

```json
// .claude/settings.json
{
  "assistant": true,           // Enable KAIROS mode
  "defaultView": "chat"        // Enable Brief view by default (optional)
}
```

### 18.2 CLI Flags

| Flag | Effect |
|------|--------|
| `--assistant` | Force KAIROS (daemon mode, skips gate) |
| `--brief` | Enable SendUserMessage tool |
| `--channels <servers>` | MCP servers for channel notifications |
| `--dangerously-load-development-channels <servers>` | Dev channel bypass |
| `--permission-mode <mode>` | Permission mode (unchanged by KAIROS) |

### 18.3 Environment Variables

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_BRIEF` | Force brief entitlement (`1`/`true`) |
| `CLAUDE_CODE_PROACTIVE` | Force proactive mode |
| `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` | Extra memory policy text |

### 18.4 CLI Subcommands

```bash
claude                           # Normal launch (KAIROS if settings.json says so)
claude --assistant               # Daemon mode (forced KAIROS)
claude assistant <sessionId>     # Viewer: connect to remote session
claude assistant                 # Viewer: discover available sessions
```

---
---

# Appendices: Full Implementation Detail

The following appendices provide source-level detail for every KAIROS subsystem, including
exact code snippets, line numbers, schemas, error handling, and edge cases.

---

## Appendix A: SendUserMessage Tool — Full Implementation Detail

### A.1 Tool Schema

**Input** (`src/tools/BriefTool/BriefTool.ts:20-37`):

```typescript
z.strictObject({
  message: z.string()
    .describe('The message for the user. Supports markdown formatting.'),
  attachments: z.array(z.string()).optional()
    .describe('Optional file paths (absolute or relative to cwd) to attach.'),
  status: z.enum(['normal', 'proactive'])
    .describe("Use 'proactive' when surfacing something the user hasn't asked for."),
})
```

**Output** (`src/tools/BriefTool/BriefTool.ts:42-63`):

```typescript
z.object({
  message: z.string(),
  attachments: z.array(z.object({
    path: z.string(),
    size: z.number(),
    isImage: z.boolean(),
    file_uuid: z.string().optional(),
  })).optional(),
  sentAt: z.string().optional()
    .describe('ISO timestamp captured at tool execution.'),
})
```

### A.2 Tool Execution

```typescript
// src/tools/BriefTool/BriefTool.ts:186-203
async call({ message, attachments, status }, context) {
  const sentAt = new Date().toISOString()
  logEvent('tengu_brief_send', {
    proactive: status === 'proactive',
    attachment_count: attachments?.length ?? 0,
  })
  if (!attachments || attachments.length === 0) {
    return { data: { message, sentAt } }
  }
  const resolved = await resolveAttachments(attachments, {
    replBridgeEnabled: appState.replBridgeEnabled,
    signal: context.abortController.signal,
  })
  return { data: { message, attachments: resolved, sentAt } }
}
```

### A.3 Tool Prompt Text

```typescript
// src/tools/BriefTool/prompt.ts:1-4
export const BRIEF_TOOL_NAME = 'SendUserMessage'
export const LEGACY_BRIEF_TOOL_NAME = 'Brief'
export const DESCRIPTION = 'Send a message to the user'
```

The proactive section (`src/tools/BriefTool/prompt.ts:12-22`) instructs the model:
- `SendUserMessage` is **where your replies go** — text outside it is hidden from most users
- Every user message requires a reply through `SendUserMessage`, even "hi" or "thanks"
- Ack first ("On it — checking the test output"), then work, then send the result
- Checkpoints earn their place by carrying information, not filler ("running tests...")

### A.4 Three-Mode UI Rendering

**Transcript mode** (`src/tools/BriefTool/UI.tsx:26-37`): Black circle gutter + markdown + attachments.

**Brief-only (chat) mode** (`src/tools/BriefTool/UI.tsx:41-52`): "Claude" label with `briefLabelClaude` color, 2-column indent, optional timestamp.

**Default mode** (`src/tools/BriefTool/UI.tsx:55-68`): No gutter mark, `minWidth={2}` spacer, plain markdown. `dropTextInBriefTurns()` in `Messages.tsx` hides redundant assistant text in turns that called Brief.

### A.5 Message Filtering Pipeline

1. **`filterForBriefTool()`** (`src/components/Messages.tsx:93-158`): In brief-only mode, drops all assistant messages except Brief tool_use blocks. Keeps system messages (except `api_metrics`), user input, and matching tool_results.

2. **`dropTextInBriefTurns()`** (`src/components/Messages.tsx:169-206`): In default mode, finds turns containing a Brief tool_use and drops assistant text blocks from those turns (the Brief output replaces the text).

### A.6 `/brief` Slash Command

The `/brief` command (`src/commands/brief.ts:47-130`) toggles `isBriefOnly` on AppState and `userMsgOptIn` on bootstrap state. When KAIROS is active, the meta-message injection is skipped (KAIROS system prompt already mandates SendUserMessage). Entitlement check only gates the on-transition — off is always allowed.

---

## Appendix B: Channel Notification System — Full Implementation Detail

### B.1 Notification Schema

```typescript
// src/services/mcp/channelNotification.ts:37-47
z.object({
  method: z.literal('notifications/claude/channel'),
  params: z.object({
    content: z.string(),
    meta: z.record(z.string(), z.string()).optional(),
  }),
})
```

### B.2 XML Wrapping and Injection Prevention

```typescript
// src/services/mcp/channelNotification.ts:103-116
const SAFE_META_KEY = /^[a-zA-Z_][a-zA-Z0-9_]*$/

export function wrapChannelMessage(serverName, content, meta) {
  const attrs = Object.entries(meta ?? {})
    .filter(([k]) => SAFE_META_KEY.test(k))  // strict identifier validation
    .map(([k, v]) => ` ${k}="${escapeXmlAttr(v)}"`)
    .join('')
  return `<channel source="${escapeXmlAttr(serverName)}"${attrs}>\n${content}\n</channel>`
}
```

Security: `escapeXmlAttr()` escapes `& < > " '`. Meta keys not matching the identifier regex are silently dropped.

### B.3 Permission Relay Protocol

**Outbound request** (`src/services/mcp/channelNotification.ts:85-95`):
```typescript
{ request_id, tool_name, description, input_preview }
```
Input preview is truncated to 200 chars. Request IDs are 5-letter strings excluding 'l' (phone readability), with profanity blocklist.

**Inbound response** (`src/services/mcp/channelNotification.ts:64-72`):
```typescript
{ method: 'notifications/claude/channel/permission',
  params: { request_id, behavior: 'allow' | 'deny' } }
```

### B.4 Seven-Gate Access Control

`gateChannelServer()` (`src/services/mcp/channelNotification.ts:191-316`) checks in order:
1. **Capability**: Server declares `claude/channel` experimental capability
2. **Runtime**: `tengu_harbor` GrowthBook gate enabled
3. **OAuth**: Access token present (API key users blocked)
4. **Org policy**: Teams/Enterprise `channelsEnabled: true`
5. **Session list**: Server name in `--channels` list
6. **Plugin marketplace**: Source marketplace matches requested
7. **Allowlist**: Plugin or server on approved list (or dev flag)

### B.5 Additional Gates

- `tengu_harbor` — overall channels on/off
- `tengu_harbor_permissions` — permission relay on/off
- `tengu_harbor_ledger` — plugin allowlist entries

---

## Appendix C: Perpetual Bridge — Full Implementation Detail

### C.1 Bridge Pointer Schema

```typescript
// src/bridge/bridgePointer.ts:42-48
z.object({
  sessionId: z.string(),
  environmentId: z.string(),
  source: z.enum(['standalone', 'repl']),
})
```

TTL: `BRIDGE_POINTER_TTL_MS = 4 * 60 * 60 * 1000` (4 hours). Stale pointers are cleared on read.

### C.2 Perpetual Init Flow

1. **Read pointer** (`src/bridge/replBridge.ts:311`): Only in perpetual mode. Only reuses `source: 'repl'` pointers.
2. **Request env reuse** (`src/bridge/replBridge.ts:342`): `reuseEnvironmentId: prior?.environmentId`
3. **Try reconnect in place** (`src/bridge/replBridge.ts:381-419`): Calls `api.reconnectSession()` with both session ID forms (compat `session_*` and infra `cse_*`).
4. **Reuse or create** (`src/bridge/replBridge.ts:421-477`): If reconnect succeeds, reuse session and mark initial messages as previously-flushed. Otherwise, clear pointer and create fresh session.

### C.3 Teardown Comparison

**Perpetual** (`src/bridge/replBridge.ts:1595-1613`): Sets `transport = null`, drops flush gate, refreshes pointer mtime. No stopWork, no archive, no deregister, no pointer clear.

**Standard** (`src/bridge/replBridge.ts:1617-1668`): Sends result message, calls stopWork + archiveSession in parallel (1.5s cap), closes transport, deregisters environment, clears pointer.

### C.4 Hourly Pointer Refresh

```typescript
// src/bridge/replBridge.ts:1510-1526
const pointerRefreshTimer = perpetual
  ? setInterval(() => {
      if (reconnectPromise) return  // skip during reconnect race
      void writeBridgePointer(dir, { sessionId, environmentId, source: 'repl' })
    }, 60 * 60_000)
  : null
```

Prevents 4-hour TTL expiry on long-running sessions.

### C.5 Reconnect-After-Environment-Lost

The `doReconnect()` function (`src/bridge/replBridge.ts:587-836`) implements a two-strategy recovery:
- **Strategy 1**: Idempotent re-register with prior env ID → `tryReconnectInPlace()`
- **Strategy 2**: Archive old session → create fresh session on new environment

Max `MAX_ENVIRONMENT_RECREATIONS` attempts. Counter resets on success. Multiple abort-signal checks prevent zombie operations during teardown.

### C.6 Worktree-Aware Pointer Fanout

`readBridgePointerAcrossWorktrees()` (`src/bridge/bridgePointer.ts:129-184`) scans git worktree siblings for bridge pointers when `--continue` is invoked from a different worktree. Capped at `MAX_WORKTREE_FANOUT` to bound I/O.

---

## Appendix D: Cron/Scheduled Tasks — Full Implementation Detail

### D.1 CronCreate Tool Schema

```typescript
// src/tools/ScheduleCronTool/CronCreateTool.ts:27-42
z.strictObject({
  cron: z.string()   // 5-field cron: "M H DoM Mon DoW"
  prompt: z.string() // Prompt to enqueue at each fire
  recurring: z.boolean().optional()  // default true; false = one-shot
  durable: z.boolean().optional()    // default false; true = persist to disk
})
```

### D.2 Persistence Model

- **Durable**: `.claude/scheduled_tasks.json` — survives restarts
- **Session-only**: `STATE.sessionCronTasks` in bootstrap state — dies with process
- **Teammate crons**: Always session-only (teammates don't persist)

### D.3 Scheduler Lock

Lock file: `.claude/scheduled_tasks.lock` — contains `{ sessionId, pid, acquiredAt }`. Only the lock owner fires file-backed tasks. Other sessions probe every 5 seconds and take over if the owner's PID is dead. Lock is exclusive via `tryCreateExclusive()` + verify-after-write.

### D.4 Jitter Strategy

**Recurring tasks**: Forward delay = `jitterFrac(taskId) * recurringFrac * interval`, capped at `recurringCapMs` (default 15 min). Prevents thundering herd on identical schedules.

**One-shot tasks**: Backward lead on `:00`/`:30` minute marks — fire up to 90 seconds early. `oneShotMinuteMod: 30` identifies "round-time" targets.

Jitter fraction is **deterministic per task ID** (hash-based), stable across restarts.

### D.5 Auto-Expiry

Recurring tasks auto-delete after `recurringMaxAgeMs` (default 7 days) unless marked `permanent`. Aged-out tasks fire one final time, then delete.

### D.6 Teammate Routing

When a teammate creates a cron, `agentId` is stored on the task. The scheduler routes fires to the teammate's `pendingUserMessages` queue via `injectUserMessageToTeammate()`. If the teammate is gone, the orphaned cron is cleaned up.

### D.7 `/loop` Skill

The `/loop` skill (`src/skills/bundled/loop.ts`) parses interval + prompt, converts to cron, and calls CronCreate. Default interval: 10 minutes. Enabled only when `isKairosCronEnabled()`.

---

## Appendix E: AutoDream and Consolidation — Full Implementation Detail

### E.1 Gate Sequence (Cheapest First)

```typescript
// src/services/autoDream/autoDream.ts:95-100
function isGateOpen(): boolean {
  if (getKairosActive()) return false  // KAIROS uses disk-skill dream
  if (getIsRemoteMode()) return false
  if (!isAutoMemoryEnabled()) return false
  return isAutoDreamEnabled()
}
```

Then per-turn: time gate (hours since last consolidation ≥ `minHours`) → scan throttle (10 min between session scans) → session gate (transcript count ≥ `minSessions`) → lock acquisition.

### E.2 Consolidation Lock

Lock file: `.claude/auto_mem/.consolidate-lock`. PID written as body. Stale threshold: 1 hour (even if PID is live — guards against PID reuse). `tryAcquireConsolidationLock()` implements write-then-verify to handle two concurrent reclaimers. `rollbackConsolidationLock()` rewinds mtime on failure.

### E.3 Dream Prompt

The consolidation prompt (`src/services/autoDream/consolidationPrompt.ts:10-65`) is a four-phase workflow:
1. **Orient**: `ls` memory dir, read `MEMORY.md`, skim topic files
2. **Gather**: Read daily logs, check for drifted memories, grep transcripts narrowly
3. **Consolidate**: Write/update topic files, convert relative dates to absolute, delete contradicted facts
4. **Prune**: Update `MEMORY.md` index to stay under 200 lines / 25KB

### E.4 DreamTask Lifecycle

- `registerDreamTask()`: Creates task with `phase: 'starting'`, `status: 'running'`
- `addDreamTurn()`: Tracks tool uses, transitions phase to `'updating'` on first file write
- `completeDreamTask()`: Sets `status: 'completed'`, `notified: true`
- `failDreamTask()`: Sets `status: 'failed'`
- `kill()`: Aborts the agent, rolls back consolidation lock mtime

### E.5 Disk-Skill Dream (KAIROS Alternative)

```typescript
// src/skills/bundled/index.ts:35-40
if (feature('KAIROS') || feature('KAIROS_DREAM')) {
  const { registerDreamSkill } = require('./dream.js')
  registerDreamSkill()
}
```

KAIROS registers a `/dream` skill instead of using the auto-dream forked subagent. The skill runs within the assistant's own context.

### E.6 Tool Permissions for Dream Agent

`createAutoMemCanUseTool()` (`src/services/extractMemories/extractMemories.ts:171-222`) restricts the forked dream agent to: Read/Grep/Glob (unrestricted), Bash (read-only only), Edit/Write (only within memory directory).

---

## Appendix F: Trust Dialog — Full Implementation Detail

### F.1 Trust Computation

`checkHasTrustDialogAccepted()` (`src/utils/config.ts:697-743`) checks:
1. Session-level trust flag (for home directory case)
2. Global config `projects[projectPath].hasTrustDialogAccepted`
3. All parent directories up to filesystem root

Trust only transitions `false → true` (never reversed). The `true` result is latched; `false` is re-checked every call.

### F.2 Persistence Pathways

**Project directories**: Trust saved to `~/.claude/claude.json` under `projects.<git-root-or-cwd>`. Persists across sessions.

**Home directory** (`~`): Trust stored in `STATE.sessionTrustAccepted` (memory only). Forces re-trust on each invocation — prevents malicious `~/.claude/settings.json` from permanently forcing KAIROS.

### F.3 Trust Dialog UI

The dialog (`src/components/TrustDialog/TrustDialog.tsx:207-264`) shows:
- Working directory path (bold)
- Safety explanation text
- Link to security guide
- Binary choice: "Yes, I trust this folder" / "No, exit"

### F.4 Settings Schema Validation

```typescript
// src/utils/settings/types.ts:872-887
...(feature('KAIROS') ? {
  assistant: z.boolean().optional()
    .describe('Start Claude in assistant mode'),
  assistantName: z.string().optional()
    .describe('Display name for the assistant'),
} : {}),
```

The `assistant` field is only present in the schema when `feature('KAIROS')` is enabled at build time.

### F.5 Analytics

- `tengu_trust_dialog_shown`: Logged with risk factors (MCP servers, hooks, bash execution, API key helpers, dangerous env vars)
- `tengu_trust_dialog_accept`: Logged with same risk factors on acceptance

---

## Appendix G: Permission Handler Race — Full Implementation Detail

### G.1 Four-Way Race Architecture

`handleInteractivePermission()` (`src/hooks/toolPermission/handlers/interactiveHandler.ts:57-535`) races:

| Racer | Lines | Trigger |
|-------|-------|---------|
| Bridge (CCR/claude.ai) | 234-298 | `bridgeCallbacks.sendRequest()` |
| Channels (KAIROS) | 300-407 | `channelCallbacks.onResponse()` |
| Permission hooks | 410-431 | `ctx.runHooks()` |
| Async bash classifier | 433-530 | `executeAsyncClassifierCheck()` |

All racers use `claim()` — an atomic guard from `createResolveOnce()` — ensuring exactly one winner. Every win point cleans up all other racers.

### G.2 Channel Permission Request Flow

1. Generate `shortRequestId()` (5-letter, no 'l', profanity-filtered)
2. Filter channel clients to those in `--channels` list
3. Send `notifications/claude/channel/permission_request` to all connected clients (fire-and-forget)
4. Subscribe via `channelCallbacks.onResponse()` — on match, `claim()` → resolve
5. On abort signal, unsubscribe map entry + listener

### G.3 Handler Dispatch Order

`useCanUseTool.tsx:93-169` dispatches sequentially:
1. **Coordinator** (`awaitAutomatedChecksBeforeDialog: true`): hooks + classifier sequentially. No channels, no bridge, no UI.
2. **Swarm worker**: Forward to leader via mailbox. Classifier runs locally first.
3. **Interactive**: Full four-way race.

Coordinator mode is set on background agents (`isAsync && !shouldAvoidPrompts`). KAIROS channels **only participate in the interactive handler** — never in coordinator or swarm worker handlers.

---

## Appendix H: Team Initialization and Agent Spawning — Full Implementation Detail

### H.1 Teammate Mode Snapshot

`captureTeammateModeSnapshot()` (`src/utils/swarm/backends/teammateModeSnapshot.ts:55-67`) captures the mode at startup. CLI override (`setCliTeammateModeOverride`) takes precedence. The snapshot is immutable for the session — runtime config changes don't affect it.

Type: `'auto' | 'tmux' | 'in-process'`

### H.2 Team Context Data Structure

```typescript
{
  teamName: string,
  teamFilePath: string,
  leadAgentId: string,
  selfAgentId: string | undefined,
  selfAgentName: string | undefined,
  isLeader: boolean,
  teammates: { [agentId: string]: TeammateInfo },
}
```

### H.3 Spawn Backend Selection

`handleSpawn()` (`src/tools/shared/spawnMultiAgent.ts:1040-1078`) dispatches:
- If `isInProcessEnabled()` → `handleSpawnInProcess()`
- Else, try `detectAndGetBackend()`:
  - If backend available → `handleSpawnSplitPane()` or `handleSpawnSeparateWindow()`
  - If auto mode and no backend → `markInProcessFallback()` → `handleSpawnInProcess()`
  - If explicit tmux mode and no backend → error with install instructions

### H.4 In-Process Spawning

`handleSpawnInProcess()` (`src/tools/shared/spawnMultiAgent.ts:840-1034`):
1. Generate unique name and deterministic agent ID
2. Assign teammate color
3. Create independent `AbortController`
4. Create `TeammateContext` for `AsyncLocalStorage`
5. Register task via `spawnInProcessTeammate()`
6. Start execution loop via `startInProcessTeammate()` (fire-and-forget)
7. Update AppState `teamContext.teammates` map
8. Auto-register leader if no prior `spawnTeam` call

### H.5 Teammate State Inheritance

- **tmux**: CLI args propagation (`--agent-id`, `--agent-name`, `--team-name`, `--agent-color`, `--parent-session-id`)
- **in-process**: `AsyncLocalStorage` context isolation via `runWithTeammateContext()`
- Both: Excluded from KAIROS re-initialization by `!options.agentId` check

---

## Appendix I: Proactive Mode Composition — Full Implementation Detail

### I.1 Module Loading

```typescript
// src/utils/systemPrompt.ts:14-20
const proactiveModule =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('../proactive/index.js')
    : null
```

Both `PROACTIVE` and `KAIROS` feature flags enable proactive module loading.

### I.2 Tick Mechanism

Ticks are XML-wrapped timestamps (`<tick>HH:MM:SS</tick>`) injected periodically. The `SleepTool` controls inter-tick wait time. Multiple ticks may batch into a single message.

### I.3 SleepTool Gating

```typescript
// src/tools.ts:25-28
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

`SleepTool.isEnabled()` returns `isProactiveActive()`. Proactive must be activated BEFORE `getTools()` to include Sleep in the tool list.

### I.4 System Prompt Composition Order

1. **Proactive prompt** (`src/main.tsx:2197-2204`): Appended if proactive + not coordinator. Includes dynamic `briefVisibility` string.
2. **KAIROS addendum** (`src/main.tsx:2206-2208`): Appended if KAIROS enabled. Contains assistant-specific instructions.

The proactive prompt's autonomous work section (`src/constants/prompts.ts:860-914`) includes: tick handling, pacing (Sleep usage), first wake-up behavior, bias toward action, conciseness rules, terminal focus sensitivity, and (conditionally) the Brief proactive section.

### I.5 Agent Composition

In proactive mode, custom agent instructions **append** to the default prompt rather than replacing it (`src/utils/systemPrompt.ts:103-113`).

---

## Appendix J: Session Viewer and History — Full Implementation Detail

### J.1 CLI Argument Parsing

`claude assistant [sessionId]` is detected at `src/main.tsx:685-700`. If sessionId is present, it's stored in `_pendingAssistantChat.sessionId`. If bare `claude assistant`, `_pendingAssistantChat.discover = true`.

### J.2 Discovery Flow

1. Call `discoverAssistantSessions()` (compiled module)
2. If zero sessions: launch install wizard (`launchAssistantInstallWizard`)
3. If one session: auto-select
4. If multiple: show chooser dialog (`launchAssistantSessionChooser`)

### J.3 Viewer Initialization

At `src/main.tsx:3259-3354`:
1. Set `kairosActive = true`, `userMsgOptIn = true`, `isRemoteMode = true`
2. Create `RemoteSessionConfig` with `viewerOnly: true`
3. Launch REPL with `initialTools: []` (no local tools), `isBriefOnly: true`

### J.4 History Lazy-Loading

`useAssistantHistory` (`src/hooks/useAssistantHistory.ts:72-239`):
- **Initial**: Fetch latest page via `anchor_to_latest`, chain fill-viewport loads (max 10 pages)
- **Scroll-up**: Trigger at `PREFETCH_THRESHOLD_ROWS = 40` rows from top
- **Scroll anchor**: Snapshot height before prepend, compensate in `useLayoutEffect`
- **Sentinels**: "loading older messages…" / "start of session" at index 0

### J.5 WebSocket Event Streaming

`SessionsWebSocket` (`src/remote/SessionsWebSocket.ts:74-163`):
- URL: `wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe`
- Auth via headers (Bearer token)
- Auto-reconnect with backoff
- Ping interval for keepalive

### J.6 Message Posting

`sendEventToRemoteSession()` (`src/utils/teleport/api.ts:349-417`):
- POST to `/v1/sessions/{sessionId}/events`
- 30-second timeout (cold-start margin)
- Echo filtering via `BoundedUUIDSet` (UUID recorded pre-POST)

### J.7 Permission Handling

`RemoteSessionManager.respondToPermissionRequest()` sends `SDKControlResponse` via WebSocket with `behavior: 'allow' | 'deny'`.

---

## Appendix K: UI Rendering Pipeline — Full Implementation Detail

### K.1 AppState Fields

- `isBriefOnly: boolean` — brief-only view active (`src/state/AppStateStore.ts:96`)
- `kairosEnabled: boolean` — KAIROS fully enabled (`src/state/AppStateStore.ts:116`)

### K.2 Brief Layout Detection

`UserPromptMessage.tsx:51-61` computes `useBriefLayout` from:
```
(getKairosActive() || getUserMsgOptIn() && entitlement) && isBriefOnly && !isTranscriptMode && !viewingAgentTaskId
```

When true: no background color, "You" label, 2-column indent.

### K.3 Spinner Branching

`SpinnerWithVerb` (`src/components/Spinner.tsx:62-81`) switches to `BriefSpinner` (compact single-line status) when brief layout is active. Shows verb, connection status, and background task count.

### K.4 Streaming Text Visibility

In brief-only mode, streaming text and streaming thinking are hidden:
```typescript
{streamingText && !isBriefOnly && <Box>...</Box>}
{isStreamingThinkingVisible && streamingThinking && !isBriefOnly && <Box>...</Box>}
```

### K.5 Agent Tool Async Forcing

```typescript
// src/tools/AgentTool/AgentTool.tsx:566
const assistantForceAsync = feature('KAIROS') ? appState.kairosEnabled : false;
```

When KAIROS is enabled, agents are forced into async mode.

---

## Appendix L: Fast Mode and Model Selection — Full Implementation Detail

### L.1 KAIROS Fast Mode Exemption

```typescript
// src/utils/fastMode.ts:96-110
if (getIsNonInteractiveSession() && preferThirdPartyAuthentication() && !getKairosActive()) {
  // Block fast mode in SDK unless explicitly opted in
}
```

KAIROS sessions are exempt from the SDK fast-mode block — they're first-party orchestration. `kairosActive` is set before the check runs.

### L.2 Model Resolution Priority

1. Model override during session (`/model` command)
2. Model override at startup (`--model` flag)
3. `ANTHROPIC_MODEL` environment variable
4. Settings (`model` in user settings)
5. Built-in default

### L.3 Fast Mode Support

Only Opus 4.6 supports fast mode (`src/utils/fastMode.ts:167-176`). Fast mode has a state machine: `active` → `cooldown` (rate limit / overloaded) → `active` (on timer expiry).

---

## Appendix M: Memory Path System — Full Implementation Detail

### M.1 Path Resolution

```typescript
// src/memdir/paths.ts:223-235
export const getAutoMemPath = memoize((): string => {
  const override = getAutoMemPathOverride() ?? getAutoMemPathSetting()
  if (override) return override
  const projectsDir = join(getMemoryBaseDir(), 'projects')
  return (join(projectsDir, sanitizePath(getAutoMemBase()), 'memory') + sep).normalize('NFC')
}, () => getProjectRoot())
```

### M.2 Daily Log Path

```typescript
// src/memdir/paths.ts:246-251
export function getAutoMemDailyLogPath(date = new Date()): string {
  const yyyy = date.getFullYear().toString()
  const mm = (date.getMonth() + 1).toString().padStart(2, '0')
  const dd = date.getDate().toString().padStart(2, '0')
  return join(getAutoMemPath(), 'logs', yyyy, mm, `${yyyy}-${mm}-${dd}.md`)
}
```

### M.3 Auto Memory Enable Decision Tree

`isAutoMemoryEnabled()` (`src/memdir/paths.ts:21-55`) priority:
1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY` env var (truthy → OFF, falsy → ON)
2. `CLAUDE_CODE_SIMPLE` (`--bare`) → OFF
3. CCR without `CLAUDE_CODE_REMOTE_MEMORY_DIR` → OFF
4. `autoMemoryEnabled` in settings (explicit opt-out)
5. Default: enabled

### M.4 Memory Prompt Assembly Pipeline

`loadMemoryPrompt()` (`src/memdir/memdir.ts:419-507`) dispatches:
1. **KAIROS + auto enabled + kairosActive**: → `buildAssistantDailyLogPrompt(skipIndex)`
2. **TEAMMEM enabled**: → `buildCombinedMemoryPrompt()`
3. **Auto only**: → `buildMemoryLines()`
4. **Disabled**: → log `tengu_memdir_disabled`, return null

### M.5 What NOT to Save

Five exclusion rules (`src/memdir/memoryTypes.ts:183-195`):
- Code patterns, architecture, file paths (derivable from code)
- Git history (authoritative via `git log` / `git blame`)
- Debugging solutions (fix is in the code, context in commit message)
- Anything in CLAUDE.md files
- Ephemeral task details

These apply **even when the user explicitly asks** — the agent should ask what was surprising or non-obvious instead.

---

## Appendix N: GrowthBook Analytics — Full Implementation Detail

### N.1 Caching Architecture

- **In-memory**: `remoteEvalFeatureValues` Map — authoritative after `processRemoteEvalPayload`
- **Disk**: `~/.claude/claude.json` → `cachedGrowthBookFeatures` — synced on every successful payload
- **Exposure dedup**: `loggedExposures` Set prevents duplicate exposure events in hot paths

### N.2 Accessor Functions

- `getFeatureValue_CACHED_MAY_BE_STALE<T>(feature, default)`: Pure read from memory or disk cache. Preferred for startup-critical and sync contexts.
- `getFeatureValue_CACHED_WITH_REFRESH<T>(feature, default, refreshMs)`: **Deprecated** — now delegates to `_CACHED_MAY_BE_STALE`. Disk cache is synced on every payload load.

### N.3 Refresh Intervals

- **Production**: 6 hours
- **Internal (ants)**: 20 minutes

### N.4 Complete KAIROS Gate Reference

| Gate | Type | Purpose |
|------|------|---------|
| `tengu_kairos` | Boolean | Main activation entitlement |
| `tengu_kairos_brief` | Boolean | Brief tool availability |
| `tengu_kairos_cron` | Boolean | Cron scheduler enable/disable |
| `tengu_kairos_cron_durable` | Boolean | Durable cron kill switch |
| `tengu_kairos_cron_config` | JSON | Cron jitter parameters |
| `tengu_harbor` | Boolean | Channel notifications |
| `tengu_harbor_permissions` | Boolean | Channel permission relay |
| `tengu_harbor_ledger` | JSON | Channel plugin allowlist |
| `tengu_moth_copse` | Boolean | Skip MEMORY.md index in prompt |
| `tengu_coral_fern` | Boolean | Memory search instructions |
| `tengu_onyx_plover` | JSON | AutoDream config (minHours, minSessions) |

### N.5 Analytics Metadata

When KAIROS is active, all analytics events include:
```typescript
{ kairosActive: true }            // metadata.ts:736
{ is_assistant_mode: true }       // metadata.ts:967 (1P format)
{ assistantActivationPath: ... }  // main.tsx:2518
```

Datadog tag: `kairosActive` included in allowed tag set (`src/services/analytics/datadog.ts:72`).
