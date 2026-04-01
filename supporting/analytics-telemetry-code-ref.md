# Analytics & Telemetry — Code Reference

This section extracts all code-level details: specific file paths, function/class/variable names, line references, TypeScript types, import paths, and implementation specifics. It serves as a validation checklist for the design.

---

### Entry Point and Queue

- **File**: `src/services/analytics/index.ts`
- Zero imports from application modules
- Defines: event queue, sink interface, two marker types
- `attachAnalyticsSink()` drains queued events via `queueMicrotask()`
- Called from both preAction hook (subcommands) and `setup()` (default command)

---

### Datadog Integration

- **File**: `src/services/analytics/datadog.ts`
- HTTP Logs Intake API: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- Module-level `logBatch` array
- `MAX_BATCH_SIZE = 100`
- Flush timer: 15 seconds, `.unref()`
- Event allowlist (~40 names):
  - `tengu_init`, `tengu_started`, `tengu_exit`, `tengu_cancel`
  - `tengu_api_error`, `tengu_api_success`, `tengu_query_error`
  - `tengu_oauth_error/success`, `tengu_oauth_token_refresh_*` (8 events)
  - `tengu_tool_use_granted_in_prompt_*`, `tengu_tool_use_rejected_in_prompt`, `tengu_tool_use_error/success`
  - `tengu_brief_*`, `tengu_voice_*`, `tengu_flicker`, `tengu_model_fallback_triggered`
  - `chrome_bridge_*` (7 events)
  - `tengu_team_mem_sync_*` (4 events)
- Tag fields: `arch`, `clientType`, `errorType`, `http_status_range`, `http_status`, `kairosActive`, `model`, `platform`, `provider`, `skillMode`, `subscriptionType`, `toolName`, `userBucket`, `userType`, `version`, `versionBase`
- Tags prepended with `event:<name>`; snake_case conversion
- MCP tools: `mcp__*` normalized to `"mcp"`
- Model names: `getCanonicalName()` validated against `MODEL_COSTS`; unrecognized -> `"other"`
- Dev version truncation: `2.0.53-dev.20251124.t173302.sha526cc6a` -> `2.0.53-dev.20251124`
- User bucketing: SHA-256 hash of user ID modulo 30
- Third-party exclusion: Bedrock, Vertex, Foundry dropped
- Gate: `tengu_log_datadog_events` via `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()`
- Gate initialized during `initializeAnalyticsGates()` at startup
- Shutdown: `shutdownDatadog()` called from `gracefulShutdown()` before `process.exit()` (because `forceExit()` prevents `beforeExit` handler)

---

### First-Party Event Logging

- **Files**: `src/services/analytics/firstPartyEventLogger.ts`, `src/services/analytics/firstPartyEventLoggingExporter.ts`
- Functions: `logEventTo1P()`, `logEventTo1PAsync()`, `reinitialize1PEventLoggingIfConfigChanged()`
- OTel classes: `LoggerProvider`, `BatchLogRecordProcessor`
- Exporter class: `FirstPartyEventLoggingExporter`
- Protobuf types: `ClaudeCodeInternalEvent` (`events_mono/claude_code/v1/`), `GrowthbookExperimentEvent` (`events_mono/growthbook/v1/`)
- API endpoint: `POST /api/event_logging/batch`
- Base URL: `https://api.anthropic.com`
- Failed events path: `~/.claude/telemetry/1p_failed_events.{sessionId}.{batchUuid}.json`
- Module-level `BATCH_UUID`
- GrowthBook config key: `tengu_1p_event_batch_config`
  - `scheduledDelayMillis` (default 10,000)
  - `maxExportBatchSize` (default 200)
  - `maxQueueSize` (default 8,192)
  - `skipAuth` (default false)
  - `maxAttempts` (default 8)
  - `path` (default `/api/event_logging/batch`)
  - `baseUrl` (default `https://api.anthropic.com`)
- Backoff: `delay = baseBackoffDelayMs * attempts^2`, cap at `maxBackoffDelayMs` (30s)
- Event sampling config key: `tengu_event_sampling_config`
- Sampling logic in: `src/services/analytics/sink.ts` (referenced at line 57-85 in firstPartyEventLogger.ts per source comment)
- `is1PEventLoggingEnabled()` -- killswitch check avoided here to prevent circular imports with `growthbook.ts`

---

### GrowthBook

- **File**: `src/services/analytics/growthbook.ts`
- Constructor: `new GrowthBook({ remoteEval: true, cacheKeyAttributes: ['id', 'organizationUUID'] })`
- Init timeout: `5000` ms
- `getUserAttributes()` returns: `id`, `sessionId`, `deviceID`, `platform`, `organizationUUID`, `accountUUID`, `userType`, `subscriptionType`, `rateLimitTier`, `firstTokenTime`, `email`, `appVersion`, `apiBaseUrlHost`
- `processRemoteEvalPayload()` -- fixes `value` -> `defaultValue` API format workaround; populates `remoteEvalFeatureValues` Map
- `syncRemoteEvalToDisk()` -> `~/.claude/claude.json`: `cachedGrowthBookFeatures`
- `refreshed.emit()` -- event emitter for subscribers
- Override env var: `CLAUDE_INTERNAL_FC_OVERRIDES` (JSON, ant-only)
- Config overrides: `getGlobalConfig().growthBookOverrides` (per-feature, ant-only)
- Accessor functions:
  - `getFeatureValue_CACHED_MAY_BE_STALE<T>()`
  - `getDynamicConfig_CACHED_MAY_BE_STALE<T>()`
  - `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()`
  - `checkSecurityRestrictionGate()`
  - `checkGate_CACHED_OR_BLOCKING()`
  - `getDynamicConfig_BLOCKS_ON_INIT<T>()`
  - `getFeatureValue_DEPRECATED<T>()`
- `_CACHED_WITH_REFRESH` variant: deprecated, delegates to `_CACHED_MAY_BE_STALE`
- Refresh intervals: 6 hours (external), 20 minutes (ant)
- `client.refreshFeatures()` -> `processRemoteEvalPayload()` -> sync + emit
- Exposure tracking: `logGrowthBookExperimentTo1P()`, `loggedExposures` Set, `pendingExposures`
- Auth change: `refreshGrowthBookAfterAuthChange()` -> `resetGrowthBook()` -> `refreshed.emit()` -> `initializeGrowthBook()` -> tracks `reinitializingPromise`

---

### Sink Killswitch

- **File**: `src/services/analytics/sinkKillswitch.ts`
- Config key: `tengu_frond_boric` (deliberately mangled name)
- TypeScript type:
  ```typescript
  type SinkName = 'datadog' | 'firstParty'
  // Config shape: { datadog?: boolean, firstParty?: boolean }
  // true = killed, false/absent = alive
  ```
- Function: `isSinkKilled(sink: SinkName): boolean`
- Checked at dispatch sites, not in `is1PEventLoggingEnabled()`

---

### Metadata Collection

- **File**: `src/services/analytics/metadata.ts`
- Main function: `getEventMetadata({ model?, betas? })`
- Fields sourced from:
  - `getMainLoopModel()` -- for model
  - Bootstrap state -- for sessionId, isInteractive, clientType
  - `USER_TYPE` env var -- for userType
  - OAuth -- for subscriptionType
  - SHA-256 hash of repo remote URL, first 16 chars -- `rh` field
- `buildEnvContext()` -- memoized, runs `Promise.all()` over 4 async operations
- `setKairosActive()` called in `main.tsx` -- deliberately outside memoized context
- Process metrics: `process.memoryUsage()`, `process.cpuUsage()`
- Agent identification via `AsyncLocalStorage` context
- `to1PEventFormat()` -- camelCase to snake_case, type-checked against proto-generated `EnvironmentMetadata` type
- Comment at `src/services/analytics/metadata.ts:813-819` documents compile-error guarantee
- `sanitizeToolNameForAnalytics()` -- MCP names -> `'mcp_tool'`; exceptions for:
  - Entrypoint `local-agent`
  - MCP server type `claudeai-proxy`
  - Official MCP registry URL match
  - Built-in servers (feature-gated set)
- `getFileExtensionForAnalytics()` -- extensions >10 chars -> `'other'`
- `getFileExtensionsFromBashCommand()` -- splits on compound operators, identifies file-command tokens (`rm`, `mv`, `cp`, `grep`, etc.)

---

### Permission Decision Audit

- **File**: `src/hooks/toolPermission/permissionLogging.ts`
- Central function: `logPermissionDecision()`
- In-session store: `toolUseContext.toolDecisions` Map (lines 221-228)
- Tool decision storage also referenced at `src/Tool.ts:258-265`
- `coreSchemas` is the SDK-facing type export from `src/entrypoints/sdk/coreSchemas.ts` that exposes hook event types and permission decision metadata to SDK consumers
- Event names:
  - `tengu_tool_use_granted_in_config`
  - `tengu_tool_use_granted_by_classifier`
  - `tengu_tool_use_granted_in_prompt_permanent`
  - `tengu_tool_use_granted_in_prompt_temporary`
  - `tengu_tool_use_granted_by_permission_hook`
  - `tengu_tool_use_denied_in_config`
  - `tengu_tool_use_rejected_in_prompt`
  - `tengu_auto_mode_decision`
  - `tengu_auto_mode_denial_limit_exceeded`
- Types: `PermissionApprovalSource`, `PermissionRejectionSource` (discriminated unions)
- Code-edit tools triggering language counter: `Edit`, `Write`, `NotebookEdit`
- Language derived via `getLanguageName()` from file path

---

### OpenTelemetry Events

- **File**: `src/utils/telemetry/events.ts`
- Logger obtained from `getEventLogger()` in bootstrap state
- Event body format: `claude_code.${eventName}`
- Attributes include: `getTelemetryAttributes()`, `event.name`, `event.timestamp`, `event.sequence`, `prompt.id`, `workspace.host_paths`
- `logOTelEvent()` function
- Redaction: `redactIfDisabled()` returns `'<REDACTED>'` when `OTEL_LOG_USER_PROMPTS` is not set
- Tool input serialization: `extractToolInputForTelemetry()` when `OTEL_LOG_TOOL_DETAILS` is enabled
  - String threshold: 512 chars; truncated to 128 chars + `[N chars]`
  - Max JSON: 4 KB
  - Max collection items: 20
  - Max nesting: 2
  - Keys starting with `_` filtered out

---

### PII Marker Types

- **File**: `src/services/analytics/index.ts`
- Types:
  ```typescript
  type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never
  type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never
  ```
- `LogEventMetadata` restricts values to `boolean | number | undefined`
- `_PROTO_*` field prefix convention
- Known proto keys: `_PROTO_skill_name`, `_PROTO_plugin_name`, `_PROTO_marketplace_name`
- `stripProtoFields()` -- applied in `sink.ts` before Datadog dispatch, and defensively in the 1P exporter

---

### Cost Tracking

- **File**: `src/cost-tracker.ts`
- `addToTotalSessionCost()` -- accumulates in bootstrap state
- `calculateUSDCost(model, usage)` -- uses `MODEL_COSTS` table
- OTel counters: `getCostCounter()`, `getTokenCounter()` with model and speed attributes
- `saveCurrentSessionCosts()` -- persists to project config
- Persisted fields: `lastCost`, `lastAPIDuration`, `lastAPIDurationWithoutRetries`, `lastToolDuration`, `lastDuration`, `lastLinesAdded`, `lastLinesRemoved`, `lastTotalInputTokens`, `lastTotalOutputTokens`, `lastTotalCacheReadInputTokens`, `lastTotalCacheCreationInputTokens`, `lastTotalWebSearchRequests`, `lastModelUsage`, `lastFpsAverage`, `lastFpsLow1Pct`, `lastSessionId`
- Token counter attributes: model name, token type (`input`, `output`, `cacheRead`, `cacheCreation`), speed (`fast`)
