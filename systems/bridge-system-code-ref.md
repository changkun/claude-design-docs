# Bridge System — Code Reference

## File Inventory

All files under `src/bridge/`:

| File | Purpose |
|------|---------|
| `bridgeMain.ts` (~115KB) | Standalone bridge daemon loop |
| `replBridge.ts` (~100KB) | In-process REPL bridge core |
| `remoteBridgeCore.ts` | Env-less bridge core |
| `replBridgeTransport.ts` | Transport abstraction interface + v1/v2 factories |
| `bridgeMessaging.ts` | Shared message routing, type guards, BoundedUUIDSet |
| `workSecret.ts` | Work secret decoding, SDK URL builders, worker registration |
| `jwtUtils.ts` | JWT decode, token refresh scheduler |
| `trustedDevice.ts` | Trusted device token enrollment + retrieval |
| `capacityWake.ts` | Shared capacity-wake primitive |
| `flushGate.ts` | FlushGate state machine for initial history flush |
| `bridgePointer.ts` | Crash-recovery pointer read/write |
| `pollConfig.ts` | Poll interval config with Zod schema |
| `pollConfigDefaults.ts` | Default poll config values |
| `types.ts` | SessionHandle, BridgeConfig, WorkSecret, SpawnMode, etc. |
| `bridgeApi.ts` | Bridge API client implementation |
| `bridgeConfig.ts` | Bridge configuration |
| `bridgeDebug.ts` | Debug utilities |
| `bridgeEnabled.ts` | Bridge enable checks |
| `bridgePermissionCallbacks.ts` | Permission callback wiring |
| `bridgeStatusUtil.ts` | Status line utilities |
| `bridgeUI.ts` | Bridge UI rendering |
| `codeSessionApi.ts` | Thin HTTP wrappers for code sessions |
| `createSession.ts` | Session creation logic |
| `debugUtils.ts` | Debug/logging utilities |
| `envLessBridgeConfig.ts` | Env-less bridge timing config |
| `inboundAttachments.ts` | Inbound attachment handling |
| `inboundMessages.ts` | Inbound message processing |
| `initReplBridge.ts` | REPL bridge initialization + gate checks |
| `replBridgeHandle.ts` | REPL bridge handle |
| `sessionIdCompat.ts` | Session ID compatibility (cse_ <-> session_) |
| `sessionRunner.ts` | Child process spawning + activity tracking |

## Type Definitions

**`src/bridge/types.ts:178-190`** -- `SessionHandle`:
```typescript
type SessionHandle = {
  sessionId: string
  done: Promise<SessionDoneStatus>
  kill(): void
  forceKill(): void
  activities: SessionActivity[]
  currentActivity: SessionActivity | null
  accessToken: string
  lastStderr: string[]
  writeStdin(data: string): void
  updateAccessToken(token: string): void
}
```

**`src/bridge/types.ts:33-51`** -- `WorkSecret`:
```typescript
type WorkSecret = {
  version: number
  session_ingress_token: string
  api_base_url: string
  sources: Array<{ type: string; git_info?: { type: string; repo: string; ref?: string; token?: string } }>
  auth: Array<{ type: string; token: string }>
  claude_code_args?: Record<string, string> | null
  mcp_config?: unknown | null
  environment_variables?: Record<string, string> | null
  use_code_sessions?: boolean
}
```

**`src/bridge/replBridgeTransport.ts:23-70`** -- `ReplBridgeTransport`:
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

## Key Functions and Their Locations

| Function | File | Line(s) |
|----------|------|---------|
| `createV1ReplTransport()` | `replBridgeTransport.ts` | 78-103 |
| `createV2ReplTransport()` | `replBridgeTransport.ts` | 119-369 |
| `handleIngressMessage()` | `bridgeMessaging.ts` | 132-208 |
| `handleServerControlRequest()` | `bridgeMessaging.ts` | 243-391 |
| `isEligibleBridgeMessage()` | `bridgeMessaging.ts` | 77-88 |
| `isSDKMessage()` | `bridgeMessaging.ts` | 36-43 |
| `isSDKControlResponse()` | `bridgeMessaging.ts` | 46-56 |
| `isSDKControlRequest()` | `bridgeMessaging.ts` | 59-70 |
| `BoundedUUIDSet` (class) | `bridgeMessaging.ts` | 429-461 |
| `makeResultMessage()` | `bridgeMessaging.ts` | 399-416 |
| `decodeWorkSecret()` | `workSecret.ts` | 6-32 |
| `buildSdkUrl()` | `workSecret.ts` | 41-48 |
| `buildCCRv2SdkUrl()` | `workSecret.ts` | 81-87 |
| `sameSessionId()` | `workSecret.ts` | 62-73 |
| `registerWorker()` | `workSecret.ts` | 97-127 |
| `decodeJwtPayload()` | `jwtUtils.ts` | 21-32 |
| `decodeJwtExpiry()` | `jwtUtils.ts` | 38-49 |
| `createTokenRefreshScheduler()` | `jwtUtils.ts` | 72-256 |
| `createCapacityWake()` | `capacityWake.ts` | 28-56 |
| `handleTransportPermanentClose()` | `replBridge.ts` | 887-966 |
| `recoverFromAuthFailure()` | `remoteBridgeCore.ts` | 530-590 |
| `getTrustedDeviceToken()` | `trustedDevice.ts` | 54-59 |
| `enrollTrustedDevice()` | `trustedDevice.ts` | 98-210 |
| `writeBridgePointer()` | `bridgePointer.ts` | 62-74 |
| `readBridgePointer()` | `bridgePointer.ts` | 83-113 |
| `getPollIntervalConfig()` | `pollConfig.ts` | 102-110 |
| `validateBridgeId()` | `bridgeApi.ts` (+ `replBridge.ts`, `bridgeMain.ts`) | (multiple files) |

## Constants

| Constant | File | Value |
|----------|------|-------|
| `SPAWN_SESSIONS_DEFAULT` | `bridgeMain.ts:83` | `32` |
| `MAX_ENVIRONMENT_RECREATIONS` | `replBridge.ts:583` (and `:1920`) | `3` |
| `TOKEN_REFRESH_BUFFER_MS` | `jwtUtils.ts:52` | `5 * 60 * 1000` (5 min) |
| `FALLBACK_REFRESH_INTERVAL_MS` | `jwtUtils.ts:55` | `30 * 60 * 1000` (30 min) |
| `MAX_REFRESH_FAILURES` | `jwtUtils.ts:58` | `3` |
| `REFRESH_RETRY_DELAY_MS` | `jwtUtils.ts:61` | `60_000` (60s) |
| `BRIDGE_POINTER_TTL_MS` | `bridgePointer.ts:40` | `4 * 60 * 60 * 1000` (4h) |
| `DEFAULT_BACKOFF.connInitialMs` | `bridgeMain.ts:73` | `2_000` |
| `DEFAULT_BACKOFF.connCapMs` | `bridgeMain.ts:74` | `120_000` (2 min) |
| `DEFAULT_BACKOFF.connGiveUpMs` | `bridgeMain.ts:75` | `600_000` (10 min) |
| `DEFAULT_BACKOFF.generalInitialMs` | `bridgeMain.ts:76` | `500` |
| `DEFAULT_BACKOFF.generalCapMs` | `bridgeMain.ts:77` | `30_000` |
| `DEFAULT_BACKOFF.generalGiveUpMs` | `bridgeMain.ts:78` | `600_000` (10 min) |
| `uuid_dedup_buffer_size` default | `envLessBridgeConfig.ts:50` | `2000` |

## Feature Gate Names (exact strings)

| Gate String | Purpose |
|-------------|---------|
| `tengu_ccr_bridge` | Master enable for Remote Control |
| `tengu_bridge_repl_v2` | Enable env-less bridge architecture |
| `tengu_ccr_bridge_multi_session` | Enable multi-session spawn modes |
| `tengu_bridge_repl_v2_cse_shim_enabled` | `cse_*` to `session_*` ID retag |
| `tengu_sessions_elevated_auth_enforcement` | Trusted device token enforcement |
| `tengu_cobalt_harbor` | Auto-connect Remote Control at startup |
| `tengu_ccr_mirror` | CCR mirror mode (outbound-only) |

## GrowthBook Config Keys

| Key | Purpose |
|-----|---------|
| `tengu_bridge_poll_interval_config` | Poll/heartbeat intervals (Zod-validated) |
| `tengu_bridge_repl_v2_config` | Env-less timing config |
| `tengu_bridge_min_version` | Minimum CLI version for v1 path |
| `tengu_bridge_initial_history_cap` | Max initial messages to replay |

## Environment Variables

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

## Compile-Time Feature Flags

| Flag | Purpose |
|------|---------|
| `BRIDGE_MODE` | Include bridge code in the build |
| `CCR_AUTO_CONNECT` | Auto-connect default behavior |
| `CCR_MIRROR` | Mirror mode support |
| `KAIROS` | Assistant-mode worker type differentiation |

## Import Dependencies Worth Noting

- `replBridgeTransport.ts` imports: `CCRClient` from `../cli/transports/ccrClient.js`, `HybridTransport` from `../cli/transports/HybridTransport.js`, `SSETransport` from `../cli/transports/SSETransport.js`
- `remoteBridgeCore.ts` uses: `import { feature } from 'bun:bundle'` for compile-time gating
- `trustedDevice.ts` uses `memoize` from `lodash-es` for keychain read caching
- `trustedDevice.ts` lazy-requires `../utils/auth.js` to avoid the ~1300-module transitive import from `sessionStorage.ts -> commands.ts`
- `pollConfig.ts` uses `z` from `zod/v4` for schema validation
- `bridgePointer.ts` uses `z` from `zod/v4` for pointer schema validation

## Telemetry Event Names

| Event | Where Emitted |
|-------|--------------|
| `tengu_bridge_repl_started` | Bridge session initialization |
| `tengu_bridge_repl_ws_connected` | Transport connected |
| `tengu_bridge_repl_ws_closed` | Transport closed |
| `tengu_bridge_session_started` | Child session spawned |
| `tengu_bridge_session_done` | Child session exited |
| `tengu_bridge_heartbeat_error` | Heartbeat failure |
| `tengu_bridge_token_refreshed` | Token proactively refreshed |
| `tengu_bridge_reconnected` | Recovered from connection loss |
| `tengu_bridge_poll_give_up` | Error budget exhausted |
| `tengu_bridge_fatal_error` | Non-retryable error |
| `tengu_bridge_repl_connect_timeout` | Connect deadline exceeded |
| `tengu_bridge_repl_teardown` | Bridge torn down |
| `tengu_bridge_message_received` | Inbound message received |
| `tengu_bridge_repl_reconnect_failed` | Reconnection attempt failed |

## Close Code Constants (as used in code)

| Code | Source | Meaning |
|------|--------|---------|
| `1000` | Standard WS | Clean close |
| `401` | HTTP status | JWT expired |
| `4090` | `replBridgeTransport.ts:220` | Epoch mismatch |
| `4091` | `replBridgeTransport.ts:365` | CCR initialize failed |
| `4092` | `replBridgeTransport.ts:313` | SSE reconnect budget exhausted |
