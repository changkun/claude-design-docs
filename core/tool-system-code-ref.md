# Tool System — Code Reference

### File Paths

| Component | Path |
|---|---|
| Tool type definition | `src/Tool.ts` |
| Tool registry | `src/tools.ts` |
| Tool availability constants | `src/constants/tools.ts` |
| Tool execution engine | `src/services/tools/toolExecution.ts` |
| Tool implementations directory | `src/tools/` |
| MCP client (tool instance creation) | `src/services/mcp/client.ts` |
| Permission utilities | `src/utils/permissions/permissions.ts` |
| Permission types | `src/types/permissions.ts` |
| Permission hook (React) | `src/hooks/useCanUseTool.tsx` |
| Tool search utilities | `src/utils/toolSearch.ts` |
| Tool search prompt/deferral logic | `src/tools/ToolSearchTool/prompt.ts` |
| Tool progress types | `src/types/tools.ts` |

### Specific Tool Implementations

| Tool | Path | Approx. Size |
|---|---|---|
| BashTool | `src/tools/BashTool/BashTool.tsx` | ~3000+ lines across directory |
| Bash permissions | `src/tools/BashTool/bashPermissions.ts` | ~800 lines |
| FileEditTool | `src/tools/FileEditTool/FileEditTool.ts` | - |
| AgentTool | `src/tools/AgentTool/AgentTool.tsx` | 1500+ lines |
| MCPTool template | `src/tools/MCPTool/MCPTool.ts` | - |
| ToolSearchTool | `src/tools/ToolSearchTool/` | - |

### Type Definitions

**Tool type** (at `src/Tool.ts`):
```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = { /* ~40-50 properties */ }
```

**ToolResult** (at `src/Tool.ts`):
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

**PermissionBehavior** (at `src/types/permissions.ts`):
```typescript
export type PermissionBehavior = 'allow' | 'deny' | 'ask'
```
Note: `'passthrough'` is used informally but is not part of this union type.

**ToolProgress** (at `src/types/tools.ts`):
```typescript
export type ToolProgress<P extends ToolProgressData> = {
  toolUseID: string
  data: P
}
export type ToolCallProgress<P extends ToolProgressData = ToolProgressData> = (
  progress: ToolProgress<P>,
) => void
```

### Key Functions

**`buildTool`** at `src/Tool.ts:757-792`:
```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

**`TOOL_DEFAULTS`** at `src/Tool.ts`:
```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,
  isReadOnly: () => false,
  isDestructive: () => false,
  checkPermissions: (input) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',
  userFacingName: () => '',
}
```

**`getAllBaseTools()`** at `src/tools.ts`:
```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool, FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool, TodoWriteTool, WebSearchTool,
    TaskStopTool, AskUserQuestionTool, SkillTool, EnterPlanModeTool,
    // ... conditionally included tools ...
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
  ]
}
```

**`assembleToolPool()`** at `src/tools.ts`:
```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

**MCP tool instance creation** at `src/services/mcp/client.ts:1765-1862`:
```typescript
return {
  ...MCPTool,
  name: skipPrefix ? tool.name : fullyQualifiedName,
  mcpInfo: { serverName: client.name, toolName: tool.name },
  isMcp: true,
  searchHint: tool._meta?.['anthropic/searchHint'],
  alwaysLoad: tool._meta?.['anthropic/alwaysLoad'] === true,
  inputJSONSchema: tool.inputSchema as Tool['inputJSONSchema'],
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
  async call(args, context, _canUseTool, parentMessage, onProgress?) { /* MCP callTool */ },
}
```

### Named Constants (at `src/constants/tools.ts`)

**Agent-disallowed tools**:
```typescript
export const ALL_AGENT_DISALLOWED_TOOLS = new Set([
  TASK_OUTPUT_TOOL_NAME,
  EXIT_PLAN_MODE_V2_TOOL_NAME,
  ENTER_PLAN_MODE_TOOL_NAME,
  AGENT_TOOL_NAME,
  ASK_USER_QUESTION_TOOL_NAME,
  TASK_STOP_TOOL_NAME,
  WORKFLOW_TOOL_NAME,
])
```

**Async agent allowed tools**:
```typescript
export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  FILE_READ_TOOL_NAME, WEB_SEARCH_TOOL_NAME, TODO_WRITE_TOOL_NAME,
  GREP_TOOL_NAME, WEB_FETCH_TOOL_NAME, GLOB_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, NOTEBOOK_EDIT_TOOL_NAME,
  SKILL_TOOL_NAME, SYNTHETIC_OUTPUT_TOOL_NAME, TOOL_SEARCH_TOOL_NAME,
  ENTER_WORKTREE_TOOL_NAME, EXIT_WORKTREE_TOOL_NAME,
])
```

**Coordinator mode allowed tools**:
```typescript
export const COORDINATOR_MODE_ALLOWED_TOOLS = new Set([
  AGENT_TOOL_NAME,
  TASK_STOP_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
```

### Feature Gates for Conditional Tools

| Tool | Gate Expression |
|---|---|
| ConfigTool, TungstenTool, REPLTool | `process.env.USER_TYPE === 'ant'` |
| WebBrowserTool | `feature('WEB_BROWSER_TOOL')` |
| TaskCreate/Get/Update/List | `isTodoV2Enabled()` |
| LSPTool | `ENABLE_LSP_TOOL` env var |
| EnterWorktreeTool, ExitWorktreeTool | `isWorktreeModeEnabled()` |
| TeamCreateTool, TeamDeleteTool | `isAgentSwarmsEnabled()` |
| ToolSearchTool | `isToolSearchEnabledOptimistic()` |
| SleepTool | `feature('PROACTIVE')` or `feature('KAIROS')` |
| SnipTool | `feature('HISTORY_SNIP')` |
| PowerShellTool | `isPowerShellToolEnabled()` |
| WorkflowTool | `feature('WORKFLOW_SCRIPTS')` |
| ListPeersTool | `feature('UDS_INBOX')` |

### Result Size Limits

| Tool | `maxResultSizeChars` |
|---|---|
| BashTool | 30,000 |
| FileEditTool | 100,000 |
| MCPTool | 100,000 |
| FileReadTool | `Infinity` |

Overflow output persisted to `~/.claude/tool-results/`.

### Progress Threshold Constants

| Constant | Value | Description |
|---|---|---|
| `PROGRESS_THRESHOLD_MS` | 2,000 ms | Show "background hint" for long-running tools |
| `ASSISTANT_BLOCKING_BUDGET_MS` | 15,000 ms | Auto-background blocking bash in KAIROS mode |

### MCP Name Prefixing

- Default format: `mcp__<normalized_server>__<tool>` (e.g., `mcp__claude_ai_Slack__slack_send_message`)
- Disabled by: `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` environment variable
- MCP annotations used: `_meta['anthropic/searchHint']`, `_meta['anthropic/alwaysLoad']`, `annotations.readOnlyHint`, `annotations.destructiveHint`, `annotations.openWorldHint`

### Deferral Logic (at `src/tools/ToolSearchTool/prompt.ts`)

Function: `isDeferredTool(tool)` -- priority order:
1. `tool.alwaysLoad === true` -> not deferred
2. `tool.isMcp === true` -> deferred
3. `tool.name === TOOL_SEARCH_TOOL_NAME` -> not deferred
4. Fork-subagent enabled + `tool.name === AGENT_TOOL_NAME` -> not deferred
5. `tool.name` is `BriefTool` or `SendUserFileTool` -> not deferred
6. `tool.shouldDefer === true` -> deferred

### Execution Pipeline Entry Point

Function: `runToolUse()` at `src/services/tools/toolExecution.ts` -- async generator.

Key helper functions referenced:
- `findToolByName(tools, name)` -- tool lookup
- `checkReadOnlyConstraints()` -- BashTool read-only analysis
- `filterToolsForAgent()` -- subagent tool filtering
- `createSubagentContext` -- restricted context for subagents
- `filterToolsByDenyRules()` -- deny rule application
- `hasPermissionsToUseTool()` -- at `src/utils/permissions/permissions.ts`
- `checkWritePermissionForTool()` -- file write permission
- `checkReadPermissionForTool()` -- file read permission
- `parseForSecurity` -- BashTool AST command parsing
- `processToolResultBlock` -- result size check and persistence

### BashTool Input Schema Fields

- `command`: string (the shell command)
- `timeout`: optional number
- `description`: optional string
- `run_in_background`: optional boolean
- `dangerouslyDisableSandbox`: optional boolean
- `_simulatedSedEdit`: internal-only (stripped from model-facing schema via `.omit()`, double-stripped at execution time)

### Tool Rendering Methods (React)

All tools implement or inherit these rendering methods:
- `renderToolUseMessage(input, options)` -> `React.ReactNode`
- `renderToolResultMessage?(content, progressMessages, options)` -> `React.ReactNode`
- `renderToolUseProgressMessage?(progressMessages, options)` -> `React.ReactNode`
- `renderToolUseRejectedMessage?(input, options)` -> `React.ReactNode`
- `renderToolUseErrorMessage?(result, options)` -> `React.ReactNode`
- `renderToolUseQueuedMessage?()` -> `React.ReactNode`
- `renderToolUseTag?(input)` -> `React.ReactNode`
- `renderGroupedToolUse?(toolUses, options)` -> `React.ReactNode | null`

Color theming: `userFacingNameBackgroundColor?(input): keyof Theme | undefined`
