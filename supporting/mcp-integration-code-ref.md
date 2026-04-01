# MCP Integration — Code Reference

This section contains all implementation-specific details: file paths, function/class/variable names, line references, code snippets, TypeScript types, import paths, and codebase-specific information.

---

### Source Files

| Area | Primary Files |
|---|---|
| MCP service module | `src/services/mcp/` (~22 source files) |
| MCP tool wrapper | `src/tools/MCPTool/` |
| Resource list tool | `src/tools/ListMcpResourcesTool/` |
| Resource read tool | `src/tools/ReadMcpResourceTool/` |

### Types and State Model

**File:** `src/services/mcp/types.ts`

- `MCPServerConnection`: Discriminated union with states `connected`, `failed`, `needs-auth`, `pending`, `disabled`
- `ConfigScope` (lines 10-20): Union type of `'claudeai' | 'dynamic' | 'user' | 'project' | 'local' | 'enterprise' | 'managed'`
- `McpServerConfigSchema`: Zod union of all transport-specific schemas

### Transport Implementations

**File:** `src/services/mcp/client.ts` (lines 595-961)

- `SSEClientTransport` from `@modelcontextprotocol/sdk`
- `StreamableHTTPClientTransport` from `@modelcontextprotocol/sdk`
- `StdioClientTransport` from `@modelcontextprotocol/sdk`
- `wrapFetchWithTimeout()`: Wraps fetch with 60s timeout using `setTimeout` + `AbortController`; excludes GET requests
- `getMcpServerHeaders()`: Combines static and dynamic headers for SSE/HTTP
- `isMcpSessionExpiredError()`: Detects HTTP 404 + JSON-RPC error code -32001
- `getConnectionTimeoutMs()`: Returns 30s default, overridable via `MCP_TIMEOUT` env var

**File:** `src/utils/mcpWebSocketTransport.js`

- Custom `WebSocketTransport` wrapper
- `createNodeWsClient()`: Node.js `ws` package WebSocket creation
- `getWebSocketTLSOptions()`: mTLS configuration
- `getWebSocketProxyAgent()`: Proxy configuration
- Subprotocol: `"mcp"`

**File:** `src/services/mcp/InProcessTransport.ts`

- `createLinkedTransportPair()`: Creates two `InProcessTransport` instances connected back-to-back
- Uses `queueMicrotask()` for message delivery
- `isClaudeInChromeMCPServer()`: Detection predicate for Chrome MCP
- `isComputerUseMCPServer()`: Detection predicate for Computer Use MCP

**File:** `src/services/mcp/SdkControlTransport.ts`

- `SdkControlClientTransport`: Wraps JSON-RPC in control messages over structured I/O
- Config type: `sdk`
- `connectToServer()` throws for SDK configs (handled via bridge instead)

### Client Instance

**File:** `src/services/mcp/client.ts` (lines 985-1001)

- Client from `@modelcontextprotocol/sdk`
- Client info: `{ name: 'claude-code', version: <build macro> }`
- Capabilities: `{ roots: {}, elicitation: {} }`
- Registers `ListRootsRequestSchema` handler returning `file://` URI of working directory

### Connection Caching

**File:** `src/services/mcp/client.ts`

- `connectToServer()`: Wrapped with lodash `memoize`
- `getServerCacheKey(name, serverRef)`: Key = server name + `JSON.stringify(config)`
- `clearServerCache()`: Explicit eviction + cleanup
- `ensureConnectedClient()`: Safe entry point; throws if not connected
- `MCP_FETCH_CACHE_SIZE = 20`: LRU cache bound
- `fetchToolsForClient`: Cached `tools/list`
- `fetchResourcesForClient`: Cached `resources/list`
- `fetchCommandsForClient`: Cached `prompts/list`

### Configuration

**File:** `src/services/mcp/config.ts`

- `getAllMcpConfigs()` (lines 1258-1290): Complete configuration pipeline
- `filterMcpServersByPolicy()` (lines 364-551): Policy enforcement public API
- `deniedMcpServers`, `allowedMcpServers`, `allowManagedMcpServersOnly`: Policy settings
- Enterprise config file path: `/etc/claude-code/managed-mcp.json`
- User config: `~/.claude/settings.json` (`mcpServers` property)
- Local config: `.claude/settings.local.json` (`mcpServers` property)
- Project config: `.mcp.json` (walked up from `getCwd()` to filesystem root)

**File:** `src/services/mcp/envExpansion.ts`

- `expandEnvVars()`: Processes `${VAR}` and `${VAR:-default}` syntax

**File:** `src/services/mcp/normalization.ts`

- Server name regex: `^[a-zA-Z0-9_-]{1,64}$`
- Claude.ai server name prefix: `"claude.ai "`

### Authentication

**File:** `src/services/mcp/auth.ts`

- `ClaudeAuthProvider`: Implements `OAuthClientProvider` from MCP SDK
- Token storage key: SHA-256 hash of server config
- Retry backoff: 1s, 2s, 4s for `TemporarilyUnavailableError`, `TooManyRequestsError`, `ServerError`
- `MCP_CLIENT_METADATA_URL`: URL for DCR client metadata
- `fetchAuthServerMetadata()`: Multi-step OAuth discovery
- OAuth lockfile: `~/.claude/mcp-auth-{hash}.lock`
- `findAvailablePort()`: Port selection for local callback server

**File:** `src/services/mcp/xaa.ts`, `src/services/mcp/xaaIdpLogin.ts`

- XAA (SEP-990): Cross-App Access implementation
- RFC 8693 token exchange, RFC 7523 JWT Bearer grant
- `settings.xaaIdp`: IdP configuration (issuer, clientId, callback port)

**File:** `src/services/mcp/client.ts` (lines 372-422)

- `createClaudeAiProxyFetch()`: Proxy fetch wrapper for claude.ai connectors
- `oauthConfig.MCP_PROXY_URL`: Proxy URL
- `handleOAuth401Error()`: Force-refresh token on 401
- `wrapFetchWithStepUpDetection()`: Intercepts HTTP 403 with `WWW-Authenticate: Bearer`

**Auth cache file:** `mcp-needs-auth-cache.json` (15-minute TTL)
- `hasMcpDiscoveryButNoToken()`: Secondary needs-auth check

### Connection Lifecycle

**File:** `src/services/mcp/MCPConnectionManager.tsx`

- `MCPConnectionManager`: React context provider
- `useManageMCPConnections()`: Hook for connection management
- `reconnectMcpServer(serverName)`: Manual reconnection
- `toggleMcpServer(serverName)`: Enable/disable toggle

**File:** `src/services/mcp/useManageMCPConnections.ts`

- Lines 858-1024: Two-phase startup initialization
- Lines 207-308: Batched state update logic
- `pendingUpdatesRef`: Queue for pending state updates
- 16ms `setTimeout` debounce window for batch flushing
- `processBatched()`: Backed by `pMap` for parallel connection
- `getMcpServerConnectionBatchSize()`: Returns 3 (default local concurrency)
- `getRemoteMcpServerConnectionBatchSize()`: Returns 20 (default remote concurrency)
- `getMcpToolsCommandsAndResources()`: Tool/command/resource loading
- `resourceToolsAdded` flag: Prevents duplicate resource tool injection
- `initializeServersAsPending()`: Detects and cleans up stale servers

### Reconnection Constants

- `MAX_RECONNECT_ATTEMPTS = 5`
- `INITIAL_BACKOFF_MS = 1000`
- `MAX_BACKOFF_MS = 30000`
- Formula: `min(1000 * 2^(attempt-1), 30000)`

### Error Handling

**File:** `src/services/mcp/client.ts` (lines 1249-1371)

- Terminal error codes: `ECONNRESET`, `ETIMEDOUT`, `EPIPE`, `EHOSTUNREACH`, `ECONNREFUSED`
- Consecutive terminal error threshold: 3
- `closeTransportAndRejectPending()`: Calls `client.close()` (not `client.onclose?.()`)
- Rejects pending `callTool()` promises with `McpError -32000`
- `hasTriggeredClose`: Re-entry guard flag

### Graceful Shutdown

**File:** `src/services/mcp/client.ts` (lines 1404-1580)

- `registerCleanup()`: Registers transport-specific cleanup
- STDIO signal sequence: `SIGINT` (100ms) -> `SIGTERM` (400ms) -> `SIGKILL`
- Absolute failsafe timeout: 600ms
- Environment variable: `CLAUDE_CODE_SHELL_PREFIX`

### Tool Wrapping

**File:** `src/tools/MCPTool/MCPTool.ts`

- `MCPTool`: Built with `buildTool()`
- `isMcp: true` flag
- Input schema: `z.object({}).passthrough()`
- `mcpInfo: { serverName, toolName }`

**File:** `src/services/mcp/client.ts` (lines 1743-1998)

- `fetchToolsForClient()`: Transforms MCP tools to internal Tool objects
- Tool name format: `mcp__<serverName>__<toolName>`
- SDK prefix env var: `CLAUDE_AGENT_SDK_MCP_NO_PREFIX`
- `userFacingName()`: `"serverName - toolTitle (MCP)"`
- `MAX_MCP_DESCRIPTION_LENGTH = 2048`
- Annotations mapped: `readOnlyHint`, `destructiveHint`, `openWorldHint`
- `_meta['anthropic/searchHint']`: Search hint metadata
- `_meta['anthropic/alwaysLoad']`: Always-load metadata
- `callMCPToolWithUrlElicitationRetry()`: Tool call with auth retry
- `McpSessionExpiredError`: Single retry on session expiry
- `TelemetrySafeError`: Error wrapper for analytics
- `DEFAULT_MCP_TOOL_TIMEOUT_MS = 100_000_000` (~27.8 hours)
- `MCP_TOOL_TIMEOUT` env var: Override

**File:** `src/tools/MCPTool/classifyForCollapse.ts`

- `classifyMcpToolForCollapse()`: Classification function
- `SEARCH_TOOLS`: ~140 entries static allowlist
- `READ_TOOLS`: ~400 entries static allowlist
- Normalization: camelCase/kebab-case to snake_case

### Resource Tools

**File:** `src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts`

- Returns: `{ uri, name, mimeType, description, server }[]`
- Flags: `isConcurrencySafe`, `isReadOnly`, `shouldDefer`
- Input parameter: `server` (optional, filter to one server)

**File:** `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts`

- MCP method: `resources/read`
- `persistBinaryContent()`: Writes base64 blobs to disk with MIME-derived extension
- Returns: `{ contents: [{ uri, mimeType, text?, blobSavedTo? }] }`

### Deduplication

- `dedupPluginMcpServers()`: Plugin vs. manual dedup by signature (stdio command array or URL)
- `dedupClaudeAiMcpServers()`: Claude.ai vs. manual dedup by URL signature
- `areMcpConfigsEqual()`: Config hash comparison for stale detection
- Plugin server key format: `plugin:<pluginName>:<serverName>`
- CCR proxy URL stripping in signature matching

### Stale Server Cleanup

**File:** `src/services/mcp/utils.ts`

- `excludeStalePluginClients`: Stale server detection utility

### System Prompt Integration

**File:** `src/utils/mcpInstructionsDelta.ts`

- `getMcpInstructionsDelta()`: Compares connected servers against announced history
- `mcp_instructions_delta`: Attachment message type for persisted deltas
- `isMcpInstructionsDeltaEnabled()`: Feature gate for delta mode
- `DANGEROUS_uncachedSystemPromptSection`: Legacy mode system prompt section
- `ClientSideInstruction`: Client-authored instruction type

### IDE Tool Filtering

- `isIncludedMcpTool()`: Only allows `executeCode` and `getDiagnostics` from IDE MCP servers

### Auth Tool

- `McpAuthTool`: Synthetic tool injected for `needs-auth` servers

### Subprocess Environment

- `subprocessEnv()`: Returns sanitized parent environment for STDIO servers

### Plugin Loading

- `loadAllPluginsCacheOnly`: Cache-only mode for fast startup
