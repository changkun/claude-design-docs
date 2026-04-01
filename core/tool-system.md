# Claude Code: Tool System — Design Specification

This document analyzes the extensible tool architecture of Claude Code — Anthropic's
agentic CLI tool — focusing on how tools are defined, registered, composed, permission-gated,
and executed across the lifecycle of an agentic turn.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Tool Interface Contract](#2-tool-interface-contract)
- [3. Tool Registry](#3-tool-registry)
- [4. Tool Categories](#4-tool-categories)
- [5. Tool Availability Matrix](#5-tool-availability-matrix)
- [6. Permission Integration](#6-permission-integration)
- [7. MCP Tool Composition](#7-mcp-tool-composition)
- [8. Deferred Loading](#8-deferred-loading)
- [9. Tool Execution Flow](#9-tool-execution-flow)
- [10. Tool Progress Reporting](#10-tool-progress-reporting)
- [11. Representative Tool Patterns](#11-representative-tool-patterns)
- [12. Design Principles](#12-design-principles)

---

## 1. Overview

The tool system is the mechanism through which Claude Code acts on the world. Every
filesystem read, shell command, file edit, subagent launch, web search, and MCP
integration flows through a single, uniform tool abstraction. The system's design
goals are:

1. **Uniform interface** — every capability, whether built-in or dynamically discovered
   via MCP, exposes the same `Tool` type to the model, the permission system, and the
   rendering pipeline.
2. **Fail-closed permissions** — no tool executes without passing through validation,
   permission checking, and optionally user confirmation. The defaults are conservative:
   tools are assumed to write, assumed not concurrency-safe, and assumed to need
   permission checks.
3. **Extensibility without modification** — MCP servers can add arbitrary tools at
   runtime without changing any built-in code. The composition layer (`assembleToolPool`)
   merges them with built-in tools transparently.
4. **Lazy discovery** — when the tool count grows large (especially with MCP servers),
   most tools are "deferred" behind a `ToolSearchTool` that loads their full schemas
   on demand, keeping the initial prompt lean.

The tool system spans roughly 50 tool implementations across `src/tools/`, a central
registry in `src/tools.ts`, type definitions in `src/Tool.ts`, availability constants
in `src/constants/tools.ts`, and the execution engine in
`src/services/tools/toolExecution.ts`.

---

## 2. Tool Interface Contract

> **Source:** `src/Tool.ts`

### 2.1 The Tool Type

The `Tool` type is the core abstraction. It is a generic parameterized by three type
variables — `Input` (a Zod object schema), `Output` (the result data type), and `P`
(a progress event type) — and contains roughly 50 properties spanning identity,
schema, execution, permissions, rendering, and metadata.

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // Identity
  readonly name: string
  aliases?: string[]
  searchHint?: string

  // Schema
  readonly inputSchema: Input
  readonly inputJSONSchema?: ToolInputJSONSchema   // MCP tools use raw JSON Schema
  outputSchema?: z.ZodType<unknown>

  // Execution
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  validateInput?(input, context): Promise<ValidationResult>
  checkPermissions(input, context): Promise<PermissionResult>

  // Behavioral metadata
  isConcurrencySafe(input): boolean
  isEnabled(): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isOpenWorld?(input): boolean
  interruptBehavior?(): 'cancel' | 'block'
  isSearchOrReadCommand?(input): { isSearch; isRead; isList? }
  requiresUserInteraction?(): boolean

  // MCP metadata
  isMcp?: boolean
  isLsp?: boolean
  mcpInfo?: { serverName: string; toolName: string }

  // Deferred loading
  readonly shouldDefer?: boolean
  readonly alwaysLoad?: boolean

  // Permission integration
  getPath?(input): string
  preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>
  backfillObservableInput?(input): void
  toAutoClassifierInput(input): unknown

  // Result size management
  maxResultSizeChars: number
  readonly strict?: boolean

  // Prompt generation
  description(input, options): Promise<string>
  prompt(options): Promise<string>

  // Rendering (UI)
  userFacingName(input): string
  userFacingNameBackgroundColor?(input): keyof Theme | undefined
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  renderToolUseProgressMessage?(progressMessages, options): React.ReactNode
  renderToolUseRejectedMessage?(input, options): React.ReactNode
  renderToolUseErrorMessage?(result, options): React.ReactNode
  renderToolUseQueuedMessage?(): React.ReactNode
  renderToolUseTag?(input): React.ReactNode
  renderGroupedToolUse?(toolUses, options): React.ReactNode | null
  isTransparentWrapper?(): boolean
  isResultTruncated?(output): boolean

  // Summary and search
  getToolUseSummary?(input): string | null
  getActivityDescription?(input): string | null
  extractSearchText?(out): string

  // Result mapping
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
  inputsEquivalent?(a, b): boolean
}
```

The `Tools` type is `readonly Tool[]` — a named alias used consistently across the
codebase to track where tool collections are assembled, passed, and filtered.

### 2.2 The buildTool Factory

> **Source:** `src/Tool.ts:757-792`

Rather than requiring every tool implementation to fill in all ~50 properties, the
`buildTool` function provides fail-closed defaults for commonly-stubbed methods:

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,    // assume not safe
  isReadOnly: () => false,            // assume writes
  isDestructive: () => false,
  checkPermissions: (input) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',    // skip classifier
  userFacingName: () => '',
}
```

Tool definitions use the `ToolDef` type, which makes all defaultable keys optional.
The `buildTool` function merges the definition with `TOOL_DEFAULTS` via object spread:

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

This ensures:
- All tool exports are complete `Tool` objects — callers never need `?.() ?? default`
- Defaults are fail-closed: `isConcurrencySafe` defaults to `false`, `isReadOnly`
  defaults to `false`
- Security-relevant tools must **explicitly override** `toAutoClassifierInput` — the
  default skips classifier analysis

### 2.3 The ToolResult Type

Every tool `call()` returns a `ToolResult<T>`:

```typescript
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

The `data` field carries the tool's primary output. `newMessages` allows a tool to
inject additional messages into the conversation (e.g., AgentTool appending subagent
transcripts). `contextModifier` allows non-concurrency-safe tools to mutate the
`ToolUseContext` between turns (e.g., changing the working directory). `mcpMeta`
passes through MCP protocol metadata to SDK consumers.

### 2.4 The ToolUseContext

Every tool receives a `ToolUseContext` — a rich environment struct carrying:

- **`options`**: static session configuration (tools list, model, MCP clients,
  thinking config, agent definitions, budget)
- **`abortController`**: cooperative cancellation signal
- **`readFileState`**: an LRU cache of recently-accessed files (for stale-write
  detection and post-compaction restoration)
- **`messages`**: the full conversation history
- **State accessors**: `getAppState()`, `setAppState()`, `setToolJSX()`, etc.
- **UI callbacks**: `addNotification`, `sendOSNotification`, `setStreamMode`
- **Tracking state**: `toolDecisions`, `queryTracking`, `fileReadingLimits`
- **Content replacement**: `contentReplacementState` for the tool result budget

This struct is the tool's view of the entire session. Subagent tools receive a
modified copy via `createSubagentContext` that restricts state mutation (e.g.,
`setAppState` becomes a no-op for async agents to prevent cross-thread race conditions).

---

## 3. Tool Registry

> **Source:** `src/tools.ts`

### 3.1 getAllBaseTools — The Exhaustive Source of Truth

`getAllBaseTools()` returns the complete list of all tools that **could be available**
in the current environment. It is the single source of truth for tool existence.

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    AskUserQuestionTool,
    SkillTool,
    EnterPlanModeTool,
    // ... conditionally included tools ...
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
  ]
}
```

Tools are included conditionally based on:
- **Feature flags** (`feature('KAIROS')`, `feature('COORDINATOR_MODE')`, etc.)
- **Environment variables** (`process.env.USER_TYPE === 'ant'`)
- **Runtime checks** (`isAgentSwarmsEnabled()`, `isWorktreeModeEnabled()`)
- **Test mode** (`process.env.NODE_ENV === 'test'`)

Conditional tools use `require()` (not static `import`) for dead code elimination —
the bundler can tree-shake entire tool implementations when their feature flag is off.

### 3.2 getTools — Permission-Filtered Built-in Tools

`getTools(permissionContext)` starts from `getAllBaseTools()` and applies three filters:

1. **Special tool exclusion**: `ListMcpResourcesTool`, `ReadMcpResourceTool`, and
   `SyntheticOutputTool` are excluded (they are added conditionally elsewhere)
2. **Deny rule filtering**: `filterToolsByDenyRules()` removes tools that match a
   blanket deny rule (no `ruleContent` — i.e., the entire tool is denied, not just
   specific inputs)
3. **REPL mode filtering**: when REPL mode is enabled, primitive tools (Bash, Read,
   Edit, etc.) are hidden — they are accessible only through the REPL VM context
4. **isEnabled() check**: each tool's `isEnabled()` is called; disabled tools are
   dropped

A "simple mode" (`CLAUDE_CODE_SIMPLE`) reduces the tool set to just Bash, FileRead,
and FileEdit.

### 3.3 assembleToolPool — The Full Tool Set

`assembleToolPool(permissionContext, mcpTools)` is the **single source of truth** for
combining built-in and MCP tools:

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // Sort each partition for prompt-cache stability
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

Key design decisions:
- **Built-in tools take precedence**: `uniqBy` preserves insertion order, so when a
  name collides between built-in and MCP, built-in wins
- **Sorted partitions**: built-ins are sorted as a contiguous prefix, MCP tools as
  a separate suffix, preventing interleaving that would invalidate the server-side
  prompt cache
- **Deny rules apply to MCP tools**: server-prefix rules like `mcp__server` strip
  all tools from a server before the model sees them

### 3.4 getMergedTools — Count and Token Estimation

`getMergedTools(permissionContext, mcpTools)` produces a simple concatenation of
built-in and MCP tools. Used where ordering doesn't matter — threshold calculations
for ToolSearch enablement, token counting, and any context where MCP tools should
be considered alongside built-ins.

---

## 4. Tool Categories

The ~50 tool implementations in `src/tools/` group into seven functional categories:

### 4.1 File System Tools

| Tool | Purpose |
|---|---|
| `FileReadTool` | Read files (text, images, PDFs, notebooks, directories) |
| `FileEditTool` | Exact string replacement in existing files |
| `FileWriteTool` | Create or overwrite files |
| `NotebookEditTool` | Edit Jupyter notebook cells |
| `GlobTool` | Pattern-based file search |
| `GrepTool` | Content search via ripgrep |

These tools form the bedrock of codebase interaction. `FileReadTool` handles an
especially wide range of formats — text files with line-numbered output, images
(resized and returned as base64), PDFs (extracted via page range), Jupyter notebooks
(all cells with outputs), and directories (listed as file trees). `FileEditTool`
implements exact-string-replacement semantics with uniqueness enforcement, stale-write
detection via `readFileState`, and git diff tracking for undo support.

### 4.2 Execution Tools

| Tool | Purpose |
|---|---|
| `BashTool` | Execute shell commands (with sandbox support) |
| `PowerShellTool` | Execute PowerShell commands (Windows) |
| `REPLTool` | VM-sandboxed execution wrapping primitive tools (ant-only) |
| `TungstenTool` | Virtual terminal interaction (ant-only) |

`BashTool` is the most complex tool implementation (~3000 lines). It handles timeout
management, background task spawning, sandbox mode (macOS Seatbelt), sed edit
interception, image output detection, and an elaborate permission system that parses
shell commands into AST nodes for fine-grained rule matching.

### 4.3 Agent and Task Tools

| Tool | Purpose |
|---|---|
| `AgentTool` | Launch subagents (sync, async, forked, remote, teammate) |
| `TaskOutputTool` | Read output from background tasks |
| `TaskStopTool` | Stop running tasks |
| `TaskCreateTool` / `TaskGetTool` / `TaskUpdateTool` / `TaskListTool` | TODO v2 task management |
| `TodoWriteTool` | Write to the plan/todo panel |

`AgentTool` is the second most complex implementation. It supports five distinct launch
modes — synchronous foreground, asynchronous background, forked (prompt-cache-sharing),
remote (CCR environments), and teammate (multi-agent swarm). Each mode has its own
lifecycle, progress reporting, and tool filtering.

### 4.4 Planning and Mode Tools

| Tool | Purpose |
|---|---|
| `EnterPlanModeTool` | Switch to plan mode |
| `ExitPlanModeV2Tool` | Exit plan mode |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Enter/exit git worktree isolation |

### 4.5 Integration Tools

| Tool | Purpose |
|---|---|
| `MCPTool` | Base template for all MCP tool instances |
| `ListMcpResourcesTool` | List resources from MCP servers |
| `ReadMcpResourceTool` | Read a specific MCP resource |
| `McpAuthTool` | Handle MCP server authentication |
| `LSPTool` | Language Server Protocol queries |
| `WebFetchTool` | Fetch and process web content |
| `WebSearchTool` | Web search via API |

### 4.6 Communication and Notification Tools

| Tool | Purpose |
|---|---|
| `SendMessageTool` | Send messages between agents |
| `AskUserQuestionTool` | Prompt the user for input |
| `BriefTool` | Primary communication channel (KAIROS) |
| `SendUserFileTool` | Deliver files to the user (KAIROS) |
| `PushNotificationTool` | Push notifications (KAIROS) |
| `ListPeersTool` | List peer agents (UDS inbox) |

### 4.7 Configuration and Discovery Tools

| Tool | Purpose |
|---|---|
| `ToolSearchTool` | Discover and load deferred tool schemas |
| `SkillTool` | Invoke slash-command skills |
| `ConfigTool` | Read/write settings (ant-only) |
| `VerifyPlanExecutionTool` | Verify plan execution (debug) |
| `SnipTool` | Snip conversation segments |
| `WorkflowTool` | Execute bundled workflows |

---

## 5. Tool Availability Matrix

> **Source:** `src/constants/tools.ts`

### 5.1 Always-Available Tools

The following tools are always present in `getAllBaseTools()` regardless of feature
flags (though they may still be filtered by deny rules or `isEnabled()`):

AgentTool, TaskOutputTool, BashTool, FileReadTool, FileEditTool, FileWriteTool,
NotebookEditTool, WebFetchTool, TodoWriteTool, WebSearchTool, TaskStopTool,
AskUserQuestionTool, SkillTool, EnterPlanModeTool, ExitPlanModeV2Tool,
SendMessageTool, BriefTool.

GlobTool and GrepTool are present unless `hasEmbeddedSearchTools()` returns true
(ant-native builds embed bfs/ugrep in the binary).

### 5.2 Conditionally Available Tools

| Tool | Gate |
|---|---|
| ConfigTool, TungstenTool, REPLTool | `process.env.USER_TYPE === 'ant'` |
| SuggestBackgroundPRTool | `process.env.USER_TYPE === 'ant'` |
| WebBrowserTool | `feature('WEB_BROWSER_TOOL')` |
| TaskCreate/Get/Update/List | `isTodoV2Enabled()` |
| OverflowTestTool | `feature('OVERFLOW_TEST_TOOL')` |
| CtxInspectTool | `feature('CONTEXT_COLLAPSE')` |
| TerminalCaptureTool | `feature('TERMINAL_PANEL')` |
| LSPTool | `ENABLE_LSP_TOOL` env var |
| EnterWorktreeTool, ExitWorktreeTool | `isWorktreeModeEnabled()` |
| TeamCreateTool, TeamDeleteTool | `isAgentSwarmsEnabled()` |
| ListPeersTool | `feature('UDS_INBOX')` |
| WorkflowTool | `feature('WORKFLOW_SCRIPTS')` |
| SleepTool | `feature('PROACTIVE')` or `feature('KAIROS')` |
| CronCreate/Delete/ListTool | `feature('AGENT_TRIGGERS')` |
| RemoteTriggerTool | `feature('AGENT_TRIGGERS_REMOTE')` |
| MonitorTool | `feature('MONITOR_TOOL')` |
| SendUserFileTool | `feature('KAIROS')` |
| PushNotificationTool | `feature('KAIROS')` or `feature('KAIROS_PUSH_NOTIFICATION')` |
| SubscribePRTool | `feature('KAIROS_GITHUB_WEBHOOKS')` |
| PowerShellTool | `isPowerShellToolEnabled()` |
| SnipTool | `feature('HISTORY_SNIP')` |
| ToolSearchTool | `isToolSearchEnabledOptimistic()` |

### 5.3 Agent Disallowed Tools

Several tools are explicitly blocked for subagents to prevent recursion and
preserve abstraction boundaries:

```typescript
export const ALL_AGENT_DISALLOWED_TOOLS = new Set([
  TASK_OUTPUT_TOOL_NAME,
  EXIT_PLAN_MODE_V2_TOOL_NAME,
  ENTER_PLAN_MODE_TOOL_NAME,
  AGENT_TOOL_NAME,               // unless ant-only (nested agents)
  ASK_USER_QUESTION_TOOL_NAME,
  TASK_STOP_TOOL_NAME,
  WORKFLOW_TOOL_NAME,             // prevents recursive workflow execution
])
```

### 5.4 Async Agent Allowed Tools

Background/async agents receive a strict allowlist rather than a denylist:

```typescript
export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  FILE_READ_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  TODO_WRITE_TOOL_NAME,
  GREP_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  GLOB_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
  NOTEBOOK_EDIT_TOOL_NAME,
  SKILL_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
  TOOL_SEARCH_TOOL_NAME,
  ENTER_WORKTREE_TOOL_NAME,
  EXIT_WORKTREE_TOOL_NAME,
])
```

Notably absent: `AgentTool` (prevents recursion), `TaskOutputTool` (main thread
abstraction), `MCPTool` (TBD), `TungstenTool` (singleton conflict).

### 5.5 Coordinator Mode Tools

In coordinator mode, the coordinator itself gets only orchestration tools:

```typescript
export const COORDINATOR_MODE_ALLOWED_TOOLS = new Set([
  AGENT_TOOL_NAME,
  TASK_STOP_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
```

Workers get the full tool set minus the coordinator-only tools.

---

## 6. Permission Integration

> **Source:** `src/Tool.ts`, `src/types/permissions.ts`, `src/hooks/useCanUseTool.tsx`,
> `src/services/tools/toolExecution.ts`

### 6.1 Permission Behaviors

The permission system uses a four-behavior model:

```typescript
export type PermissionBehavior = 'allow' | 'deny' | 'ask'
```

A fourth informal behavior — `'passthrough'` — is used by tools like MCPTool and
the `buildTool` default to signal "I have no opinion; defer to the general permission
system."

### 6.2 The checkPermissions Method

Every tool implements `checkPermissions(input, context) -> PermissionResult`. This
is **tool-specific** permission logic, called after input validation passes. The
general permission system (`hasPermissionsToUseTool` in `src/utils/permissions/permissions.ts`)
wraps this with mode-based rules, deny/allow lists, and classifier checks.

Tool-specific implementations vary widely:

- **FileEditTool / FileWriteTool**: delegate to `checkWritePermissionForTool()` which
  evaluates the target path against the working directory and permission rules
- **FileReadTool**: delegates to `checkReadPermissionForTool()`
- **BashTool**: the most complex — parses the command into an AST, checks each
  subcommand against prefix rules, evaluates sandbox eligibility, checks redirect
  paths, and returns compound permission results with per-subcommand reasons
- **MCPTool**: returns `{ behavior: 'passthrough' }` with suggested allow rules,
  deferring entirely to the general system
- **Default (from buildTool)**: returns `{ behavior: 'allow' }` — tools that don't
  override `checkPermissions` are auto-allowed

### 6.3 The preparePermissionMatcher Method

For hook `if` conditions (patterns like `Bash(git *)`), tools can provide a custom
matcher. This is called **once** per hook-input pair; expensive parsing happens
here. The returned closure is called per pattern.

BashTool's implementation parses the command into an AST, extracts subcommands (stripping
`VAR=val` prefixes), and matches each subcommand against the pattern. For compound
commands (`ls && git push`), **any** subcommand matching triggers the hook — this
prevents security hooks from being bypassed by chaining.

### 6.4 The backfillObservableInput Method

Before hooks and permission checks see the input, `backfillObservableInput` can
mutate a **cloned** copy to add legacy/derived fields. The canonical use is
FileEditTool expanding `~` and relative paths in `file_path` so hook allowlists
can't be bypassed via path tricks. The original API-bound input is never mutated
(preserves prompt cache). Hook-returned `updatedInput` objects are not re-backfilled —
they own their shape.

### 6.5 Permission Modes

The system supports several permission modes that affect how `checkPermissions` results
are interpreted:

| Mode | Behavior |
|---|---|
| `default` | Writes prompt the user; reads auto-allowed |
| `acceptEdits` | File edits auto-allowed; other writes prompt |
| `bypassPermissions` | Everything auto-allowed (dangerous) |
| `plan` | All writes require plan approval |
| `dontAsk` | Auto-deny anything that would prompt |
| `auto` | Classifier decides allow/deny without user prompting |

### 6.6 The toAutoClassifierInput Method

In `auto` mode, a security classifier evaluates tool calls. Each tool provides a
compact representation of its input via `toAutoClassifierInput()`:

- **BashTool**: returns `input.command` (the raw shell command)
- **FileEditTool**: returns `"${file_path}: ${new_string}"`
- **Default**: returns `''` (skips classifier — only security-relevant tools override)

### 6.7 Permission Decision Flow

The full permission flow in `useCanUseTool`:

```
tool.checkPermissions(input, context)
        │
        ▼
hasPermissionsToUseTool(tool, input, context)
        │
        ├── behavior: 'allow' → resolve immediately
        │
        ├── behavior: 'deny' → resolve denied (log auto-mode denial if applicable)
        │
        └── behavior: 'ask'
              │
              ├── coordinator mode? → handleCoordinatorPermission
              ├── swarm worker? → handleSwarmWorkerPermission
              ├── classifier available? → speculative check + race
              └── fallback → handleInteractivePermission (show dialog)
```

---

## 7. MCP Tool Composition

> **Source:** `src/services/mcp/client.ts:1765-1862`, `src/tools/MCPTool/MCPTool.ts`

### 7.1 The MCPTool Template

`MCPTool.ts` defines a template object — a minimal `Tool` with stub implementations
for `name`, `description`, `prompt`, and `call`. All of these are marked as
"overridden in mcpClient.ts." The template exists so that MCP tools share common
rendering logic (`renderToolUseMessage`, `renderToolResultMessage`,
`renderToolUseProgressMessage`) while having per-tool identity and behavior.

```typescript
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',
  maxResultSizeChars: 100_000,
  async checkPermissions(): Promise<PermissionResult> {
    return { behavior: 'passthrough', message: 'MCPTool requires permission.' }
  },
  // ... rendering methods shared across all MCP tools
} satisfies ToolDef<InputSchema, Output>)
```

### 7.2 MCP Tool Instance Creation

When an MCP server reports its tools, `client.ts` creates per-tool instances via
object spread:

```typescript
return {
  ...MCPTool,
  name: skipPrefix ? tool.name : fullyQualifiedName,  // e.g., 'mcp__slack__send_message'
  mcpInfo: { serverName: client.name, toolName: tool.name },
  isMcp: true,
  searchHint: tool._meta?.['anthropic/searchHint'],
  alwaysLoad: tool._meta?.['anthropic/alwaysLoad'] === true,
  inputJSONSchema: tool.inputSchema as Tool['inputJSONSchema'],
  // Per-tool overrides:
  async description() { return tool.description ?? '' },
  isConcurrencySafe() { return tool.annotations?.readOnlyHint ?? false },
  isReadOnly() { return tool.annotations?.readOnlyHint ?? false },
  isDestructive() { return tool.annotations?.destructiveHint ?? false },
  isOpenWorld() { return tool.annotations?.openWorldHint ?? false },
  async checkPermissions() {
    return {
      behavior: 'passthrough',
      suggestions: [{ type: 'addRules', rules: [...], behavior: 'allow', destination: 'localSettings' }],
    }
  },
  async call(args, context, _canUseTool, parentMessage, onProgress?) {
    // MCP protocol callTool invocation with retry logic
  },
}
```

Key properties:
- **`inputJSONSchema`**: MCP tools use raw JSON Schema (from the MCP protocol) rather
  than Zod schemas. The API layer handles the translation.
- **`mcpInfo`**: always present, preserving the original server/tool names. Used for
  permission rule matching even when `name` is prefixed or unprefixed.
- **MCP annotations**: `readOnlyHint`, `destructiveHint`, `openWorldHint` from the
  MCP tool definition map directly to tool behavioral metadata.
- **Permission passthrough**: MCP tools always return `passthrough` with suggested
  allow rules. The general permission system handles the actual allow/deny decision
  based on the user's configured rules.

### 7.3 Name Prefixing

By default, MCP tool names are prefixed: `mcp__<normalized_server>__<tool>` (e.g.,
`mcp__claude_ai_Slack__slack_send_message`). The `CLAUDE_AGENT_SDK_MCP_NO_PREFIX`
environment variable disables prefixing, allowing MCP tools to override built-in
tools by name.

---

## 8. Deferred Loading

> **Source:** `src/tools/ToolSearchTool/`, `src/utils/toolSearch.ts`

### 8.1 The Problem

When many MCP servers are connected, the tool count can grow to hundreds. Including
full JSON Schemas for all tools in the system prompt consumes significant context
window space and prompt cache capacity.

### 8.2 The Solution: ToolSearchTool

When tool search is enabled (determined by `isToolSearchEnabledOptimistic()` — a
function that checks whether the total tool count exceeds a threshold), most tools
are **deferred**. Their names appear in the prompt (via `<available-deferred-tools>`
or `<system-reminder>` messages), but their full schemas are withheld until the model
explicitly loads them via `ToolSearchTool`.

### 8.3 Deferral Rules

`isDeferredTool(tool)` at `src/tools/ToolSearchTool/prompt.ts` determines which
tools are deferred:

1. If `tool.alwaysLoad === true` — **never deferred** (MCP `_meta['anthropic/alwaysLoad']`)
2. If `tool.isMcp === true` — **always deferred** (workflow-specific)
3. If `tool.name === TOOL_SEARCH_TOOL_NAME` — **never deferred** (bootstrapping)
4. If fork-subagent is enabled and `tool.name === AGENT_TOOL_NAME` — **never deferred**
   (must be available turn 1)
5. If `tool.name` is `BriefTool` or `SendUserFileTool` (KAIROS) — **never deferred**
   (primary communication channels)
6. Otherwise: deferred if `tool.shouldDefer === true`

### 8.4 ToolSearch Query Forms

The model can load deferred tools three ways:

- **`"select:Read,Edit,Grep"`** — fetch exact tools by name (comma-separated)
- **`"notebook jupyter"`** — keyword search, scored against tool name + description + `searchHint`
- **`"+slack send"`** — require "slack" in the name, rank by remaining terms

Results are returned as `<functions>` blocks containing full JSON Schema definitions —
the same format as the tool list at the top of the prompt. Once a tool's schema
appears in a result, it is callable exactly like any tool defined at prompt start.

### 8.5 Cache Invalidation

ToolSearchTool memoizes tool descriptions by name. When the set of deferred tools
changes (e.g., MCP server connects mid-session), the cache key (sorted deferred tool
names) changes and the cache is cleared.

---

## 9. Tool Execution Flow

> **Source:** `src/services/tools/toolExecution.ts`

### 9.1 Entry Point: runToolUse

When the model emits a `tool_use` content block, `runToolUse()` is called as an
async generator. The full execution flow:

```
tool_use block arrives
        │
        ▼
1. Tool Lookup
   ├── findToolByName(tools, name) — check available tools
   └── Fallback: check aliases in getAllBaseTools() for deprecated names
        │
        ▼
2. Abort Check
   └── If aborted → yield cancel message, return
        │
        ▼
3. Input Validation (Zod)
   ├── tool.inputSchema.safeParse(input)
   └── Failure → yield InputValidationError
       └── If deferred + not discovered → append "call ToolSearch first" hint
        │
        ▼
4. Input Validation (Semantic)
   ├── tool.validateInput?(parsedInput, context)
   └── Failure → yield validation error message
        │
        ▼
5. Input Backfill
   ├── Clone input if backfillObservableInput exists
   └── Backfill legacy/derived fields on clone (paths expanded, etc.)
        │
        ▼
6. Speculative Classifier Check (Bash only)
   └── Start classifier in parallel with hooks/permissions
        │
        ▼
7. PreToolUse Hooks
   ├── Run registered hooks (may modify input, add messages, stop execution)
   ├── Collect hook permission decisions
   └── Emit hook progress and timing summaries
        │
        ▼
8. Permission Resolution
   ├── resolveHookPermissionDecision (merge hook + tool permission)
   ├── canUseTool (mode-based, classifier, interactive dialog)
   └── Denied? → yield error message, run PermissionDenied hooks, return
        │
        ▼
9. Tool Execution
   ├── tool.call(callInput, context, canUseTool, assistantMessage, onProgress)
   └── Progress events streamed via onProgress callback
        │
        ▼
10. Result Processing
    ├── mapToolResultToToolResultBlockParam(result.data, toolUseID)
    ├── processToolResultBlock (size check, persistence to disk if too large)
    ├── Track git commit attribution
    ├── PostToolUse hooks
    └── Yield tool_result message + any newMessages from tool
```

### 9.2 Error Recovery

If tool execution throws:
- `ShellError` or `AbortError`: handled specifically with appropriate messages
- `McpAuthError`: triggers MCP authentication flow
- Generic errors: wrapped in `<tool_use_error>` XML and returned to the model

### 9.3 Result Size Management

Each tool declares `maxResultSizeChars` — the threshold above which output is
persisted to disk and replaced with a preview:

- `BashTool`: 30,000 chars
- `FileEditTool`, `MCPTool`: 100,000 chars
- `FileReadTool`: `Infinity` (never persisted — would create circular Read loops)

When exceeded, `processToolResultBlock` writes the full output to
`~/.claude/tool-results/` and returns a truncated preview with the file path.

---

## 10. Tool Progress Reporting

> **Source:** `src/Tool.ts`, `src/types/tools.ts`

### 10.1 Progress Types

Tools report progress via the `onProgress` callback passed to `call()`:

```typescript
export type ToolProgress<P extends ToolProgressData> = {
  toolUseID: string
  data: P
}

export type ToolCallProgress<P extends ToolProgressData = ToolProgressData> = (
  progress: ToolProgress<P>,
) => void
```

Each tool defines its own progress data type. Known progress types include:

| Type | Tool | Data |
|---|---|---|
| `BashProgress` | BashTool | stdout/stderr chunks, exit code, timing |
| `AgentToolProgress` | AgentTool | subagent status, partial results |
| `MCPProgress` | MCPTool | MCP call status (started/completed/error) |
| `WebSearchProgress` | WebSearchTool | search result streaming |
| `SkillToolProgress` | SkillTool | skill execution progress |
| `TaskOutputProgress` | TaskOutputTool | background task output |
| `ShellProgress` | BashTool/AgentTool | Shell execution progress forwarded from subagents |

### 10.2 Progress Flow

Progress events flow: tool `call()` -> `onProgress` callback -> `Stream` ->
query loop -> UI renderer. The `renderToolUseProgressMessage` method on each tool
determines how progress is displayed. BashTool shows live stdout, AgentTool shows
subagent status, and MCPTool shows server/tool name with elapsed time.

### 10.3 Background Hints

Both BashTool and AgentTool show a "background hint" after `PROGRESS_THRESHOLD_MS`
(2 seconds) — indicating to the user that a long-running command can be sent to the
background. In assistant mode (KAIROS), blocking bash commands auto-background after
`ASSISTANT_BLOCKING_BUDGET_MS` (15 seconds).

---

## 11. Representative Tool Patterns

### 11.1 BashTool — The Most Complex Tool

> **Source:** `src/tools/BashTool/BashTool.tsx` (~3000+ lines across the directory)

BashTool exemplifies every feature the tool system supports:

**Input schema**: command string + optional timeout, description, run_in_background,
dangerouslyDisableSandbox, and an internal-only `_simulatedSedEdit` field (stripped
from the model-facing schema via `.omit()`).

**Behavioral metadata**:
- `isConcurrencySafe`: true only if `isReadOnly` returns true
- `isReadOnly`: delegates to `checkReadOnlyConstraints()` which AST-parses the command
- `toAutoClassifierInput`: returns the raw command string
- `isSearchOrReadCommand`: classifies commands by their base command (grep, cat, ls, etc.)
  with semantic-neutral commands (echo, printf) skipped

**Permission system**: `bashPermissions.ts` (~800 lines) implements the most sophisticated
permission logic in the codebase. It parses commands into ASTs, evaluates subcommands
against prefix rules, checks redirect paths, detects sed edits, and runs a speculative
classifier check in parallel with hook execution.

**Execution**: handles foreground/background execution, timeout management, sandbox
mode, sed edit interception (where sed in-place edits are intercepted, previewed to
the user, and applied via direct file write rather than shell execution), and image
output detection.

**Progress**: streams stdout/stderr chunks during execution, shows elapsed time,
and supports user-initiated backgrounding via Ctrl+B.

### 11.2 FileEditTool — Precise String Replacement

> **Source:** `src/tools/FileEditTool/FileEditTool.ts`

FileEditTool demonstrates the file-mutation pattern:

**Input validation (`validateInput`)**: enforces uniqueness of `old_string` in the
file (unless `replace_all` is true), detects stale writes by comparing the file's
current modification time against `readFileState`, rejects edits to team memory
files that would introduce secrets, and suggests `NotebookEditTool` for `.ipynb` files.

**Permissions**: delegates to `checkWritePermissionForTool()` with path-based rules.
Uses `preparePermissionMatcher` to match wildcard patterns against the file path.
Uses `backfillObservableInput` to expand `~` and relative paths before hooks see them.

**Execution**: reads the file, finds the exact string, replaces it, preserves line
endings (CRLF/LF) and encoding (UTF-8/Latin-1), writes back, updates `readFileState`,
notifies VS Code, tracks file history for undo, and computes a git diff for display.

**Rendering**: shows the diff in the terminal with syntax highlighting. The rejected
rendering shows what would have been changed.

### 11.3 AgentTool — Subagent Orchestration

> **Source:** `src/tools/AgentTool/AgentTool.tsx` (1500+ lines)

AgentTool demonstrates the subagent launch pattern:

**Schema gating**: the input schema is dynamically composed based on feature flags.
The `isolation` field offers `'worktree'` (all users) or `'worktree' | 'remote'`
(ant-only). The `run_in_background` field is omitted when background tasks are
disabled. The `cwd` field is only present when KAIROS is enabled.

**Launch modes**:
1. **Synchronous**: runs `runAgent()` in the foreground, blocking until completion
2. **Async background**: registers with `LocalAgentTask`, runs in a separate async
   context, returns immediately with an `agentId` and `outputFile` path
3. **Fork subagent**: shares the parent's prompt cache via `CacheSafeParams`,
   preserving cache efficiency
4. **Remote**: launches in a CCR (Claude Code Remote) environment
5. **Teammate**: spawns via tmux with agent swarm coordination

**Tool filtering**: subagents receive a filtered tool set via `filterToolsForAgent()`
which applies `ALL_AGENT_DISALLOWED_TOOLS`, `ASYNC_AGENT_ALLOWED_TOOLS`, or
`IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` depending on the agent type.

**Progress**: forwards both its own `AgentToolProgress` events (status, partial
results) and `ShellProgress` events from the subagent's bash executions, so the
SDK receives continuous `tool_progress` updates.

### 11.4 MCPTool — Protocol Bridge

> **Source:** `src/tools/MCPTool/MCPTool.ts`, `src/services/mcp/client.ts`

MCPTool demonstrates the template/composition pattern:

The base `MCPTool.ts` defines a minimal template with shared rendering. Per-server
tool instances are created by spreading the template and overriding identity,
description, schema, permissions, and execution. The `call()` override invokes
the MCP protocol's `callTool` method with retry logic for session disconnects.

The `inputJSONSchema` field (rather than `inputSchema`) allows MCP tools to bypass
Zod entirely — their schemas come from the MCP server as raw JSON Schema objects.

---

## 12. Design Principles

### 12.1 Uniform Abstraction

Every capability — from reading a file to sending a Slack message via MCP — is a
`Tool`. The model, permission system, rendering pipeline, and telemetry all operate
on this single type. This uniformity means new capabilities (MCP servers, skills,
workflows) integrate without changes to the core execution engine.

### 12.2 Fail-Closed Defaults

`buildTool` defaults are deliberately conservative:
- `isConcurrencySafe: false` — tools must opt into parallel execution
- `isReadOnly: false` — tools are assumed to have side effects
- `toAutoClassifierInput: ''` — tools are invisible to the security classifier
  unless they explicitly opt in
- `checkPermissions` defaults to `allow`, but this is overridden by the general
  permission system for any tool that touches the filesystem or shell

Security-relevant tools **must** override the defaults. The design makes it easy
to add a safe tool (just define `call` and rendering) while requiring explicit
opt-in for security-sensitive behavior.

### 12.3 Separation of Validation, Permission, and Execution

Tool execution proceeds through three distinct phases:
1. **Validation** (`validateInput`) — is this input structurally and semantically
   valid?
2. **Permission** (`checkPermissions` + general system) — is this input allowed
   by the current security policy?
3. **Execution** (`call`) — perform the action

This separation means validation errors never reach the permission dialog, and
permission denials never reach the tool's execution logic. Each layer can evolve
independently.

### 12.4 Progressive Schema Loading

The deferred loading system (`ToolSearchTool` + `shouldDefer`) solves the tension
between capability breadth and prompt efficiency. The model starts with a lean
prompt containing only essential tool schemas, and loads additional schemas on
demand. This is invisible to the user and transparent to the model — loaded tools
are callable exactly like initially-present tools.

### 12.5 Composition Over Inheritance

MCP tools are created via object spread from a template, not class inheritance.
This keeps the type system simple (everything is a plain `Tool` object), avoids
prototype chain complexity, and makes it easy to override individual properties
without affecting others.

### 12.6 Cache-Aware Assembly

`assembleToolPool` sorts built-in and MCP tools into separate contiguous partitions
to maximize prompt cache stability. Adding or removing an MCP tool only invalidates
the MCP suffix of the tool list — the built-in prefix (and its cache) is preserved.

### 12.7 Defense in Depth for Bash

BashTool has the most elaborate security surface:
- AST parsing for command analysis (`parseForSecurity`)
- Subcommand-level permission matching (not just whole-command matching)
- Sed edit interception (preview + direct write instead of shell execution)
- Sandbox mode (macOS Seatbelt) for untrusted commands
- `_simulatedSedEdit` stripped from model-facing schema and re-stripped at execution
  time (double defense against injection)
- Speculative classifier checks running in parallel with hooks
- Read-only constraint checking via command semantics analysis

### 12.8 Transparent Extensibility

The tool system's architecture means adding a new tool requires:
1. Create a directory under `src/tools/`
2. Define a `buildTool({...})` export
3. Add it to `getAllBaseTools()` (with appropriate feature gating)
4. Implement rendering (or rely on defaults)

No changes to the execution engine, permission system, or query loop are needed.
MCP tools require even less — they are automatically created from server-reported
tool definitions with no source code changes at all.
