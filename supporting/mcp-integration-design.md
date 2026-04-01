# MCP Integration — Design Document

This section contains the architecture, strategy, data flow, state machines, invariants, trade-offs, and conceptual content independent of specific code implementation.

---

### 1. Overview

#### 1.1 What MCP Is

The Model Context Protocol (MCP) is an open standard that defines a client-server protocol for connecting AI applications to external tools and data sources. An MCP **server** exposes tools (callable functions), resources (readable data), and prompts (templated interactions) over a standardized JSON-RPC transport. An MCP **client** discovers these capabilities at connection time, then invokes them on behalf of the model.

#### 1.2 MCP's Role in Claude Code

MCP is Claude Code's **primary extensibility mechanism**. Without MCP, the agent has a fixed set of built-in tools (Read, Write, Edit, Bash, Glob, Grep, etc.). With MCP, any external service -- Slack, GitHub, Jira, databases, custom internal tools -- can expose its API as first-class tools that the model can invoke directly.

The MCP subsystem handles:

- Transport negotiation across multiple distinct transport types
- Configuration aggregation from seven scoped sources
- OAuth and cross-app authentication flows
- Connection lifecycle management with automatic reconnection
- Tool wrapping that converts MCP tools into Claude Code's internal Tool type
- Resource access through dedicated list and read tools
- Deduplication and policy enforcement across manual, plugin, and claude.ai servers

#### 1.3 Server Connection States

Every MCP server connection exists in one of five states, modeled as a discriminated union:

| State | Meaning |
|---|---|
| `connected` | Active connection with client handle, capabilities, instructions, and cleanup function |
| `failed` | Connection attempt failed; carries error message |
| `needs-auth` | Server requires OAuth authentication before connecting |
| `pending` | Connection in progress; optionally carries reconnect attempt count |
| `disabled` | User has disabled this server; no connection attempted |

---

### 2. Transport Layer

Claude Code supports multiple transport mechanisms for MCP communication. The transport is selected based on the `type` field in the server configuration.

#### 2.1 SSE (Server-Sent Events)

**Config type:** `sse`

The legacy streaming transport. A long-lived EventSource connection receives server-to-client messages; POST requests carry client-to-server messages. The EventSource connection is explicitly excluded from the 60-second request timeout -- only POST requests get timeouts.

SSE connections receive an OAuth provider, combined static and dynamic headers, and proxy-aware fetch configuration.

#### 2.2 Streamable HTTP

**Config type:** `http`

The current recommended transport. Per the MCP Streamable HTTP spec, every POST must advertise `Accept: application/json, text/event-stream`. HTTP connections support session IDs. When a server returns HTTP 404 with JSON-RPC error code -32001 ("Session not found"), the client detects session expiry and triggers reconnection.

#### 2.3 WebSocket

**Config type:** `ws`

Full-duplex transport for low-latency bidirectional communication. The WebSocket is opened with the `mcp` subprotocol. Two code paths exist: Bun's native WebSocket (which accepts headers and proxy options as constructor arguments) and Node's `ws` package. WebSocket connections support mTLS and proxy configuration.

#### 2.4 STDIO (Standard I/O)

**Config type:** `stdio` (or omitted -- the default)

The classic local-process transport. Spawns a child process with the configured command and arguments. Communication happens over the process's stdin/stdout pipes. stderr is captured into a 64MB-capped buffer for error logging.

The subprocess environment is constructed by merging a sanitized parent environment with any server-specific env overrides. An optional shell prefix environment variable can wrap the command in a shell.

#### 2.5 In-Process Transport

An optimization for specific built-in MCP servers (e.g., Chrome MCP, Computer Use MCP) that avoids spawning a subprocess. A linked transport pair is created: two in-process transport instances connected back-to-back where `send()` on one side delivers to `onmessage` on the other via microtask scheduling (to avoid stack depth issues with synchronous request/response cycles). `close()` on either side calls `onclose` on both.

Certain built-in servers are detected during connection setup and routed to in-process mode even though their config says `type: 'stdio'`.

#### 2.6 SDK Control Transport

A specialized bridge for MCP servers running in the SDK (Agent SDK) process rather than the CLI process. Unlike regular transports, this wraps JSON-RPC messages in control messages that travel through the structured I/O channel between CLI and SDK:

```
CLI MCP Client -> SdkControlClientTransport -> stdout control message ->
SDK StructuredIO -> SDK MCP Server -> response -> CLI resolves pending promise
```

SDK MCP servers have config type `sdk` and are handled entirely through this bridge; the normal connection path throws if it encounters an SDK config directly.

#### 2.7 Timeout Architecture

Two timeout layers protect against hung connections:

1. **Connection timeout**: 30 seconds by default (configurable). Races the connection attempt against a timer and kills the transport on timeout.

2. **Per-request timeout**: 60 seconds per POST request. Uses `setTimeout` + `AbortController` rather than `AbortSignal.timeout()` to avoid lazy GC of native timer memory (~2.4KB per request in certain runtimes). GET requests are excluded since they are long-lived SSE streams.

---

### 3. Client Architecture

#### 3.1 The MCP Client Instance

Each connected server gets a client instance from the MCP SDK. The client is constructed with:

- **Client info:** name, version, website URL
- **Capabilities:** roots (workspace root directory), elicitation (user input requests from server)

After connection, the client registers a handler that returns the working directory as a `file://` URI -- allowing servers to discover the project root.

#### 3.2 Memoized Connection Cache

The connection function is memoized, keyed by server name + serialized config. This means:

- Repeated calls with the same server config return the cached connection
- When a connection drops (`onclose` fires), the cache entry is deleted so the next call creates a fresh connection
- An explicit cache-clearing function can evict and clean up a cached connection

#### 3.3 Fetch Caches

Three LRU-cached fetch functions (bounded to a configurable size, keyed by server name) store the results of server discovery:

| Function | MCP Method | Returns |
|---|---|---|
| Tools fetch | `tools/list` | Tool definitions |
| Resources fetch | `resources/list` | Server resources |
| Commands fetch | `prompts/list` | Command definitions |

All three caches are invalidated on `onclose` and on the corresponding `list_changed` notification from the server.

#### 3.4 Safe Connection Entry Point

A safe entry-point function is provided for code that needs a valid connection. It calls the memoized connect function (which returns the cached result if still connected, or reconnects if the cache was cleared) and throws if the result is not `connected`. SDK servers are returned as-is since they have a separate lifecycle.

---

### 4. Configuration

#### 4.1 Configuration Scopes

Server configurations are tagged with a scope that identifies their origin:

| Scope | Source | Precedence (lowest to highest) |
|---|---|---|
| `claudeai` | Claude.ai web UI connector toggles | 1 (lowest) |
| `dynamic` | Plugin-provided MCP servers | 2 |
| `user` | User-level settings file | 3 |
| `project` | `.mcp.json` files (up the directory tree) | 4 |
| `local` | Local settings file | 5 |
| `enterprise` | Enterprise managed config file (exclusive) | 6 (highest) |
| `managed` | Enterprise policy settings | (policy layer) |

When an enterprise MCP config file exists, it has **exclusive control** -- all other scopes are suppressed.

#### 4.2 Configuration File Formats

**Project scope (`.mcp.json`):** Placed in the project root or any parent directory. The system walks upward from the working directory to the filesystem root, collecting `.mcp.json` files at each level with closer files having higher priority.

**Settings files (user, local, enterprise):** MCP servers are stored in an `mcpServers` property. Validated against a schema that is a union of all transport-specific schemas.

#### 4.3 Environment Variable Expansion

Configuration values support `${VAR}` and `${VAR:-default}` syntax. The expansion function processes stdio configs (command, args, env) and remote configs (url, headers). Missing variables are collected and reported as warnings, not errors -- the original `${VAR}` token is preserved for debugging.

#### 4.4 Server Name Normalization

Server names are normalized to match the pattern `^[a-zA-Z0-9_-]{1,64}$`. Invalid characters (dots, spaces) are replaced with underscores. For claude.ai servers (names starting with `"claude.ai "`), consecutive underscores are collapsed and leading/trailing underscores are stripped to prevent interference with the `__` delimiter used in MCP tool names.

#### 4.5 Policy Enforcement

Enterprise administrators can control MCP servers through allow/deny lists:

- **Deny list**: Absolute block list. Checked by name, command array, or URL pattern (with wildcard support). Denylist always takes precedence.
- **Allow list**: Positive allowlist. When present, only listed servers can connect. Supports the same three matching modes. An empty allowlist blocks all servers.
- **Managed-only mode**: When enabled, only policy-level settings control the allowlist -- user-defined allowlists are ignored.

#### 4.6 The Configuration Pipeline

```
+--------------------------------------------------+
|  1. Check enterprise config (exclusive control?)  |
+--------------------------------------------------+
|  2. Load scopes: user, project, local             |
|     (project scope: walk up directory tree)        |
|     (project servers: require user approval)       |
+--------------------------------------------------+
|  3. Load plugin MCP servers (from enabled plugins) |
|     - Dedup against manual servers (signature)     |
|     - Dedup among plugins (first-loaded wins)      |
+--------------------------------------------------+
|  4. Merge: plugin < user < project < local        |
+--------------------------------------------------+
|  5. Apply policy filtering (allow/deny lists)      |
+--------------------------------------------------+
|  6. Fetch claude.ai connectors (async, overlapped) |
|     - Dedup against enabled manual servers         |
|     - Merge with lowest precedence                 |
+--------------------------------------------------+
```

The pipeline is designed for startup speed: local config loading is fast (only local file reads), while the claude.ai connector fetch is kicked off in parallel and awaited only at the dedup step.

---

### 5. Authentication

#### 5.1 OAuth Provider

The OAuth provider implements the MCP SDK's OAuth interface for SSE and HTTP servers:

- **Token storage**: OAuth tokens (access, refresh, client info, discovery state) are persisted in the system's secure storage (macOS Keychain, Linux secret service), keyed by a hash of the server config
- **Token refresh**: Automatic refresh with retry logic for transient errors with exponential backoff (1s, 2s, 4s)
- **Dynamic Client Registration (DCR)**: When the authorization server doesn't recognize the client, DCR creates a new registration
- **Token revocation**: RFC 7009-compliant revocation with fallback to Bearer auth for non-compliant servers

#### 5.2 OAuth Discovery

Multi-step discovery chain:

1. **Configured URL**: If a metadata URL is set in the server config, fetch directly
2. **RFC 9728**: Probe `/.well-known/oauth-protected-resource` on the MCP server, read `authorization_servers[0]`, then RFC 8414 against that URL
3. **Fallback**: RFC 8414 path-aware discovery directly against the MCP server URL (for legacy servers that co-host auth metadata)

#### 5.3 Browser-Based OAuth Flow

When a server requires authentication:

1. Find an available port for the local callback server
2. Generate PKCE code verifier/challenge
3. Open the user's browser to the authorization URL
4. Run a local HTTP server to receive the callback
5. Exchange the authorization code for tokens
6. Store tokens in secure storage

A lockfile serializes concurrent OAuth flows for the same server to prevent races between multiple instances.

#### 5.4 Cross-App Access (XAA)

XAA enables browser-free authentication for enterprise environments by chaining two token exchanges:

1. **RFC 8693 Token Exchange at the IdP**: Exchange the user's `id_token` for an ID-JAG (Identity Assertion Grant) -- an identity-bound JWT that proves "this user consented to this app"
2. **RFC 7523 JWT Bearer Grant at the AS**: Exchange the ID-JAG for an access_token at the MCP server's authorization server

This eliminates the browser consent screen entirely. The IdP connection details (issuer, clientId, callback port) are configured once and shared across all XAA-enabled servers.

#### 5.5 Step-Up Authentication

HTTP 403 responses with `WWW-Authenticate: Bearer` challenges that indicate higher authentication requirements are intercepted. The auth provider is flagged for step-up and the SDK's auth flow is re-triggered with the additional requirements.

#### 5.6 Proxy Authentication for Claude.ai Connectors

Claude.ai connector servers route through a proxy. A wrapped fetch function:

1. Attaches the user's OAuth bearer token
2. On 401, force-refreshes the token
3. Retries the request with the new token (but only if the token actually changed -- avoids double round-trips)

#### 5.7 Auth Caching

A cache file records servers that returned 401, with a 15-minute TTL. During startup, servers with cached needs-auth entries skip the connection attempt entirely -- avoiding wasted network round-trips. A secondary check catches servers where OAuth discovery state exists but no tokens are present.

---

### 6. Connection Lifecycle

#### 6.1 Connection Manager

The connection manager is a React context provider that wraps the REPL component tree. It exposes manual reconnection and enable/disable toggle functions for the UI.

#### 6.2 Startup: Two-Phase Loading

Connection initialization runs in two phases to minimize perceived startup latency:

**Phase 1 -- Local configs (fast):**
1. Load configs from all local scopes (user, project, local, enterprise, plugins)
2. Initialize all servers as `pending`
3. Begin connecting enabled servers

**Phase 2 -- claude.ai configs (may be slow):**
1. Await the claude.ai connector fetch (started in parallel with Phase 1)
2. Deduplicate against enabled manual servers by URL signature
3. Add new claude.ai servers as `pending` and begin connecting

Both phases use batched processing with different concurrency limits:
- Local servers (stdio/sdk): concurrency of 3 (default)
- Remote servers: concurrency of 20 (default)

#### 6.3 Batched State Updates

Individual server connection results are queued and flushed to application state in a single update call via a 16ms debounce window. This batching prevents N separate UI re-renders when N servers connect simultaneously.

Each update merges into state by:
1. Finding or creating the client entry by name
2. Replacing tools with the server's namespace prefix
3. Replacing commands that belong to the server
4. Updating the resources map

#### 6.4 Automatic Reconnection

When a connected remote server's transport closes, the system initiates exponential backoff reconnection:

- **Max attempts:** 5
- **Initial backoff:** 1 second
- **Max backoff:** 30 seconds
- **Formula:** `min(initialBackoff * 2^(attempt-1), maxBackoff)`

Each attempt transitions the server to `pending` with the attempt count. On success, notification handlers are re-registered. On final failure, the server transitions to `failed`.

STDIO and SDK servers do not auto-reconnect (they represent crashed processes that need user intervention).

#### 6.5 Error-Triggered Close

The MCP SDK's transport fires `onerror` on connection failures but does not always call `onclose`. The client bridges this gap:

- **Terminal errors** (ECONNRESET, ETIMEDOUT, EPIPE, EHOSTUNREACH, ECONNREFUSED): counted up to 3 consecutive, then transport is forcibly closed
- **Session expired** (HTTP 404 + JSON-RPC -32001): immediate close and reconnect
- **Max SSE reconnection**: immediate close
- **Re-entry guard**: prevents multiple close calls

The forced close calls the client's close method rather than directly triggering onclose. This ensures pending tool call promises are rejected with an error, not left hanging.

#### 6.6 Graceful Shutdown

**STDIO servers**: An escalating signal sequence:
1. SIGINT, wait 100ms
2. SIGTERM, wait 400ms
3. SIGKILL (forced termination)
4. 600ms absolute failsafe timeout

**In-process servers**: Close both the server and client handles.

**Remote servers**: Close the transport connection.

#### 6.7 Notification Handlers

Connected servers register handlers for three MCP notifications:

| Notification | Handler |
|---|---|
| `tools/list_changed` | Invalidate tools cache, re-fetch, update state |
| `prompts/list_changed` | Invalidate commands cache, re-fetch commands and skills |
| `resources/list_changed` | Invalidate resources cache, re-fetch resources and skills |

---

### 7. Tool Wrapping

#### 7.1 The Tool Skeleton

A skeleton tool is defined whose every meaningful property is overridden at runtime when wrapping real MCP tools. The skeleton provides:

- An MCP marker flag for the permission system
- Open-world input schema (accepts any JSON object)
- String output schema (MCP results are serialized to string)
- Passthrough permission check that delegates to the permission pipeline

#### 7.2 Tool Construction

Each MCP tool from a server's tool listing is transformed into an internal Tool object by overriding the skeleton:

**Identity:**
- Fully qualified name: `mcp__<serverName>__<toolName>` (or raw name for SDK servers with prefix stripping)
- Preserved server/tool info for permission checking
- User-facing display name: `"serverName - toolTitle (MCP)"`

**Metadata (from MCP tool annotations):**
- Concurrency safety / read-only hint
- Destructive hint
- Open-world hint
- Search/read classification for UI collapsing
- Search hint for tool discovery
- Always-load flag to bypass deferred loading

**Description and prompt:**
- Capped at 2048 characters to prevent context window flooding from verbose OpenAPI-generated servers
- Input schema passed through as-is

**Call implementation:**
- Establishes a valid connection before each call
- Supports elicitation retry on authentication challenges
- Retries once on session expiry (session recovery)
- Emits progress events (started, completed, failed) for UI feedback
- Wraps SDK errors into telemetry-safe error types

**Permission suggestions:**
- Returns a rule suggestion pointing at the fully qualified tool name, enabling "always allow" for specific MCP tools

#### 7.3 Tool Collapse Classification

A classification function determines whether an MCP tool invocation should collapse in the UI (like built-in search/read operations). It maintains two static allowlists -- search tools (~140 entries) and read tools (~400 entries) -- covering common MCP servers (Slack, GitHub, Linear, Datadog, Sentry, Notion, Jira, Asana, Grafana, PagerDuty, etc.).

Tool names are normalized (camelCase/kebab-case to snake_case) before lookup. Unknown tools conservatively do not collapse.

#### 7.4 Tool Call Timeout

MCP tool calls have a near-infinite default timeout of approximately 27.8 hours, overridable via environment variable. This reflects the reality that MCP tools may run arbitrary long-lived operations.

---

### 8. Resource Access

#### 8.1 List Resources Tool

A built-in tool that lists available resources across all connected MCP servers (or a specific server). It:

- Filters to connected clients only
- Uses LRU-cached resource listings (warm from startup prefetch)
- Returns resource metadata (URI, name, MIME type, description, server)
- Handles per-server failures gracefully without sinking the whole result
- Marked as concurrency-safe, read-only, and deferred

#### 8.2 Read Resource Tool

A built-in tool that reads a specific resource by URI from a named server. It:

- Validates that the server exists, is connected, and supports resources
- Handles binary blob responses: decodes base64, writes raw bytes to disk with a MIME-derived extension, and replaces the blob with a file path reference (prevents base64 content from consuming context window space)
- Returns structured content with URI, MIME type, and either text content or a file path

#### 8.3 Resource Tool Injection

Resource tools are added to the tool pool only when at least one connected server declares resource capability. They are added exactly once -- a flag prevents duplicate injection.

---

### 9. Scoped Cleanup

#### 9.1 Scoped vs. Shared MCP Servers

MCP servers enter the system through two paths:

**Inline definitions (scoped):** Servers defined in `.mcp.json` or settings files with full configuration. Their lifecycle is tied to that config's scope.

**Name references (shared):** Plugin servers use a namespaced key with their own config. Dynamic servers arrive via CLI flags or SDK control messages.

#### 9.2 Stale Server Detection

When configs change (plugin reload, session reset), the initialization process detects stale servers -- those present in state but absent from the new config -- and disconnects them:

1. Cancel any pending reconnect timer
2. Clear `onclose` handler (prevents reconnect loop with old config)
3. Clear the connection cache to close the connection
4. Remove from state

#### 9.3 Agent Cleanup

When the session ends, the cleanup process clears all reconnect timers and flushes pending batched updates. Each connected server's cleanup function handles transport-specific teardown.

---

### 10. Tool Pool Composition

#### 10.1 Deduplication with Built-in Tools

MCP tools use fully qualified names (`mcp__<serverName>__<toolName>`), which naturally avoids name collisions with built-in tools. SDK MCP servers with prefix stripping can intentionally override built-in tools -- server/tool info is still preserved for permission checking.

#### 10.2 Server-Level Deduplication

Three deduplication mechanisms prevent duplicate connections:

1. **Plugin vs. manual**: Plugin servers whose signature (stdio command or URL) matches a manually-configured server are suppressed. Between plugins, first-loaded wins. Signatures strip proxy URL wrappers to match the underlying vendor URL.

2. **Claude.ai vs. manual**: Claude.ai connectors whose URL matches an enabled manual server are suppressed. Only enabled manual servers count as dedup targets.

3. **Config-based**: Servers with changed config hashes are treated as stale and reconnected with the new config.

#### 10.3 Deny Rule Filtering

Tool visibility is filtered at multiple levels:

- **Enterprise policy**: Servers blocked by allow/deny lists never connect
- **User disable**: Disabled servers are tracked in state but skip connection
- **IDE tool filtering**: Only specific tools are included from IDE MCP servers; other IDE tools are hidden

#### 10.4 Auth Tool Injection

When a server is in the `needs-auth` state, a synthetic auth tool is injected into its tool list. When called by the model, this triggers the OAuth authentication flow for that server.

---

### 11. System Prompt Integration

#### 11.1 MCP Server Instructions

MCP servers can provide instructions in their initialization response. These are server-authored behavioral hints that inform the model about effective server usage.

Instructions are integrated through a **delta-based announcement** system:

- A comparison function checks currently connected servers (with instructions) against what has been announced in conversation history
- Newly connected servers produce "added" blocks; disconnected servers produce "removed" entries
- Deltas are persisted as attachment messages
- Instructions are capped at 2048 characters

Two modes exist (feature-gated):
- **Delta mode**: Announce via persisted delta attachments -- cache-friendly, survives compaction
- **Legacy mode**: Rebuild an uncached system prompt section every turn -- cache-busts on late connects

#### 11.2 Client-Side Instructions

The system allows attaching client-authored instruction blocks to specific servers, independent of the server's own instructions. This is used for first-party servers where the client has context the server doesn't know about.

#### 11.3 Tool Descriptions in the Prompt

Each MCP tool's description is included in the model's system prompt as part of the tool definitions, subject to the 2048-character cap. Tool input schemas are passed through directly, providing structured type information.

---

### 12. Design Principles

#### 12.1 Transport Polymorphism

Multiple transports present a uniform interface. Transport selection happens once during connection -- everything downstream (tool calls, resource reads, notification handling) is transport-agnostic. New transports can be added without modifying tool wrapping, state management, or UI layers.

#### 12.2 Fail-Open Configuration, Fail-Closed Policy

The configuration system is permissive: missing env vars produce warnings, missing config files are silently ignored, and individual server failures do not block startup. But the policy system is strict: the deny list is absolute, the allow list is exhaustive when present, and enterprise config is exclusive.

#### 12.3 Cache-Aware Memoization

Connection and fetch results are memoized at appropriate granularity:
- Connections are memoized by name + config -- reconnects on config change
- Fetch results are LRU-cached by server name -- survive reconnects, invalidated on list_changed notifications
- Auth discovery is memoized per server key

Cache invalidation is explicit and surgical: transport close clears both the connection cache and all three fetch caches for that server.

#### 12.4 Graceful Degradation

Every failure boundary is isolated:
- One server's failure does not prevent others from connecting
- One server's tool fetch failure returns empty tools, not an exception
- Auth cache prevents repeated 401 probes (15-minute TTL)
- Reconnection has backoff cap and attempt limit
- Terminal error counter prevents infinite error loops
- Binary content that cannot be persisted falls back to an error text message

#### 12.5 Batched State Management

Server connection results arrive asynchronously from parallel connection attempts. Updates are queued and flushed in a single state update every 16ms, keeping the UI responsive during startup.

#### 12.6 Security Layering

MCP servers are untrusted external code. Defenses include:
- **Unicode sanitization** on all tool and prompt data from servers
- **Description truncation** (2048-char cap) prevents context window flooding
- **Policy enforcement** with URL pattern matching
- **Project server approval** requirement before connecting
- **OAuth security**: PKCE, lockfile serialization, secure token storage, sensitive parameter redaction
- **Namespace isolation**: prefix prevents tool impersonation
- **Permission passthrough**: full permission pipeline applies to MCP tools

#### 12.7 Startup Latency Optimization

The configuration pipeline is engineered for fast startup:
- Local configs are loaded synchronously (file reads only)
- The cloud connector fetch is kicked off in parallel and awaited only at the dedup step
- Plugin loading uses cache-only mode
- Local and remote servers are connected with separate concurrency limits
- Work-stealing scheduling (instead of fixed-size batch boundaries)
