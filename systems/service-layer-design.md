# Service Layer — Design Document

## 1. Overview

The service layer is the backend foundation of Claude Code. It sits between the query loop / UI layer above and the raw APIs / OS below, providing the abstractions that make an agentic CLI possible: reliable API communication, multi-protocol tool server management, token-aware context compression, cross-session memory synchronization, enterprise policy enforcement, and telemetry routing.

The services are organized into six functional groups:

```
+------------------------------------------------------------------+
|                     Query Loop / UI Layer                          |
+------------------------------------------------------------------+
|  Communication         |  Execution          |  Intelligence      |
|  api                   |  tools              |  compact            |
|  mcp                   |  plugins            |  extractMemories    |
|  oauth                 |  lsp                |  SessionMemory      |
|  analytics             |                     |  autoDream          |
+------------------------------------------------------------------+
|  Synchronization       |  Configuration      |  Enhancement        |
|  teamMemorySync        |  remoteManagedSettings|  MagicDocs         |
|  settingsSync          |  policyLimits       |  PromptSuggestion   |
|                        |                     |  AgentSummary       |
|                        |                     |  toolUseSummary     |
|                        |                     |  tips               |
+------------------------------------------------------------------+
```

**Core Contract:** Every service follows a common contract: fail-open by default (network failures never block the user), background execution where possible, and progressive enhancement via feature gates. The only exception is the API service itself, which must succeed for any work to happen.

---

## 2. API Service

The API service is the most critical module in the service layer. Every model interaction flows through it. It wraps the Anthropic SDK with retry logic, multi-provider support, streaming management, prompt cache monitoring, and cost tracking.

### 2.1 Client Construction

The client factory supports five provider backends, selected by environment variables:

| Provider | Auth Mechanism |
|----------|----------------|
| First-party Anthropic (default) | API key or OAuth access token |
| AWS Bedrock | AWS credentials / bearer token |
| Azure Foundry | API key or Azure AD token provider |
| Google Vertex AI | GCP credentials |
| Claude.ai (OAuth) | OAuth access token |

Before client creation, the factory performs OAuth token refresh, configures custom headers, injects a client request ID for timeout correlation, and applies proxy configuration.

The factory is called on every retry attempt when the previous error was an authentication failure (401, 403 token-revoked, cloud credential errors, or stale connection errors). This ensures fresh credentials on each attempt.

### 2.2 Streaming and the Main Query Function

The core streaming interface accepts system prompts, messages, tools, thinking configuration, and model parameters, then streams message events back to the caller. Key behaviors:

- **Beta header negotiation**: Dynamically assembles beta headers based on model capabilities
- **Prompt cache management**: Places cache control markers on system prompt blocks with configurable scope (global vs. organization) and TTL (5-minute vs. 1-hour). Extended caching eligibility is determined by an allowlist and overage detection.
- **Media limiting**: Enforces a maximum of 100 media items per request by stripping excess images/documents from older messages
- **Tool schema conversion**: Transforms internal tool objects to API-format schemas, with special handling for deferred tools, MCP instructions delta, and advisor tools
- **Cost tracking**: After each response, computes USD cost and adds to session total

The function also maintains a cache-safe parameters structure used by forked agents (compaction, memory extraction, agent summarization) to share the parent's prompt cache.

### 2.3 Retry Logic

The retry system is not a simple exponential backoff -- it implements a multi-strategy retry engine:

**Retry decisions:**
- Mock errors: never retry
- Persistent mode (unattended sessions): always retry 429/529
- CCR mode (remote sessions): retry 401/403 (transient infra blips)
- Overloaded errors: always retry
- Context overflow errors: retry with adjusted max tokens
- Server `x-should-retry` header: obey unless Claude.ai subscriber (non-Enterprise)
- Connection errors, 408, 409, 429, 5xx: retry with conditions

**Fast mode fallback:** When fast mode is active and a 429/529 arrives:
1. If `retry-after` < 20 seconds: wait and retry with fast mode still active (preserves prompt cache)
2. If `retry-after` >= 20 seconds or unknown: switch to standard speed model with a minimum 10-minute cooldown floor
3. If overage disabled: permanently disable fast mode

**529 (overloaded) handling:**
- Background query sources (summaries, suggestions, classifiers) bail immediately -- no retry amplification during capacity cascades
- Foreground sources retry up to 3 times
- After 3 consecutive 529s: trigger fallback model if configured, else surface custom error message

**Persistent retry mode (unattended sessions):**
- Retries 429/529 indefinitely
- Uses higher backoff (up to 5 minutes) with a 6-hour reset cap
- Chunks long sleeps into 30-second heartbeat intervals, yielding status messages so the host environment does not mark the session idle

**Backoff formula:** `min(BASE * 2^(attempt-1), maxDelay) + random(0, 0.25 * BASE)` where BASE = 500ms and default maxDelay = 32 seconds.

### 2.4 Prompt Cache Break Detection

A two-phase diagnostic system that detects and explains unexpected drops in prompt cache utilization:

**Phase 1 (pre-call):** Hashes system prompt, tool schemas, cache control markers, beta headers, model name, fast mode, effort value, and extra body params. Computes per-tool schema hashes. Stores pending changes for phase 2.

**Phase 2 (post-call):** Compares current cache read tokens against previous value. Triggers if cache read dropped >5% AND absolute drop > 2,000 tokens. Classifies the break: client-side change (with specific reason), possible 5-minute TTL expiry, possible 1-hour TTL expiry, or "likely server-side (prompt unchanged, <5min gap)."

The system tracks up to 10 independent query sources, with special handling: compact shares tracking state with the main REPL thread (same server-side cache key), and subagents are isolated by agent ID.

---

## 3. MCP Service

The Model Context Protocol (MCP) service manages connections to external tool servers. It is the most complex service by file count, reflecting the protocol's breadth: multiple transport types, OAuth authentication for remote servers, configuration from 7+ sources, channel notifications, and dynamic tool registration.

### 3.1 Transport Types

MCP supports eight transport configurations:

| Transport | Use Case |
|-----------|----------|
| stdio | Local subprocess (command + args + env) |
| sse | Server-Sent Events over HTTP(S) |
| sse-ide | IDE extension (VS Code / JetBrains) |
| http | Streamable HTTP (bidirectional) |
| ws | WebSocket (bidirectional) |
| ws-ide | IDE WebSocket transport |
| sdk | SDK control protocol |
| claudeai-proxy | Claude.ai proxy servers |

Additionally, an in-process transport provides a linked transport pair for running MCP servers in-process without spawning a subprocess. Messages sent on one side are delivered to the other's handler via microtask queuing to avoid stack depth issues.

### 3.2 Connection Lifecycle

Each server progresses through a state machine:

```
pending -> connected
        -> failed (with error message)
        -> needs-auth (OAuth required)
        -> disabled (admin-disabled)
```

The connection type is a discriminated union of these five states. Connected servers carry a client instance, capabilities, server info, instructions, and a cleanup function.

Connection management includes:
- Transport creation, MCP client initialization, capability negotiation, tool/resource registration
- Full connection pool lifecycle management, including reconnection with exponential backoff
- Tool registration: each MCP tool is wrapped to handle schema normalization, result truncation, binary content persistence, and image resizing

### 3.3 Configuration Sources

MCP server configs are aggregated from seven scoped sources, from highest to lowest priority:

| Scope | Source |
|-------|--------|
| enterprise | Managed MCP file |
| managed | Remote managed settings |
| user | Global user config |
| project | Project config |
| local | Local config |
| dynamic | Plugin-provided servers |
| claudeai | Claude.ai proxy configs |

A project-root configuration file provides an alternative per-project configuration path. Environment variables in server configs support `${VAR}` expansion.

### 3.4 OAuth for MCP Servers

Remote MCP servers can require OAuth 2.0 authentication. The auth provider supports:
- Authorization Server Metadata discovery
- Dynamic client registration
- PKCE code flow
- Token refresh
- Secure token storage via platform secure storage
- Cross-App Access for enterprise identity provider federation

### 3.5 Channel Notifications

For assistant mode, MCP servers can act as communication channels, relaying messages between the agent and external platforms (Slack, Discord, etc.) via channel notifications. A seven-gate access control pipeline verifies capability, runtime gate, OAuth, org policy, session list, plugin marketplace, and allowlist before activating a channel.

### 3.6 Elicitation

MCP servers can request information from users via the elicitation protocol. The handler manages the lifecycle of elicitation requests, including hook execution, result validation, and timeout management.

---

## 4. Authentication Service

### 4.1 OAuth 2.0 Flow

The authentication service implements OAuth 2.0 Authorization Code Flow with PKCE, supporting dual code acquisition paths:

**Automatic flow:** Opens the user's browser to the authorization URL with a localhost redirect URI. A local HTTP server starts on a random port to capture the authorization code.

**Manual flow:** For environments without browser access, the user copies a URL, authenticates in any browser, and pastes the resulting authorization code back into the CLI.

Both paths race concurrently -- whichever delivers the code first wins.

### 4.2 Token Management

The flow produces tokens containing:
- Access token / refresh token
- Absolute expiration timestamp
- Parsed scope set
- Subscription type / rate limit tier
- Account UUID, email, organization UUID

Token refresh is called before every API client construction. The retry logic triggers refresh on 401 and 403 "token revoked" errors.

### 4.3 Cryptographic Primitives

- Code verifier: Random 128-byte base64url string for PKCE
- Code challenge: SHA-256 hash of verifier, base64url-encoded
- State: Random 32-byte hex string for CSRF protection

---

## 5. LSP Service

The Language Server Protocol service provides real-time code intelligence by managing external LSP servers, routing requests based on file types, and delivering diagnostics as conversation attachments.

### 5.1 Architecture

The service uses a factory-function pattern with closures for state encapsulation:

| Component | Role |
|-----------|------|
| Server Manager | Routes requests by file extension, manages server lifecycle |
| Server Instance | Wraps a single LSP server process with JSON-RPC protocol |
| Client | Low-level JSON-RPC communication over stdio |
| Diagnostic Registry | Stores async diagnostics for conversation delivery |

### 5.2 Server Manager

The server manager loads LSP server configurations from settings, builds an extension-to-server mapping, and provides:
- Lazy-start of appropriate server for a file path
- Request routing by extension
- File state synchronization with LSP servers (didOpen, didChange, didSave, didClose)

### 5.3 Diagnostic Registry

When LSP servers send diagnostics notifications, the registry stores them for delivery as conversation attachments:
1. Register pending diagnostic with timestamp
2. Retrieve pending (undelivered) diagnostics
3. Convert to attachment format
4. Deliver into the next query turn

Volume limiting: max 10 diagnostics per file, 30 total per delivery.
Cross-turn deduplication: LRU cache (500 files) of delivered diagnostic keys prevents re-delivery.

### 5.4 Passive Feedback

After file edits, the LSP service can provide passive feedback to the model by checking for new diagnostics introduced by the edit. This enables the model to self-correct without explicit user intervention.

---

## 6. Analytics Service

The analytics service provides telemetry routing with strong PII guarantees, event sampling, feature flag management, and multi-sink delivery.

### 6.1 Event Logging Architecture

The analytics pipeline follows a queue-then-drain pattern:

```
logEvent(name, metadata)          <- Public API (zero dependencies)
    |
    v (pre-init: queue)
eventQueue[]                      <- Buffered until sink attached
    |
    v attachAnalyticsSink()       <- Called during app startup
    |
    +---> shouldSampleEvent()     <- Per-event sampling
    |
    +---> Datadog (if gate enabled)
    |      - Strip PII-tagged keys
    |
    +---> 1P Event Logger (OpenTelemetry)
           - Full payload including PII-tagged keys
```

**PII handling** is enforced through two marker types (typed as `never`) that force explicit casts, documenting that the developer verified the value is safe:
- One type for general-access storage verification
- One type for privileged PII-tagged proto column routing

PII-tagged keys are stripped from all general-access sinks in a single centralized call. Only the first-party exporter routes them to privileged columns.

### 6.2 Datadog Integration

Datadog receives a curated subset of events (approximately 35 event types). Events are batched (max 100) and flushed every 15 seconds. Selected metadata fields are promoted to Datadog tags for dashboard filtering.

### 6.3 First-Party Event Logger

The first-party logger uses OpenTelemetry's logger provider with a batch processor and a custom exporter that POSTs events to Anthropic's event logging endpoint.

Event sampling is configurable per-event via dynamic config. Events not in the config are logged at 100% rate. When an event is sampled, its sample rate is attached for downstream bias correction.

### 6.4 Feature Flags (GrowthBook)

GrowthBook provides runtime feature flags and experiment targeting:

- **User attributes** for targeting: id, session, platform, organization, subscription type, rate limit tier, email, app version, CI metadata
- **Remote evaluation**: Server-side feature computation with client-side caching
- **Dual-layer caching**: In-memory and on-disk
- **Exposure deduplication**: Prevents duplicate exposure events in hot paths
- **Refresh intervals**: 6 hours (production), 20 minutes (internal)
- **Refresh listeners**: Systems that bake feature values into long-lived objects can subscribe to rebuild when config changes

Two accessor patterns:
- Cached feature value accessor: Pure read from memory or disk cache (synchronous/performance-critical)
- Cached gate check: Boolean gate check

### 6.5 Event Metadata

Every event is enriched with approximately 50 fields: platform, model, provider, user type, subscription type, permission mode, session ID, assistant state, tool details, and more. Metadata types are restricted to boolean/number to prevent accidental code/filepath logging.

### 6.6 Sink Killswitch

A safety mechanism that allows individual analytics sinks to be disabled via feature gates, providing immediate rollback capability.

---

## 7. Tool Execution Service

### 7.1 Execution Engine

The tool execution service is the bridge between the model's tool-use requests and actual tool implementations:

**Validation pipeline:**
1. Schema validation against the tool's input schema
2. Per-tool semantic validation (file existence, encoding, size limits, UNC path detection)
3. Errors wrapped and returned to the model

**Permission pipeline:**
1. Full permission stack evaluation (rules, tool checks, scanner, mode transformation, hooks, classifier)
2. Permission decision logged to analytics, OpenTelemetry, and in-session store
3. Denied tools return error messages with optional memory correction hints

**Execution pipeline:**
1. Pre-tool hooks run
2. Tool executes with progress streaming
3. Post-tool hooks run (success or failure variants)
4. Tool result processed: large outputs persisted to disk, binary content saved as file references, file state cache updated

**Error classification:** Extracts telemetry-safe error information from minified builds. Uses safe error messages, Node.js errno codes, or known error type names -- never mangled identifiers.

### 7.2 Tool Hooks

Hook execution wraps each tool call:

| Hook Phase | Purpose |
|-----------|---------|
| Pre-tool | Can modify input, add context, block execution |
| Post-tool (success) | Can modify output, trigger side effects |
| Post-tool (failure) | Error handling, cleanup |
| Permission | Allow/deny/defer decisions |

Hook timing is tracked: a summary is shown inline when total hook duration exceeds 500ms. A debug warning fires when any phase blocks for over 2 seconds.

### 7.3 Speculative Classifier

The tool execution service integrates with a speculative classifier for bash commands. It begins evaluating the next likely bash command in the background while the current tool is still running, reducing perceived latency for the permission prompt.

---

## 8. Context Services

These services manage the model's context window -- compressing, extracting, and maintaining state across turns and sessions.

### 8.1 Compact Service (4 Strategies)

**Strategy 1: Session Memory Compact:** Lightweight, no API call. Keeps the most recent N messages (configurable thresholds for min tokens, min messages, max tokens). Preserves tool_use/tool_result pairs.

**Strategy 2: Full Summarization:** Sends old messages to the LLM. Pre-processes by stripping images/documents and re-injectable attachments. Output capped at 20,000 tokens. Includes post-compact restoration of recently-accessed files, active skills, plan files, and background agent status.

**Strategy 3: Reactive Compact:** Emergency compaction on "prompt too long" API errors. Last-resort strategy when auto-compact's proactive threshold miscalculated.

**Strategy 4: Micro Compact:** Finer-grained approach using the API's cache edits mechanism to delete specific segments. Time-based configuration.

Additional concerns:
- Auto-compact decision logic with threshold calculation and circuit breaker (3 consecutive failures)
- Message grouping for compaction boundaries
- Post-compaction cleanup operations
- User-facing warnings when context approaches limits

### 8.2 Extract Memories Service

A background extraction agent that analyzes the session transcript for durable learnings worth persisting to the auto-memory directory:

- **Trigger:** Fires when the model stops (no pending tool calls)
- **Execution:** Forked agent sharing the parent's prompt cache
- **Tool restrictions:** Read-only filesystem access; write only within the auto-memory directory
- **Deduplication:** Skips extraction if the main agent already wrote memories in the current turn range
- **Gating:** Build-time flag + runtime feature gate

The extraction prompt instructs the agent on what to extract (user preferences, corrections, project patterns) and what to skip (code patterns, git history, ephemeral task details).

### 8.3 Session Memory Service

Session Memory automatically maintains a markdown file with notes about the current conversation. Unlike auto-memory (cross-session), session memory is within-session only:

- Runs periodically via post-sampling hook
- Uses a forked subagent with restricted tools
- Writes to a session-scoped markdown file
- Template-based updates
- Configurable thresholds for initialization and update frequency

---

## 9. Sync Services

### 9.1 Team Memory Sync

Synchronizes team memory files between the local filesystem and a server API, enabling shared memory across all authenticated org members for a given repository.

**Sync semantics:**
- Pull: server wins per-key (overwrites local)
- Push: delta upload only -- keys whose content hash differs from server checksums
- File deletions do NOT propagate (next pull restores)

**State management:** All mutable state (ETag tracking, watcher suppression) lives in an explicit state object created by the caller and threaded through every call -- no module-level mutable state.

**Secret scanning:** Before upload, content is scanned against approximately 30 high-confidence rules (AWS tokens, GCP API keys, Anthropic API keys, GitHub PATs, Slack tokens, etc.). Files containing secrets are silently skipped.

**Batched uploads:** Bodies are capped at 200KB per PUT to avoid gateway rejection (observed 256-512KB gateway limit). Larger payloads are split into sequential PUTs -- server upsert-merge semantics make this safe.

**Watcher:** Filesystem watcher on the team memory directory triggers push on file changes. Suppresses notifications during pull operations to avoid feedback loops.

### 9.2 Settings Sync

Syncs user settings and memory files across Claude Code environments:

- **Interactive CLI:** Uploads local settings to remote (incremental, only changed entries)
- **Cloud Code Runner:** Downloads remote settings to local before plugin installation

Upload runs in the background on startup. Gated behind build flag and runtime feature gate. Only for OAuth-authenticated interactive sessions.

Per-file size cap: 500KB (matches backend limit).

---

## 10. Configuration Services

### 10.1 Remote Managed Settings

Manages enterprise-controlled settings that override local configuration. This is how organizations enforce tool policies, permission rules, and feature restrictions.

**Eligibility:**
- Console users (API key): All eligible
- OAuth users: Only Enterprise/C4E and Team subscribers

**Fetch and caching:**
- ETag-based conditional requests minimize bandwidth
- Session-level in-memory cache for fast access
- Disk cache for cross-session persistence
- Background polling every hour
- 30-second timeout with deadlock prevention for the loading promise

**Fail-open:** If the fetch fails, the system continues without remote settings. Managed settings enhance security but never break the tool.

### 10.2 Policy Limits

Fetches organization-level policy restrictions from the API. Follows the same patterns as remote managed settings (fail-open, ETag caching, background polling, retry logic) but with a different payload: restrictions that disable specific CLI features.

---

## 11. Plugin Service

### 11.1 Plugin Operations

Core plugin CRUD operations (install, uninstall, enable, disable, update) designed as pure library functions usable by both CLI commands and interactive UI:

- **Install:** Resolves from marketplace, downloads, validates manifest, copies to versioned cache, registers in settings
- **Uninstall:** Checks reverse dependencies, removes installation, cleans up data directory, marks version orphaned
- **Update:** Checks for newer versions, downloads, installs alongside old version, updates settings path
- **Enable/Disable:** Toggles the plugin's enabled state in settings

Valid installable scopes: user, project, local (not managed -- those come from managed settings).

---

## 12. Minor Services

### 12.1 AutoDream

Background memory consolidation that periodically distills accumulated session memories into organized topic files.

- **Gate sequence** (cheapest first): assistant mode exclusion -> remote mode exclusion -> auto-memory enabled -> feature gate -> time gate -> session count gate -> lock acquisition
- **Consolidation lock:** File lock with PID, 1-hour stale threshold, write-then-verify for concurrent reclaim
- **Consolidation workflow:** Four-phase (orient -> gather -> consolidate -> prune)

### 12.2 MagicDocs

Automatically maintains markdown documentation files marked with special headers. When such a file is read during a session, a background agent periodically updates it with new learnings from the conversation.

- Detection: Regex pattern on file read
- Tracking: Map of detected docs by path
- Update: Forked subagent restricted to file edit tools
- Trigger: Post-sampling hook, fires when model stops with pending tool calls

### 12.3 Prompt Suggestion

Generates suggested follow-up prompts after the model completes a response. Uses a forked agent for speculative generation.

- Gated via feature flag + env var override
- Speculative background generation, canceled if the user types before completion

### 12.4 Agent Summary

Periodic background summarization for coordinator-mode sub-agents. Forks the sub-agent's conversation every 30 seconds to generate a 1-2 sentence progress summary for UI display.

- Shares the parent's cache-safe parameters for prompt cache efficiency
- Summary prompt: "Describe your most recent action in 3-5 words using present tense (-ing). Name the file or function."

### 12.5 Tool Use Summary

Generates human-readable summaries of completed tool batches using a lightweight model. Used by the SDK to provide high-level progress updates to clients.

- Input: Tool names, truncated inputs/outputs (300 chars each), optional assistant context
- Examples: "Searched in auth/", "Fixed NPE in UserService", "Ran failing tests"

### 12.6 Tips

Displays contextual tips during spinner waits:
- Registry: Defines available tips and their relevance conditions
- History: Tracks which tips have been shown and when (session count)
- Scheduler: Selects tips using a longest-time-since-shown algorithm, respecting per-tip cooldown periods

### 12.7 Standalone Services

| Service | Purpose |
|---------|---------|
| Away Summary | Generates summaries when user returns after being away |
| Rate Limit Tracking | Tracks rate limit state from response headers |
| Diagnostic Tracking | Shared diagnostic file types for LSP integration |
| Internal Logging | Internal diagnostic logging for developers |
| Notifier | OS-level notifications |
| Prevent Sleep | Prevents system sleep during long-running operations |
| Rate Limit Messages | User-facing rate limit message formatting |
| Token Estimation | Token count estimation utilities |
| VCR | Record/replay for API responses (testing infrastructure) |
| Voice | Voice input support |

---

## 13. Cross-Cutting Patterns

### 13.1 Fail-Open Design

Every service that fetches remote configuration is designed to fail open: if the network call fails, the system continues with cached or default values. The only service that must succeed is the API service itself.

This ensures that network partitions, API outages, or DNS failures never prevent the user from using the CLI. Enterprise policy enforcement degrades gracefully rather than locking out users.

### 13.2 ETag Caching

Remote managed settings and policy limits both use ETag-based conditional requests:
1. First request: Server responds with body + ETag header
2. Subsequent requests: Client sends If-None-Match
3. If unchanged: 304 Not Modified (no body, minimal bandwidth)
4. If changed: 200 OK with new body + new ETag

This reduces bandwidth by 99%+ for hourly polling scenarios.

### 13.3 Background Processing

Services extensively use background processing to avoid blocking the user. Settings sync, remote managed settings, policy limits, team memory sync, extract memories, session memory, magic docs, agent summary, prompt suggestion, and analytics all run asynchronously.

### 13.4 Progressive Enhancement via Feature Gates

Two-tier gating strategy:

**Build-time gates:** Dead-code elimination at compile time. Zero runtime cost when disabled.

**Runtime gates (GrowthBook):** Server-side kill switches with disk-cached fallback.

This means experimental features are stripped from public builds (zero binary bloat) while deployed features can be instantly toggled without a release.

### 13.5 Forked Agent Pattern

Multiple services use a forked agent pattern for background intelligence:

| Service | Fork Purpose |
|---------|-------------|
| Extract Memories | Analyze transcript for durable learnings |
| Session Memory | Update session notes |
| Magic Docs | Update documentation files |
| Agent Summary | Generate progress summaries |
| Prompt Suggestion | Generate follow-up suggestions |
| Compact (full) | Generate conversation summary |
| AutoDream | Consolidate accumulated memories |

All forks share the parent's prompt cache, making them cost-efficient. Tools are restricted per-fork to prevent unintended side effects.

### 13.6 Retry with Backoff

Exponential backoff with jitter is reused across services:

```
delay = min(500ms * 2^(attempt-1), maxDelay) + random(0, 0.25 * delay)
```

Services typically retry 3-5 times with 10-30 second timeouts.

### 13.7 Marker Types for Safety

The analytics service introduces marker types (typed as `never`) that force developers to explicitly verify data safety before logging. They can only be satisfied via an explicit cast, which serves as a code-review signal. Three markers exist: one for general-access safety verification, one for PII-tagged columns, and one for telemetry-safe error messages.

---

## 14. Design Principles

1. **Fail-Open Everywhere:** No remote service failure should prevent the user from working. The only hard dependency is the Anthropic API itself.

2. **Cache-Conscious:** Every service that interacts with the API respects prompt caching. The API service tracks cache breaks and explains them. Forked agents share the parent's prompt cache. Compact preserves cache-safe parameters. Tool schemas are hashed per-tool.

3. **Progressive Rollout:** Features move through: build-time gate (stripped from public builds) -> runtime feature gate (server-side toggle) -> full rollout.

4. **PII by Design:** Analytics metadata types enforce compile-time verification. PII-tagged values flow through dedicated proto columns with privileged access. PII keys are stripped from all general-access sinks.

5. **Background Intelligence:** Expensive intelligence operations run as forked background agents that share the parent's prompt cache. They never block the user's foreground interaction.

6. **Composition over Hierarchy:** Services are composed at the call site rather than through inheritance. Explicit state is threaded through function calls rather than module-level singletons. This gives tests natural isolation and makes data flow visible.

7. **Defensive Networking:** Every network call has a timeout, retry logic with exponential backoff and jitter, ETag caching where applicable, conditional fetches, graceful degradation on failure, and heartbeat yields during long waits.

8. **Semantic Error Classification:** Errors are not treated as opaque strings. The retry logic distinguishes between authentication errors (refresh and retry), capacity errors (backoff), connection errors (stale socket detection), context overflow (adjust tokens), and permanent failures (surface to user).
