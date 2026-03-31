# Claude Code: Bridge System — Design Specification

This document analyzes the bridge system architecture of Claude Code — Anthropic's
agentic CLI tool — focusing on how it integrates local CLI sessions with the
claude.ai web interface through two distinct connection architectures, a unified
transport abstraction, and layered authentication and failure recovery strategies.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Two Core Architectures](#2-two-core-architectures)
- [3. Transport Abstraction](#3-transport-abstraction)
- [4. Message Handling](#4-message-handling)
- [5. Environment-Based Flow](#5-environment-based-flow)
- [6. Environment-Less Flow](#6-environment-less-flow)
- [7. Session Management](#7-session-management)
- [8. Authentication and Security](#8-authentication-and-security)
- [9. Message Flow Diagrams](#9-message-flow-diagrams)
- [10. Failure Recovery](#10-failure-recovery)
- [11. Configuration and Feature Gates](#11-configuration-and-feature-gates)
- [12. Design Principles](#12-design-principles)

---

## 1. Overview

The bridge system is Claude Code's **IDE and remote session integration layer**
— the subsystem that makes a local CLI session visible and controllable from the
claude.ai web interface ("Remote Control"). When a user starts Claude Code locally
and enables Remote Control, the bridge connects the CLI process to Anthropic's
Cloud Code Runner (CCR) infrastructure so that the same conversation can be viewed
and driven from a browser or mobile device.

The system solves three fundamental problems:

1. **Bidirectional synchronization** — messages typed locally must appear on the
   web, and prompts typed on the web must flow back to the local CLI for execution
2. **Authentication bridging** — the CLI authenticates with OAuth tokens, but the
   CCR worker endpoints require session-scoped JWTs; the bridge mediates between
   these credential domains
3. **Resilience across connectivity gaps** — laptops sleep, WiFi drops, and VPN
   tunnels reset; the bridge must recover transparently without losing the session

The bridge codebase comprises ~33 files under `src/bridge/`, totaling roughly
500KB of TypeScript. The two largest files — `bridgeMain.ts` (standalone bridge
loop, ~115KB) and `replBridge.ts` (in-process REPL bridge, ~100KB) — implement
the two operational modes. The remaining files provide shared infrastructure:
transport abstraction, message routing, JWT management, session spawning, polling
configuration, and feature gating.

---

## 2. Two Core Architectures

The bridge offers two fundamentally different connection architectures. Both
achieve the same end result — a local CLI session visible on claude.ai — but
differ in how the CLI discovers and connects to the session infrastructure.

### 2.1 Environment-Based Architecture (v1)

> **Source:** `src/bridge/bridgeMain.ts`, `src/bridge/replBridge.ts`

The original design, built on the **Environments API** — a poll-based
work-dispatch layer. The CLI registers as a "bridge environment" with the server,
then enters a long-poll loop waiting for work items (sessions) to be dispatched.

```
CLI                     Environments API              CCR/Session-Ingress
 │                           │                              │
 ├─ POST /environments/bridge ──→                           │
 │        (register)         │                              │
 │                           │                              │
 ├─ POST /environments/{id}/work ←──→                      │
 │        (poll loop)        │                              │
 │                           │  ← user opens session on     │
 │                           │     claude.ai                │
 │  ←── work item            │                              │
 │  (session_id + secret)    │                              │
 │                           │                              │
 ├─ decode work secret       │                              │
 ├─ acknowledge work ────────→                              │
 │                           │                              │
 ├─────────────────── connect transport ──────────────────→ │
 │                           │                              │
 ├─ heartbeat work ──────────→                              │
 │        (periodic)         │                              │
```

This architecture supports two operational modes:

- **Standalone bridge** (`bridgeMain.ts`): `claude remote-control` runs as a
  persistent daemon. It spawns child `claude --print` processes for each session,
  communicating via NDJSON over stdin/stdout. Supports multi-session capacity
  (up to 32 concurrent sessions) with configurable spawn modes (single-session,
  worktree, same-dir).

- **REPL bridge** (`replBridge.ts`): The in-process path used by the interactive
  CLI. When a user types `/remote-control` or has `remoteControlAtStartup`
  enabled, the REPL initializes a bridge connection within the same process. No
  child spawning — messages flow directly between the query engine and the
  transport.

### 2.2 Environment-Less Architecture (v2)

> **Source:** `src/bridge/remoteBridgeCore.ts`

The newer design, gated behind `tengu_bridge_repl_v2`. Eliminates the Environments
API layer entirely — no register, poll, acknowledge, heartbeat, or deregister
lifecycle. The CLI connects directly to the session-ingress layer:

```
CLI                         CCR Code Sessions API
 │                                  │
 ├─ POST /v1/code/sessions ────────→│  (OAuth, no env_id)
 │  ←── session.id (cse_*)         │
 │                                  │
 ├─ POST /v1/code/sessions/{id}/bridge ──→  (OAuth → worker_jwt)
 │  ←── {worker_jwt, expires_in,    │
 │       api_base_url, worker_epoch}│
 │                                  │
 ├─ createV2ReplTransport ─────────→│  (SSE + CCRClient)
 │      (connect)                   │
 │                                  │
 ├─ token refresh scheduler         │
 │  (proactive /bridge re-call)     │
```

The key insight is that the `/bridge` endpoint serves as both credential exchange
AND worker registration — each call bumps the server-side epoch, replacing the
separate `POST /worker/register` step. This reduces the connection setup from
five round-trips (register env, poll, decode, ack, connect) to two (create
session, fetch bridge credentials).

**Important naming distinction**: "env-less" is distinct from "CCR v2" (the
`/worker/*` transport protocol). The environment-based path can *also* use CCR v2
transport via `CLAUDE_CODE_USE_CCR_V2`. The `tengu_bridge_repl_v2` gate controls
which *architecture* is used (poll loop vs. direct connect), not which transport
protocol runs underneath.

---

## 3. Transport Abstraction

> **Source:** `src/bridge/replBridgeTransport.ts`

The `ReplBridgeTransport` interface abstracts the read/write transport so that
both bridge architectures can swap transport implementations without changing
their message-handling logic.

### 3.1 The Interface

```typescript
type ReplBridgeTransport = {
  write(message: StdoutMessage): Promise<void>
  writeBatch(messages: StdoutMessage[]): Promise<void>
  close(): void
  isConnectedStatus(): boolean
  getStateLabel(): string
  setOnData(callback: (data: string) => void): void
  setOnClose(callback: (closeCode?: number) => void): void
  setOnConnect(callback: () => void): void
  connect(): void
  getLastSequenceNum(): number
  readonly droppedBatchCount: number
  reportState(state: SessionState): void
  reportMetadata(metadata: Record<string, unknown>): void
  reportDelivery(eventId: string, status: 'processing' | 'processed'): void
  flush(): Promise<void>
}
```

### 3.2 v1: HybridTransport (WebSocket)

> **Adapter:** `createV1ReplTransport()`

Wraps `HybridTransport` — WebSocket for reads, HTTP POST for writes to
Session-Ingress. This is the legacy path used when the server does not set
`use_code_sessions` in the work secret.

- `getLastSequenceNum()` always returns `0` — Session-Ingress WS does not use
  SSE sequence numbers; replay-on-reconnect is handled server-side
- `reportState()`, `reportMetadata()`, `reportDelivery()` are no-ops — v1 does
  not support worker state reporting
- `flush()` resolves immediately — HybridTransport POSTs are already awaited
  per-write

### 3.3 v2: SSETransport + CCRClient

> **Factory:** `createV2ReplTransport()`

Composes two components:

- **SSETransport** (reads): Server-Sent Events stream at
  `/worker/events/stream`. Supports automatic reconnection with sequence-number
  resumption (`from_sequence_num` / `Last-Event-ID`). The SSE read loop runs
  as a fire-and-forget promise; `connect()` does not await it.

- **CCRClient** (writes): HTTP POST to `/worker/events` via a
  `SerialBatchEventUploader` that batches up to 100 events per request. Also
  handles heartbeats (`PUT /worker`), state reporting, metadata updates, and
  delivery acknowledgments.

Registration happens inside the factory when no pre-existing epoch is provided.
When the caller passes an `epoch` (from the `/bridge` endpoint), the explicit
`registerWorker()` call is skipped — the `/bridge` call *is* the register.

**Epoch mismatch handling**: If the CCRClient receives a 409 (epoch superseded),
it closes both the CCR client and SSE stream, fires `onClose(4090)`, and throws
to unwind the caller. The bridge's close handler then enters recovery.

**Close code semantics**:
| Code | Meaning |
|------|---------|
| 401  | JWT expired — recoverable via OAuth refresh + `/bridge` re-call |
| 4090 | Epoch mismatch — another worker registered |
| 4091 | CCR initialize failed |
| 4092 | SSE reconnect budget exhausted (10-minute internal retry window) |

---

## 4. Message Handling

> **Source:** `src/bridge/bridgeMessaging.ts`

All message routing logic is extracted into pure functions — no closure over
bridge-specific state. Both architectures share identical parsing, filtering,
deduplication, and control-request handling.

### 4.1 Type Guards

Three type predicates classify incoming messages:

- **`isSDKMessage()`**: Validates the discriminated union on `type` — sufficient
  because SDKMessage narrows via the union
- **`isSDKControlResponse()`**: `type === 'control_response'` with a `response`
  field — permission decisions from the web UI
- **`isSDKControlRequest()`**: `type === 'control_request'` with `request_id`
  and `request` — server lifecycle commands (initialize, set_model, interrupt)

### 4.2 Ingress Routing

`handleIngressMessage()` parses each raw string from the transport and routes it:

1. **Control responses** (permission decisions) → `onPermissionResponse` callback
2. **Control requests** (server commands) → `handleServerControlRequest()`
3. **SDK messages with echoed UUIDs** → silently dropped (see 4.3)
4. **SDK messages with re-delivered UUIDs** → silently dropped (see 4.3)
5. **User-type messages** → `onInboundMessage` callback (prompts from the web)
6. **Other types** → ignored (assistant messages are not re-injected)

### 4.3 Echo Deduplication (BoundedUUIDSet)

> **Source:** `src/bridge/bridgeMessaging.ts:429-461`

Messages the CLI sends to the server come back on the read stream as echoes.
Without deduplication, the CLI would re-process its own output as new input.

`BoundedUUIDSet` is a FIFO-bounded set backed by a circular ring buffer:

```typescript
class BoundedUUIDSet {
  private readonly ring: (string | undefined)[]
  private readonly set = new Set<string>()
  private writeIdx = 0

  add(uuid: string): void {
    // Evict oldest entry at writeIdx, insert new
  }
  has(uuid: string): boolean {
    return this.set.has(uuid)
  }
}
```

Capacity is 2000 entries (configurable via `uuid_dedup_buffer_size`), providing
O(1) lookup with O(capacity) memory. Two instances are maintained per bridge
session:

- **`recentPostedUUIDs`**: Seeded with initial message UUIDs. Catches echoes of
  outbound messages.
- **`recentInboundUUIDs`**: Catches re-delivered inbound prompts after transport
  swaps where sequence-number negotiation fails.

### 4.4 Server Control Requests

`handleServerControlRequest()` responds to server-initiated control messages.
The server kills the connection if responses are not sent within ~10-14 seconds.

| Subtype | Action | Response |
|---------|--------|----------|
| `initialize` | Return minimal capabilities (empty commands, models, account) | success |
| `set_model` | Invoke `onSetModel` callback | success |
| `set_max_thinking_tokens` | Invoke `onSetMaxThinkingTokens` callback | success |
| `set_permission_mode` | Invoke `onSetPermissionMode` callback; return policy verdict | success or error |
| `interrupt` | Invoke `onInterrupt` callback | success |
| unknown | No action | error with diagnostic message |

In **outbound-only mode** (CCR mirror), all mutable requests return an error
response except `initialize` (which must succeed or the server disconnects).

### 4.5 Message Eligibility

`isEligibleBridgeMessage()` filters which internal messages are forwarded to the
transport. Only three message types pass:

- `user` messages (non-virtual)
- `assistant` messages (non-virtual)
- `system` messages with `subtype === 'local_command'`

Virtual messages (REPL inner calls) are excluded — the bridge consumer sees the
REPL `tool_use`/`result` which summarizes the work.

---

## 5. Environment-Based Flow

> **Source:** `src/bridge/bridgeMain.ts` (standalone), `src/bridge/replBridge.ts` (REPL)

### 5.1 Registration

The bridge registers with the Environments API via
`POST /v1/environments/bridge`, sending:

- Machine name, directory, branch, git repo URL
- `max_sessions` — advertised capacity for multi-session mode
- `metadata.worker_type` — `'claude_code'` or `'claude_code_assistant'`
- Optional `environment_id` for idempotent re-registration (resume scenarios)

The server returns an `environment_id` and `environment_secret` — the latter
is used to authenticate subsequent poll requests.

### 5.2 Poll Loop

The core of the environment-based architecture is a `while (!signal.aborted)`
loop that polls `POST /environments/{id}/work`. Poll intervals are
GrowthBook-configurable via `tengu_bridge_poll_interval_config`:

| State | Interval |
|-------|----------|
| No sessions active | `poll_interval_ms_not_at_capacity` (default varies) |
| Some sessions, not full | `multisession_poll_interval_ms_partial_capacity` |
| At capacity | `multisession_poll_interval_ms_at_capacity` (0 = disabled) |

When at capacity with heartbeat mode enabled
(`non_exclusive_heartbeat_interval_ms > 0`), the loop enters a nested heartbeat
cycle: it heartbeats active work items without polling, periodically breaking
out to poll based on a deadline.

### 5.3 Work Secret Decoding

> **Source:** `src/bridge/workSecret.ts`

When the poll returns a work item, its `secret` field is a base64url-encoded
JSON blob:

```typescript
type WorkSecret = {
  version: number                          // Must be 1
  session_ingress_token: string            // JWT for session-ingress auth
  api_base_url: string                     // Base URL for transport
  sources: Array<{ type: string; ... }>    // Git sources
  auth: Array<{ type: string; token: string }>
  use_code_sessions?: boolean              // Server-driven CCR v2 selector
  environment_variables?: Record<string, string>
}
```

`decodeWorkSecret()` validates the version and required fields, throwing on
malformed secrets. A decode failure marks the work item as completed (via
`stopWork`) and adds it to `completedWorkIds` to prevent XAUTOCLAIM
re-delivery.

### 5.4 Acknowledgment

After committing to handle the work (capacity check passed, secret decoded),
the bridge explicitly acknowledges via `api.acknowledgeWork()`. Acknowledgment
happens *after* the capacity guard — acking before spawn would permanently lose
the work item if the at-capacity check breaks without spawning.

### 5.5 Session Spawning (Standalone Bridge)

> **Source:** `src/bridge/sessionRunner.ts`

In standalone mode (`bridgeMain.ts`), a `SessionSpawner` creates child `claude
--print` processes:

```typescript
const args = [
  '--print',
  '--sdk-url', opts.sdkUrl,
  '--session-id', opts.sessionId,
  '--input-format', 'stream-json',
  '--output-format', 'stream-json',
  '--replay-user-messages',
]
```

The child communicates over three pipes:
- **stdin**: Receives token refresh messages
  (`update_environment_variables` NDJSON)
- **stdout**: Emits NDJSON events (parsed for activity tracking, permission
  requests, user message detection)
- **stderr**: Captured in a ring buffer (last 10 lines) for error diagnostics

The `SessionHandle` provides:
- `done`: A promise resolving to `'completed' | 'failed' | 'interrupted'`
- `kill()` / `forceKill()`: SIGTERM then SIGKILL after grace period
- `updateAccessToken()`: Sends fresh JWT to child via stdin
- `activities`: Ring buffer of last 10 `SessionActivity` records

### 5.6 Transport Connection (REPL Bridge)

In REPL mode (`replBridge.ts`), no child process is spawned. Instead, the bridge
creates a transport directly and wires callbacks:

- `setOnConnect` → flush initial history, drain queued messages, notify state
- `setOnData` → `handleIngressMessage()` for routing
- `setOnClose` → `handleTransportPermanentClose()` for recovery

The transport choice (v1 WS vs. v2 SSE+CCR) is driven by the work secret's
`use_code_sessions` flag or the `CLAUDE_BRIDGE_USE_CCR_V2` env override.

### 5.7 Heartbeat

Active work items are heartbeated via `api.heartbeatWork()`, which uses the
session-ingress JWT (not the environment secret) for authentication. Heartbeat
results are categorized:

- `ok` — at least one heartbeat succeeded
- `auth_failed` — 401/403 (JWT expired); triggers `reconnectSession()` to
  re-queue the work item with a fresh JWT
- `fatal` — 404/410 (environment expired/deleted)
- `failed` — all heartbeats failed for other reasons

### 5.8 Cleanup

When a session ends, `onSessionDone()` performs:

1. Remove from all tracking maps
2. Clear per-session timeout timer and token refresh timer
3. Wake the capacity signal (so the poll loop can accept new work)
4. Call `stopWork()` to notify the server (skipped for interrupted sessions)
5. Clean up git worktree if one was created
6. In multi-session mode: archive session and continue; in single-session mode:
   abort the poll loop

On bridge shutdown: SIGTERM all children, wait up to 30s, SIGKILL stragglers,
stop all work items, archive all sessions, deregister the environment.

---

## 6. Environment-Less Flow

> **Source:** `src/bridge/remoteBridgeCore.ts`

### 6.1 Session Creation

`POST /v1/code/sessions` with OAuth authentication (no environment ID). The
request body includes `title`, `bridge: {}` (positive signal for the oneof
runner), and optional `tags`. Returns a `cse_*`-prefixed session ID.

### 6.2 Bridge Credential Exchange

`POST /v1/code/sessions/{id}/bridge` exchanges the OAuth token for:

- `worker_jwt` — opaque JWT for CCR worker endpoints (not decoded client-side)
- `expires_in` — TTL in seconds
- `api_base_url` — CCR endpoint base URL
- `worker_epoch` — monotonic counter; each `/bridge` call bumps it

This single endpoint replaces the three-step v1 process of register environment,
poll for work, and decode work secret.

### 6.3 Token Refresh Scheduler

> **Source:** `src/bridge/jwtUtils.ts`

`createTokenRefreshScheduler()` proactively refreshes tokens before expiry:

```
Token received (expires_in = 4h)
  ↓
Schedule refresh at expires_in - buffer (5min default)
  ↓                   ↓ (3h55min later)
                    doRefresh()
                      ↓
                    getAccessToken() → refresh OAuth if needed
                      ↓
                    onRefresh(sessionId, oauthToken)
                      ↓
                    Schedule follow-up (30min fallback)
```

**Generation counters** prevent stale `doRefresh()` calls from setting follow-up
timers after a session has been cancelled or rescheduled. Each `schedule()` and
`cancel()` bumps the generation; `doRefresh()` checks its generation matches
before proceeding.

**Failure handling**: Up to 3 consecutive failures before giving up. Each failure
schedules a 60-second retry. Counter resets on successful token retrieval.

For the environment-less path, the refresh callback re-fetches `/bridge`
credentials and rebuilds the transport (since each `/bridge` call bumps epoch,
a JWT-only swap would leave the old CCRClient heartbeating with a stale epoch).

### 6.4 Transport Creation and Rebuild

The v2 transport is created via `createV2ReplTransport()` with the worker JWT,
session URL, and epoch from the `/bridge` response. On proactive refresh or 401
recovery, `rebuildTransport()`:

1. Activates the `FlushGate` to queue writes during rebuild
2. Captures the SSE sequence-number high-water mark from the old transport
3. Closes the old transport
4. Creates a new transport with fresh credentials and the captured sequence number
5. Re-wires callbacks and calls `connect()`
6. Drains the flush gate into the new transport

**Atomicity**: `authRecoveryInFlight` is a boolean latch set synchronously before
any `await`. Both the proactive refresh and 401 recovery paths check this flag to
prevent double execution — critical because laptop wake can fire both paths
simultaneously, and each `/bridge` call bumps epoch.

### 6.5 Initial History Flush

When the bridge connects with existing conversation history, a `FlushGate`
prevents ordering races:

```
start() → enqueue() returns true, messages queued
  ↓
  flushHistory(initialMessages) via writeBatch()
  ↓
  .finally() → end() → returns queued items → drain via writeBatch()
```

The gate ensures the server receives `[history..., live...]` in order. If a 401
interrupts the flush, `initialFlushDone` resets so the new `onConnect` re-flushes.

---

## 7. Session Management

### 7.1 SessionHandle

> **Source:** `src/bridge/types.ts:178-190`

The `SessionHandle` is the bridge's interface to a running session process:

```typescript
type SessionHandle = {
  sessionId: string
  done: Promise<SessionDoneStatus>    // Resolves on process exit
  kill(): void                         // SIGTERM
  forceKill(): void                    // SIGKILL
  activities: SessionActivity[]        // Ring buffer (last 10)
  currentActivity: SessionActivity | null
  accessToken: string                  // Mutable via updateAccessToken
  lastStderr: string[]                 // Ring buffer (last 10 lines)
  writeStdin(data: string): void
  updateAccessToken(token: string): void  // Sends to child via stdin
}
```

`updateAccessToken()` is the critical mechanism for token refresh in the
standalone bridge: it sends an `update_environment_variables` NDJSON message
to the child's stdin, which the child's `StructuredIO` handler picks up and
applies to `process.env`.

### 7.2 Activity Tracking

`extractActivities()` in `sessionRunner.ts` parses the child's NDJSON stdout
to build a real-time activity feed:

- `tool_start` — extracted from assistant messages containing `tool_use` blocks;
  human-readable summaries via `toolSummary()` (e.g., "Reading package.json",
  "Running npm test")
- `text` — assistant text blocks (first 80 chars)
- `result` — session completed or errored
- `error` — error subtype with message

The standalone bridge displays these in a live status line showing elapsed time,
current activity, and a trail of the last 5 tool activities.

### 7.3 Multi-Session Capacity Management

> **Source:** `src/bridge/bridgeMain.ts`

The standalone bridge supports up to `SPAWN_SESSIONS_DEFAULT = 32` concurrent
sessions, gated by `tengu_ccr_bridge_multi_session`. Three spawn modes:

| Mode | Directory | Lifecycle |
|------|-----------|-----------|
| `single-session` | cwd | Bridge exits when session ends |
| `worktree` | Isolated git worktree per session | Bridge persists |
| `same-dir` | Shared cwd | Bridge persists |

Capacity is tracked via `activeSessions` (a `Map<string, SessionHandle>`).
When at capacity, the poll loop enters a heartbeat-only or slow-poll mode.
A `CapacityWake` signal (see below) wakes the loop immediately when a session
completes.

### 7.4 CapacityWake

> **Source:** `src/bridge/capacityWake.ts`

A shared primitive for both bridge paths. When at capacity, the poll loop
sleeps with a merged abort signal that fires on either:
- The outer loop signal (shutdown)
- The capacity-wake controller (session ended)

`wake()` aborts the current controller and arms a fresh one, so the poll loop
immediately re-checks for new work.

---

## 8. Authentication and Security

### 8.1 OAuth Flow

The bridge authenticates to the Environments API and Sessions API using the
user's claude.ai OAuth token. `getBridgeAccessToken()` returns the current token;
`handleOAuth401Error()` handles refresh on 401 responses.

Pre-flight checks in `initReplBridge()`:
1. OAuth token exists (`getBridgeAccessToken()`)
2. Organization policy allows Remote Control (`isPolicyAllowed('allow_remote_control')`)
3. Proactive token refresh if expired (`checkAndRefreshOAuthTokenIfNeeded()`)
4. Cross-process dead-token backoff (global config tracks consecutive failures
   per `expiresAt` — after 3 failures, skip silently)

### 8.2 JWT Tokens (Session-Ingress)

Work secrets contain a `session_ingress_token` — a JWT scoped to the specific
session. This token authenticates:
- WebSocket connections to Session-Ingress (v1)
- CCR `/worker/*` endpoints (v2)
- Work heartbeat calls

`decodeJwtPayload()` and `decodeJwtExpiry()` parse the JWT's `exp` claim
without signature verification. The `sk-ant-si-` prefix is stripped if present.
These utilities drive the proactive token refresh scheduler.

The environment-less path receives an opaque `worker_jwt` from the `/bridge`
endpoint, along with an explicit `expires_in` — the JWT is not decoded
client-side in this path.

### 8.3 Trusted Device Tokens

> **Source:** `src/bridge/trustedDevice.ts`

Bridge sessions have `SecurityTier=ELEVATED` on the server. When the
`tengu_sessions_elevated_auth_enforcement` gate is enabled, the CLI sends an
`X-Trusted-Device-Token` header on bridge API calls.

**Enrollment**: `enrollTrustedDevice()` runs during `/login` via
`POST /auth/trusted_devices`. The server gates enrollment on
`account_session.created_at < 10min`, so enrollment must happen immediately after
authentication. The token is persistent (90-day rolling expiry) and stored in the
OS keychain via `getSecureStorage()`.

**Retrieval**: `getTrustedDeviceToken()` checks the GrowthBook gate live (so gate
flips take effect without restart) but memoizes the keychain read (which spawns a
macOS `security` subprocess at ~40ms). The memo cache is cleared after enrollment
and on logout.

### 8.4 Work Secret Validation

`decodeWorkSecret()` performs structural validation:
- Version must be `1`
- `session_ingress_token` must be a non-empty string
- `api_base_url` must be a string

### 8.5 Bridge ID Validation

`validateBridgeId()` prevents path traversal and injection in server-provided IDs
used in URL path segments. Only `[a-zA-Z0-9_-]+` is allowed — dots, slashes, and
other special characters are rejected.

---

## 9. Message Flow Diagrams

### 9.1 Environment-Based: User Types on claude.ai

```
claude.ai                CCR Server              Bridge (CLI)
    │                        │                       │
    ├─ user types prompt ───→│                       │
    │                        ├─ dispatch work ──────→│
    │                        │  (poll response)      │
    │                        │                       ├─ decode secret
    │                        │←── acknowledge work ──┤
    │                        │                       │
    │                        │                       ├─ connect transport
    │                        │                       │  (v1: WS, v2: SSE+CCR)
    │                        │                       │
    │   ←── SSE/WS event ───│←── user message ──────┤
    │                        │   (onInboundMessage)  │
    │                        │                       │
    │                        │                       ├─ execute query
    │                        │                       │
    │   ←── SSE/WS event ───│←── assistant message ─┤
    │                        │   (writeMessages)     │
    │                        │                       │
    │                        │                       ├─ tool execution
    │                        │                       │
    │   ←── SSE/WS event ───│←── tool result ───────┤
    │                        │                       │
    │   ←── SSE/WS event ───│←── result event ──────┤
    │                        │                       │
```

### 9.2 Environment-Less: REPL Enables Remote Control

```
REPL                     CCR Server              claude.ai
  │                          │                       │
  ├─ POST /v1/code/sessions ─→                       │
  │  ←── session_id (cse_*)  │                       │
  │                          │                       │
  ├─ POST /sessions/{id}/bridge ─→                   │
  │  ←── {jwt, epoch, url}  │                       │
  │                          │                       │
  ├─ createV2ReplTransport ──→                       │
  │  (SSE connect + CCR init)│                       │
  │                          │                       │
  ├─ flush initial history ──→                       │
  │  (writeBatch)            │──── events ──────────→│
  │                          │                       │
  │                          │   user types on web   │
  │  ←── SSE event ──────────│←──────────────────────┤
  │  (handleIngressMessage)  │                       │
  │                          │                       │
  │  [4h55m later]           │                       │
  ├─ proactive JWT refresh   │                       │
  │  POST /sessions/{id}/bridge ─→                   │
  │  ←── fresh {jwt, epoch}  │                       │
  │  rebuildTransport()      │                       │
  │  (SSE resumes from seq#) │                       │
```

### 9.3 Permission Flow

```
CLI (tool execution)     Bridge Transport         claude.ai
  │                          │                       │
  ├─ control_request ────────→                       │
  │  (can_use_tool)          │──── event ───────────→│
  │                          │    reportState(       │
  │                          │    requires_action)   │
  │                          │                       │
  │                          │   user clicks Allow   │
  │                          │                       │
  │  ←── control_response ───│←──────────────────────┤
  │  (onPermissionResponse)  │                       │
  │                          │    reportState(       │
  │                          │    running)           │
  │                          │                       │
  ├─ tool executes           │                       │
```

---

## 10. Failure Recovery

### 10.1 Transport Failures (REPL Bridge)

> **Source:** `src/bridge/replBridge.ts:887-966`

`handleTransportPermanentClose()` handles terminal transport failures. By the
time this fires, the underlying transport has already exhausted its internal
retry budget (10 minutes for SSE, configurable reconnect attempts for WS).

**Close code 1000 (clean close)**: Session ended normally. Tear down the bridge.

**All other codes** (budget exhausted, server rejection): Trigger
`reconnectEnvironmentWithSession()`, which tries two strategies:

1. **Reconnect-in-place**: Re-register with `reuseEnvironmentId` set to the
   current env ID. If the server returns the same ID, call `reconnectSession()`
   to force-stop stale workers and re-queue the session. The session ID stays the
   same — the URL on the user's device remains valid.

2. **Fresh session fallback**: If the server returns a different ID (original
   env TTL-expired) or reconnect fails, archive the old session and create a new
   one on the now-registered environment. Clear `previouslyFlushedUUIDs` so
   initial messages are re-sent. Reset SSE sequence number and inbound UUID
   dedup set.

A reentrancy guard (`reconnectPromise`) ensures concurrent callers share the
same reconnection attempt. A counter caps at `MAX_ENVIRONMENT_RECREATIONS = 3`
per burst, resetting on success.

### 10.2 Transport Failures (Environment-Less)

> **Source:** `src/bridge/remoteBridgeCore.ts:530-590`

The env-less path has no environment to reconnect. 401 on the SSE stream
triggers `recoverFromAuthFailure()`:

1. Set `authRecoveryInFlight = true` synchronously
2. Refresh OAuth via `onAuth401()`
3. Re-fetch `/bridge` credentials
4. `rebuildTransport()` with fresh JWT and epoch

Non-401 terminal failures (4090 epoch mismatch, 4091 init failure, 4092 budget
exhausted) transition to `'failed'` state with no recovery — the session is
dead.

### 10.3 Poll Loop Failures (Environment-Based)

> **Source:** `src/bridge/bridgeMain.ts:1236-1401`

The poll loop uses **dual-track exponential backoff** — connection errors and
general errors are tracked independently with separate budgets:

| Track | Initial | Cap | Give-up |
|-------|---------|-----|---------|
| Connection | 2s | 2min | 10min |
| General | 500ms | 30s | 10min |

Switching error types resets the other track's counter.

**Fatal errors** (`BridgeFatalError`): 401/403 responses are non-retryable.
The loop breaks immediately. Server-enforced expiry gets a clean status message;
suppressible 403s (scope/permission issues) are logged silently.

### 10.4 Sleep Detection

Both the poll loop and the transport use sleep detection to reset error budgets
after laptop wake:

```typescript
function pollSleepDetectionThresholdMs(backoff: BackoffConfig): number {
  return backoff.connCapMs * 2  // 2x the connection backoff cap
}
```

If the gap since the last error exceeds this threshold, the machine likely slept.
Error tracking resets so the bridge retries with a fresh budget rather than
immediately giving up after wake.

### 10.5 Adaptive Backoff with Jitter

All backoff calculations apply jitter (typically +/- 50%) to prevent
thundering-herd effects when many bridges reconnect simultaneously after a
server outage:

```
delay = base * 2^attempt          // exponential
delay = min(delay, cap)           // capped
delay = delay * (0.5 + random())  // jittered
```

---

## 11. Configuration and Feature Gates

### 11.1 GrowthBook Runtime Gates

| Gate | Purpose |
|------|---------|
| `tengu_ccr_bridge` | Master enable for Remote Control |
| `tengu_bridge_repl_v2` | Enable env-less bridge architecture |
| `tengu_ccr_bridge_multi_session` | Enable multi-session spawn modes |
| `tengu_bridge_repl_v2_cse_shim_enabled` | `cse_*` to `session_*` ID retag |
| `tengu_sessions_elevated_auth_enforcement` | Trusted device token enforcement |
| `tengu_cobalt_harbor` | Auto-connect Remote Control at startup |
| `tengu_ccr_mirror` | CCR mirror mode (outbound-only) |

### 11.2 GrowthBook Dynamic Configs

| Config | Purpose |
|--------|---------|
| `tengu_bridge_poll_interval_config` | Poll/heartbeat intervals, Zod-validated |
| `tengu_bridge_repl_v2_config` | Env-less timing config (retry, heartbeat, timeouts) |
| `tengu_bridge_min_version` | Minimum CLI version for v1 path |
| `tengu_bridge_initial_history_cap` | Max initial messages to replay (default 200) |

### 11.3 Compile-Time Feature Flags

| Flag | Purpose |
|------|---------|
| `BRIDGE_MODE` | Include bridge code in the build at all |
| `CCR_AUTO_CONNECT` | Auto-connect default behavior |
| `CCR_MIRROR` | Mirror mode support |
| `KAIROS` | Assistant-mode worker type differentiation |

### 11.4 Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_BRIDGE_OAUTH_TOKEN` | Override OAuth token (ant-only) |
| `CLAUDE_BRIDGE_BASE_URL` | Override API base URL (ant-only) |
| `CLAUDE_BRIDGE_SESSION_INGRESS_URL` | Override session ingress URL (ant-only) |
| `CLAUDE_BRIDGE_USE_CCR_V2` | Force CCR v2 transport in env-based path |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | Session JWT for child processes |
| `CLAUDE_CODE_USE_CCR_V2` | Enable CCR v2 in child processes |
| `CLAUDE_CODE_WORKER_EPOCH` | Worker epoch for child processes |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | Set to `'bridge'` in spawned children |
| `CLAUDE_TRUSTED_DEVICE_TOKEN` | Override trusted device token |

### 11.5 Poll Configuration Schema

The poll config (`pollConfig.ts`) uses Zod validation with defense-in-depth
constraints:

- `.min(100)` on seek-work intervals prevents fat-fingered GrowthBook values
- At-capacity intervals use `0-or->=100` refinement: 0 means "disabled", 1-99
  rejected to prevent unit confusion (ops entering seconds instead of ms)
- Object-level refinements require at least one at-capacity liveness mechanism
  (heartbeat OR poll interval) to prevent tight-loop drift configs

---

## 12. Design Principles

### 12.1 Two Architectures, One Abstraction

The `ReplBridgeTransport` interface and the shared `bridgeMessaging.ts` module
ensure that both connection architectures share identical message handling,
deduplication, and control-request logic. The architectural choice (env-based vs.
env-less) is confined to the construction site — once a transport is created, the
rest of the bridge cannot tell which architecture produced it.

### 12.2 Bootstrap Isolation

A recurring constraint throughout the bridge codebase is **import graph
discipline**. `replBridge.ts` (the core) reads nothing from bootstrap state or
session storage — all context comes from `BridgeCoreParams`. This isolation
exists because `sessionStorage.ts` transitively imports `src/commands.ts` (the
entire slash command + React component tree, ~1300 modules). Keeping the core
free of bootstrap imports lets daemon callers (Agent SDK) use `initBridgeCore`
without bloating their bundle.

The same pattern appears in `codeSessionApi.ts` (thin HTTP wrappers without
analytics/transport dependencies), `bridgeMessaging.ts` (pure functions, no
closures), and the injected callback pattern throughout `BridgeCoreParams`
(e.g., `createSession`, `archiveSession`, `toSDKMessages` are injected rather
than imported).

### 12.3 Crash Recovery via Bridge Pointer

> **Source:** `src/bridge/bridgePointer.ts`

After session creation, the bridge writes a crash-recovery pointer to disk
containing `{sessionId, environmentId, source}`. If the process is killed at any
point after this write, a subsequent `claude remote-control --continue` from the
same directory can detect and resume the session. In perpetual mode (KAIROS
assistant sessions), the pointer survives clean exits.

### 12.4 Defense in Depth for Deduplication

Message deduplication uses multiple independent layers:

1. **Primary**: The hook's `lastWrittenIndexRef` tracks the array index boundary
2. **Secondary**: `recentPostedUUIDs` (2000-cap ring buffer) catches echoes
3. **Tertiary**: `initialMessageUUIDs` (unbounded set) catches initial message
   echoes that may have been evicted from the ring buffer
4. **Inbound**: `recentInboundUUIDs` catches re-delivered prompts after
   transport swaps
5. **SSE sequence numbers**: `lastTransportSequenceNum` carried across transport
   swaps so the server resumes from the correct position

No single layer is sufficient; their combination covers race conditions,
transport swaps, sequence-number negotiation failures, and server-side replay
edge cases.

### 12.5 Graceful Degradation

- **Poll errors**: Dual-track exponential backoff with independent give-up
  timeouts and sleep detection resets
- **Transport failures**: Internal retry budget (10 min) before surfacing to
  the bridge, then multi-strategy reconnection
- **Token refresh failures**: Up to 3 retries with 60s intervals before
  giving up; generation counters prevent stale callbacks
- **Feature gates**: GrowthBook configs fall back to validated defaults on
  parse failure (Zod rejects the entire object, not individual fields)
- **Shutdown budget**: `gracefulShutdown` races cleanup against a 2s cap;
  archive timeouts (1.5s) fit within this budget

### 12.6 Epoch-Aware Transport Management

Every CCR v2 transport is bound to a specific epoch. The server rejects requests
with stale epochs (409). This means:

- Token refresh must rebuild the entire transport, not just swap the JWT
- The `FlushGate` must activate during rebuild to prevent writes with stale
  epoch from being silently dropped
- `authRecoveryInFlight` must be claimed synchronously before any `await` to
  prevent double `/bridge` calls (each of which bumps epoch)

### 12.7 Idempotent Server Interactions

- `archiveSession` returns 409 if already archived — safe to double-call
- Environment registration with `reuseEnvironmentId` is idempotent
- `stopWork` is called with `force=false` for recoverable exits, `force=true`
  for shutdown
- `reconnectSession` force-stops stale workers before re-queuing

### 12.8 Observable Operations

Every significant bridge operation is instrumented with telemetry events
(`logEvent`), diagnostic logs (`logForDiagnosticsNoPII`), and debug logs
(`logForDebugging`). Key events include:

| Event | Meaning |
|-------|---------|
| `tengu_bridge_repl_started` | Bridge session initialized |
| `tengu_bridge_repl_ws_connected` | Transport connected (with cause) |
| `tengu_bridge_repl_ws_closed` | Transport closed (with code) |
| `tengu_bridge_session_started` | Child session spawned |
| `tengu_bridge_session_done` | Child session exited |
| `tengu_bridge_heartbeat_error` | Heartbeat failure |
| `tengu_bridge_token_refreshed` | Token proactively refreshed |
| `tengu_bridge_reconnected` | Recovered from connection loss |
| `tengu_bridge_poll_give_up` | Error budget exhausted |
| `tengu_bridge_fatal_error` | Non-retryable error |
| `tengu_bridge_repl_connect_timeout` | Connect deadline exceeded |
| `tengu_bridge_repl_teardown` | Bridge torn down (with archive status) |
