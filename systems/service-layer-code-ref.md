# Service Layer — Code Reference

## File Paths and Module Structure

**Root directory:** `src/services/`

| Module Directory | Key Files |
|-----------------|-----------|
| `src/services/api/` | `client.ts`, `claude.ts`, `withRetry.ts`, `promptCacheBreakDetection.ts`, `bootstrap.ts`, `errors.ts`, `errorUtils.ts`, `usage.ts`, `filesApi.ts`, `grove.ts`, `sessionIngress.ts`, `firstTokenDate.ts`, `overageCreditGrant.ts`, `ultrareviewQuota.ts` |
| `src/services/mcp/` | `types.ts`, `client.ts`, `config.ts`, `auth.ts`, `channelNotification.ts`, `channelPermissions.ts`, `elicitationHandler.ts`, `InProcessTransport.ts` |
| `src/services/oauth/` | `index.ts`, `crypto.ts` |
| `src/services/lsp/` | `LSPServerManager.ts`, `LSPDiagnosticRegistry.ts`, `passiveFeedback.ts` |
| `src/services/analytics/` | `index.ts`, `sink.ts`, `datadog.ts`, `firstPartyEventLogger.ts`, `firstPartyEventLoggingExporter.ts`, `growthbook.ts`, `metadata.ts`, `sinkKillswitch.ts` |
| `src/services/tools/` | `toolExecution.ts`, `toolHooks.ts` |
| `src/services/compact/` | `sessionMemoryCompact.ts`, `compact.ts`, `microCompact.ts`, `apiMicrocompact.ts`, `timeBasedMCConfig.ts`, `autoCompact.ts`, `grouping.ts`, `postCompactCleanup.ts`, `compactWarningHook.ts`, `compactWarningState.ts` |
| `src/services/extractMemories/` | `prompts.ts` (plus main module) |
| `src/services/SessionMemory/` | (main module) |
| `src/services/teamMemorySync/` | `secretScanner.ts`, `watcher.ts` (plus main module) |
| `src/services/settingsSync/` | (main module) |
| `src/services/remoteManagedSettings/` | `syncCache.ts`, `syncCacheState.ts`, `securityCheck.ts` |
| `src/services/policyLimits/` | (main module) |
| `src/services/plugins/` | `pluginOperations.ts`, `PluginInstallationManager.ts`, `pluginCliCommands.ts` |
| `src/services/autoDream/` | `consolidationLock.ts`, `consolidationPrompt.ts` |
| `src/services/MagicDocs/` | (main module) |
| `src/services/PromptSuggestion/` | `speculation.ts` (plus main module) |
| `src/services/AgentSummary/` | (main module) |
| `src/services/toolUseSummary/` | (main module) |
| `src/services/tips/` | `tipRegistry.ts`, `tipHistory.ts`, `tipScheduler.ts` |

**Standalone service files:**
- `src/services/awaySummary.ts`
- `src/services/claudeAiLimits.ts`
- `src/services/diagnosticTracking.ts`
- `src/services/internalLogging.ts`
- `src/services/notifier.ts`
- `src/services/preventSleep.ts`
- `src/services/rateLimitMessages.ts`
- `src/services/tokenEstimation.ts`
- `src/services/vcr.ts`
- `src/services/voice.ts`
- `src/services/voiceStreamSTT.ts`

## Function and Class Names

**API Service:**
- `getAnthropicClient()` -- factory function in `client.ts`
- `query()` -- core streaming interface in `claude.ts`
- `withRetry()` -- async generator retry wrapper in `withRetry.ts`
- `shouldRetry()` -- retry decision function in `withRetry.ts`
- `recordPromptState()` -- Phase 1 cache diagnostics in `promptCacheBreakDetection.ts`
- `checkResponseForCacheBreak()` -- Phase 2 cache diagnostics in `promptCacheBreakDetection.ts`
- `should1hCacheTTL()` -- extended caching eligibility in `claude.ts`
- `toolToAPISchema()` -- tool schema conversion in `claude.ts`
- `calculateUSDCost()` -- cost computation
- `addToTotalSessionCost()` -- session cost accumulator
- `checkAndRefreshOAuthTokenIfNeeded()` -- in `src/utils/auth.ts`
- `getRetryDelay()` -- reusable backoff formula in `withRetry.ts`

**MCP Service:**
- `McpServerConfigSchema` -- discriminated union Zod schema in `types.ts`
- `McpStdioServerConfig`, `McpSSEServerConfig`, `McpSSEIDEServerConfig`, `McpHTTPServerConfig`, `McpWebSocketServerConfig`, `McpWebSocketIDEServerConfig`, `McpSdkServerConfig`, `McpClaudeAIProxyServerConfig` -- transport config types in `types.ts`
- `MCPServerConnection` -- discriminated union of 5 connection states in `client.ts`
- `connectToMCPServer()` -- connection function in `client.ts`
- `useManageMCPConnections` -- React hook in `client.ts`
- `MCPTool` -- tool wrapper class
- `InProcessTransport` -- in `InProcessTransport.ts`, uses `queueMicrotask()`
- `ClaudeAuthProvider` -- OAuth provider in `auth.ts`
- `expandEnvVarsInString()` -- env var expansion in `config.ts`
- `gateChannelServer()` -- seven-gate access control in `channelPermissions.ts`

**OAuth Service:**
- `OAuthService` -- main class in `index.ts`
- `AuthCodeListener` -- local HTTP server for auth code capture
- `OAuthTokens` -- token type (fields: `accessToken`, `refreshToken`, `expiresAt`, `scopes`, `subscriptionType`, `rateLimitTier`, `tokenAccount`)
- `generateCodeVerifier()` -- 128-byte random base64url in `crypto.ts`
- `generateCodeChallenge()` -- SHA-256 base64url in `crypto.ts`
- `generateState()` -- 32-byte hex in `crypto.ts`

**LSP Service:**
- `createLSPServerManager()` -- factory function in `LSPServerManager.ts`
- `LSPServerManager` -- component with methods: `ensureServerStarted()`, `sendRequest()`, `openFile()`, `changeFile()`, `saveFile()`, `closeFile()`
- `LSPServerInstance` -- single server process wrapper
- `LSPClient` -- JSON-RPC communication
- `LSPDiagnosticRegistry` -- with methods: `registerPendingLSPDiagnostic()`, `checkForLSPDiagnostics()`, `getLSPDiagnosticAttachments()`

**Analytics Service:**
- `logEvent()` -- public API in `index.ts`
- `attachAnalyticsSink()` -- sink attachment in `sink.ts`
- `shouldSampleEvent()` -- per-event sampling
- `stripProtoFields()` -- PII key removal
- `trackDatadogEvent()` -- Datadog delivery
- `logEventTo1P()` -- first-party delivery
- `FirstPartyEventLoggingExporter` -- custom OpenTelemetry exporter in `firstPartyEventLoggingExporter.ts`
- `getEventMetadata()` -- event enrichment in `metadata.ts`
- `getFeatureValue_CACHED_MAY_BE_STALE<T>()` -- cached feature accessor in `growthbook.ts`
- `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` -- cached gate check in `growthbook.ts` (note: Statsig name is legacy; delegates to GrowthBook internally)
- `remoteEvalFeatureValues` -- in-memory Map in `growthbook.ts`

**Tool Execution Service:**
- `canUseTool()` -- permission callback in `toolExecution.ts`
- `logPermissionDecision()` -- analytics logging in `toolExecution.ts`
- `classifyToolError()` -- telemetry-safe error extraction in `toolExecution.ts`
- `runPreToolUseHooks` -- pre-tool hook runner in `toolHooks.ts`
- `runPostToolUseHooks` -- post-tool success hook runner in `toolHooks.ts`
- `runPostToolUseFailureHooks` -- post-tool failure hook runner in `toolHooks.ts`
- `startSpeculativeClassifierCheck()` -- background classifier in `toolExecution.ts`

**Compact Service:**
- `shouldAutoCompact()` -- decision logic in `autoCompact.ts`

**Team Memory Sync:**
- `SyncState` -- mutable state object
- API endpoints: `GET /api/claude_code/team_memory?repo={owner/repo}`, `GET ...&view=hashes`, `PUT /api/claude_code/team_memory?repo={owner/repo}`

**Settings Sync:**
- `uploadUserSettingsInBackground()` -- background upload function
- `SYNC_KEYS` -- sync key definitions

**Session Memory:**
- `registerPostSamplingHook()` -- hook registration
- `getSessionMemoryPath()` -- path resolver
- `buildSessionMemoryUpdatePrompt()` -- prompt builder

**Extract Memories:**
- `runForkedAgent()` -- forked agent executor (shared pattern)

## TypeScript Types and Schemas

- `BetaRawMessageStreamEvent` -- API stream event type (from Anthropic SDK)
- `Anthropic`, `AnthropicBedrock`, `AnthropicFoundry`, `AnthropicVertex` -- SDK classes
- `Tool` -- internal tool type
- `CacheSafeParams` -- structure for fork-safe prompt cache sharing
- `Attachment[]` -- diagnostic delivery format
- `Stream` -- progress streaming type for tool execution
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` -- type: `never`
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED` -- type: `never`
- `TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` -- type: `never`
- `TelemetrySafeError` -- error wrapper type
- `SystemAPIErrorMessage` -- heartbeat message type during persistent retry
- `ToolPermissionContext` -- permission evaluation context
- `AgentProgress` -- coordinator UI progress type

## Constants

- `API_MAX_MEDIA_PER_REQUEST = 100`
- `MAX_529_RETRIES = 3`
- `BASE_DELAY_MS = 500`
- Default `maxDelayMs = 32000`
- `MAX_TRACKED_SOURCES = 10` (prompt cache break tracking)
- `HOOK_TIMING_DISPLAY_THRESHOLD_MS = 500`
- Session Memory Compact: `minTokens: 10,000`, `minTextBlockMessages: 5`, `maxTokens: 40,000`
- Full Summarization output cap: `20,000 tokens`
- Post-compact restoration budgets: files `50K tokens`, active skills `25K tokens`
- Diagnostic volume limits: `10 per file`, `30 total per delivery`
- Diagnostic LRU cache: `500 files`
- Team memory upload batch cap: `200KB`
- Settings sync per-file cap: `500KB`
- Remote managed settings polling: `POLLING_INTERVAL_MS = 60 * 60 * 1000` (1 hour)
- Remote managed settings timeout: `30 seconds`
- Datadog batch size: `100`, flush interval: `15 seconds`
- Datadog endpoint: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- Datadog allowed events: `~35 event types` (in `DATADOG_ALLOWED_EVENTS`)
- Datadog tag fields: `TAG_FIELDS` list including `arch`, `clientType`, `errorType`, `http_status`, `model`, `platform`, `provider`, `subscriptionType`, `toolName`, `userType`, `kairosActive`
- GrowthBook refresh: `6 hours` (production), `20 minutes` (internal)
- GrowthBook disk cache location: `~/.claude/claude.json` key `cachedGrowthBookFeatures`
- Persistent retry: max backoff `5 minutes`, reset cap `6 hours`, heartbeat interval `30 seconds`
- Fast mode retry-after threshold: `20 seconds`
- Fast mode cooldown floor: `10 minutes`
- AutoDream consolidation lock stale threshold: `1 hour`
- Agent Summary interval: `30 seconds`
- Tool Use Summary: truncated at `300 chars` each, target `~30 characters` for summary

## Environment Variables

- `CLAUDE_CODE_USE_BEDROCK` -- Bedrock provider selection
- `CLAUDE_CODE_USE_FOUNDRY` -- Azure Foundry provider selection
- `CLAUDE_CODE_USE_VERTEX` -- Vertex AI provider selection
- `ANTHROPIC_CUSTOM_HEADERS` -- custom API headers
- `CLAUDE_CODE_UNATTENDED_RETRY` -- persistent retry mode

## Feature Gate Names

**Build-time:**
`EXTRACT_MEMORIES`, `KAIROS`, `TEAMMEM`, `REACTIVE_COMPACT`, `HISTORY_SNIP`, `CONTEXT_COLLAPSE`, `MCP_SKILLS`, `UPLOAD_USER_SETTINGS`, `UNATTENDED_RETRY`, `BASH_CLASSIFIER`, `TRANSCRIPT_CLASSIFIER`

**Runtime (GrowthBook):**
- `tengu_passport_quail` -- extract memories gate
- `tengu_chomp_inflection` -- prompt suggestions gate
- `tengu_log_datadog_events` -- Datadog analytics gate
- `tengu_event_sampling_config` -- per-event sampling configuration
- `tengu_enable_settings_sync_push` -- settings sync gate

## Configuration File Paths

- `~/.claude/settings.json` -- global user config
- `.claude/settings.json` -- project config
- `.claude/settings.local.json` -- local config
- `managed-mcp.json` -- enterprise MCP config
- `managed-settings.json` -- managed plugin source
- `.mcp.json` -- project-root MCP config (alternative path)
- `~/.claude/claude.json` -- GrowthBook disk cache

## API Endpoints

- `/api/claude_code/bootstrap` -- client bootstrap data
- `/api/claude_code/team_memory` -- team memory sync (GET/PUT)
- `https://http-intake.logs.us5.datadoghq.com/api/v2/logs` -- Datadog log intake

## Analytics Event Names

- `tengu_prompt_cache_break` -- prompt cache break diagnostic event

## Header Names

- `x-client-request-id` -- client request ID for timeout correlation
- `x-should-retry` -- server retry hint header
- `_PROTO_*` -- PII-tagged payload key prefix

## MagicDocs Detection Pattern

- Regex: `^#\s*MAGIC\s+DOC:\s*(.+)$`

## Secret Scanner Rule Sources

- Approximately 30 rules derived from the public gitleaks configuration, adapted to JavaScript regex. Targets: AWS tokens, GCP API keys, Anthropic API keys, GitHub PATs, Slack tokens, and others.

## Cross-Referenced Specifications

- Context Management Design Specification (referenced in Section 8.1) -- see [context-management.md](../core/context-management.md)
- KAIROS Design Specification (referenced in Section 12.1) -- see [kairos.md](kairos.md)
- Oversight spec (referenced in Section 7.2) -- see [oversight.md](../core/oversight.md)
