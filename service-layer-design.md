# Claude Code: Service Layer — Design Specification

This document analyzes the service layer architecture of Claude Code — Anthropic's
agentic CLI tool — covering the ~21 backend service modules that power API
communication, tool execution, context management, synchronization, analytics,
and progressive feature rollout.

## Table of Contents

- [1. Overview](#1-overview)
- [2. API Service](#2-api-service)
- [3. MCP Service](#3-mcp-service)
- [4. Authentication Service](#4-authentication-service)
- [5. LSP Service](#5-lsp-service)
- [6. Analytics Service](#6-analytics-service)
- [7. Tool Execution Service](#7-tool-execution-service)
- [8. Context Services](#8-context-services)
- [9. Sync Services](#9-sync-services)
- [10. Configuration Services](#10-configuration-services)
- [11. Plugin Service](#11-plugin-service)
- [12. Minor Services](#12-minor-services)
- [13. Cross-Cutting Patterns](#13-cross-cutting-patterns)
- [14. Design Principles](#14-design-principles)

---

## 1. Overview

The service layer is the backend substrate of Claude Code. It sits between the
query loop / UI layer above and the raw APIs / OS below, providing the
abstractions that make an agentic CLI possible: reliable API communication,
multi-protocol tool server management, token-aware context compression,
cross-session memory synchronization, enterprise policy enforcement, and
telemetry routing.

The services are organized into `src/services/`, spanning approximately 130
source files across 21 module directories plus several standalone modules.
Architecturally, they split into six functional groups:

```
┌──────────────────────────────────────────────────────────────────┐
│                     Query Loop / UI Layer                        │
├──────────────────────────────────────────────────────────────────┤
│  Communication         │  Execution          │  Intelligence     │
│  ─────────────         │  ─────────          │  ────────────     │
│  api/                  │  tools/             │  compact/         │
│  mcp/                  │  plugins/           │  extractMemories/ │
│  oauth/                │  lsp/               │  SessionMemory/   │
│  analytics/            │                     │  autoDream/       │
├──────────────────────────────────────────────────────────────────┤
│  Synchronization       │  Configuration      │  Enhancement      │
│  ───────────────       │  ─────────────      │  ───────────      │
│  teamMemorySync/       │  remoteManagedSettings/│  MagicDocs/    │
│  settingsSync/         │  policyLimits/      │  PromptSuggestion/│
│                        │                     │  AgentSummary/    │
│                        │                     │  toolUseSummary/  │
│                        │                     │  tips/            │
└──────────────────────────────────────────────────────────────────┘
```

Every service follows a common contract: **fail-open by default** (network
failures never block the user), **background execution** where possible, and
**progressive enhancement** via feature gates. The only exception is the API
service itself, which must succeed for any work to happen.

---

## 2. API Service

> **Source:** `src/services/api/`

The API service is the most critical module in the service layer — every model
interaction flows through it. It wraps the Anthropic SDK with retry logic,
multi-provider support, streaming management, prompt cache monitoring, and
cost tracking.

### 2.1 Client Construction

> **Source:** `src/services/api/client.ts`

`getAnthropicClient()` is the factory function that constructs an Anthropic SDK
client. It supports **five provider backends**, selected by environment variables:

| Provider | Env Var | SDK Class | Auth Mechanism |
|----------|---------|-----------|----------------|
| First-party Anthropic | (default) | `Anthropic` | API key or OAuth `authToken` |
| AWS Bedrock | `CLAUDE_CODE_USE_BEDROCK` | `AnthropicBedrock` | AWS credentials / bearer token |
| Azure Foundry | `CLAUDE_CODE_USE_FOUNDRY` | `AnthropicFoundry` | API key or Azure AD token provider |
| Google Vertex AI | `CLAUDE_CODE_USE_VERTEX` | `AnthropicVertex` | GCP `GoogleAuth` credentials |
| Claude.ai (OAuth) | `isClaudeAISubscriber()` | `Anthropic` | OAuth access token via `authToken` |

Before client creation, the factory performs OAuth token refresh
(`checkAndRefreshOAuthTokenIfNeeded()`), configures custom headers from
`ANTHROPIC_CUSTOM_HEADERS`, injects a client request ID (`x-client-request-id`)
for timeout correlation, and applies proxy configuration.

The factory is called on **every retry attempt** when the previous error was an
authentication failure (401, 403 token-revoked, Bedrock/Vertex credential
errors, or stale ECONNRESET/EPIPE connections). This ensures fresh credentials
on each attempt.

### 2.2 Streaming and the Main Query Function

> **Source:** `src/services/api/claude.ts`

The `query()` function is the core streaming interface. It accepts system
prompts, messages, tools, thinking configuration, and model parameters, then
streams `BetaRawMessageStreamEvent` objects back to the caller. Key behaviors:

- **Beta header negotiation**: Dynamically assembles beta headers based on model
  capabilities (thinking, structured outputs, effort, fast mode, AFK mode,
  prompt caching scope, context management, tool search, advisor)
- **Prompt cache management**: Places `cache_control` markers on system prompt
  blocks with configurable scope (global vs. organization) and TTL (5-minute vs.
  1-hour). The `should1hCacheTTL()` function determines extended caching
  eligibility using an allowlist and overage detection
- **Media limiting**: Enforces `API_MAX_MEDIA_PER_REQUEST` (100) by stripping
  excess images/documents from older messages
- **Tool schema conversion**: Transforms internal `Tool` objects to API-format
  schemas via `toolToAPISchema()`, with special handling for deferred tools
  (tool search), MCP instructions delta, and advisor tools
- **Cost tracking**: After each response, computes USD cost via
  `calculateUSDCost()` and adds to session total via `addToTotalSessionCost()`

The function also maintains the `CacheSafeParams` structure used by forked
agents (compaction, memory extraction, agent summarization) to share the
parent's prompt cache.

### 2.3 Retry Logic

> **Source:** `src/services/api/withRetry.ts`

`withRetry()` is an async generator that wraps any API operation with
sophisticated retry behavior. It is not a simple exponential backoff — it
implements a multi-strategy retry engine:

**Retry decisions** (`shouldRetry()`):
- Mock errors (from `/mock-limits`): never retry
- Persistent mode (unattended sessions): always retry 429/529
- CCR mode (remote sessions): retry 401/403 (transient infra blips)
- Overloaded errors (`"type":"overloaded_error"` in message): always retry
- Context overflow errors: retry with adjusted `maxTokensOverride`
- `x-should-retry` header: obey unless Claude.ai subscriber (non-Enterprise)
- Connection errors, 408, 409, 429, 5xx: retry with conditions

**Fast mode fallback**: When fast mode is active and a 429/529 arrives:
1. If `retry-after` < 20 seconds: wait and retry with fast mode still active
   (preserves prompt cache — same model name)
2. If `retry-after` >= 20 seconds or unknown: trigger cooldown (switches to
   standard speed model) with a minimum 10-minute floor
3. If overage disabled: permanently disable fast mode

**529 (overloaded) handling**:
- Background query sources (summaries, suggestions, classifiers) bail
  immediately — no retry amplification during capacity cascades
- Foreground sources retry up to `MAX_529_RETRIES = 3`
- After 3 consecutive 529s: trigger fallback model if configured, else surface
  custom error message

**Persistent retry mode** (`CLAUDE_CODE_UNATTENDED_RETRY`):
- For unattended sessions, retries 429/529 indefinitely
- Uses higher backoff (up to 5 minutes) with a 6-hour reset cap
- Chunks long sleeps into 30-second heartbeat intervals, yielding
  `SystemAPIErrorMessage` objects so the host environment does not mark the
  session idle

**Backoff formula**: `min(BASE_DELAY_MS * 2^(attempt-1), maxDelayMs) + random(0, 0.25 * baseDelay)` where `BASE_DELAY_MS = 500` and default `maxDelayMs = 32000`.

### 2.4 Prompt Cache Break Detection

> **Source:** `src/services/api/promptCacheBreakDetection.ts`

A two-phase diagnostic system that detects and explains unexpected drops in
prompt cache utilization:

**Phase 1 (pre-call)** — `recordPromptState()`:
- Hashes system prompt, tool schemas, cache control markers, beta headers,
  model name, fast mode, effort value, extra body params
- Computes per-tool schema hashes for pinpointing which tool's description
  changed
- Stores pending changes (system prompt changed, tools added/removed, model
  changed, etc.) for phase 2

**Phase 2 (post-call)** — `checkResponseForCacheBreak()`:
- Compares current `cacheReadTokens` against previous value
- Triggers if cache read dropped >5% AND absolute drop > 2,000 tokens
- Classifies the break: client-side change (with specific reason), possible
  5-minute TTL expiry, possible 1-hour TTL expiry, or "likely server-side
  (prompt unchanged, <5min gap)"
- Logs `tengu_prompt_cache_break` event with ~30 diagnostic fields
- Writes a unified diff file for debugging (ant-only)

The system tracks up to `MAX_TRACKED_SOURCES = 10` independent query sources,
with special handling: compact shares tracking state with `repl_main_thread`
(same server-side cache key), and subagents are isolated by `agentId`.

### 2.5 Other API Modules

| Module | Role |
|--------|------|
| `bootstrap.ts` | Fetches client bootstrap data (additional model options) from `/api/claude_code/bootstrap` |
| `errors.ts` | Canonical error messages (repeated 529, overloaded) |
| `errorUtils.ts` | Extracts connection error details (ECONNRESET, EPIPE codes) |
| `usage.ts` | Usage tracking and reporting |
| `filesApi.ts` | File upload/download via API |
| `grove.ts` | Grove (internal tool platform) integration |
| `sessionIngress.ts` | Session ingress authentication for remote sessions |
| `promptCacheBreakDetection.ts` | Cache break diagnostics (see above) |
| `firstTokenDate.ts` | Tracks first-ever API token date for user segmentation |
| `overageCreditGrant.ts` | Handles overage credit grants for subscribers |
| `ultrareviewQuota.ts` | Manages ultra-review (extended thinking) quota |

---

## 3. MCP Service

> **Source:** `src/services/mcp/`

The Model Context Protocol (MCP) service manages connections to external tool
servers. It is the most complex service by file count (~20 modules), reflecting
the protocol's breadth: multiple transport types, OAuth authentication for
remote servers, configuration from 7+ sources, channel notifications, and
dynamic tool registration.

### 3.1 Transport Types

> **Source:** `src/services/mcp/types.ts`

MCP supports **eight transport configurations**, resolved from a discriminated
union (`McpServerConfigSchema`):

| Transport | Config Type | Use Case |
|-----------|------------|----------|
| `stdio` | `McpStdioServerConfig` | Local subprocess (command + args + env) |
| `sse` | `McpSSEServerConfig` | Server-Sent Events over HTTP(S) |
| `sse-ide` | `McpSSEIDEServerConfig` | IDE extension (VS Code / JetBrains) |
| `http` | `McpHTTPServerConfig` | Streamable HTTP (bidirectional) |
| `ws` | `McpWebSocketServerConfig` | WebSocket (bidirectional) |
| `ws-ide` | `McpWebSocketIDEServerConfig` | IDE WebSocket transport |
| `sdk` | `McpSdkServerConfig` | SDK control protocol |
| `claudeai-proxy` | `McpClaudeAIProxyServerConfig` | Claude.ai proxy servers |

Additionally, `InProcessTransport` (`src/services/mcp/InProcessTransport.ts`)
provides a linked transport pair for running MCP servers in-process without
spawning a subprocess. Messages sent on one side are delivered to the other's
`onmessage` handler via `queueMicrotask()` to avoid stack depth issues.

### 3.2 Connection Lifecycle

> **Source:** `src/services/mcp/client.ts`

Each server progresses through a state machine:

```
pending → connected
        → failed (with error message)
        → needs-auth (OAuth required)
        → disabled (admin-disabled)
```

The `MCPServerConnection` type is a discriminated union of these five states.
Connected servers carry a `Client` instance (from `@modelcontextprotocol/sdk`),
capabilities, server info, instructions, and a cleanup function.

Connection management happens through:
- `connectToMCPServer()`: Creates transport, initializes MCP client, negotiates
  capabilities, registers tools/resources
- `useManageMCPConnections` (React hook): Manages the full connection pool
  lifecycle, including reconnection with exponential backoff
- Tool registration: Each MCP tool is wrapped in an `MCPTool` instance that
  handles schema normalization, result truncation, binary content persistence,
  and image resizing

### 3.3 Configuration Sources

> **Source:** `src/services/mcp/config.ts`

MCP server configs are aggregated from seven scoped sources:

| Scope | Source | Priority |
|-------|--------|----------|
| `enterprise` | Managed MCP file (`managed-mcp.json`) | Highest |
| `managed` | Remote managed settings | High |
| `user` | Global user config (`~/.claude/settings.json`) | Medium |
| `project` | Project config (`.claude/settings.json`) | Medium |
| `local` | Local config (`.claude/settings.local.json`) | Medium |
| `dynamic` | Plugin-provided servers | Variable |
| `claudeai` | Claude.ai proxy configs | Variable |

The `.mcp.json` file in the project root provides an alternative per-project
configuration path. Environment variables in server configs support `${VAR}`
expansion via `expandEnvVarsInString()`.

### 3.4 OAuth for MCP Servers

> **Source:** `src/services/mcp/auth.ts`

Remote MCP servers can require OAuth 2.0 authentication. The `ClaudeAuthProvider`
implements the MCP SDK's auth provider interface, supporting:
- Authorization Server Metadata discovery (`authServerMetadataUrl`)
- Dynamic client registration
- PKCE code flow
- Token refresh
- Secure token storage via macOS Keychain / platform secure storage
- Cross-App Access (XAA/SEP-990) for enterprise identity provider federation

### 3.5 Channel Notifications

> **Source:** `src/services/mcp/channelNotification.ts`, `channelPermissions.ts`

For KAIROS (assistant mode), MCP servers can act as communication channels,
relaying messages between the agent and external platforms (Slack, Discord, etc.)
via `notifications/claude/channel`. A seven-gate access control pipeline
(`gateChannelServer()`) verifies capability, runtime gate, OAuth, org policy,
session list, plugin marketplace, and allowlist before activating a channel.

### 3.6 Elicitation

> **Source:** `src/services/mcp/elicitationHandler.ts`

MCP servers can request information from users via the elicitation protocol.
The handler manages the lifecycle of elicitation requests, including hook
execution, result validation, and timeout management.

---

## 4. Authentication Service

> **Source:** `src/services/oauth/`

### 4.1 OAuth 2.0 Flow

> **Source:** `src/services/oauth/index.ts`

The `OAuthService` class implements OAuth 2.0 Authorization Code Flow with PKCE,
supporting dual code acquisition paths:

**Automatic flow**: Opens the user's browser to the authorization URL with a
localhost redirect URI. An `AuthCodeListener` starts a local HTTP server on a
random port to capture the authorization code from the redirect.

**Manual flow**: For environments without browser access, the user copies a
URL, authenticates in any browser, and pastes the resulting authorization code
back into the CLI.

Both paths race concurrently — whichever delivers the code first wins.

### 4.2 Token Management

The flow produces `OAuthTokens` containing:
- `accessToken` / `refreshToken` — standard OAuth tokens
- `expiresAt` — absolute expiration timestamp
- `scopes` — parsed scope set
- `subscriptionType` / `rateLimitTier` — fetched from profile API
- `tokenAccount` — account UUID, email, organization UUID

Token refresh is handled externally by `checkAndRefreshOAuthTokenIfNeeded()`
(in `src/utils/auth.ts`), which is called before every API client construction.
The retry logic in `withRetry.ts` triggers refresh on 401 and 403
"token revoked" errors.

### 4.3 Cryptographic Primitives

> **Source:** `src/services/oauth/crypto.ts`

- `generateCodeVerifier()`: Random 128-byte base64url string for PKCE
- `generateCodeChallenge()`: SHA-256 hash of verifier, base64url-encoded
- `generateState()`: Random 32-byte hex string for CSRF protection

---

## 5. LSP Service

> **Source:** `src/services/lsp/`

The Language Server Protocol service provides real-time code intelligence by
managing external LSP servers, routing requests based on file types, and
delivering diagnostics as conversation attachments.

### 5.1 Architecture

The service uses a factory-function pattern with closures for state encapsulation:

| Component | Role |
|-----------|------|
| `LSPServerManager` | Routes requests by file extension, manages server lifecycle |
| `LSPServerInstance` | Wraps a single LSP server process with JSON-RPC protocol |
| `LSPClient` | Low-level JSON-RPC communication over stdio |
| `LSPDiagnosticRegistry` | Stores async diagnostics for conversation delivery |

### 5.2 Server Manager

> **Source:** `src/services/lsp/LSPServerManager.ts`

`createLSPServerManager()` loads LSP server configurations from settings,
builds an extension-to-server mapping, and provides:
- `ensureServerStarted(filePath)`: Lazy-starts the appropriate server
- `sendRequest(filePath, method, params)`: Routes LSP requests by extension
- `openFile()` / `changeFile()` / `saveFile()` / `closeFile()`: Synchronizes
  file state with LSP servers via `textDocument/didOpen`, `didChange`, `didSave`,
  `didClose` notifications

### 5.3 Diagnostic Registry

> **Source:** `src/services/lsp/LSPDiagnosticRegistry.ts`

When LSP servers send `textDocument/publishDiagnostics` notifications, the
registry stores them for delivery as conversation attachments:

1. `registerPendingLSPDiagnostic()` stores diagnostics with timestamp
2. `checkForLSPDiagnostics()` retrieves pending (undelivered) diagnostics
3. `getLSPDiagnosticAttachments()` converts to `Attachment[]`
4. The attachment system delivers them into the next query turn

Volume limiting: max 10 diagnostics per file, 30 total per delivery.
Cross-turn deduplication: LRU cache (500 files) of delivered diagnostic keys
prevents re-delivery.

### 5.4 Passive Feedback

> **Source:** `src/services/lsp/passiveFeedback.ts`

After file edits, the LSP service can provide passive feedback to the model
by checking for new diagnostics introduced by the edit. This enables the
model to self-correct without explicit user intervention.

---

## 6. Analytics Service

> **Source:** `src/services/analytics/`

The analytics service provides telemetry routing with strong PII guarantees,
event sampling, feature flag management, and multi-sink delivery.

### 6.1 Event Logging Architecture

> **Source:** `src/services/analytics/index.ts`, `sink.ts`

The analytics pipeline follows a **queue-then-drain** pattern:

```
logEvent(name, metadata)          ← Public API (zero dependencies)
    │
    ▼ (pre-init: queue)
eventQueue[]                      ← Buffered until sink attached
    │
    ▼ attachAnalyticsSink()       ← Called during app startup
    │
    ├──→ shouldSampleEvent()      ← Per-event sampling via GrowthBook config
    │
    ├──→ Datadog (if gate enabled)
    │      └── stripProtoFields() ← Remove PII-tagged keys
    │      └── trackDatadogEvent()
    │
    └──→ 1P Event Logger (OpenTelemetry)
           └── logEventTo1P()     ← Full payload including _PROTO_* keys
```

**PII handling** is enforced through two marker types:
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` (type: `never`) —
  forces explicit cast, documenting that the developer verified the value is safe
  for general-access storage
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED` (type: `never`) — for values
  routed to privileged PII-tagged proto columns via `_PROTO_*` payload keys

The `stripProtoFields()` function removes all `_PROTO_*` keys before sending to
general-access backends (Datadog). Only the 1P exporter sees them and routes
them to privileged BQ columns.

### 6.2 Datadog Integration

> **Source:** `src/services/analytics/datadog.ts`

Datadog receives a curated subset of events (defined in `DATADOG_ALLOWED_EVENTS`,
approximately 35 event types). Events are batched (max 100) and flushed every
15 seconds to `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`.

Selected metadata fields are promoted to Datadog tags (`TAG_FIELDS`) for
dashboard filtering: `arch`, `clientType`, `errorType`, `http_status`, `model`,
`platform`, `provider`, `subscriptionType`, `toolName`, `userType`,
`kairosActive`, etc.

### 6.3 First-Party Event Logger

> **Source:** `src/services/analytics/firstPartyEventLogger.ts`,
> `firstPartyEventLoggingExporter.ts`

The first-party (1P) logger uses OpenTelemetry's `LoggerProvider` with a
`BatchLogRecordProcessor` and a custom `FirstPartyEventLoggingExporter` that
POSTs events to Anthropic's event logging endpoint.

Event sampling is configurable per-event via the `tengu_event_sampling_config`
GrowthBook dynamic config. Events not in the config are logged at 100% rate.
When an event is sampled, its `sample_rate` is attached to the metadata for
downstream bias correction.

### 6.4 GrowthBook Feature Flags

> **Source:** `src/services/analytics/growthbook.ts`

GrowthBook provides runtime feature flags and experiment targeting. The
integration supports:

- **User attributes** for targeting: id, sessionId, platform, organizationUUID,
  subscriptionType, rateLimitTier, email, appVersion, GitHub Actions metadata
- **Remote evaluation**: Server-side feature computation with client-side caching
- **Dual-layer caching**: In-memory (`remoteEvalFeatureValues` Map) and on-disk
  (`~/.claude/claude.json` → `cachedGrowthBookFeatures`)
- **Exposure deduplication**: `loggedExposures` Set prevents duplicate exposure
  events in hot paths
- **Refresh intervals**: 6 hours (production), 20 minutes (internal)
- **Refresh listeners**: Systems that bake feature values into long-lived
  objects can subscribe to rebuild when config changes

Two accessor patterns:
- `getFeatureValue_CACHED_MAY_BE_STALE<T>(feature, default)`: Pure read from
  memory or disk cache — used in performance-critical and synchronous contexts
- `checkStatsigFeatureGate_CACHED_MAY_BE_STALE(gate)`: Boolean gate check

### 6.5 Event Metadata

> **Source:** `src/services/analytics/metadata.ts`

`getEventMetadata()` enriches every event with ~50 fields: platform, model,
provider, user type, subscription type, permission mode, session ID, KAIROS
state, tool details, and more. Metadata types are restricted to
`boolean | number` to prevent accidental code/filepath logging.

### 6.6 Sink Killswitch

> **Source:** `src/services/analytics/sinkKillswitch.ts`

A safety mechanism that allows individual analytics sinks to be disabled via
GrowthBook gates, providing immediate rollback capability if a sink causes
issues.

---

## 7. Tool Execution Service

> **Source:** `src/services/tools/`

### 7.1 Execution Engine

> **Source:** `src/services/tools/toolExecution.ts`

The tool execution service is the bridge between the model's tool-use requests
and actual tool implementations. It handles:

**Validation pipeline**:
1. Zod schema validation against the tool's input schema
2. Per-tool `validateInput()` for semantic checks (file existence, encoding,
   size limits, UNC path detection)
3. Errors wrapped in `<tool_use_error>` tags and returned to the model

**Permission pipeline**:
1. `canUseTool()` callback evaluates the full permission stack (rules, tool
   checks, scanner, mode transformation, hooks, classifier)
2. Permission decision logged via `logPermissionDecision()` to analytics,
   OpenTelemetry, and in-session store
3. Denied tools return error messages with optional memory correction hints

**Execution pipeline**:
1. Pre-tool hooks run (`runPreToolUseHooks`)
2. Tool `call()` method executes with progress streaming via `Stream`
3. Post-tool hooks run (`runPostToolUseHooks` or `runPostToolUseFailureHooks`)
4. Tool result processed: large outputs persisted to disk, binary content
   saved as file references, file state cache updated

**Error classification** (`classifyToolError()`): Extracts telemetry-safe
error information from minified builds. Uses `TelemetrySafeError` messages,
Node.js errno codes, or known error type names — never mangled identifiers.

### 7.2 Tool Hooks

> **Source:** `src/services/tools/toolHooks.ts`

Hook execution wraps each tool call:

| Hook Phase | Purpose |
|-----------|---------|
| Pre-tool | Can modify input, add context, block execution |
| Post-tool (success) | Can modify output, trigger side effects |
| Post-tool (failure) | Error handling, cleanup |
| Permission | Allow/deny/defer decisions (see Oversight spec) |

Hook timing is tracked: a summary is shown inline when total hook duration
exceeds `HOOK_TIMING_DISPLAY_THRESHOLD_MS = 500ms`. A debug warning fires when
any phase blocks for over 2 seconds.

### 7.3 Speculative Classifier

The tool execution service integrates with the speculative classifier check for
bash commands. `startSpeculativeClassifierCheck()` begins evaluating the next
likely bash command in the background while the current tool is still running,
reducing perceived latency for the permission prompt.

---

## 8. Context Services

These services manage the model's context window — compressing, extracting, and
maintaining state across turns and sessions.

### 8.1 Compact Service (4 Strategies)

> **Source:** `src/services/compact/`

The compact service is documented in depth in the Context Management Design
Specification. In summary, it provides four compaction strategies:

**Strategy 1: Session Memory Compact** (`sessionMemoryCompact.ts`):
Lightweight, no API call. Keeps the most recent N messages
(configurable: `minTokens: 10,000`, `minTextBlockMessages: 5`,
`maxTokens: 40,000`). Preserves tool_use/tool_result pairs.

**Strategy 2: Full Summarization** (`compact.ts`):
Sends old messages to the LLM. Pre-processes by stripping images/documents
and re-injectable attachments. Output capped at 20,000 tokens. Includes
post-compact restoration of recently-accessed files (50K tokens), active
skills (25K tokens), plan files, and background agent status.

**Strategy 3: Reactive Compact** (in `query.ts`):
Emergency compaction on "prompt too long" API errors. Last-resort strategy
when auto-compact's proactive threshold miscalculated.

**Strategy 4: Micro Compact** (`microCompact.ts`, `apiMicrocompact.ts`):
Finer-grained approach using the API's `cache_edits` mechanism to delete
specific segments. Time-based configuration via `timeBasedMCConfig.ts`.

Additional modules:
- `autoCompact.ts`: Decision logic (`shouldAutoCompact()`), threshold
  calculation, circuit breaker (3 consecutive failures)
- `grouping.ts`: Message grouping for compaction boundaries
- `postCompactCleanup.ts`: Post-compaction cleanup operations
- `compactWarningHook.ts` / `compactWarningState.ts`: User-facing warnings
  when context approaches limits

### 8.2 Extract Memories Service

> **Source:** `src/services/extractMemories/`

A background extraction agent that analyzes the session transcript for durable
learnings worth persisting to the auto-memory directory. Key design:

- **Trigger**: Fires when the model stops (no pending tool calls)
- **Execution**: Forked agent sharing the parent's prompt cache
- **Tool restrictions**: Read-only filesystem access; Edit/Write only within
  the auto-memory directory
- **Deduplication**: Skips extraction if the main agent already wrote memories
  in the current turn range
- **Gating**: Build-time `EXTRACT_MEMORIES` flag + runtime `tengu_passport_quail`
  GrowthBook gate

The extraction prompt (`prompts.ts`) instructs the agent on what to extract
(user preferences, corrections, project patterns) and what to skip (code
patterns, git history, ephemeral task details).

### 8.3 Session Memory Service

> **Source:** `src/services/SessionMemory/`

Session Memory automatically maintains a markdown file with notes about the
current conversation. Unlike auto-memory (cross-session), session memory is
within-session only:

- Runs periodically via `registerPostSamplingHook()`
- Uses a forked subagent with restricted tools
- Writes to a session-scoped markdown file at `getSessionMemoryPath()`
- Template-based updates via `buildSessionMemoryUpdatePrompt()`
- Configurable thresholds for initialization and update frequency

---

## 9. Sync Services

### 9.1 Team Memory Sync

> **Source:** `src/services/teamMemorySync/`

Synchronizes team memory files between the local filesystem and a server API,
enabling shared memory across all authenticated org members for a given
repository.

**API contract**:
- `GET /api/claude_code/team_memory?repo={owner/repo}` — full data + checksums
- `GET ...&view=hashes` — metadata + checksums only (bandwidth optimization)
- `PUT /api/claude_code/team_memory?repo={owner/repo}` — upload entries (upsert)

**Sync semantics**:
- Pull: server wins per-key (overwrites local)
- Push: delta upload only — keys whose content hash differs from server checksums
- File deletions do NOT propagate (next pull restores)

**State management**: All mutable state (ETag tracking, watcher suppression)
lives in a `SyncState` object created by the caller and threaded through
every call — no module-level mutable state.

**Secret scanning** (`secretScanner.ts`): Before upload, content is scanned
against ~30 high-confidence rules from gitleaks (AWS tokens, GCP API keys,
Anthropic API keys, GitHub PATs, Slack tokens, etc.). Files containing secrets
are silently skipped. Rule IDs are derived from the public gitleaks config with
JS regex adaptations.

**Batched uploads**: Bodies are capped at 200KB per PUT to avoid gateway
rejection (observed 256-512KB gateway limit). Larger payloads are split into
sequential PUTs — server upsert-merge semantics make this safe.

**Watcher** (`watcher.ts`): Filesystem watcher on the team memory directory
triggers push on file changes. Suppresses notifications during pull operations
to avoid feedback loops.

### 9.2 Settings Sync

> **Source:** `src/services/settingsSync/`

Syncs user settings and memory files across Claude Code environments:

- **Interactive CLI**: Uploads local settings to remote (incremental, only
  changed entries)
- **CCR (Cloud Code Runner)**: Downloads remote settings to local before plugin
  installation

Upload runs in the background on startup (`uploadUserSettingsInBackground()`).
Gated behind `UPLOAD_USER_SETTINGS` build flag and `tengu_enable_settings_sync_push`
GrowthBook gate. Only for OAuth-authenticated interactive sessions.

Sync keys (`SYNC_KEYS`) define which settings entries participate in
synchronization. Per-file size cap: 500KB (matches backend limit).

---

## 10. Configuration Services

### 10.1 Remote Managed Settings

> **Source:** `src/services/remoteManagedSettings/`

Manages enterprise-controlled settings that override local configuration.
This is how organizations enforce tool policies, permission rules, and
feature restrictions across their Claude Code deployments.

**Eligibility**:
- Console users (API key): All eligible
- OAuth users: Only Enterprise/C4E and Team subscribers

**Fetch and caching** (`syncCache.ts`, `syncCacheState.ts`):
- ETag-based conditional requests minimize bandwidth
- Session-level in-memory cache for fast access
- Disk cache for cross-session persistence
- Background polling every hour (`POLLING_INTERVAL_MS = 60 * 60 * 1000`)
- 30-second timeout with deadlock prevention for the loading promise

**Security check** (`securityCheck.jsx`): Validates fetched settings against
security constraints before applying. The check runs before settings are
merged into the active configuration.

**Fail-open**: If the fetch fails, the system continues without remote settings.
This is a deliberate design choice — managed settings should enhance security
but never break the tool.

### 10.2 Policy Limits

> **Source:** `src/services/policyLimits/`

Fetches organization-level policy restrictions from the API. Follows the same
patterns as remote managed settings (fail-open, ETag caching, background
polling, retry logic) but with a different payload: `restrictions` that
disable specific CLI features.

The policy limits response schema defines which features can be restricted,
and the restrictions are checked at various points in the codebase before
enabling features.

---

## 11. Plugin Service

> **Source:** `src/services/plugins/`

### 11.1 Plugin Operations

> **Source:** `src/services/plugins/pluginOperations.ts`

Core plugin CRUD operations (install, uninstall, enable, disable, update)
designed as pure library functions usable by both CLI commands and interactive
UI:

- **Install**: Resolves from marketplace, downloads, validates manifest,
  copies to versioned cache, registers in settings
- **Uninstall**: Checks reverse dependencies, removes installation, cleans
  up data directory, marks version orphaned
- **Update**: Checks for newer versions, downloads, installs alongside old
  version, updates settings path
- **Enable/Disable**: Toggles the plugin's enabled state in settings

Valid installable scopes: `user`, `project`, `local` (not `managed` — those
are installed from `managed-settings.json`).

### 11.2 Plugin Installation Manager

> **Source:** `src/services/plugins/PluginInstallationManager.ts`

Manages the plugin installation lifecycle, including dependency resolution,
marketplace interaction, and versioned caching.

### 11.3 Plugin CLI Commands

> **Source:** `src/services/plugins/pluginCliCommands.ts`

Implements the `claude plugin` CLI subcommands (`install`, `uninstall`,
`enable`, `disable`, `update`, `list`) by wrapping the core operations
with terminal I/O and process exit handling.

---

## 12. Minor Services

### 12.1 AutoDream

> **Source:** `src/services/autoDream/`

Background memory consolidation that periodically distills accumulated session
memories into organized topic files. See the KAIROS Design Specification for
full detail. Key points:

- **Gate sequence** (cheapest first): KAIROS exclusion → remote mode exclusion
  → auto-memory enabled → GrowthBook gate → time gate → session count gate →
  lock acquisition
- **Consolidation lock** (`consolidationLock.ts`): File lock with PID, 1-hour
  stale threshold, write-then-verify for concurrent reclaim
- **Consolidation prompt** (`consolidationPrompt.ts`): Four-phase workflow
  (orient → gather → consolidate → prune)

### 12.2 MagicDocs

> **Source:** `src/services/MagicDocs/`

Automatically maintains markdown documentation files marked with
`# MAGIC DOC: [title]` headers. When such a file is read during a session, a
background agent periodically updates it with new learnings from the
conversation.

- Detection: Regex pattern `^#\s*MAGIC\s+DOC:\s*(.+)$` on file read
- Tracking: Map of detected magic docs by path
- Update: Forked subagent with `runAgent()`, restricted to File Edit tools
- Trigger: Post-sampling hook, fires when model stops with pending tool calls

### 12.3 Prompt Suggestion

> **Source:** `src/services/PromptSuggestion/`

Generates suggested follow-up prompts after the model completes a response.
Uses a forked agent to speculatively generate user-intent suggestions.

- **Gating**: `tengu_chomp_inflection` GrowthBook gate + env var override
- **Speculation** (`speculation.ts`): Starts speculative generation in the
  background, canceled if the user types before it completes
- **Variant**: Currently uses `user_intent` prompt variant

### 12.4 Agent Summary

> **Source:** `src/services/AgentSummary/`

Periodic background summarization for coordinator-mode sub-agents. Forks the
sub-agent's conversation every 30 seconds to generate a 1-2 sentence progress
summary for UI display.

- Shares the parent's `CacheSafeParams` for prompt cache efficiency
- Summary prompt: "Describe your most recent action in 3-5 words using present
  tense (-ing). Name the file or function."
- Stored on `AgentProgress` for the coordinator UI

### 12.5 Tool Use Summary

> **Source:** `src/services/toolUseSummary/`

Generates human-readable summaries of completed tool batches using Haiku. Used
by the SDK to provide high-level progress updates to clients.

- System prompt: "Write a short summary label... truncates around 30 characters,
  so think git-commit-subject"
- Input: Tool names, truncated inputs/outputs (300 chars each), optional
  assistant context
- Examples: "Searched in auth/", "Fixed NPE in UserService", "Ran failing tests"

### 12.6 Tips

> **Source:** `src/services/tips/`

Displays contextual tips during spinner waits. Three modules:

- `tipRegistry.ts`: Defines available tips and their relevance conditions
- `tipHistory.ts`: Tracks which tips have been shown and when (session count)
- `tipScheduler.ts`: Selects tips using a longest-time-since-shown algorithm,
  respecting per-tip cooldown periods

### 12.7 Standalone Services

| Module | Purpose |
|--------|---------|
| `awaySummary.ts` | Generates summaries when user returns after being away |
| `claudeAiLimits.ts` | Tracks Claude.ai rate limit state from response headers |
| `diagnosticTracking.ts` | Shared diagnostic file types for LSP integration |
| `internalLogging.ts` | Internal diagnostic logging for ant developers |
| `notifier.ts` | OS-level notifications (macOS Notification Center, etc.) |
| `preventSleep.ts` | Prevents system sleep during long-running operations |
| `rateLimitMessages.ts` | User-facing rate limit message formatting |
| `tokenEstimation.ts` | Token count estimation utilities |
| `vcr.ts` | Record/replay for API responses (testing infrastructure) |
| `voice.ts` / `voiceStreamSTT.ts` | Voice input support |

---

## 13. Cross-Cutting Patterns

### 13.1 Fail-Open Design

Every service that fetches remote configuration (remote managed settings,
policy limits, team memory sync, settings sync, GrowthBook) is designed to
**fail open**: if the network call fails, the system continues with cached
or default values. The only service that must succeed is the API service
itself — without it, no model interaction is possible.

This design ensures that network partitions, API outages, or DNS failures
never prevent the user from using the CLI. Enterprise policy enforcement
degrades gracefully rather than locking out users.

### 13.2 ETag Caching

Remote managed settings and policy limits both use ETag-based conditional
requests. The flow:

1. First request: Server responds with body + `ETag` header
2. Subsequent requests: Client sends `If-None-Match: <etag>`
3. If unchanged: Server responds `304 Not Modified` (no body, minimal bandwidth)
4. If changed: Server responds `200 OK` with new body + new ETag

This pattern reduces bandwidth by 99%+ for polling scenarios (hourly
background checks).

### 13.3 Background Processing

Services extensively use background processing to avoid blocking the user:

- **Settings sync**: `uploadUserSettingsInBackground()` — fire-and-forget
- **Remote managed settings**: Background polling every hour
- **Policy limits**: Background polling every hour
- **Team memory sync**: Filesystem watcher triggers async push
- **Extract memories**: Post-sampling hook runs asynchronously
- **Session memory**: Post-sampling hook runs asynchronously
- **Magic docs**: Post-sampling hook runs asynchronously
- **Agent summary**: 30-second interval timer
- **Prompt suggestion**: Speculative generation in background
- **Analytics**: Batch log processor with 15-second flush interval

### 13.4 Progressive Enhancement via Feature Gates

Services use a two-tier gating strategy:

**Build-time gates** (`feature('FLAG_NAME')` from `bun:bundle`):
- Dead-code elimination at compile time
- Zero runtime cost when disabled
- Used for: `EXTRACT_MEMORIES`, `KAIROS`, `TEAMMEM`, `REACTIVE_COMPACT`,
  `HISTORY_SNIP`, `CONTEXT_COLLAPSE`, `MCP_SKILLS`, `UPLOAD_USER_SETTINGS`,
  `UNATTENDED_RETRY`, `BASH_CLASSIFIER`, `TRANSCRIPT_CLASSIFIER`

**Runtime gates** (GrowthBook):
- Server-side kill switches with disk-cached fallback
- Used for: `tengu_passport_quail` (extract mode), `tengu_chomp_inflection`
  (prompt suggestions), `tengu_log_datadog_events` (Datadog), all KAIROS gates

This two-tier approach means experimental features are stripped from public
builds (zero binary bloat) while deployed features can be instantly toggled
without a release.

### 13.5 Forked Agent Pattern

Multiple services use the forked agent pattern (`runForkedAgent()`) for
background intelligence:

| Service | Fork Purpose |
|---------|-------------|
| Extract Memories | Analyze transcript for durable learnings |
| Session Memory | Update session notes |
| Magic Docs | Update documentation files |
| Agent Summary | Generate progress summaries |
| Prompt Suggestion | Generate follow-up suggestions |
| Compact (full) | Generate conversation summary |
| AutoDream | Consolidate accumulated memories |

All forks share the parent's prompt cache via `CacheSafeParams`, making them
cost-efficient. Tools are restricted per-fork to prevent unintended side effects.

### 13.6 Retry with Backoff

The `getRetryDelay()` function from `src/services/api/withRetry.ts` is reused
across services (team memory sync, settings sync, remote managed settings,
policy limits). It provides exponential backoff with jitter:

```
delay = min(500ms * 2^(attempt-1), maxDelay) + random(0, 0.25 * delay)
```

Services typically set `MAX_RETRIES = 3-5` and `TIMEOUT_MS = 10,000-30,000`.

### 13.7 Marker Types for Safety

The analytics service introduces two marker types (`never`-typed) that force
developers to explicitly verify data safety before logging:

- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` — for
  general-access backends
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED` — for privileged columns
- `TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` — for error
  messages safe to log

These types are `never`, so they can only be satisfied via an explicit `as`
cast, which serves as a code-review signal that the developer verified the
content.

---

## 14. Design Principles

### 14.1 Fail-Open Everywhere

No remote service failure should prevent the user from working. Remote managed
settings, policy limits, team memory sync, settings sync, analytics, and
GrowthBook all fail silently. The only hard dependency is the Anthropic API
itself.

### 14.2 Cache-Conscious

Every service that interacts with the API respects prompt caching:
- The API service tracks cache breaks and explains them
- Forked agents share the parent's prompt cache
- Compact preserves cache-safe parameters
- Tool schemas are hashed per-tool to identify specific cache-breaking changes

### 14.3 Progressive Rollout

Features move through a pipeline: build-time gate (stripped from public builds)
→ runtime GrowthBook gate (server-side toggle) → full rollout. This allows
experimentation with zero risk to the general user population.

### 14.4 PII by Design

Analytics metadata types enforce compile-time verification that values don't
contain code or filepaths. PII-tagged values flow through dedicated proto
columns with privileged access. `_PROTO_*` keys are stripped from all
general-access sinks in a single centralized call.

### 14.5 Background Intelligence

Expensive intelligence operations (memory extraction, session memory, magic
docs, agent summaries, prompt suggestions) run as forked background agents
that share the parent's prompt cache. They never block the user's foreground
interaction.

### 14.6 Composition over Hierarchy

Services are composed at the call site rather than through inheritance.
The `SyncState` object in team memory sync, the `CacheSafeParams` in forked
agents, and the `ToolPermissionContext` in tool execution all follow the
pattern of threading explicit state through function calls rather than
maintaining module-level singletons. This gives tests natural isolation
and makes data flow visible.

### 14.7 Defensive Networking

Every network call has:
- A timeout (10-30 seconds for config fetches, 600 seconds for API)
- Retry logic with exponential backoff and jitter
- ETag caching where applicable
- Conditional fetches to minimize bandwidth
- Graceful degradation on failure
- Heartbeat yields during long waits (persistent retry mode)

### 14.8 Semantic Error Classification

Errors are not treated as opaque strings. The retry logic distinguishes
between authentication errors (refresh and retry), capacity errors (backoff),
connection errors (stale socket detection), context overflow (adjust tokens),
and permanent failures (surface to user). The tool execution service classifies
errors into telemetry-safe strings that survive minification.
