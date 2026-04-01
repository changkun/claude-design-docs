# Bridge System — Design Document

## 1. Overview

The bridge system is Claude Code's IDE and remote session integration layer -- the subsystem that makes a local CLI session visible and controllable from the claude.ai web interface ("Remote Control"). When a user starts Claude Code locally and enables Remote Control, the bridge connects the CLI process to Anthropic's Cloud Code Runner (CCR) infrastructure so that the same conversation can be viewed and driven from a browser or mobile device.

The system solves three fundamental problems:

1. **Bidirectional synchronization** -- messages typed locally must appear on the web, and prompts typed on the web must flow back to the local CLI for execution.
2. **Authentication bridging** -- the CLI authenticates with OAuth tokens, but the CCR worker endpoints require session-scoped JWTs; the bridge mediates between these credential domains.
3. **Resilience across connectivity gaps** -- laptops sleep, WiFi drops, and VPN tunnels reset; the bridge must recover transparently without losing the session.

The bridge codebase comprises approximately 31 TypeScript files totaling roughly 479KB.

---

## 2. Two Core Architectures

The bridge offers two fundamentally different connection architectures. Both achieve the same end result -- a local CLI session visible on claude.ai -- but differ in how the CLI discovers and connects to the session infrastructure.

### 2.1 Environment-Based Architecture (v1)

The original design, built on the **Environments API** -- a poll-based work-dispatch layer. The CLI registers as a "bridge environment" with the server, then enters a long-poll loop waiting for work items (sessions) to be dispatched.

```
CLI                     Environments API              CCR/Session-Ingress
 |                           |                              |
 +-- POST /environments/bridge -->                           |
 |        (register)         |                              |
 |                           |                              |
 +-- POST /environments/{id}/work <-->                      |
 |        (poll loop)        |                              |
 |                           |  <-- user opens session on    |
 |                           |     claude.ai                |
 |  <-- work item            |                              |
 |  (session_id + secret)    |                              |
 |                           |                              |
 +-- decode work secret       |                              |
 +-- acknowledge work -------->                              |
 |                           |                              |
 +------------------ connect transport ------------------>  |
 |                           |                              |
 +-- heartbeat work ---------->                              |
 |        (periodic)         |                              |
```

This architecture supports two operational modes:

- **Standalone bridge**: `claude remote-control` runs as a persistent daemon. It spawns child `claude --print` processes for each session, communicating via NDJSON over stdin/stdout. Supports multi-session capacity (up to 32 concurrent sessions) with configurable spawn modes (single-session, worktree, same-dir).

- **REPL bridge**: The in-process path used by the interactive CLI. When a user types `/remote-control` or has `remoteControlAtStartup` enabled, the REPL initializes a bridge connection within the same process. No child spawning -- messages flow directly between the query engine and the transport.

### 2.2 Environment-Less Architecture (v2)

The newer design, gated behind a feature flag. Eliminates the Environments API layer entirely -- no register, poll, acknowledge, heartbeat, or deregister lifecycle. The CLI connects directly to the session-ingress layer:

```
CLI                         CCR Code Sessions API
 |                                  |
 +-- POST /v1/code/sessions -------->|  (OAuth, no env_id)
 |  <-- session.id (cse_*)         |
 |                                  |
 +-- POST /v1/code/sessions/{id}/bridge -->  (OAuth -> worker_jwt)
 |  <-- {worker_jwt, expires_in,    |
 |       api_base_url, worker_epoch}|
 |                                  |
 +-- createV2ReplTransport --------->|  (SSE + CCRClient)
 |      (connect)                   |
 |                                  |
 +-- token refresh scheduler         |
 |  (proactive /bridge re-call)     |
```

The key insight is that the `/bridge` endpoint serves as both credential exchange AND worker registration -- each call bumps the server-side epoch, replacing the separate worker registration step. This reduces the connection setup from five round-trips (register env, poll, decode, ack, connect) to two (create session, fetch bridge credentials).

**Important naming distinction**: "env-less" is distinct from "CCR v2" (the `/worker/*` transport protocol). The environment-based path can *also* use CCR v2 transport. The env-less feature gate controls which *architecture* is used (poll loop vs. direct connect), not which transport protocol runs underneath.

---

## 3. Transport Abstraction

The transport abstraction ensures that both bridge architectures can swap transport implementations without changing their message-handling logic.

### 3.1 The Interface

The transport interface provides:
- **Write path**: `write(message)`, `writeBatch(messages)`, `flush()`
- **Read path**: `setOnData(callback)`, `setOnConnect(callback)`, `setOnClose(callback)`
- **Lifecycle**: `connect()`, `close()`, `isConnectedStatus()`, `getStateLabel()`
- **Dedup support**: `getLastSequenceNum()`, `droppedBatchCount`
- **Worker state**: `reportState()`, `reportMetadata()`, `reportDelivery()`

### 3.2 v1: HybridTransport (WebSocket)

Wraps WebSocket for reads and HTTP POST for writes to Session-Ingress. This is the legacy path used when the server does not indicate CCR v2 in the work secret.

- `getLastSequenceNum()` always returns `0` -- Session-Ingress WS does not use SSE sequence numbers; replay-on-reconnect is handled server-side.
- `reportState()`, `reportMetadata()`, `reportDelivery()` are no-ops -- v1 does not support worker state reporting.
- `flush()` resolves immediately -- POSTs are already awaited per-write.

### 3.3 v2: SSETransport + CCRClient

Composes two components:

- **SSETransport** (reads): Server-Sent Events stream at `/worker/events/stream`. Supports automatic reconnection with sequence-number resumption (`from_sequence_num` / `Last-Event-ID`). The SSE read loop runs as a fire-and-forget promise; `connect()` does not await it.

- **CCRClient** (writes): HTTP POST to `/worker/events` via a batch uploader that batches up to 100 events per request. Also handles heartbeats, state reporting, metadata updates, and delivery acknowledgments.

Registration happens inside the factory when no pre-existing epoch is provided. When the caller passes an epoch (from the `/bridge` endpoint), the explicit worker registration call is skipped -- the `/bridge` call *is* the register.

**Epoch mismatch handling**: If the CCRClient receives a 409 (epoch superseded), it closes both the CCR client and SSE stream, fires `onClose(4090)`, and throws to unwind the caller. The bridge's close handler then enters recovery.

**Close code semantics**:
| Code | Meaning |
|------|---------|
| 401  | JWT expired -- recoverable via OAuth refresh + `/bridge` re-call |
| 4090 | Epoch mismatch -- another worker registered |
| 4091 | CCR initialize failed |
| 4092 | SSE reconnect budget exhausted (10-minute internal retry window) |

---

## 4. Message Handling

All message routing logic is extracted into pure functions -- no closure over bridge-specific state. Both architectures share identical parsing, filtering, deduplication, and control-request handling.

### 4.1 Type Guards

Three type predicates classify incoming messages:

- **SDK message**: Validates the discriminated union on `type`
- **Control response**: `type === 'control_response'` with a `response` field -- permission decisions from the web UI
- **Control request**: `type === 'control_request'` with `request_id` and `request` -- server lifecycle commands (initialize, set_model, interrupt)

### 4.2 Ingress Routing

The ingress handler parses each raw string from the transport and routes it:

1. **Control responses** (permission decisions) -> `onPermissionResponse` callback
2. **Control requests** (server commands) -> server control request handler
3. **SDK messages with echoed UUIDs** -> silently dropped (see 4.3)
4. **SDK messages with re-delivered UUIDs** -> silently dropped (see 4.3)
5. **User-type messages** -> `onInboundMessage` callback (prompts from the web)
6. **Other types** -> ignored (assistant messages are not re-injected)

### 4.3 Echo Deduplication (BoundedUUIDSet)

Messages the CLI sends to the server come back on the read stream as echoes. Without deduplication, the CLI would re-process its own output as new input.

The dedup mechanism is a FIFO-bounded set backed by a circular ring buffer. Capacity is 2000 entries (configurable), providing O(1) lookup with O(capacity) memory. Two instances are maintained per bridge session:

- **`recentPostedUUIDs`**: Seeded with initial message UUIDs. Catches echoes of outbound messages.
- **`recentInboundUUIDs`**: Catches re-delivered inbound prompts after transport swaps where sequence-number negotiation fails.

### 4.4 Server Control Requests

The server control handler responds to server-initiated control messages. The server kills the connection if responses are not sent within ~10-14 seconds.

| Subtype | Action | Response |
|---------|--------|----------|
| `initialize` | Return minimal capabilities (empty commands, models, account) | success |
| `set_model` | Invoke callback | success |
| `set_max_thinking_tokens` | Invoke callback | success |
| `set_permission_mode` | Invoke callback; return policy verdict | success or error |
| `interrupt` | Invoke callback | success |
| unknown | No action | error with diagnostic message |

In **outbound-only mode** (CCR mirror), all mutable requests return an error response except `initialize` (which must succeed or the server disconnects).

### 4.5 Message Eligibility

Only three message types pass the eligibility filter for forwarding to the transport:

- `user` messages (non-virtual)
- `assistant` messages (non-virtual)
- `system` messages with subtype `local_command`

Virtual messages (REPL inner calls) are excluded -- the bridge consumer sees the REPL `tool_use`/`result` which summarizes the work.

---

## 5. Environment-Based Flow

### 5.1 Registration

The bridge registers with the Environments API, sending machine name, directory, branch, git repo URL, max sessions, worker type, and optional environment ID for idempotent re-registration. The server returns an environment_id and environment_secret.

### 5.2 Poll Loop

The core of the environment-based architecture is a poll loop. Poll intervals are configurable via feature flags:

| State | Interval |
|-------|----------|
| No sessions active | `poll_interval_ms_not_at_capacity` |
| Some sessions, not full | `multisession_poll_interval_ms_partial_capacity` |
| At capacity | `multisession_poll_interval_ms_at_capacity` (0 = disabled) |

When at capacity with heartbeat mode enabled, the loop enters a nested heartbeat cycle.

### 5.3 Work Secret Decoding

When the poll returns a work item, its `secret` field is a base64url-encoded JSON blob containing: version number, session ingress JWT, API base URL, git sources, auth tokens, and an optional flag for CCR v2 session mode. The decoder validates the version and required fields, throwing on malformed secrets.

### 5.4 Acknowledgment

After committing to handle the work (capacity check passed, secret decoded), the bridge explicitly acknowledges. Acknowledgment happens *after* the capacity guard -- acking before spawn would permanently lose the work item if the at-capacity check breaks without spawning.

### 5.5 Session Spawning (Standalone Bridge)

In standalone mode, a session spawner creates child `claude --print` processes. The child communicates over three pipes:
- **stdin**: Receives token refresh messages (NDJSON)
- **stdout**: Emits NDJSON events (parsed for activity tracking, permission requests, user message detection)
- **stderr**: Captured in a ring buffer (last 10 lines) for error diagnostics

The session handle provides: a done promise resolving to `completed`/`failed`/`interrupted`, kill/forceKill methods, token update mechanism, and an activity ring buffer.

### 5.6 Transport Connection (REPL Bridge)

In REPL mode, no child process is spawned. Instead, the bridge creates a transport directly and wires callbacks for connection, data, and close events. The transport choice (v1 WS vs. v2 SSE+CCR) is driven by the work secret's flag or an environment variable override.

### 5.7 Heartbeat

Active work items are heartbeated using the session-ingress JWT. Results are categorized:
- `ok` -- at least one heartbeat succeeded
- `auth_failed` -- 401/403, triggers session reconnection with fresh JWT
- `fatal` -- 404/410 (environment expired/deleted)
- `failed` -- all heartbeats failed for other reasons

### 5.8 Cleanup

When a session ends: remove from tracking, clear timers, wake capacity signal, notify server, clean up git worktree if applicable. On bridge shutdown: SIGTERM all children, wait up to 30s, SIGKILL stragglers, stop all work items, archive all sessions, deregister the environment.

---

## 6. Environment-Less Flow

### 6.1 Session Creation

`POST /v1/code/sessions` with OAuth authentication (no environment ID). Returns a `cse_*`-prefixed session ID.

### 6.2 Bridge Credential Exchange

`POST /v1/code/sessions/{id}/bridge` exchanges the OAuth token for: worker JWT, expires_in TTL, API base URL, and a monotonic worker epoch. This single endpoint replaces the three-step v1 process.

### 6.3 Token Refresh Scheduler

The scheduler proactively refreshes tokens before expiry:

1. Token received with expiry
2. Schedule refresh at `expires_in - buffer` (5min default)
3. When timer fires, call `getAccessToken()` to refresh OAuth if needed
4. Invoke `onRefresh` callback
5. Schedule follow-up (30min fallback)

**Generation counters** prevent stale refresh calls from setting follow-up timers after a session has been cancelled or rescheduled.

**Failure handling**: Up to 3 consecutive failures before giving up. Each failure schedules a 60-second retry. Counter resets on successful token retrieval.

For the environment-less path, the refresh callback re-fetches `/bridge` credentials and rebuilds the transport (since each `/bridge` call bumps epoch).

### 6.4 Transport Creation and Rebuild

On proactive refresh or 401 recovery, the rebuild process:

1. Activates the FlushGate to queue writes during rebuild
2. Captures the SSE sequence-number high-water mark from the old transport
3. Closes the old transport
4. Creates a new transport with fresh credentials and the captured sequence number
5. Re-wires callbacks and calls `connect()`
6. Drains the flush gate into the new transport

**Atomicity**: A boolean latch is set synchronously before any `await`. Both the proactive refresh and 401 recovery paths check this flag to prevent double execution -- critical because laptop wake can fire both paths simultaneously, and each `/bridge` call bumps epoch.

### 6.5 Initial History Flush

When the bridge connects with existing conversation history, a FlushGate prevents ordering races:

```
start() -> enqueue() returns true, messages queued
  |
  flushHistory(initialMessages) via writeBatch()
  |
  .finally() -> end() -> returns queued items -> drain via writeBatch()
```

The gate ensures the server receives `[history..., live...]` in order. If a 401 interrupts the flush, the initial flush flag resets so the new `onConnect` re-flushes.

---

## 7. Session Management

### 7.1 SessionHandle

The bridge's interface to a running session process provides: session ID, done promise, kill/forceKill, activity ring buffer (last 10), mutable access token, last stderr lines, stdin writer, and token updater. The `updateAccessToken()` method sends an NDJSON message to the child's stdin for the child's handler to pick up and apply.

### 7.2 Activity Tracking

The standalone bridge parses the child's NDJSON stdout to build a real-time activity feed:
- `tool_start` -- extracted from assistant messages containing tool_use blocks
- `text` -- assistant text blocks (first 80 chars)
- `result` -- session completed or errored
- `error` -- error subtype with message

### 7.3 Multi-Session Capacity Management

The standalone bridge supports up to 32 concurrent sessions. Three spawn modes:

| Mode | Directory | Lifecycle |
|------|-----------|-----------|
| `single-session` | cwd | Bridge exits when session ends |
| `worktree` | Isolated git worktree per session | Bridge persists |
| `same-dir` | Shared cwd | Bridge persists |

### 7.4 CapacityWake

A shared primitive for both bridge paths. When at capacity, the poll loop sleeps with a merged abort signal that fires on either: the outer loop signal (shutdown) or the capacity-wake controller (session ended). `wake()` aborts the current controller and arms a fresh one.

---

## 8. Authentication and Security

### 8.1 OAuth Flow

The bridge authenticates to the Environments API and Sessions API using the user's claude.ai OAuth token. Pre-flight checks include: token existence, organization policy, proactive refresh if expired, and cross-process dead-token backoff (3 failures per expiry window).

### 8.2 JWT Tokens (Session-Ingress)

Work secrets contain a session_ingress_token JWT scoped to the specific session. This authenticates: WebSocket connections (v1), CCR `/worker/*` endpoints (v2), and work heartbeat calls.

The environment-less path receives an opaque worker JWT from the `/bridge` endpoint, along with an explicit `expires_in` -- the JWT is not decoded client-side in this path.

### 8.3 Trusted Device Tokens

Bridge sessions have `SecurityTier=ELEVATED` on the server. When the enforcement gate is enabled, the CLI sends an `X-Trusted-Device-Token` header.

**Enrollment**: Runs during `/login` via `POST /auth/trusted_devices`. The server gates enrollment on account age < 10 minutes. The token is persistent (90-day rolling expiry) and stored in the OS keychain.

**Retrieval**: Checks the feature gate live (so flips take effect without restart) but memoizes the keychain read (which spawns a macOS `security` subprocess at ~40ms).

### 8.4 Work Secret Validation

Structural validation: version must be 1, session_ingress_token must be a non-empty string, api_base_url must be a string.

### 8.5 Bridge ID Validation

Prevents path traversal and injection in server-provided IDs used in URL path segments. Only `[a-zA-Z0-9_-]+` is allowed.

---

## 9. Message Flow Diagrams

### 9.1 Environment-Based: User Types on claude.ai

```
claude.ai                CCR Server              Bridge (CLI)
    |                        |                       |
    +-- user types prompt -->|                       |
    |                        +-- dispatch work ----->|
    |                        |  (poll response)      |
    |                        |                       +-- decode secret
    |                        |<-- acknowledge work --+
    |                        |                       |
    |                        |                       +-- connect transport
    |                        |                       |
    |   <-- SSE/WS event ---|<-- user message ------+
    |                        |   (onInboundMessage)  |
    |                        |                       |
    |                        |                       +-- execute query
    |                        |                       |
    |   <-- SSE/WS event ---|<-- assistant message -+
    |                        |   (writeMessages)     |
    |                        |                       |
    |   <-- SSE/WS event ---|<-- tool result -------+
    |                        |                       |
    |   <-- SSE/WS event ---|<-- result event ------+
```

### 9.2 Environment-Less: REPL Enables Remote Control

```
REPL                     CCR Server              claude.ai
  |                          |                       |
  +-- POST /v1/code/sessions ->                       |
  |  <-- session_id (cse_*)  |                       |
  |                          |                       |
  +-- POST /sessions/{id}/bridge ->                   |
  |  <-- {jwt, epoch, url}  |                       |
  |                          |                       |
  +-- createV2ReplTransport -->                       |
  |  (SSE connect + CCR init)|                       |
  |                          |                       |
  +-- flush initial history -->                       |
  |  (writeBatch)            |--- events ----------->|
  |                          |                       |
  |                          |   user types on web   |
  |  <-- SSE event ----------|<----------------------+
  |  (handleIngressMessage)  |                       |
  |                          |                       |
  |  [~5h later]             |                       |
  +-- proactive JWT refresh   |                       |
  |  POST /sessions/{id}/bridge ->                   |
  |  <-- fresh {jwt, epoch}  |                       |
  |  rebuildTransport()      |                       |
  |  (SSE resumes from seq#) |                       |
```

### 9.3 Permission Flow

```
CLI (tool execution)     Bridge Transport         claude.ai
  |                          |                       |
  +-- control_request ------->                       |
  |  (can_use_tool)          |--- event ------------>|
  |                          |    reportState(       |
  |                          |    requires_action)   |
  |                          |                       |
  |                          |   user clicks Allow   |
  |                          |                       |
  |  <-- control_response ---|<----------------------+
  |  (onPermissionResponse)  |                       |
  |                          |    reportState(       |
  |                          |    running)           |
  |                          |                       |
  +-- tool executes           |                       |
```

---

## 10. Failure Recovery

### 10.1 Transport Failures (REPL Bridge)

The transport permanent-close handler handles terminal transport failures. By the time it fires, the underlying transport has exhausted its internal retry budget.

**Close code 1000 (clean close)**: Session ended normally. Tear down the bridge.

**All other codes**: Trigger environment reconnection, which tries two strategies:

1. **Reconnect-in-place**: Re-register with the current env ID. If the server returns the same ID, force-stop stale workers and re-queue the session. The session ID stays the same.

2. **Fresh session fallback**: If the server returns a different ID or reconnect fails, archive the old session and create a new one. Clear dedup state and reset sequence number.

A reentrancy guard ensures concurrent callers share the same reconnection attempt. A counter caps at 3 per burst, resetting on success.

### 10.2 Transport Failures (Environment-Less)

The env-less path has no environment to reconnect. 401 on the SSE stream triggers auth failure recovery:

1. Set recovery latch synchronously
2. Refresh OAuth
3. Re-fetch `/bridge` credentials
4. Rebuild transport with fresh JWT and epoch

Non-401 terminal failures (epoch mismatch, init failure, budget exhausted) transition to `'failed'` state with no recovery.

### 10.3 Poll Loop Failures (Environment-Based)

The poll loop uses **dual-track exponential backoff** -- connection errors and general errors are tracked independently:

| Track | Initial | Cap | Give-up |
|-------|---------|-----|---------|
| Connection | 2s | 2min | 10min |
| General | 500ms | 30s | 10min |

Switching error types resets the other track's counter.

**Fatal errors**: 401/403 responses are non-retryable. The loop breaks immediately.

### 10.4 Sleep Detection

Both the poll loop and the transport use sleep detection to reset error budgets after laptop wake. If the gap since the last error exceeds twice the connection backoff cap, the machine likely slept. Error tracking resets so the bridge retries with a fresh budget.

### 10.5 Adaptive Backoff with Jitter

All backoff calculations apply jitter (typically +/- 50%) to prevent thundering-herd effects:

```
delay = base * 2^attempt          // exponential
delay = min(delay, cap)           // capped
delay = delay * (0.5 + random())  // jittered
```

---

## 11. Configuration and Feature Gates

### 11.1 Runtime Feature Gates

| Gate | Purpose |
|------|---------|
| Master enable | Enable Remote Control |
| Env-less architecture | Enable env-less bridge path |
| Multi-session | Enable multi-session spawn modes |
| Session ID shim | `cse_*` to `session_*` ID retag |
| Elevated auth | Trusted device token enforcement |
| Auto-connect | Auto-connect Remote Control at startup |
| Mirror mode | CCR mirror (outbound-only) |

### 11.2 Dynamic Configs

| Config | Purpose |
|--------|---------|
| Poll interval config | Poll/heartbeat intervals, validated |
| Env-less timing config | Retry, heartbeat, timeout settings |
| Minimum CLI version | Min version for v1 path |
| Initial history cap | Max initial messages to replay (default 200) |

### 11.3 Poll Configuration Schema

The poll config uses Zod validation with defense-in-depth constraints:
- `.min(100)` on seek-work intervals prevents fat-fingered values
- At-capacity intervals use `0-or->=100` refinement: 0 means "disabled", 1-99 rejected to prevent unit confusion
- Object-level refinements require at least one at-capacity liveness mechanism (heartbeat OR poll interval)

---

## 12. Design Principles

### 12.1 Two Architectures, One Abstraction

The transport interface and the shared messaging module ensure both connection architectures share identical message handling, deduplication, and control-request logic. The architectural choice is confined to the construction site -- once a transport is created, the rest of the bridge cannot tell which architecture produced it.

### 12.2 Bootstrap Isolation

A recurring constraint throughout the bridge codebase is **import graph discipline**. The bridge core reads nothing from bootstrap state or session storage -- all context comes from injected parameters. This isolation exists because session storage transitively imports the entire slash command + React component tree (~1300 modules). Keeping the core free of bootstrap imports lets daemon callers use the bridge core without bloating their bundle.

The same pattern appears in the session API module (thin HTTP wrappers), the messaging module (pure functions, no closures), and the injected callback pattern throughout the core parameters.

### 12.3 Crash Recovery via Bridge Pointer

After session creation, the bridge writes a crash-recovery pointer to disk containing session ID, environment ID, and source. If the process is killed, a subsequent `claude remote-control --continue` can detect and resume the session. The pointer's staleness is checked against the file's mtime (not an embedded timestamp), with a 4-hour TTL matching the backend's rolling TTL semantics.

### 12.4 Defense in Depth for Deduplication

Message deduplication uses multiple independent layers:

1. **Primary**: The hook's array index boundary tracker
2. **Secondary**: 2000-cap ring buffer catches echoes of outbound messages
3. **Tertiary**: Unbounded set catches initial message echoes that may have been evicted from the ring buffer
4. **Inbound**: Separate ring buffer catches re-delivered prompts after transport swaps
5. **SSE sequence numbers**: Carried across transport swaps so the server resumes from the correct position

No single layer is sufficient; their combination covers race conditions, transport swaps, sequence-number negotiation failures, and server-side replay edge cases.

### 12.5 Graceful Degradation

- **Poll errors**: Dual-track exponential backoff with independent give-up timeouts and sleep detection resets
- **Transport failures**: Internal retry budget (10 min) before surfacing to the bridge, then multi-strategy reconnection
- **Token refresh failures**: Up to 3 retries with 60s intervals before giving up; generation counters prevent stale callbacks
- **Feature gates**: Configs fall back to validated defaults on parse failure
- **Shutdown budget**: Cleanup races against a 2s cap; archive timeouts (1.5s) fit within this budget

### 12.6 Epoch-Aware Transport Management

Every CCR v2 transport is bound to a specific epoch. The server rejects requests with stale epochs (409). This means:
- Token refresh must rebuild the entire transport, not just swap the JWT
- The FlushGate must activate during rebuild to prevent writes with stale epoch
- The recovery latch must be claimed synchronously before any `await` to prevent double `/bridge` calls

### 12.7 Idempotent Server Interactions

- Session archival returns 409 if already archived -- safe to double-call
- Environment registration with reuse ID is idempotent
- Work stop supports `force=false` for recoverable exits, `force=true` for shutdown
- Session reconnect force-stops stale workers before re-queuing

### 12.8 Observable Operations

Every significant bridge operation is instrumented with telemetry events, diagnostic logs, and debug logs. Key events include: bridge started, transport connected/closed, session started/done, heartbeat error, token refreshed, reconnected, poll give-up, fatal error, connect timeout, and teardown.
