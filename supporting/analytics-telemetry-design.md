# Analytics & Telemetry — Design Document

This section extracts the pure design, architecture, strategy, data flow, invariants, and trade-offs -- all content understandable without reading source code.

---

### 1. Overview

Claude Code's analytics system provides observability into every meaningful decision the agent makes -- from API calls and tool invocations to permission decisions and feature flag evaluations. The system answers three categories of questions:

1. **Operational health**: Is the system working? What are error rates, latencies, and resource consumption patterns across the user population?
2. **Safety auditing**: Did the oversight layers make correct decisions? How often does the classifier approve versus deny? What is the user override frequency?
3. **Product understanding**: How are features adopted? Which tools are used most? What is the cost distribution across subscription tiers?

The analytics infrastructure is deliberately designed with **zero dependencies on the rest of the codebase**. The main entry point imports nothing from application modules -- it defines only a queue, a sink interface, and two marker types. This isolation prevents import cycles and ensures events can be logged from any module at any initialization stage, even before the analytics backend is attached.

Events logged before the sink is attached are queued in memory. When the sink attaches during app startup, queued events drain asynchronously via microtask scheduling to avoid adding latency to the startup path. The attachment is idempotent -- calling it from both the preAction hook (for subcommands) and setup (for the default command) is safe.

---

### 2. Three Telemetry Sinks

Every analytics event fans out to up to three independent sinks, each serving a different audience and retention model:

```
logEvent(eventName, metadata)
        |
        v
+------------------------+
|     Event Sampling      |  (per-event-type, configurable via feature flags)
|   sample_rate 0-1       |  dropped if not selected
+-----------+------------+
            |
    +-------+-----------------+
    |       |                 |
    v       v                 v
+--------+ +--------------+ +---------------------+
|Datadog | | 1P Event     | | OpenTelemetry        |
| Logs   | | Logging      | | (customer OTLP)      |
|        | | (BigQuery)   | |                      |
+--------+ +--------------+ +---------------------+
```

**Sink 1: Datadog (Operational Dashboards)** -- Receives a curated subset of events (allowlisted by name) for real-time operational monitoring. Events are batched and flushed every 15 seconds or when the batch reaches 100 entries.

**Sink 2: First-Party Event Logging (BigQuery Analytics)** -- The primary analytics backend. All events flow through an OTel SDK batch log processor, are serialized as protobuf messages, and exported to Anthropic's event logging endpoint. Failed events are persisted to disk for retry with quadratic backoff.

**Sink 3: OpenTelemetry (Customer Telemetry)** -- A separate OTel logger emits structured events to customer-configured OTLP endpoints. Completely independent of the first-party pipeline. Events carry monotonically increasing sequence numbers and are prefixed with `claude_code.*`.

**In-Session Store** -- An in-process Map records every permission decision by tool use ID. This enables deduplication of OTel span attribution, post-hoc inspection by downstream code, and SDK consumers reading decision metadata.

---

### 3. Datadog Integration

#### 3.1 Architecture

Datadog integration uses the HTTP Logs Intake API with a client token for authentication. Events are buffered in a module-level array and flushed on a timer or when the batch reaches capacity (100 events).

```
Event arrives -> push to batch -> batch full? -> flush immediately
                                   | no
                                   v
                             schedule flush (15s timer, non-blocking)
```

#### 3.2 Event Allowlist

Only a curated set of approximately 40 event names are forwarded to Datadog. High-volume events (like per-token usage) are excluded to control costs. Categories covered:

| Category | Examples |
|----------|---------|
| Lifecycle | init, started, exit, cancel |
| API | errors, success, query errors |
| OAuth | error/success, token refresh lifecycle (8 events) |
| Permissions | granted/rejected variants, tool use error/success |
| Features | brief mode, voice, flicker, model fallback |
| Chrome Bridge | 7 bridge events |
| Team Memory | 4 sync events |

#### 3.3 Tag Set and Cardinality Controls

Datadog logs carry structured tags for filtering: arch, clientType, errorType, HTTP status, model, platform, provider, skillMode, subscriptionType, toolName, userBucket, userType, version. The event name is prepended as a tag because the DD `message` field is reserved and not queryable from dashboard widgets.

Several transformations reduce tag cardinality:
- **MCP tool names**: All MCP-prefixed tool names normalized to a single value -- server-specific suffixes stripped
- **Model names**: Unrecognized models become a generic placeholder
- **Dev versions**: The time-of-day component and SHA component are stripped; the date component is retained

#### 3.4 User Bucketing

Rather than tracking individual user IDs (which would explode cardinality), events carry a `userBucket` -- a deterministic hash of the user ID modulo 30. This allows estimating unique affected users during incidents without exposing identity.

#### 3.5 Third-Party Provider Exclusion

Events are silently dropped for third-party API providers (Bedrock, Vertex, Foundry). Only first-party Anthropic API traffic is tracked.

#### 3.6 Gating

Datadog dispatch is gated behind a feature flag, checked via a cached accessor. The gate state is initialized during startup from either the live GrowthBook evaluation or the previous session's disk-persisted state.

#### 3.7 Shutdown

The Datadog subsystem is shut down from the graceful shutdown handler before process exit, clearing the flush timer and performing a final batch flush.

---

### 4. First-Party Event Logging

#### 4.1 Architecture

The 1P event logging system is built on top of the OTel SDK's log infrastructure but operates as a **completely separate pipeline** from customer telemetry. It has its own LoggerProvider, its own BatchLogRecordProcessor, and its own exporter.

```
logEventTo1P(eventName, metadata)
    | (fire-and-forget)
    v
logEventTo1PAsync()
    | enriches with metadata + user data
    v
Logger.emit({ body, attributes })
    | (OTel batch processor)
    v
Exporter.export()
    | transforms log records -> protobuf events
    | chunks into batches
    v
POST /api/event_logging/batch
    | success -> reset backoff, retry queued events
    | failure -> persist to disk, schedule quadratic backoff retry
    v
(disk-backed retry on next export cycle or next process run)
```

#### 4.2 Event Types

Two event types are handled:
- **Internal events**: Standard analytics events (protobuf format)
- **Experiment events**: Feature flag experiment assignments

#### 4.3 Batching Configuration

Batch processor parameters are remotely configurable via feature flags:

| Parameter | Default |
|-----------|---------|
| Export interval | 10,000 ms |
| Max batch size | 200 |
| Max queue size | 8,192 |
| Max retry attempts | 8 |
| Base URL | Anthropic API |

#### 4.4 Resilience Model

The exporter implements a multi-layer resilience strategy:

1. **Append-only failed event log**: Events that fail to export are appended to a JSONL file. The append operation is atomic on most filesystems, making it concurrency-safe.
2. **Quadratic backoff retry**: Failed batches trigger retry with `delay = base * attempts^2`, capped at 30 seconds. After 8 attempts, events are dropped.
3. **Short-circuit on first failure**: When sending batches sequentially, the first batch failure causes all remaining unsent batches to be queued to disk -- the assumption is that the endpoint is down.
4. **Success-triggered drain**: When any export succeeds, queued events from disk are immediately retried.
5. **Cross-run retry**: On startup, the exporter scans for failed event files from previous runs and retries them in the background.
6. **Auth fallback**: On 401 errors, the exporter retries without auth headers. This handles expired OAuth tokens when the endpoint accepts unauthenticated events. Note: This fail-open behavior means event integrity relies on server-side validation rather than client authentication when tokens expire.

#### 4.5 Hot Reinit on Config Change

When feature flag configuration refreshes, the system compares the new batch config against the last-used config. If they differ, it:
1. Nulls the logger (concurrent calls bail at the guard)
2. Force-flushes the old processor's buffer
3. Creates a new LoggerProvider + exporter with the new config
4. Shuts down the old provider in the background

Failed events from the old exporter's disk file are automatically picked up by the new exporter because the file path is keyed by stable session identifiers.

#### 4.6 Event Sampling

Events can be sampled at configurable rates via feature flags. The sampling decision is per-event at dispatch time:
- Rate absent or 1.0: log at 100%
- Rate between 0 and 1: random selection; if selected, the `sample_rate` is added to metadata for statistical correction
- Rate = 0: drop entirely

---

### 5. GrowthBook Feature Flags (Analytics Role)

#### 5.1 Role in the Analytics System

GrowthBook controls:
- Which analytics sinks are active (Datadog gate, sink killswitches)
- Event sampling rates
- 1P batch processor configuration
- Feature rollouts that generate experiment exposure events
- Security restriction gates

#### 5.2 Initialization

GrowthBook uses **remote evaluation** -- the server pre-evaluates all feature flags for the user's attributes and returns the complete set of resolved values. The client never evaluates rules locally.

```
getUserAttributes() -> GrowthBook client (remoteEval: true)
    -> client.init(timeout: 5s)
    -> processPayload() -> populate in-memory map
    -> syncToDisk() -> persist to config file
    -> refreshed.emit() -> notify subscribers
```

#### 5.3 Four-Level Value Resolution

Feature values are resolved through a priority cascade:
1. **Env var overrides** (internal-only, for eval harnesses)
2. **Config overrides** (internal-only, for interactive testing)
3. **In-memory map** (authoritative after init)
4. **Disk cache** (survives restarts)

#### 5.4 Accessor Semantics

Accessors are named to communicate their blocking and staleness behavior:
- Non-blocking, possibly stale: for startup-critical paths, render loops, sync contexts
- Conditionally blocking: for security gates (waits if reinit in progress) or entitlement checks (fast-path if cache says true)
- Blocking on init: for JSON configs that must be fresh (e.g. kill switches)
- Blocking (legacy): for older paths that require fresh values

#### 5.5 Periodic Refresh

- 6 hours for external users
- 20 minutes for internal users

Subscribers are notified after each successful refresh, allowing long-lived objects to rebuild.

#### 5.6 Experiment Exposure Tracking

Exposures are deduplicated per session. Features accessed before init completes are added to a pending set and logged retroactively once init succeeds.

#### 5.7 Auth Change Handling

When authentication changes, the client is destroyed and recreated (auth headers cannot be updated after client creation). Security-critical gates can await the reinit promise.

#### 5.8 Sink Killswitch

A deliberately obfuscated-name dynamic config provides per-sink killswitches. When a sink is killed, new events are silently dropped and the 1P exporter's backoff retry also checks the killswitch per-POST.

---

### 6. Metadata Collection

#### 6.1 The Metadata Enrichment Pipeline

Every analytics event is enriched with approximately 40 fields:

- **Session context**: model, sessionId, userType, betas, isInteractive, clientType, subscriptionType, skillMode, observerMode
- **Environment context** (memoized, once per session): platform, arch, nodeVersion, terminal, package managers, runtimes, CI detection, remote environment type, version info, VCS info, Linux distro/kernel
- **Process metrics** (per-event): uptime, RSS, heap usage, CPU usage, delta CPU percentage
- **Agent identification**: agentId, parentSessionId, agentType (teammate/subagent/standalone), teamName
- **SWE-bench fields**: runId, instanceId, taskId

#### 6.2 Memoization Strategy

Environment context is computed once per session because it involves async I/O (package manager detection, runtime detection, Linux distro info, VCS detection). Exception: `kairosActive` is deliberately not inside the memoized context because it can be set after the first event fires.

#### 6.3 Process Metrics

Every event includes: memory usage (RSS, heap, external, array buffers, constrained memory), CPU usage (user + system), and a delta CPU percentage computed from wall-clock and CPU time since the last event.

#### 6.4 Agent Identification

Resolution order for multi-agent scenarios:
1. AsyncLocalStorage context (subagents in same process)
2. Env var / swarm helpers (teammates in separate processes)
3. Bootstrap state (parent session ID in plan-to-implementation transitions)

#### 6.5 1P Event Format Transformation

The transformation from internal camelCase to protobuf snake_case is type-checked against the proto-generated type. Adding a field to the formatter that the proto does not define is a **compile error**, preventing silent field dropping.

#### 6.6 Tool Name Sanitization

MCP tool names can reveal user-specific server configurations. They are redacted to a generic label for general-access backends. Detailed MCP tool names are logged only when the source is known to be non-user-specific (official proxied connectors, official MCP registry, built-in servers, or the local-agent entrypoint where there is no ZDR (Zero Data Retention) concept).

#### 6.7 File Extension Extraction

Extensions longer than 10 characters are replaced with a generic label to avoid logging potentially sensitive hash-based filenames.

---

### 7. Permission Decision Audit

#### 7.1 Central Logging Function

Every permission decision flows through a single entry point that fans out to four destinations:
1. Analytics event (distinct event name per approval/rejection source)
2. OTel telemetry (tool_decision event)
3. Code-edit OTel counter (language-specific metrics)
4. In-session store (per-tool-use Map)

#### 7.2 Nine Distinct Event Types

| Event | Trigger |
|-------|---------|
| Granted in config | Auto-approved by allowlist |
| Granted by classifier | Approved by auto-mode AI classifier |
| Granted in prompt (permanent) | User approved and saved permanent rule |
| Granted in prompt (temporary) | User approved for session only |
| Granted by permission hook | Approved by external hook |
| Denied in config | Blocked by denylist |
| Rejected in prompt | User rejected in permission dialog |
| Auto mode decision | Classifier decision (allow or deny, with stage info) |
| Auto mode denial limit exceeded | 3 consecutive or 20 total denials |

#### 7.3 Base Metadata

Every permission event carries: messageID, sanitized toolName, sandboxEnabled flag, and waiting-for-user-permission duration.

#### 7.4 Source Flattening

The structured discriminated union of approval/rejection sources is flattened to string labels: `classifier`, `hook`, `user_permanent`, `user_temporary`, `user_abort`, `user_reject`, `config`.

#### 7.5 Code-Edit Language Metrics

For code-editing tools, an additional OTel counter is incremented with language-specific attributes, enabling dashboards showing approval rates segmented by programming language.

---

### 8. OpenTelemetry Integration

#### 8.1 Customer Event Logger

Events are emitted as OTel log records with session-level attributes, event name, timestamp, monotonic sequence counter, prompt ID, and workspace paths.

#### 8.2 User Prompt Redaction

User prompt content is **redacted by default**. An explicit environment variable must be set to enable logging of user-provided text.

#### 8.3 Tool Input Telemetry

When enabled, tool inputs are serialized with aggressive truncation:

| Limit | Value |
|-------|-------|
| String truncation threshold | 512 chars |
| Truncated string length | 128 chars + indicator |
| Max JSON output | 4 KB |
| Max collection items | 20 |
| Max nesting depth | 2 |

Keys starting with `_` are filtered to prevent internal markers from leaking.

---

### 9. PII Handling

#### 9.1 Marker Type System

Two type-level markers enforce developer intent:
- One for general-access backends (verified not code or filepaths)
- One for PII-tagged proto columns with privileged access controls

Both types are `never` -- they can never hold a value. They exist only as type-cast annotations. The event metadata type restricts values to `boolean | number | undefined`, which **structurally prevents string values** without an explicit cast.

#### 9.2 Proto Field Routing

Fields prefixed with `_PROTO_` contain PII-tagged values destined for privileged BigQuery columns:
- At dispatch: proto fields are stripped before forwarding to Datadog
- At 1P export: known proto keys are hoisted to top-level proto fields; remaining proto keys are stripped defensively

#### 9.3 MCP Tool Name PII Classification

MCP server names are classified as medium PII. Full names are logged only for known-official sources; custom user MCPs are replaced with a generic label.

#### 9.4 OTel Redaction

- User prompts: redacted unless opt-in env var set
- Tool input details: omitted unless opt-in env var set
- Long file extensions: replaced with generic label
- Workspace paths: in events but not metrics
- Prompt ID: in events but not metrics

---

### 10. Analytics Events Catalog

#### 10.1 Lifecycle Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `tengu_init` | Analytics sink attached | `queued_event_count` (ant-only) |
| `tengu_started` | Session fully initialized | env context, model |
| `tengu_exit` | Session ending | cost, duration, token counts |
| `tengu_cancel` | User cancelled operation | -- |

#### 10.2 API Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `tengu_api_success` | API call completed | model, tokens, latency, cache stats |
| `tengu_api_error` | API call failed | status, errorType, http_status_range |
| `tengu_query_error` | Query pipeline error | error classification |
| `tengu_model_fallback_triggered` | Model fallback activated | original model, fallback model |

#### 10.3 Tool Use Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `tengu_tool_use_success` | Tool executed successfully | toolName, duration |
| `tengu_tool_use_error` | Tool execution failed | toolName, errorType |
| `tengu_tool_use_granted_*` | Permission granted (6 variants) | toolName, source, waitMs |
| `tengu_tool_use_denied_in_config` | Permission denied by config | toolName |
| `tengu_tool_use_rejected_in_prompt` | Permission denied by user/hook | toolName, hasFeedback, isHook |

#### 10.4 Auto Mode Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `tengu_auto_mode_decision` | Classifier rendered a decision | toolName, decision, stage, tokens, latency, cost |
| `tengu_auto_mode_denial_limit_exceeded` | Consecutive or total denial limit hit | consecutive count, total count |

#### 10.5 Feature-Specific Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `tengu_brief_mode_enabled` | Brief mode activated | activation path |
| `tengu_brief_mode_toggled` | `/brief` command used | enabled/disabled |
| `tengu_brief_send` | SendUserMessage executed | proactive, attachment_count |
| `tengu_compact_failed` | Context compaction failed | -- |
| `tengu_flicker` | UI flicker detected | -- |
| `tengu_voice_*` | Voice features | recording state |
| `tengu_memdir_loaded` | Memory directory stats | file count, size |
| `tengu_trust_dialog_shown` | Trust dialog displayed | risk factors |

#### 10.6 OAuth Events

| Event | When |
|-------|------|
| `tengu_oauth_success` | OAuth flow completed |
| `tengu_oauth_error` | OAuth flow failed |
| `tengu_oauth_token_refresh_*` | Token refresh lifecycle (8 events covering lock acquisition, refresh execution, completion, and release) |

#### 10.7 Cost and Token Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `tengu_advisor_tool_token_usage` | Advisor model usage | advisor_model, input/output tokens, cost_usd_micros |

---

### 11. Cost Tracking

#### 11.1 Per-Session Cost Accumulation

For each API response:
1. Compute USD cost using a model cost table
2. Accumulate per-model usage (input/output tokens, cache read/write, web search, cost)
3. Report to OTel counters with model and speed attributes
4. Recursively process advisor tool usage

#### 11.2 Session Cost Persistence

Costs are persisted to project config for `--continue` invocations. Persisted fields include: total cost, API duration (with and without retries), tool duration, wall-clock duration, lines added/removed, token counts, cache metrics, web search count, per-model breakdown, UI performance metrics (FPS average, low 1%), and session ID.

#### 11.3 OTel Cost Counters

Counters (not log events) carry attributes for model name, token type, and speed. These enable aggregation in customer dashboards.

---

### 12. Design Principles

1. **Zero-Dependency Entry Point**: Any module at any initialization stage can log events without import cycles. Pre-sink events are queued and drained asynchronously.

2. **Fail-Open Analytics**: Analytics failures never affect user-facing functionality. Try/catch at every level; disk persistence for failed events; silent fallback to disk cache on network failure.

3. **PII-by-Construction**: The type system prevents PII from entering general-access backends. String values require explicit verification casts. The `_PROTO_*` prefix routes sensitive strings to privileged columns.

4. **Progressive Rollout via Feature Flags**: Every sink, sampling rate, and batch config is remotely controllable. Kill switches for instant disabling; gradual rollout at low sampling rates; hot reconfiguration without code deploys.

5. **Separation of Internal and Customer Telemetry**: The 1P and customer OTLP pipelines are completely isolated -- separate providers, exporters, configuration. No cross-contamination.

6. **Structured Forgetting**: Events are designed to be lossy by default. High-volume events can be sampled. Failed exports are retried but dropped after max attempts. Statistical correctness (via `sample_rate` metadata) is preferred over completeness.

7. **Cardinality Discipline**: Every field entering Datadog tags or OTel metric dimensions is evaluated for cardinality impact. MCP names collapsed, models canonicalized, versions truncated, user IDs hashed into 30 buckets, file paths excluded from metrics.

8. **Observability of the Observability System**: The analytics system instruments itself -- queue depth at attachment, export failures with context, GrowthBook init/refresh status.
