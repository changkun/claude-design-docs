# Claude Code: MCP Protocol Integration — Design Specification

This document analyzes the Model Context Protocol (MCP) integration architecture of
Claude Code — Anthropic's agentic CLI tool — focusing on how it discovers, connects to,
wraps, and manages external MCP servers to extend the agent's capabilities at runtime.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Transport Layer](#2-transport-layer)
- [3. Client Architecture](#3-client-architecture)
- [4. Configuration](#4-configuration)
- [5. Authentication](#5-authentication)
- [6. Connection Lifecycle](#6-connection-lifecycle)
- [7. Tool Wrapping](#7-tool-wrapping)
- [8. Resource Access](#8-resource-access)
- [9. Scoped Cleanup](#9-scoped-cleanup)
- [10. Tool Pool Composition](#10-tool-pool-composition)
- [11. System Prompt Integration](#11-system-prompt-integration)
- [12. Design Principles](#12-design-principles)

---

## 1. Overview

### 1.1 What MCP Is

The Model Context Protocol (MCP) is an open standard that defines a client-server
protocol for connecting AI applications to external tools and data sources. An MCP
**server** exposes tools (callable functions), resources (readable data), and prompts
(templated interactions) over a standardized JSON-RPC transport. An MCP **client**
discovers these capabilities at connection time, then invokes them on behalf of the
model.

### 1.2 MCP's Role in Claude Code

MCP is Claude Code's **primary extensibility mechanism**. Without MCP, the agent
has a fixed set of built-in tools (Read, Write, Edit, Bash, Glob, Grep, etc.). With
MCP, any external service — Slack, GitHub, Jira, databases, custom internal tools —
can expose its API as first-class tools that the model can invoke directly.

The MCP subsystem is the largest service module in the codebase: 22 source files
under `src/services/mcp/`, plus tool wrappers in `src/tools/MCPTool/`,
`src/tools/ListMcpResourcesTool/`, and `src/tools/ReadMcpResourceTool/`. It handles:

- Transport negotiation across five distinct transport types
- Configuration aggregation from seven scoped sources
- OAuth and cross-app authentication flows
- Connection lifecycle management with automatic reconnection
- Tool wrapping that converts MCP tools into Claude Code's internal `Tool` type
- Resource access through dedicated list and read tools
- Deduplication and policy enforcement across manual, plugin, and claude.ai servers

### 1.3 Server Connection States

> **Source:** `src/services/mcp/types.ts`

Every MCP server connection exists in one of five states, modeled as a discriminated
union `MCPServerConnection`:

| State | Meaning |
|---|---|
| `connected` | Active connection with `Client` handle, capabilities, instructions, and cleanup function |
| `failed` | Connection attempt failed; carries error message |
| `needs-auth` | Server requires OAuth authentication before connecting |
| `pending` | Connection in progress; optionally carries reconnect attempt count |
| `disabled` | User has disabled this server; no connection attempted |

---

## 2. Transport Layer

> **Source:** `src/services/mcp/client.ts:595-961`

Claude Code supports five transport mechanisms for MCP communication. The transport
is selected based on the `type` field in the server configuration.

### 2.1 SSE (Server-Sent Events)

**Config type:** `sse`

The legacy streaming transport. Uses `SSEClientTransport` from the MCP SDK. A
long-lived EventSource connection receives server-to-client messages; POST requests
carry client-to-server messages. The EventSource connection is explicitly excluded
from the 60-second request timeout — only POST requests get timeouts.

SSE connections receive a `ClaudeAuthProvider` for OAuth, combined static and dynamic
headers via `getMcpServerHeaders()`, and proxy-aware fetch configuration.

### 2.2 Streamable HTTP

**Config type:** `http`

The current recommended transport. Uses `StreamableHTTPClientTransport` from the MCP
SDK. Per the MCP Streamable HTTP spec, every POST must advertise
`Accept: application/json, text/event-stream` — the client enforces this in
`wrapFetchWithTimeout()` to handle runtimes that drop the header.

HTTP connections support session IDs (the MCP SDK manages `Mcp-Session-Id`
automatically). When a server returns HTTP 404 with JSON-RPC error code -32001
("Session not found"), the client detects session expiry via
`isMcpSessionExpiredError()` and triggers reconnection.

### 2.3 WebSocket

**Config type:** `ws`

Full-duplex transport for low-latency bidirectional communication. Uses a custom
`WebSocketTransport` wrapper (`src/utils/mcpWebSocketTransport.js`). The WebSocket
is opened with the `mcp` subprotocol. Two code paths exist for WebSocket creation:
Bun's native `globalThis.WebSocket` (which accepts headers and proxy options as
constructor arguments) and Node's `ws` package (via `createNodeWsClient()`).

WebSocket connections support mTLS via `getWebSocketTLSOptions()` and proxy
configuration via `getWebSocketProxyAgent()`.

### 2.4 STDIO (Standard I/O)

**Config type:** `stdio` (or omitted — the default)

The classic local-process transport. `StdioClientTransport` spawns a child process
with the configured command and arguments. Communication happens over the process's
stdin/stdout pipes. stderr is captured into a 64MB-capped buffer for error logging.

The subprocess environment is constructed by merging `subprocessEnv()` (the sanitized
parent environment) with any server-specific `env` overrides. An optional
`CLAUDE_CODE_SHELL_PREFIX` environment variable can wrap the command in a shell.

### 2.5 In-Process Transport

> **Source:** `src/services/mcp/InProcessTransport.ts`

An optimization for specific built-in MCP servers (Chrome MCP, Computer Use MCP) that
avoids spawning a ~325 MB subprocess. `createLinkedTransportPair()` creates two
`InProcessTransport` instances connected back-to-back: `send()` on one side delivers
to `onmessage` on the other via `queueMicrotask()` (to avoid stack depth issues with
synchronous request/response cycles). `close()` on either side calls `onclose` on
both.

Servers that match `isClaudeInChromeMCPServer()` or `isComputerUseMCPServer()` are
detected during `connectToServer()` and routed to in-process mode even though their
config says `type: 'stdio'`.

### 2.6 SDK Control Transport

> **Source:** `src/services/mcp/SdkControlTransport.ts`

A specialized bridge for MCP servers running in the SDK (Agent SDK) process rather
than the CLI process. Unlike regular transports, `SdkControlClientTransport` wraps
JSON-RPC messages in control messages that travel through the structured I/O channel
between CLI and SDK:

```
CLI MCP Client → SdkControlClientTransport → stdout control message →
SDK StructuredIO → SDK MCP Server → response → CLI resolves pending promise
```

SDK MCP servers have config type `sdk` and are handled entirely through this bridge;
`connectToServer()` throws if it encounters an SDK config directly.

### 2.7 Timeout Architecture

Two timeout layers protect against hung connections:

1. **Connection timeout** (`getConnectionTimeoutMs()`): 30 seconds by default
   (`MCP_TIMEOUT` env override). Races `client.connect(transport)` against a
   `setTimeout` and kills the transport on timeout.

2. **Per-request timeout** (`wrapFetchWithTimeout()`): 60 seconds per POST request.
   Uses `setTimeout` + `AbortController` rather than `AbortSignal.timeout()` to
   avoid Bun's lazy GC of native timer memory (~2.4KB per request). GET requests
   are excluded since they are long-lived SSE streams.

---

## 3. Client Architecture

### 3.1 The MCP Client Instance

> **Source:** `src/services/mcp/client.ts:985-1001`

Each connected server gets a `Client` instance from `@modelcontextprotocol/sdk`. The
client is constructed with:

- **Client info:** name `claude-code`, version from build macros, website URL
- **Capabilities:** `roots` (workspace root directory), `elicitation` (user input
  requests from server)

After connection, the client registers a `ListRootsRequestSchema` handler that returns
the working directory as a `file://` URI — allowing servers to discover the project
root.

### 3.2 Memoized Connection Cache

`connectToServer()` is wrapped with lodash `memoize`, keyed by
`getServerCacheKey(name, serverRef)` (the server name + JSON-serialized config).
This means:

- Repeated calls with the same server config return the cached connection
- When a connection drops (`onclose` fires), the cache entry is deleted so the
  next call creates a fresh connection
- `clearServerCache()` explicitly evicts and cleans up a cached connection

### 3.3 Fetch Caches

Three LRU-cached fetch functions (bounded to `MCP_FETCH_CACHE_SIZE = 20`, keyed by
server name) store the results of server discovery:

| Function | MCP Method | Returns |
|---|---|---|
| `fetchToolsForClient` | `tools/list` | `Tool[]` |
| `fetchResourcesForClient` | `resources/list` | `ServerResource[]` |
| `fetchCommandsForClient` | `prompts/list` | `Command[]` |

All three caches are invalidated on `onclose` and on the corresponding
`list_changed` notification from the server.

### 3.4 ensureConnectedClient

`ensureConnectedClient()` is the safe entry point for code that needs a valid
connection. It calls `connectToServer()` (which returns the cached result if still
connected, or reconnects if the cache was cleared) and throws if the result is
not `connected`. SDK servers are returned as-is since they have a separate lifecycle.

---

## 4. Configuration

### 4.1 Configuration Scopes

> **Source:** `src/services/mcp/types.ts:10-20`, `src/services/mcp/config.ts`

Server configurations are tagged with a `ConfigScope` that identifies their origin:

| Scope | Source | Precedence (lowest → highest) |
|---|---|---|
| `claudeai` | Claude.ai web UI connector toggles | 1 (lowest) |
| `dynamic` | Plugin-provided MCP servers | 2 |
| `user` | `~/.claude/settings.json` `mcpServers` | 3 |
| `project` | `.mcp.json` files (up the directory tree) | 4 |
| `local` | `.claude/settings.local.json` `mcpServers` | 5 |
| `enterprise` | `/etc/claude-code/managed-mcp.json` (exclusive) | 6 (highest) |
| `managed` | Enterprise policy settings | (policy layer) |

When an enterprise MCP config file exists (`managed-mcp.json`), it has **exclusive
control** — all other scopes are suppressed.

### 4.2 Configuration File Formats

**`.mcp.json`** (project scope): Placed in the project root (or any parent
directory). The system walks upward from `getCwd()` to the filesystem root,
collecting `.mcp.json` files at each level with closer files having higher priority.
Format:

```json
{
  "mcpServers": {
    "server-name": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

**Settings files** (user, local, enterprise): MCP servers are stored in the
`mcpServers` property of the settings JSON. Validated against `McpServerConfigSchema`
(a Zod union of all transport-specific schemas).

### 4.3 Environment Variable Expansion

> **Source:** `src/services/mcp/envExpansion.ts`

Configuration values support `${VAR}` and `${VAR:-default}` syntax. The
`expandEnvVars()` function processes stdio configs (command, args, env) and remote
configs (url, headers). Missing variables are collected and reported as warnings,
not errors — the original `${VAR}` token is preserved for debugging.

### 4.4 Server Name Normalization

> **Source:** `src/services/mcp/normalization.ts`

Server names are normalized to match the API pattern `^[a-zA-Z0-9_-]{1,64}$`.
Invalid characters (dots, spaces) are replaced with underscores. For claude.ai
servers (names starting with `"claude.ai "`), consecutive underscores are collapsed
and leading/trailing underscores are stripped to prevent interference with the `__`
delimiter used in MCP tool names.

### 4.5 Policy Enforcement

> **Source:** `src/services/mcp/config.ts:364-551`

Enterprise administrators can control MCP servers through allow/deny lists in
managed settings:

- **`deniedMcpServers`**: Absolute block list. Checked by name, command array, or
  URL pattern (with wildcard support). Denylist always takes precedence.
- **`allowedMcpServers`**: Positive allowlist. When present, only listed servers
  can connect. Supports the same three matching modes. An empty allowlist blocks
  all servers.
- **`allowManagedMcpServersOnly`**: When true, only policy-level settings control
  the allowlist — user-defined allowlists are ignored.

`filterMcpServersByPolicy()` is the public API that applies policy filtering to a
config map, returning allowed and blocked server names.

### 4.6 The getAllMcpConfigs Pipeline

> **Source:** `src/services/mcp/config.ts:1258-1290`

The complete configuration pipeline:

```
┌──────────────────────────────────────────────────┐
│  1. Check enterprise config (exclusive control?) │
├──────────────────────────────────────────────────┤
│  2. Load scopes: user, project, local            │
│     (project scope: walk up directory tree)       │
│     (project servers: require user approval)      │
├──────────────────────────────────────────────────┤
│  3. Load plugin MCP servers (from enabled plugins)│
│     ├── Dedup against manual servers (signature)  │
│     └── Dedup among plugins (first-loaded wins)   │
├──────────────────────────────────────────────────┤
│  4. Merge: plugin < user < project < local       │
├──────────────────────────────────────────────────┤
│  5. Apply policy filtering (allow/deny lists)     │
├──────────────────────────────────────────────────┤
│  6. Fetch claude.ai connectors (async, overlapped)│
│     ├── Dedup against enabled manual servers      │
│     └── Merge with lowest precedence              │
└──────────────────────────────────────────────────┘
```

The pipeline is designed for startup speed: `getClaudeCodeMcpConfigs()` is fast
(only local file reads), while the claude.ai connector fetch is kicked off in
parallel and awaited only at the dedup step.

---

## 5. Authentication

### 5.1 The ClaudeAuthProvider

> **Source:** `src/services/mcp/auth.ts`

`ClaudeAuthProvider` implements the MCP SDK's `OAuthClientProvider` interface for
SSE and HTTP servers. It handles:

- **Token storage**: OAuth tokens (access, refresh, client info, discovery state)
  are persisted in the system's secure storage (macOS Keychain, Linux secret service)
  keyed by a SHA-256 hash of the server config
- **Token refresh**: Automatic refresh with retry logic for transient errors
  (`TemporarilyUnavailableError`, `TooManyRequestsError`, `ServerError`) with
  exponential backoff (1s, 2s, 4s)
- **Dynamic Client Registration (DCR)**: When the server's authorization server
  doesn't recognize the client, DCR creates a new client registration. Client
  metadata is fetched from `MCP_CLIENT_METADATA_URL`
- **Token revocation**: RFC 7009-compliant revocation with fallback to Bearer auth
  for non-compliant servers

### 5.2 OAuth Discovery

`fetchAuthServerMetadata()` implements a multi-step discovery chain:

1. **Configured URL**: If `oauth.authServerMetadataUrl` is set in the server config,
   fetch metadata directly from that HTTPS URL
2. **RFC 9728**: Probe `/.well-known/oauth-protected-resource` on the MCP server,
   read `authorization_servers[0]`, then RFC 8414 against that URL
3. **Fallback**: RFC 8414 path-aware discovery directly against the MCP server URL
   (for legacy servers that co-host auth metadata)

### 5.3 Browser-Based OAuth Flow

When a server requires authentication, the system:

1. Finds an available port for the local callback server (`findAvailablePort()`)
2. Generates PKCE code verifier/challenge
3. Opens the user's browser to the authorization URL
4. Runs a local HTTP server to receive the callback
5. Exchanges the authorization code for tokens
6. Stores tokens in secure storage

A lockfile (`~/.claude/mcp-auth-{hash}.lock`) serializes concurrent OAuth flows for
the same server to prevent races between multiple Claude Code instances.

### 5.4 Cross-App Access (XAA)

> **Source:** `src/services/mcp/xaa.ts`, `src/services/mcp/xaaIdpLogin.ts`

XAA (SEP-990) enables browser-free authentication for enterprise environments by
chaining two token exchanges:

1. **RFC 8693 Token Exchange at the IdP**: Exchange the user's `id_token` for an
   **ID-JAG** (Identity Assertion Grant) — an identity-bound JWT that proves "this
   user consented to this app"
2. **RFC 7523 JWT Bearer Grant at the AS**: Exchange the ID-JAG for an
   `access_token` at the MCP server's authorization server

This eliminates the browser consent screen entirely. The IdP connection details
(issuer, clientId, callback port) are configured once in `settings.xaaIdp` and
shared across all XAA-enabled servers.

### 5.5 Step-Up Authentication

`wrapFetchWithStepUpDetection()` intercepts HTTP 403 responses with
`WWW-Authenticate: Bearer` challenges that indicate the server requires a higher
authentication level. When detected, the auth provider is flagged for step-up and
the SDK's auth flow is re-triggered with the additional requirements.

### 5.6 Claude.ai Proxy Authentication

> **Source:** `src/services/mcp/client.ts:372-422`

Claude.ai connector servers route through a proxy at `oauthConfig.MCP_PROXY_URL`.
`createClaudeAiProxyFetch()` wraps fetch to:

1. Attach the user's claude.ai OAuth bearer token
2. On 401, call `handleOAuth401Error()` to force-refresh the token
3. Retry the request with the new token (but only if the token actually changed —
   avoids double round-trips when the server genuinely needs auth)

### 5.7 Auth Caching

The `mcp-needs-auth-cache.json` file records servers that returned 401, with a
15-minute TTL. During startup, servers with cached needs-auth entries skip the
connection attempt entirely — avoiding wasted network round-trips for servers that
cannot succeed until the user authenticates via `/mcp`. A secondary check
(`hasMcpDiscoveryButNoToken()`) catches servers where OAuth discovery state is stored
but no tokens exist, closing the gap the TTL leaves open.

---

## 6. Connection Lifecycle

### 6.1 MCPConnectionManager

> **Source:** `src/services/mcp/MCPConnectionManager.tsx`

`MCPConnectionManager` is a React context provider that wraps the REPL component
tree. It instantiates `useManageMCPConnections()` and exposes two functions via
context:

- `reconnectMcpServer(serverName)` — manual reconnection (from `/mcp` UI)
- `toggleMcpServer(serverName)` — enable/disable toggle (from `/mcp` UI)

### 6.2 Startup: Two-Phase Loading

> **Source:** `src/services/mcp/useManageMCPConnections.ts:858-1024`

Connection initialization runs in two phases to minimize perceived startup latency:

**Phase 1 — Claude Code configs (fast):**
1. Load configs from all local scopes (user, project, local, enterprise, plugins)
2. Initialize all servers as `pending` in AppState
3. Begin connecting enabled servers via `getMcpToolsCommandsAndResources()`

**Phase 2 — claude.ai configs (may be slow):**
1. Await the claude.ai connector fetch (started in parallel with Phase 1)
2. Deduplicate against enabled manual servers by URL signature
3. Add new claude.ai servers as `pending` and begin connecting

Both phases use `processBatched()` (backed by `pMap`) with different concurrency
limits:
- Local servers (stdio/sdk): `getMcpServerConnectionBatchSize()` = 3 (default)
- Remote servers: `getRemoteMcpServerConnectionBatchSize()` = 20 (default)

### 6.3 Batched State Updates

> **Source:** `src/services/mcp/useManageMCPConnections.ts:207-308`

Individual server connection results are queued into `pendingUpdatesRef` and flushed
to AppState in a single `setAppState` call via a 16ms `setTimeout` window. This
batching prevents N separate React re-renders when N servers connect simultaneously.

Each update merges into AppState by:
1. Finding or creating the client entry by name
2. Replacing tools with the server's prefix (`mcp__serverName__`)
3. Replacing commands that belong to the server
4. Updating the resources map

### 6.4 Automatic Reconnection

When a connected remote server's `onclose` fires, the system initiates exponential
backoff reconnection:

- **Max attempts:** 5 (`MAX_RECONNECT_ATTEMPTS`)
- **Initial backoff:** 1 second (`INITIAL_BACKOFF_MS`)
- **Max backoff:** 30 seconds (`MAX_BACKOFF_MS`)
- **Formula:** `min(1000 * 2^(attempt-1), 30000)`

Each attempt transitions the server to `pending` with the attempt count, calls
`reconnectMcpServerImpl()`, and on success re-registers all notification handlers.
On final failure, the server transitions to `failed`.

STDIO and SDK servers do not auto-reconnect (they represent crashed processes that
need user intervention).

### 6.5 Error-Triggered Close

> **Source:** `src/services/mcp/client.ts:1249-1371`

The MCP SDK's transport fires `onerror` on connection failures but does not always
call `onclose`, which Claude Code uses to trigger reconnection. The client bridges
this gap:

- **Terminal errors** (ECONNRESET, ETIMEDOUT, EPIPE, EHOSTUNREACH, ECONNREFUSED):
  counted up to 3 consecutive, then `closeTransportAndRejectPending()` is called
- **Session expired** (HTTP 404 + JSON-RPC -32001): immediate close and reconnect
- **Max SSE reconnection**: SDK reports "Maximum reconnection attempts" — immediate
  close
- **Re-entry guard**: `hasTriggeredClose` prevents multiple close calls

`closeTransportAndRejectPending()` calls `client.close()` rather than
`client.onclose?.()` directly. This ensures pending `callTool()` promises are
rejected with `McpError -32000`, not left hanging.

### 6.6 Graceful Shutdown

> **Source:** `src/services/mcp/client.ts:1404-1580`

Cleanup is registered via `registerCleanup()` for all transport types:

**STDIO servers**: An escalating signal sequence:
1. `SIGINT` → wait 100ms
2. `SIGTERM` → wait 400ms
3. `SIGKILL` → forced termination
4. 600ms absolute failsafe timeout

**In-process servers**: Close both the server and client handles.

**Remote servers**: `client.close()` closes the transport connection.

### 6.7 Notification Handlers

When a server connects, `onConnectionAttempt()` registers handlers for three
MCP notifications:

| Notification | Handler |
|---|---|
| `tools/list_changed` | Invalidate `fetchToolsForClient` cache, re-fetch, update AppState |
| `prompts/list_changed` | Invalidate `fetchCommandsForClient` cache, re-fetch commands and skills |
| `resources/list_changed` | Invalidate `fetchResourcesForClient` cache, re-fetch resources and skills |

---

## 7. Tool Wrapping

### 7.1 The MCPTool Skeleton

> **Source:** `src/tools/MCPTool/MCPTool.ts`

`MCPTool` is a skeleton tool built with `buildTool()` whose every meaningful property
is overridden at runtime when wrapping real MCP tools. The skeleton provides:

- `isMcp: true` — marks the tool as an MCP tool for the permission system
- Open-world input schema (`z.object({}).passthrough()`) — accepts any JSON object
- String output schema — MCP results are serialized to string
- Passthrough permission check — delegates to the permission pipeline

### 7.2 Tool Construction

> **Source:** `src/services/mcp/client.ts:1743-1998`

`fetchToolsForClient()` transforms each MCP tool from the server's `tools/list`
response into a Claude Code `Tool` object by spreading `MCPTool` and overriding:

**Identity:**
- `name`: Fully qualified `mcp__serverName__toolName` (or raw name for SDK servers
  with `CLAUDE_AGENT_SDK_MCP_NO_PREFIX`)
- `mcpInfo`: `{ serverName, toolName }` — preserved for permission checking
- `userFacingName()`: `"serverName - toolTitle (MCP)"`

**Metadata (from MCP tool annotations):**
- `isConcurrencySafe()` / `isReadOnly()`: From `readOnlyHint` annotation
- `isDestructive()`: From `destructiveHint` annotation
- `isOpenWorld()`: From `openWorldHint` annotation
- `isSearchOrReadCommand()`: From `classifyMcpToolForCollapse()` — per-tool
  allowlist for UI collapsing
- `searchHint`: From `_meta['anthropic/searchHint']` — guides tool discovery
- `alwaysLoad`: From `_meta['anthropic/alwaysLoad']` — bypasses deferred loading

**Description and prompt:**
- Capped at `MAX_MCP_DESCRIPTION_LENGTH = 2048` characters. OpenAPI-generated MCP
  servers have been observed dumping 15-60KB of endpoint docs; this caps the p95 tail.
- Input schema passed through as-is (`tool.inputSchema`)

**Call implementation:**
- Calls `ensureConnectedClient()` to get a valid connection
- Invokes `callMCPToolWithUrlElicitationRetry()` with abort signal, progress
  callbacks, and elicitation handler
- Retries once on `McpSessionExpiredError` (session recovery)
- Emits MCP progress events (started, completed, failed) for UI feedback
- Wraps MCP SDK errors into `TelemetrySafeError` for analytics

**Permission suggestions:**
- Returns an `addRules` suggestion pointing at the fully qualified tool name in
  `localSettings`, so the user can "always allow" specific MCP tools

### 7.3 Tool Collapse Classification

> **Source:** `src/tools/MCPTool/classifyForCollapse.ts`

`classifyMcpToolForCollapse()` determines whether an MCP tool invocation should
collapse in the UI (like built-in search/read operations). It maintains two static
allowlists — `SEARCH_TOOLS` (~140 entries) and `READ_TOOLS` (~400 entries) —
covering the most common MCP servers (Slack, GitHub, Linear, Datadog, Sentry, Notion,
Jira, Asana, Grafana, PagerDuty, and many more).

Tool names are normalized (camelCase/kebab-case → snake_case) before lookup. Unknown
tools conservatively return `{ isSearch: false, isRead: false }` — they do not
collapse.

### 7.4 Tool Call Timeout

MCP tool calls have a near-infinite default timeout of ~27.8 hours
(`DEFAULT_MCP_TOOL_TIMEOUT_MS = 100_000_000`), overridable via `MCP_TOOL_TIMEOUT`
env var. This reflects the reality that MCP tools may run arbitrary long-lived
operations.

---

## 8. Resource Access

### 8.1 ListMcpResourcesTool

> **Source:** `src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts`

A built-in tool that lists available resources across all connected MCP servers (or
a single server specified by the `server` input parameter). It:

- Filters to `connected` clients only
- Calls `ensureConnectedClient()` then `fetchResourcesForClient()` (LRU-cached,
  warm from startup prefetch)
- Returns an array of `{ uri, name, mimeType, description, server }` objects
- Gracefully handles per-server reconnect failures without sinking the whole result
- Marked as `isConcurrencySafe`, `isReadOnly`, and `shouldDefer` (deferred loading)

### 8.2 ReadMcpResourceTool

> **Source:** `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts`

A built-in tool that reads a specific resource by URI from a named server. It:

- Validates that the server exists, is connected, and supports resources
- Sends a `resources/read` request to the MCP server
- Handles binary blob responses: decodes base64, writes raw bytes to disk with a
  MIME-derived extension via `persistBinaryContent()`, and replaces the blob with a
  file path reference. This prevents base64-encoded content from consuming context
  window space.
- Returns `{ contents: [{ uri, mimeType, text?, blobSavedTo? }] }`

### 8.3 Resource Tool Injection

Resource tools (`ListMcpResourcesTool`, `ReadMcpResourceTool`) are added to the
tool pool only when at least one connected server declares `resources` capability.
They are added exactly once — the first server with resource support gets them
appended to its tool list. The `resourceToolsAdded` flag in
`getMcpToolsCommandsAndResources()` prevents duplicate injection.

---

## 9. Scoped Cleanup

### 9.1 Scoped vs. Shared MCP Servers

MCP servers enter the system through two paths:

**Inline definitions (scoped):** Servers defined in `.mcp.json` or settings files
with full configuration (command, URL, etc.). These are "owned" by the config source
and their lifecycle is tied to that config's scope.

**Name references (shared):** Plugin servers use a namespaced key
(`plugin:pluginName:serverName`) with their own config. Dynamic servers arrive via
`--mcp-config` or SDK control messages.

### 9.2 Stale Server Detection

> **Source:** `src/services/mcp/utils.ts` (via `excludeStalePluginClients`)

When configs change (plugin reload, session reset), `initializeServersAsPending()`
detects stale servers — those present in AppState but absent from the new config —
and disconnects them. For each stale server:

1. Cancel any pending reconnect timer
2. Clear `onclose` handler (prevents reconnect loop with old config)
3. Call `clearServerCache()` to close the connection
4. Remove from AppState

### 9.3 Agent Cleanup

When the REPL unmounts or a session clears, the cleanup effect in
`useManageMCPConnections` clears all reconnect timers and flushes pending batched
updates. Each connected server's `cleanup()` function (registered via
`registerCleanup()`) handles transport-specific teardown.

---

## 10. Tool Pool Composition

### 10.1 Deduplication with Built-in Tools

The MCP tool wrapping system creates tools with fully qualified names
(`mcp__server__tool`), which naturally avoids name collisions with built-in tools.
For SDK MCP servers with `CLAUDE_AGENT_SDK_MCP_NO_PREFIX=true`, tools use their raw
names and can intentionally override built-in tools — `mcpInfo` is still preserved
for permission checking.

### 10.2 Server-Level Deduplication

Three deduplication mechanisms prevent the same underlying server from being connected
twice:

1. **Plugin vs. manual** (`dedupPluginMcpServers`): Plugin servers whose
   "signature" (stdio command array or URL) matches a manually-configured server
   are suppressed. Between plugins, first-loaded wins. Signatures strip CCR proxy
   URL wrappers to match the underlying vendor URL.

2. **Claude.ai vs. manual** (`dedupClaudeAiMcpServers`): Claude.ai connectors
   whose URL signature matches an enabled manual server are suppressed. Only enabled
   manual servers count as dedup targets — a disabled manual server must not suppress
   its connector twin.

3. **Config-based** (`areMcpConfigsEqual`): When configs change (e.g., edited
   `.mcp.json`), servers with changed config hashes are treated as stale and
   reconnected with the new config.

### 10.3 Deny Rule Filtering

Tool visibility is filtered at multiple levels:

- **Enterprise policy** (`allowedMcpServers` / `deniedMcpServers`): Servers
  blocked by policy never connect
- **User disable** (`disabledMcpServers` in project config): Disabled servers are
  tracked in AppState but skip connection
- **IDE tool filtering** (`isIncludedMcpTool`): Only `executeCode` and
  `getDiagnostics` are included from IDE MCP servers — other IDE tools are hidden

### 10.4 Auth Tool Injection

When a server is in the `needs-auth` state, a synthetic `McpAuthTool` is injected
into its tool list. This tool, when called by the model, triggers the OAuth
authentication flow for that server.

---

## 11. System Prompt Integration

### 11.1 MCP Server Instructions

> **Source:** `src/utils/mcpInstructionsDelta.ts`

MCP servers can provide `instructions` in their `InitializeResult`. These are
server-authored behavioral hints (e.g., "prefer using search before listing") that
inform the model about how to use the server effectively.

Instructions are integrated into the conversation through a **delta-based
announcement** system:

- `getMcpInstructionsDelta()` compares currently connected servers (that have
  instructions) against what has been announced in the conversation history
- Newly connected servers produce "added" blocks; disconnected servers produce
  "removed" entries
- Deltas are persisted as `mcp_instructions_delta` attachment messages
- Instructions are capped at `MAX_MCP_DESCRIPTION_LENGTH = 2048` characters

Two modes exist (feature-gated):
- **Delta mode** (`isMcpInstructionsDeltaEnabled()`): Announce via persisted
  delta attachments — cache-friendly, survives compaction
- **Legacy mode**: Rebuild `DANGEROUS_uncachedSystemPromptSection` every turn —
  cache-busts on late connects

### 11.2 Client-Side Instructions

`ClientSideInstruction` allows Claude Code to attach its own instruction blocks to
specific servers, independent of the server's `InitializeResult`. This is used for
first-party servers (e.g., Chrome MCP) where the client has context the server
doesn't know about.

### 11.3 Tool Descriptions in the Prompt

Each MCP tool's description is included in the model's system prompt as part of the
tool definitions, subject to the 2048-character cap. Tool input schemas are passed
through directly, providing the model with structured type information about each
tool's parameters.

---

## 12. Design Principles

### 12.1 Transport Polymorphism

Five transports present a uniform interface to the rest of the system. The transport
selection happens once during `connectToServer()` — everything downstream (tool
calls, resource reads, notification handling) is transport-agnostic. This allows new
transports to be added without modifying the tool wrapping, state management, or UI
layers.

### 12.2 Fail-Open Configuration, Fail-Closed Policy

The configuration system is permissive: missing env vars produce warnings not errors,
missing `.mcp.json` files are silently ignored, and individual server failures do not
block startup. But the policy system is strict: the deny list is absolute, the
allow list is exhaustive when present, and enterprise config is exclusive.

### 12.3 Cache-Aware Memoization

Connection and fetch results are memoized at exactly the right granularity:
- `connectToServer` is memoized by `name + JSON(config)` — reconnects on config
  change
- `fetch*ForClient` are LRU-cached by server name — survive reconnects (same
  server, fresh connection), invalidated on `list_changed` notifications
- Auth discovery is memoized per server key — avoids repeated OAuth discovery

Cache invalidation is explicit and surgical: `onclose` clears both the connection
cache and all three fetch caches for that server.

### 12.4 Graceful Degradation

Every failure boundary is isolated:
- One server's connection failure does not prevent other servers from connecting
- One server's tool fetch failure returns empty tools, not an exception
- Auth cache prevents repeated 401 probes (15-minute TTL)
- Reconnection has a backoff cap and attempt limit
- Terminal error counter prevents infinite onerror loops
- Binary content that cannot be persisted falls back to an error text message

### 12.5 Batched State Management

Server connection results arrive asynchronously from parallel connection attempts.
Rather than triggering N React re-renders for N servers, updates are queued and
flushed in a single `setAppState` call every 16ms. This keeps the UI responsive
during startup when tens of servers may connect simultaneously.

### 12.6 Security Layering

MCP servers are untrusted external code. The system defends against this:
- **Unicode sanitization**: `recursivelySanitizeUnicode()` on all tool and prompt
  data from servers
- **Description truncation**: 2048-char cap prevents context window flooding
- **Policy enforcement**: Enterprise allow/deny lists with URL pattern matching
- **Project server approval**: `.mcp.json` servers from the project scope require
  explicit user approval before connecting
- **OAuth security**: PKCE, lockfile serialization, secure storage for tokens,
  sensitive parameter redaction in logs
- **Namespace isolation**: `mcp__` prefix prevents MCP tools from colliding with
  or impersonating built-in tools
- **Permission passthrough**: MCP tools route through the full permission pipeline
  (the same Swiss cheese model as built-in tools)

### 12.7 Startup Latency Optimization

The configuration pipeline is engineered for fast startup:
- Local configs are loaded synchronously (file reads only)
- The claude.ai connector fetch is kicked off in parallel and awaited only at the
  dedup step
- Plugin loading uses cache-only mode (`loadAllPluginsCacheOnly`)
- Local and remote servers are connected with separate concurrency limits
- `pMap` provides work-stealing scheduling instead of fixed-size batch boundaries
