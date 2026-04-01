# Claude Code: Analytics & Telemetry System — Design Specification

This document analyzes the analytics and telemetry architecture of Claude Code — Anthropic's
agentic CLI tool — focusing on how it collects, routes, enriches, and exports observability
data across its multi-sink telemetry pipeline.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Three Telemetry Sinks](#2-three-telemetry-sinks)
- [3. Datadog Integration](#3-datadog-integration)
- [4. First-Party Event Logging](#4-first-party-event-logging)
- [5. GrowthBook Feature Flags](#5-growthbook-feature-flags)
- [6. Metadata Collection](#6-metadata-collection)
- [7. Permission Decision Audit](#7-permission-decision-audit)
- [8. OpenTelemetry Integration](#8-opentelemetry-integration)
- [9. PII Handling](#9-pii-handling)
- [10. Analytics Events Catalog](#10-analytics-events-catalog)
- [11. Cost Tracking](#11-cost-tracking)
- [12. Design Principles](#12-design-principles)

---

## 1. Overview

Claude Code's analytics system provides observability into every meaningful decision the
agent makes — from API calls and tool invocations to permission decisions and feature flag
evaluations. The system exists to answer three categories of questions:

1. **Operational health**: Is the system working? What are error rates, latencies, and
   resource consumption patterns across the user population?
2. **Safety auditing**: Did the oversight layers make correct decisions? How often does
   the classifier approve versus deny? What is the user override frequency?
3. **Product understanding**: How are features adopted? Which tools are used most? What
   is the cost distribution across subscription tiers?

The analytics infrastructure is deliberately designed with **zero dependencies on the rest
of the codebase**. The main entry point (`src/services/analytics/index.ts`) imports nothing
from Claude Code's application modules — it defines only a queue, a sink interface, and
two marker types. This isolation prevents import cycles and ensures events can be logged
from any module at any initialization stage, even before the analytics backend is attached.

Events logged before the sink is attached are queued in memory. When `attachAnalyticsSink()`
is called during app startup, queued events drain asynchronously via `queueMicrotask()` to
avoid adding latency to the startup path. The attachment is idempotent — calling it from
both the preAction hook (for subcommands) and `setup()` (for the default command) is safe.

> **Source:** `src/services/analytics/index.ts`

---

## 2. Three Telemetry Sinks

Every analytics event fans out to up to three independent sinks, each serving a different
audience and retention model:

```
logEvent(eventName, metadata)
        │
        ▼
┌───────────────────────┐
│     Event Sampling     │  tengu_event_sampling_config (GrowthBook)
│   (per-event-type)     │  sample_rate 0–1, dropped if not selected
└───────────┬───────────┘
            │
    ┌───────┼───────────────────┐
    │       │                   │
    ▼       ▼                   ▼
┌────────┐ ┌──────────────┐ ┌────────────────────┐
│Datadog │ │ 1P Event     │ │ OpenTelemetry       │
│ Logs   │ │ Logging      │ │ (customer OTLP)     │
│        │ │ (BigQuery)   │ │                     │
├────────┤ ├──────────────┤ ├────────────────────┤
│Gated   │ │Always-on     │ │Separate logger     │
│behind  │ │(when not     │ │from bootstrap      │
│tengu_  │ │analytics-    │ │state; events emit  │
│log_    │ │disabled)     │ │via logOTelEvent()  │
│datadog_│ │              │ │                     │
│events  │ │Batched via   │ │Prefixed with       │
│        │ │OTel SDK      │ │claude_code.*       │
│HTTP    │ │BatchLog-     │ │                     │
│POST to │ │Record-       │ │Includes prompt.id  │
│DD Logs │ │Processor     │ │and workspace paths │
│API     │ │              │ │                     │
│        │ │Exported to   │ │                     │
│Events  │ │/api/event_   │ │                     │
│allow-  │ │logging/batch │ │                     │
│listed  │ │              │ │                     │
└────────┘ └──────────────┘ └────────────────────┘
```

### 2.1 Sink 1: Datadog (Operational Dashboards)

Datadog receives a curated subset of events (allowlisted by name) for real-time operational
monitoring. Events are batched and flushed every 15 seconds or when the batch reaches 100
entries.

### 2.2 Sink 2: First-Party Event Logging (BigQuery Analytics)

The primary analytics backend. All events flow through the OpenTelemetry SDK's
`BatchLogRecordProcessor`, are serialized as protobuf `ClaudeCodeInternalEvent` messages, and
exported to Anthropic's `/api/event_logging/batch` endpoint. Failed events are persisted to
disk (`~/.claude/telemetry/`) for retry with quadratic backoff.

### 2.3 Sink 3: OpenTelemetry (Customer Telemetry)

A separate OTel logger (from `getEventLogger()` in bootstrap state) emits structured events
to customer-configured OTLP endpoints. This is the **third-party telemetry** path —
completely independent of the first-party pipeline. Events are prefixed with `claude_code.`
and carry monotonically increasing sequence numbers.

### 2.4 In-Session Store (toolDecisions Map)

Not a telemetry sink in the export sense, but an in-process Map on `ToolUseContext` that
records every permission decision by `toolUseID`. This store enables:

- Deduplication of OTel span attribution (attaching the decision to the tool's execution span)
- Post-hoc inspection by downstream code (e.g., did this tool call get classifier-approved?)
- SDK consumers reading decision metadata via the `coreSchemas` export

> **Source:** `src/hooks/toolPermission/permissionLogging.ts:221-228`, `src/Tool.ts:258-265`

---

## 3. Datadog Integration

> **Source:** `src/services/analytics/datadog.ts`

### 3.1 Architecture

Datadog integration uses the HTTP Logs Intake API (`https://http-intake.logs.us5.datadoghq.com/api/v2/logs`)
with a client token for authentication. Events are buffered in a module-level `logBatch` array
and flushed on a timer or when the batch reaches `MAX_BATCH_SIZE = 100`.

```
Event arrives → push to logBatch → batch full? → flush immediately
                                    │ no
                                    ▼
                              schedule flush (15s timer, .unref())
```

### 3.2 Event Allowlist

Only a curated set of ~40 event names are forwarded to Datadog. High-volume events
(like per-token usage) are excluded to control costs. The allowlist covers:

| Category | Events |
|----------|--------|
| **Lifecycle** | `tengu_init`, `tengu_started`, `tengu_exit`, `tengu_cancel` |
| **API** | `tengu_api_error`, `tengu_api_success`, `tengu_query_error` |
| **OAuth** | `tengu_oauth_error/success`, `tengu_oauth_token_refresh_*` (8 events) |
| **Permissions** | `tengu_tool_use_granted_in_prompt_*`, `tengu_tool_use_rejected_in_prompt`, `tengu_tool_use_error/success` |
| **Features** | `tengu_brief_*`, `tengu_voice_*`, `tengu_flicker`, `tengu_model_fallback_triggered` |
| **Chrome Bridge** | `chrome_bridge_*` (7 events) |
| **Team Memory** | `tengu_team_mem_sync_*` (4 events) |

### 3.3 Tag Set

Datadog logs carry structured tags for filtering and aggregation. The tag fields are:

```
arch, clientType, errorType, http_status_range, http_status,
kairosActive, model, platform, provider, skillMode,
subscriptionType, toolName, userBucket, userType, version, versionBase
```

Tags are built by prepending `event:<name>` (so the event name is searchable via the DD
log search API, since the `message` field is a DD reserved field and not queryable from
dashboard widgets) followed by each TAG_FIELD that has a non-null value, converted to
snake_case.

### 3.4 Cardinality Controls

Several transformations reduce tag cardinality to keep Datadog costs bounded:

- **MCP tool names**: All `mcp__*` tool names are normalized to `"mcp"` — the server-specific
  suffix is stripped
- **Model names**: For external users, model names are mapped through `getCanonicalName()` and
  validated against `MODEL_COSTS`; unrecognized models become `"other"`
- **Dev versions**: Timestamps and SHAs are stripped from dev version strings
  (`2.0.53-dev.20251124.t173302.sha526cc6a` becomes `2.0.53-dev.20251124`)

### 3.5 User Bucketing

Rather than tracking individual user IDs (which would explode cardinality), Datadog events
carry a `userBucket` — a deterministic SHA-256 hash of the user ID modulo 30. This allows
estimating unique affected users during incidents without exposing identity.

### 3.6 Third-Party Provider Exclusion

Datadog events are silently dropped for third-party API providers (Bedrock, Vertex, Foundry).
Only first-party Anthropic API traffic is tracked.

### 3.7 Gating

Datadog event dispatch is gated behind the `tengu_log_datadog_events` GrowthBook feature flag,
checked via `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()`. The gate state is cached at
module level and initialized during `initializeAnalyticsGates()` at startup. Early events
use the cached value from the previous session's disk-persisted GrowthBook state to avoid
data loss during initialization.

### 3.8 Shutdown

`shutdownDatadog()` is called from `gracefulShutdown()` before `process.exit()`, since
`forceExit()` prevents the `beforeExit` handler from firing. It clears the flush timer and
performs a final `flushLogs()`.

---

## 4. First-Party Event Logging

> **Source:** `src/services/analytics/firstPartyEventLogger.ts`, `src/services/analytics/firstPartyEventLoggingExporter.ts`

### 4.1 Architecture

The first-party (1P) event logging system is built on top of the OpenTelemetry SDK's log
infrastructure, but operates as a **completely separate pipeline** from customer telemetry.
It has its own `LoggerProvider`, its own `BatchLogRecordProcessor`, and its own exporter.

```
logEventTo1P(eventName, metadata)
    │
    ▼ (fire-and-forget via void promise)
logEventTo1PAsync()
    │ enriches with getEventMetadata(), getCoreUserData()
    ▼
firstPartyEventLogger.emit({ body, attributes })
    │
    ▼ (OpenTelemetry BatchLogRecordProcessor)
FirstPartyEventLoggingExporter.export()
    │ transforms ReadableLogRecord[] → ClaudeCodeInternalEvent protos
    │ chunks into batches of maxBatchSize
    ▼
POST /api/event_logging/batch
    │ success → reset backoff, retry any queued events
    │ failure → append to ~/.claude/telemetry/1p_failed_events.{sessionId}.{batchUuid}.json
    │           schedule quadratic backoff retry
    ▼
(disk-backed retry on next export cycle or next process run)
```

### 4.2 Event Types

The exporter handles two event types:

| Type | Proto | When |
|------|-------|------|
| `ClaudeCodeInternalEvent` | `events_mono/claude_code/v1/` | All standard analytics events |
| `GrowthbookExperimentEvent` | `events_mono/growthbook/v1/` | Feature flag experiment assignments |

### 4.3 Batching Configuration

Batch processor parameters are remotely configurable via the `tengu_1p_event_batch_config`
GrowthBook dynamic config:

| Parameter | Default | Config Key |
|-----------|---------|------------|
| Export interval | 10,000 ms | `scheduledDelayMillis` |
| Max batch size | 200 | `maxExportBatchSize` |
| Max queue size | 8,192 | `maxQueueSize` |
| Skip auth | false | `skipAuth` |
| Max retry attempts | 8 | `maxAttempts` |
| API path | `/api/event_logging/batch` | `path` |
| Base URL | `https://api.anthropic.com` | `baseUrl` |

### 4.4 Resilience Model

The `FirstPartyEventLoggingExporter` implements a multi-layer resilience strategy:

1. **Append-only failed event log**: Events that fail to export are appended to a JSONL
   file keyed by `{sessionId}.{batchUuid}`. The append operation is atomic on most
   filesystems, making it concurrency-safe.

2. **Quadratic backoff retry**: Failed batches trigger a retry schedule with
   `delay = baseBackoffDelayMs * attempts^2`, capped at `maxBackoffDelayMs` (30s).
   After `maxAttempts` (8), events are dropped.

3. **Short-circuit on first failure**: When sending batches sequentially, the first batch
   failure causes all remaining unsent batches to be queued to disk without attempting
   network calls — the assumption is that the endpoint is down.

4. **Success-triggered drain**: When any export succeeds, queued events from disk are
   immediately retried (the endpoint is healthy again).

5. **Cross-run retry**: On startup, the exporter scans `~/.claude/telemetry/` for failed
   event files from previous runs of the same session and retries them in the background.

6. **Auth fallback**: On 401 errors, the exporter retries the same batch without auth
   headers. This handles the case where the OAuth token has expired but the endpoint
   accepts unauthenticated events.

### 4.5 Hot Reinit on Config Change

When GrowthBook refreshes, `reinitialize1PEventLoggingIfConfigChanged()` compares the new
batch config against the last-used config. If they differ, it:

1. Nulls the logger (concurrent `logEventTo1P()` calls bail at the guard)
2. Force-flushes the old processor's buffer
3. Creates a new `LoggerProvider` + exporter with the new config
4. Shuts down the old provider in the background

Failed events from the old exporter's disk file are automatically picked up by the new
exporter because the file path is keyed by module-level `BATCH_UUID` + `sessionId`, both
unchanged across reinit.

### 4.6 Event Sampling

Events can be sampled at configurable rates via `tengu_event_sampling_config`, a GrowthBook
dynamic config mapping event names to `{ sample_rate: number }`. The sampling decision is
made per-event at dispatch time in `sink.ts`:

- `sample_rate` absent or 1.0 → log at 100% (no metadata addition)
- `sample_rate` between 0 and 1 → random selection; if selected, `sample_rate` is added
  to metadata for statistical correction
- `sample_rate` = 0 → drop entirely

> **Source:** `src/services/analytics/firstPartyEventLogger.ts:57-85`

---

## 5. GrowthBook Feature Flags

> **Source:** `src/services/analytics/growthbook.ts`

### 5.1 Role in the Analytics System

GrowthBook serves as the feature flag and experiment infrastructure for Claude Code. It
controls:

- Which analytics sinks are active (Datadog gate, sink killswitches)
- Event sampling rates
- 1P batch processor configuration
- Feature rollouts that generate experiment exposure events
- Security restriction gates

### 5.2 Initialization

GrowthBook uses **remote evaluation** (`remoteEval: true`), meaning the server pre-evaluates
all feature flags for the user's attributes and returns the complete set of resolved values.
The client never evaluates rules locally.

```
getUserAttributes()
    │ id, sessionId, deviceID, platform, organizationUUID,
    │ accountUUID, userType, subscriptionType, rateLimitTier,
    │ firstTokenTime, email, appVersion, apiBaseUrlHost
    ▼
new GrowthBook({ remoteEval: true, cacheKeyAttributes: ['id', 'organizationUUID'] })
    │
    ▼
client.init({ timeout: 5000 })
    │
    ▼ (on success)
processRemoteEvalPayload()
    │ fixes API format (value → defaultValue workaround)
    │ populates remoteEvalFeatureValues Map
    │ extracts experiment assignment data
    ▼
syncRemoteEvalToDisk()  →  ~/.claude/claude.json: cachedGrowthBookFeatures
    │
    ▼
refreshed.emit()  →  notifies subscribers (reinitialize1PEventLogging, etc.)
```

### 5.3 Three-Tier Caching

Feature values are resolved through a priority cascade:

1. **Env var overrides** (`CLAUDE_INTERNAL_FC_OVERRIDES`): JSON object, ant-only, for
   eval harnesses. Highest priority.
2. **Config overrides** (`getGlobalConfig().growthBookOverrides`): Per-feature overrides
   set via `/config Gates` tab, ant-only.
3. **In-memory map** (`remoteEvalFeatureValues`): Authoritative after init completes.
   Populated by `processRemoteEvalPayload()`.
4. **Disk cache** (`~/.claude/claude.json → cachedGrowthBookFeatures`): Survives across
   process restarts. Written on every successful payload.

### 5.4 Accessor Functions

| Function | Blocks? | Use Case |
|----------|---------|----------|
| `getFeatureValue_CACHED_MAY_BE_STALE<T>()` | No | Startup-critical paths, sync contexts, render loops |
| `getDynamicConfig_CACHED_MAY_BE_STALE<T>()` | No | Object-valued configs (sampling, batch config) |
| `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` | No | Boolean gates (migration wrapper) |
| `checkSecurityRestrictionGate()` | Maybe | Security gates; waits if reinit in progress |
| `checkGate_CACHED_OR_BLOCKING()` | Maybe | User-facing features; fast-path if disk says true |
| `getFeatureValue_DEPRECATED<T>()` | Yes | Legacy; blocks on init |

The preferred accessor is `_CACHED_MAY_BE_STALE` — it reads from memory first, falls back
to disk, and never blocks. The `_CACHED_WITH_REFRESH` variant is deprecated and delegates
directly to `_CACHED_MAY_BE_STALE`.

### 5.5 Periodic Refresh

After initialization, a periodic refresh runs every:
- **6 hours** for external users
- **20 minutes** for internal (ant) users

The refresh calls `client.refreshFeatures()`, then re-runs `processRemoteEvalPayload()` to
rebuild the in-memory map, sync to disk, and notify subscribers.

### 5.6 Experiment Exposure Tracking

When a feature backed by a GrowthBook experiment is accessed, the system logs an exposure
event to the 1P pipeline via `logGrowthBookExperimentTo1P()`. Exposures are **deduplicated
per session** via a `loggedExposures` Set — each feature fires at most one exposure event.

Features accessed before init completes are added to `pendingExposures`. Once init succeeds,
all pending features have their exposures logged retroactively.

### 5.7 Auth Change Handling

`refreshGrowthBookAfterAuthChange()` destroys and recreates the GrowthBook client because
`apiHostRequestHeaders` cannot be updated after client creation. The full sequence:

1. `resetGrowthBook()` — destroy client, clear all caches
2. `refreshed.emit()` — notify subscribers to re-read (falls to disk cache)
3. `initializeGrowthBook()` — recreate with fresh auth headers
4. Track reinit promise so `checkSecurityRestrictionGate()` can await it

### 5.8 Kill Switch: The Sink Killswitch

> **Source:** `src/services/analytics/sinkKillswitch.ts`

A GrowthBook dynamic config (`tengu_frond_boric`, deliberately mangled name) provides
per-sink killswitches:

```typescript
type SinkName = 'datadog' | 'firstParty'
// Config shape: { datadog?: boolean, firstParty?: boolean }
// true = sink killed, false/absent = sink alive
```

`isSinkKilled()` is checked at dispatch sites (not in `is1PEventLoggingEnabled()` to avoid
circular imports with `growthbook.ts`). When a sink is killed:

- New events to that sink are silently dropped
- The 1P exporter's backoff retry also checks the killswitch per-POST, so disabling the
  firstParty sink stops all network traffic including retries

---

## 6. Metadata Collection

> **Source:** `src/services/analytics/metadata.ts`

### 6.1 The Metadata Enrichment Pipeline

Every analytics event is enriched with a comprehensive metadata envelope before export.
The `getEventMetadata()` function collects ~40 fields organized into several categories:

```
getEventMetadata({ model?, betas? })
    │
    ├── model                           ← from options or getMainLoopModel()
    ├── sessionId                       ← from bootstrap state
    ├── userType                        ← USER_TYPE env var
    ├── betas                           ← model-specific beta flags
    ├── isInteractive                   ← from bootstrap state
    ├── clientType                      ← from bootstrap state
    ├── subscriptionType                ← OAuth subscription tier
    ├── kairosActive                    ← assistant mode flag
    ├── rh                              ← hashed repo remote URL (first 16 chars SHA-256)
    ├── skillMode                       ← discovery / coach / both
    ├── observerMode                    ← backseat / skillcoach / both
    │
    ├── envContext (memoized, once per session)
    │   ├── platform, platformRaw, arch
    │   ├── nodeVersion, terminal
    │   ├── packageManagers, runtimes
    │   ├── isRunningWithBun, isCi, isClaubbit
    │   ├── isClaudeCodeRemote, isLocalAgentMode, isConductor
    │   ├── remoteEnvironmentType, coworkerType
    │   ├── claudeCodeContainerId, claudeCodeRemoteSessionId
    │   ├── tags
    │   ├── isGithubAction, isClaudeCodeAction
    │   ├── isClaudeAiAuth
    │   ├── version, versionBase, buildTime
    │   ├── deploymentEnvironment
    │   ├── github* (event name, runner env/OS, action ref)
    │   ├── wslVersion, linuxDistro*, linuxKernel
    │   └── vcs (git, hg, svn, etc.)
    │
    ├── processMetrics (per-event, all users)
    │   ├── uptime, rss, heapTotal, heapUsed
    │   ├── external, arrayBuffers, constrainedMemory
    │   ├── cpuUsage (user + system microseconds)
    │   └── cpuPercent (delta since last event)
    │
    ├── agentIdentification
    │   ├── agentId                     ← AsyncLocalStorage context or env var
    │   ├── parentSessionId             ← team lead's session
    │   ├── agentType                   ← teammate | subagent | standalone
    │   └── teamName                    ← swarm team name
    │
    └── SWE-bench fields
        ├── sweBenchRunId
        ├── sweBenchInstanceId
        └── sweBenchTaskId
```

### 6.2 Environment Context

`buildEnvContext()` is **memoized** — it runs once per session and caches the result. This
is critical because it runs `Promise.all()` over four async operations (package manager
detection, runtime detection, Linux distro info, VCS detection) that would be wasteful to
repeat per-event.

One notable exception: `kairosActive` is deliberately **not** inside the memoized
`buildEnvContext()`. The `setKairosActive()` call in `main.tsx` runs after the first event
may have already fired and memoized the env context. Instead, `kairosActive` is read fresh
per-event from bootstrap state.

### 6.3 Process Metrics

Every event includes a snapshot of process resource consumption: memory usage (RSS, heap,
external, array buffers, constrained memory), CPU usage (user + system microseconds), and
a **delta CPU percentage** computed from the wall-clock time and CPU time since the last
event. This enables detecting performance regressions and memory leaks in production.

### 6.4 Agent Identification

For multi-agent scenarios (swarm teammates, Agent tool subagents), the metadata includes
provenance tracking. The resolution order is:

1. **AsyncLocalStorage context** — for subagents running in the same process
2. **Env var / swarm helpers** — for swarm teammates in separate processes
3. **Bootstrap state** — for parent session ID in plan-to-implementation transitions

### 6.5 1P Event Format Transformation

`to1PEventFormat()` transforms the camelCase `EventMetadata` into the snake_case
`ClaudeCodeInternalEvent` proto format. The transformation is type-checked against the
proto-generated `EnvironmentMetadata` type — adding a field to the hand-written formatter
that the proto does not define is a **compile error**. This prevents the class of bugs
where fields are added to the formatter but silently dropped by `toJSON()` because the
proto schema was not updated.

> **Source:** `src/services/analytics/metadata.ts:813-819` (comment)

### 6.6 Tool Name Sanitization

MCP tool names follow the format `mcp__<server>__<tool>` and can reveal user-specific
server configurations. `sanitizeToolNameForAnalytics()` redacts these to `'mcp_tool'` for
general-access backends, while built-in tool names (Bash, Read, Write, etc.) pass through
unchanged.

Detailed MCP tool names are logged only when:
- The entrypoint is `local-agent` (Cowork — no ZDR concept)
- The MCP server type is `claudeai-proxy` (always official)
- The server URL matches the official MCP registry
- The server is a built-in (e.g., `computer-use`, feature-gated)

### 6.7 File Extension Extraction

For code-edit analytics, `getFileExtensionForAnalytics()` extracts and sanitizes file
extensions. Extensions longer than 10 characters are replaced with `'other'` to avoid
logging potentially sensitive hash-based filenames. For bash commands,
`getFileExtensionsFromBashCommand()` splits on compound operators, identifies file-command
tokens (`rm`, `mv`, `cp`, `grep`, etc.), and extracts extensions from non-flag arguments.

---

## 7. Permission Decision Audit

> **Source:** `src/hooks/toolPermission/permissionLogging.ts`

### 7.1 Central Logging Function

Every permission decision — whether auto-approved by config, classifier-approved in auto
mode, user-approved in a prompt, hook-approved, or denied by any source — flows through
`logPermissionDecision()`. This single entry point fans out to four destinations:

1. **Analytics event** — a distinct event name per approval/rejection source
2. **OTel telemetry** — a `tool_decision` event with decision and source attributes
3. **Code-edit OTel counter** — language-specific metrics for file editing tools
4. **In-session store** — `toolUseContext.toolDecisions` Map

### 7.2 Nine Distinct Event Types

| Event Name | Trigger |
|------------|---------|
| `tengu_tool_use_granted_in_config` | Auto-approved by allowlist in settings |
| `tengu_tool_use_granted_by_classifier` | Approved by the auto-mode AI classifier |
| `tengu_tool_use_granted_in_prompt_permanent` | User approved and saved a permanent rule |
| `tengu_tool_use_granted_in_prompt_temporary` | User approved for this session only |
| `tengu_tool_use_granted_by_permission_hook` | Approved by an external hook |
| `tengu_tool_use_denied_in_config` | Blocked by denylist in settings |
| `tengu_tool_use_rejected_in_prompt` | User rejected in the permission dialog |
| `tengu_auto_mode_decision` | Auto mode classifier decision (allow or deny, with stage info) |
| `tengu_auto_mode_denial_limit_exceeded` | Denial limit breached (3 consecutive or 20 total) |

### 7.3 Base Metadata

Every permission event carries:

| Field | Content |
|-------|---------|
| `messageID` | The message ID (verified not-PII) |
| `toolName` | Sanitized tool name (MCP → `mcp_tool`) |
| `sandboxEnabled` | Whether sandboxing is active |
| `waiting_for_user_permission_ms` | Time the user spent in the permission dialog (only when actually prompted) |

### 7.4 Source Flattening

The structured `PermissionApprovalSource` / `PermissionRejectionSource` discriminated union
is flattened to a string label for analytics and OTel:

| Source Type | Label |
|-------------|-------|
| `{ type: 'classifier' }` | `'classifier'` |
| `{ type: 'hook' }` | `'hook'` |
| `{ type: 'user', permanent: true }` | `'user_permanent'` |
| `{ type: 'user', permanent: false }` | `'user_temporary'` |
| `{ type: 'user_abort' }` | `'user_abort'` |
| `{ type: 'user_reject' }` | `'user_reject'` |
| `'config'` | `'config'` |

### 7.5 Code-Edit Language Metrics

When the tool is a code-editing tool (`Edit`, `Write`, `NotebookEdit`), an additional OTel
counter is incremented with language-specific attributes. The language is derived from the
target file path via `getLanguageName()`. This enables dashboards showing approval rates
segmented by programming language.

---

## 8. OpenTelemetry Integration

> **Source:** `src/utils/telemetry/events.ts`

### 8.1 Event Logger

The OTel event logger is obtained from `getEventLogger()` in bootstrap state — it is a
separate logger from the 1P event logger, configured by the customer's OTLP endpoint
settings. Events are emitted as OTel log records with:

```typescript
{
  body: `claude_code.${eventName}`,
  attributes: {
    ...getTelemetryAttributes(),       // session-level attributes
    'event.name': eventName,
    'event.timestamp': ISO string,
    'event.sequence': monotonic counter,
    'prompt.id': promptId,             // if available
    'workspace.host_paths': paths,     // if set
    ...metadata,                       // event-specific key-value pairs
  },
}
```

### 8.2 Tool Decision Events

The `tool_decision` event is the primary OTel event for permission auditing:

```typescript
logOTelEvent('tool_decision', {
  decision: 'accept' | 'reject',
  source: 'config' | 'classifier' | 'hook' | 'user_permanent' | 'user_temporary' | ...,
  tool_name: sanitizedToolName,
})
```

### 8.3 User Prompt Redaction

User prompt content is **redacted by default** in OTel events. The `OTEL_LOG_USER_PROMPTS`
environment variable must be explicitly set to enable logging of user-provided text.
`redactIfDisabled()` returns `'<REDACTED>'` when the flag is not set.

### 8.4 Tool Input Telemetry

When `OTEL_LOG_TOOL_DETAILS` is enabled, tool input arguments are serialized for the
`tool_result` OTel event via `extractToolInputForTelemetry()`. The serialization applies
aggressive truncation:

| Limit | Value |
|-------|-------|
| String threshold for truncation | 512 chars |
| Truncated string length | 128 chars + `[N chars]` |
| Max JSON output | 4 KB |
| Max collection items | 20 |
| Max nesting depth | 2 |

Keys starting with `_` are filtered out to prevent internal marker keys from leaking into
telemetry.

---

## 9. PII Handling

### 9.1 The Marker Type System

Claude Code uses a type-level verification system to prevent accidental PII logging. Two
marker types enforce developer intent:

```typescript
// For general-access backends (Datadog, 1P additional_metadata)
type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never

// For PII-tagged proto columns with privileged access controls
type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never
```

Both types are `never` — they can never hold a value. They exist only as type-cast
annotations that document the developer's verification that the logged value is safe. The
`logEvent()` metadata type restricts values to `boolean | number | undefined`, which
**structurally prevents string values** from being logged without an explicit cast through
one of these marker types.

### 9.2 _PROTO_* Field Routing

Fields prefixed with `_PROTO_` contain PII-tagged values (like MCP server names, skill
names, plugin names) destined for privileged BigQuery columns. These fields follow a
special routing path:

1. **At dispatch (sink.ts)**: `stripProtoFields()` removes all `_PROTO_*` keys before
   forwarding to Datadog — the general-access backend never sees them.

2. **At 1P export**: The `FirstPartyEventLoggingExporter` destructures known `_PROTO_*`
   keys (`_PROTO_skill_name`, `_PROTO_plugin_name`, `_PROTO_marketplace_name`) and hoists
   them to top-level proto fields. Then `stripProtoFields()` is applied defensively to the
   remaining `additional_metadata` to prevent unrecognized future `_PROTO_*` keys from
   landing in the general-access JSON blob.

### 9.3 MCP Tool Name PII Classification

Per the taxonomy, MCP server names are classified as **medium PII**. They are logged in
full only when the source is known to be non-user-specific:
- Local-agent (Cowork) entrypoint
- claude.ai-proxied connectors
- Official MCP registry URLs
- Built-in MCP servers (feature-gated set)

Custom user-configured MCPs have their names replaced with `'mcp_tool'`.

### 9.4 OTel Redaction

- User prompts are redacted unless `OTEL_LOG_USER_PROMPTS=1`
- Tool input details are omitted unless `OTEL_LOG_TOOL_DETAILS=1`
- File extensions longer than 10 characters are replaced with `'other'`
- Workspace host paths are included in events but not in metrics (cardinality concern)
- `prompt.id` is added to events but not metrics for the same reason

---

## 10. Analytics Events Catalog

### 10.1 Lifecycle Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `tengu_init` | Analytics sink attached | `queued_event_count` (ant-only) |
| `tengu_started` | Session fully initialized | env context, model |
| `tengu_exit` | Session ending | cost, duration, token counts |
| `tengu_cancel` | User cancelled operation | — |

### 10.2 API Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `tengu_api_success` | API call completed | model, tokens, latency, cache stats |
| `tengu_api_error` | API call failed | status, errorType, http_status_range |
| `tengu_query_error` | Query pipeline error | error classification |
| `tengu_model_fallback_triggered` | Model fallback activated | original model, fallback model |

### 10.3 Tool Use Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `tengu_tool_use_success` | Tool executed successfully | toolName, duration |
| `tengu_tool_use_error` | Tool execution failed | toolName, errorType |
| `tengu_tool_use_granted_*` | Permission granted (6 variants) | toolName, source, waitMs |
| `tengu_tool_use_denied_in_config` | Permission denied by config | toolName |
| `tengu_tool_use_rejected_in_prompt` | Permission denied by user/hook | toolName, hasFeedback, isHook |

### 10.4 Auto Mode Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `tengu_auto_mode_decision` | Classifier rendered a decision | toolName, decision, stage, tokens, latency, cost |
| `tengu_auto_mode_denial_limit_exceeded` | Consecutive or total denial limit hit | consecutive count, total count |

### 10.5 Feature-Specific Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `tengu_brief_mode_enabled` | Brief mode activated | activation path |
| `tengu_brief_mode_toggled` | `/brief` command used | enabled/disabled |
| `tengu_brief_send` | SendUserMessage executed | proactive, attachment_count |
| `tengu_compact_failed` | Context compaction failed | — |
| `tengu_flicker` | UI flicker detected | — |
| `tengu_voice_*` | Voice features | recording state |
| `tengu_memdir_loaded` | Memory directory stats | file count, size |
| `tengu_trust_dialog_shown` | Trust dialog displayed | risk factors |

### 10.6 OAuth Events

| Event | When |
|-------|------|
| `tengu_oauth_success` | OAuth flow completed |
| `tengu_oauth_error` | OAuth flow failed |
| `tengu_oauth_token_refresh_*` | Token refresh lifecycle (8 events covering lock acquisition, refresh execution, completion, and release) |

### 10.7 Cost and Token Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `tengu_advisor_tool_token_usage` | Advisor model usage | advisor_model, input/output tokens, cost_usd_micros |

---

## 11. Cost Tracking

> **Source:** `src/cost-tracker.ts`

### 11.1 Per-Session Cost Accumulation

Cost tracking is maintained in bootstrap state and accumulated through
`addToTotalSessionCost()`. For each API response, the system:

1. Computes USD cost via `calculateUSDCost(model, usage)` using the `MODEL_COSTS` table
2. Accumulates per-model usage (input tokens, output tokens, cache read/write, web search
   requests, cost)
3. Reports to OTel counters (`getCostCounter()`, `getTokenCounter()`) with model and
   speed (fast mode) attributes
4. Recursively processes advisor tool usage (models invoked server-side as part of the
   response)

### 11.2 Session Cost Persistence

Session costs are persisted to the project config (`saveCurrentSessionCosts()`) so they
survive across `--continue` invocations:

| Persisted Field | Description |
|----------------|-------------|
| `lastCost` | Total USD cost |
| `lastAPIDuration` | Total API call duration |
| `lastAPIDurationWithoutRetries` | API duration excluding retry overhead |
| `lastToolDuration` | Total tool execution duration |
| `lastDuration` | Wall-clock session duration |
| `lastLinesAdded` / `lastLinesRemoved` | Code change metrics |
| `lastTotalInputTokens` / `lastTotalOutputTokens` | Token counts |
| `lastTotalCacheReadInputTokens` / `lastTotalCacheCreationInputTokens` | Cache metrics |
| `lastTotalWebSearchRequests` | Web search count |
| `lastModelUsage` | Per-model breakdown |
| `lastFpsAverage` / `lastFpsLow1Pct` | UI performance metrics |
| `lastSessionId` | Session ID for matching on restore |

### 11.3 OTel Cost Counters

Cost and token metrics are reported to OTel counters (not log events) for aggregation in
customer dashboards. The counters carry attributes for model name, token type
(`input`, `output`, `cacheRead`, `cacheCreation`), and speed (`fast` when fast mode is
active).

---

## 12. Design Principles

### 12.1 Zero-Dependency Entry Point

The analytics entry point (`index.ts`) has **no imports** from the rest of the codebase.
This is explicitly documented and architecturally enforced. Any module, at any initialization
stage, can call `logEvent()` without worrying about import cycles or initialization order.
Events logged before the sink attaches are queued and drained asynchronously.

### 12.2 Fail-Open Analytics

Analytics failures never affect user-facing functionality. The entire pipeline is wrapped
in try/catch at every level: events that fail to send are silently queued to disk; disk
failures are silently ignored; OTel export failures return `ExportResultCode.FAILED` but
never throw; GrowthBook init timeouts fall back to disk cache. The user never sees an
analytics error.

### 12.3 PII-by-Construction

Rather than relying on runtime scrubbing, the type system prevents PII from entering
general-access backends. The `LogEventMetadata` type restricts values to
`boolean | number | undefined` — strings require an explicit cast through the verification
marker type. The `_PROTO_*` prefix convention routes sensitive strings to privileged columns
with access controls. `stripProtoFields()` is applied at a single choke point (sink.ts)
before any general-access dispatch.

### 12.4 Progressive Rollout via Feature Flags

Every analytics sink, every event sampling rate, and every batch configuration parameter
is remotely controllable via GrowthBook. This enables:

- **Kill switches**: `tengu_frond_boric` can disable individual sinks instantly
- **Gradual rollout**: New event types can be sampled at low rates before full deployment
- **Hot reconfiguration**: Batch sizes, flush intervals, and endpoints can change without
  a code deploy
- **Separate cadences**: Internal users refresh every 20 minutes; external users every
  6 hours

### 12.5 Separation of Internal and Customer Telemetry

The 1P event logging pipeline and the customer OTLP pipeline are **completely isolated**.
They have separate `LoggerProvider` instances, separate exporters, separate configuration.
Internal events never leak to customer endpoints; customer events never flow to Anthropic's
analytics. The only shared code is the event name and basic OTel attribute construction.

### 12.6 Structured Forgetting

Events are designed to be lossy by default. High-volume events can be sampled. Failed
exports are retried with backoff but dropped after `maxAttempts`. Disk-backed retry files
are cleaned up on success. The system does not attempt perfect delivery — it optimizes for
**statistical correctness** (via `sample_rate` metadata for correction) over completeness.

### 12.7 Cardinality Discipline

Every field that enters Datadog tags or OTel metric dimensions is evaluated for cardinality
impact. MCP tool names are collapsed to `"mcp"`. Model names are canonicalized. Version
strings are truncated. User IDs are hashed into 30 buckets. File paths never enter metric
dimensions. The `prompt.id` and `workspace.host_paths` are added to events but explicitly
excluded from metrics.

### 12.8 Observability of the Observability System

The analytics system instruments itself. `analytics_sink_attached` (ant-only) reports the
queue depth at attachment time. 1P export failures are logged with request-id, status, and
error code context. GrowthBook initialization success/failure, feature counts, and refresh
completions are all logged to debug output for ant users. The system is designed to be
debuggable when it itself is misbehaving.
