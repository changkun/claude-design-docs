# Tool System — Design Document

## Part A: Design Document

### 1. Overview

The tool system is the mechanism through which Claude Code acts on the world. Every filesystem read, shell command, file edit, subagent launch, web search, and MCP integration flows through a single, uniform tool abstraction. The system's design goals are:

1. **Uniform interface** -- every capability, whether built-in or dynamically discovered via MCP, exposes the same `Tool` type to the model, the permission system, and the rendering pipeline.
2. **Fail-closed permissions** -- no tool executes without passing through validation, permission checking, and optionally user confirmation. The defaults are conservative: tools are assumed to write, assumed not concurrency-safe, and assumed to need permission checks.
3. **Extensibility without modification** -- MCP servers can add arbitrary tools at runtime without changing any built-in code. The composition layer merges them with built-in tools transparently.
4. **Lazy discovery** -- when the tool count grows large (especially with MCP servers), most tools are "deferred" behind a ToolSearchTool that loads their full schemas on demand, keeping the initial prompt lean.

The tool system spans roughly 50 tool implementations, a central registry, type definitions, availability constants, and an execution engine.

---

### 2. Tool Interface Contract

#### 2.1 The Tool Type

The `Tool` type is the core abstraction. It is a generic parameterized by three type variables -- `Input` (a schema object), `Output` (the result data type), and `P` (a progress event type) -- and contains properties spanning the following facets:

- **Identity**: name, aliases, search hints
- **Schema**: input schema (structured or raw JSON Schema for MCP), optional output schema
- **Execution**: the `call()` method, input validation, permission checking
- **Behavioral metadata**: concurrency safety, read-only, destructive, open-world, interrupt behavior, user interaction requirements
- **MCP metadata**: flags for MCP/LSP origin, server/tool name pairing
- **Deferred loading**: whether to defer schema inclusion, whether to always load
- **Permission integration**: path extraction, matcher preparation, input backfilling, classifier input generation
- **Result size management**: maximum result characters, strict mode
- **Prompt generation**: tool description and prompt generation
- **Rendering (UI)**: user-facing name, background color, multiple rendering methods for different states (in-progress, rejected, errored, queued, grouped), transparency, truncation detection
- **Summary and search**: activity descriptions, search text extraction
- **Result mapping**: converting tool output to API-compatible formats, input equivalence checking

The `Tools` type is a named alias for `readonly Tool[]`, used consistently across the codebase.

#### 2.2 The buildTool Factory

Rather than requiring every tool implementation to fill in all properties, a `buildTool` factory function provides fail-closed defaults for commonly-stubbed methods:

- `isEnabled`: defaults to `true`
- `isConcurrencySafe`: defaults to `false` (assume not safe)
- `isReadOnly`: defaults to `false` (assume writes)
- `isDestructive`: defaults to `false`
- `checkPermissions`: defaults to allowing with the original input passed through
- `toAutoClassifierInput`: defaults to empty string (skip classifier)
- `userFacingName`: defaults to the tool's `name`

Tool definitions use a partial type where all defaultable keys are optional. The factory merges the definition with defaults via object spread. This ensures:
- All tool exports are complete objects -- callers never need optional chaining with fallback defaults
- Defaults are fail-closed: non-concurrency-safe, non-read-only by default
- Security-relevant tools must explicitly override `toAutoClassifierInput` -- the default skips classifier analysis

#### 2.3 The ToolResult Type

Every tool `call()` returns a result containing:

- **`data`**: the tool's primary output
- **`newMessages`**: optional additional messages to inject into the conversation (e.g., subagent transcripts)
- **`contextModifier`**: optional function to mutate the tool use context between turns (e.g., changing working directory)
- **`mcpMeta`**: optional MCP protocol metadata for SDK consumers

#### 2.4 The ToolUseContext

Every tool receives a rich environment struct carrying:

- **Static session configuration**: tools list, model, MCP clients, thinking config, agent definitions, budget
- **Cooperative cancellation**: abort controller signal
- **File state cache**: LRU cache of recently-accessed files for stale-write detection and post-compaction restoration
- **Conversation history**: the full message list
- **State accessors**: application state read/write, tool JSX management
- **UI callbacks**: notifications, OS notifications, stream mode control
- **Tracking state**: tool decisions, query tracking, file reading limits
- **Content replacement**: tool result budget state

Subagent tools receive a modified copy that restricts state mutation (e.g., `setAppState` becomes a no-op) to prevent cross-thread race conditions.

---

### 3. Tool Registry

#### 3.1 Exhaustive Tool Registration

A single function returns the complete list of all tools that could be available in the current environment. It is the single source of truth for tool existence. Tools are included conditionally based on:
- Feature flags
- Environment variables
- Runtime checks
- Test mode

Conditional tools use dynamic `require()` (not static `import`) for dead code elimination -- the bundler can tree-shake entire tool implementations when their feature flag is off.

#### 3.2 Permission-Filtered Built-in Tools

A filtering function starts from the exhaustive list and applies:

1. **Special tool exclusion**: certain tools (MCP resource listing, synthetic output) are excluded from the default set and added conditionally elsewhere
2. **Deny rule filtering**: removes tools matching a blanket deny rule (the entire tool is denied)
3. **REPL mode filtering**: when REPL mode is enabled, primitive tools are hidden -- they are accessible only through the REPL VM context
4. **isEnabled() check**: each tool's `isEnabled()` is called; disabled tools are dropped

A "simple mode" reduces the tool set to just shell execution, file reading, and file editing.

#### 3.3 Full Tool Assembly

The assembly function is the single point where built-in and MCP tools are combined:

- **Built-in tools take precedence**: when a name collides between built-in and MCP, built-in wins (by insertion-order deduplication)
- **Sorted partitions**: built-ins are sorted as a contiguous prefix, MCP tools as a separate suffix, preventing interleaving that would invalidate the server-side prompt cache
- **Deny rules apply to MCP tools**: server-prefix rules strip all tools from a server before the model sees them

#### 3.4 Count and Token Estimation

A separate merge function produces a simple concatenation of built-in and MCP tools, used where ordering does not matter -- threshold calculations for ToolSearch enablement, token counting, and contexts where MCP tools should be considered alongside built-ins.

---

### 4. Tool Categories

The tool implementations group into seven functional categories:

#### 4.1 File System Tools
Read, edit, write, notebook cell editing, pattern-based file search, content search. The file reader handles text (with line numbers), images (resized, base64), PDFs (page ranges), Jupyter notebooks (all cells), and directories (file trees). The file editor implements exact-string-replacement with uniqueness enforcement, stale-write detection, and git diff tracking for undo.

#### 4.2 Execution Tools
Shell command execution (with sandbox), PowerShell (Windows), VM-sandboxed execution wrapping primitive tools, virtual terminal interaction. The shell tool is the most complex implementation, handling timeout management, background task spawning, sandbox mode (macOS Seatbelt), sed edit interception, image output detection, and elaborate command-level permission analysis.

#### 4.3 Agent and Task Tools
Subagent launching (sync, async, forked, remote, teammate), background task output reading, task stopping, TODO/task management. The subagent tool supports five distinct launch modes, each with its own lifecycle, progress reporting, and tool filtering.

#### 4.4 Planning and Mode Tools
Enter/exit plan mode, enter/exit git worktree isolation.

#### 4.5 Integration Tools
MCP tool template, MCP resource listing/reading, MCP authentication, Language Server Protocol queries, web content fetching and processing, web search.

#### 4.6 Communication and Notification Tools
Inter-agent messaging, user prompting, primary communication channels (KAIROS), file delivery to users, push notifications, peer agent listing.

#### 4.7 Configuration and Discovery Tools
Deferred tool discovery and loading, slash-command skill invocation, settings read/write, plan execution verification, conversation segment snipping, bundled workflow execution.

---

### 5. Tool Availability Matrix

#### 5.1 Always-Available Tools
A core set of tools is always present in the exhaustive list regardless of feature flags, though they may still be filtered by deny rules or `isEnabled()`. This includes: subagent launching, background task output, shell execution, file read/edit/write, notebook editing, web fetch, TODO writing, web search, task stopping, user prompting, skill invocation, plan mode entry/exit, inter-agent messaging, and primary communication.

File search tools (glob and grep) are present unless embedded search tools are available in the build.

#### 5.2 Conditionally Available Tools
Many tools are gated behind feature flags, environment variables, or runtime checks. These include configuration tools, virtual terminals, REPL, web browser, task management v2, overflow testing, context inspection, terminal capture, LSP integration, worktree management, swarm agent management, peer listing, workflow execution, sleep, cron scheduling, remote triggers, monitoring, file delivery, push notifications, PR subscription, PowerShell, conversation snipping, and tool search itself.

#### 5.3 Agent Disallowed Tools
Several tools are explicitly blocked for subagents to prevent recursion and preserve abstraction boundaries: background task output, plan mode entry/exit, nested subagent launching (unless specifically enabled), user prompting, task stopping, and workflow execution.

#### 5.4 Async Agent Allowed Tools
Background/async agents receive a strict allowlist rather than a denylist. The allowlist includes file operations, search, shell execution, web tools, skill invocation, synthetic output, tool search, and worktree management. Notably absent: subagent launching (prevents recursion), background task output (main thread abstraction), MCP tools (TBD), and virtual terminal (singleton conflict).

#### 5.5 Coordinator Mode Tools
In coordinator mode, the coordinator gets only orchestration tools: subagent launching, task stopping, inter-agent messaging, and synthetic output. Workers get the full tool set minus coordinator-only tools.

---

### 6. Permission Integration

#### 6.1 Permission Behaviors

The permission system uses a three-behavior model: `allow`, `deny`, `ask`. A fourth informal behavior -- `passthrough` -- is used by certain tools (notably MCP tools and the factory default) to signal "I have no opinion; defer to the general permission system."

#### 6.2 Tool-Specific Permission Logic

Every tool implements a permission check method. This is called after input validation passes. Tool-specific implementations vary:

- **File edit/write tools**: evaluate the target path against the working directory and permission rules
- **File read tool**: delegates to read permission checking
- **Shell tool**: parses the command into an AST, checks each subcommand against prefix rules, evaluates sandbox eligibility, checks redirect paths, returns compound permission results with per-subcommand reasons
- **MCP tools**: return `passthrough` with suggested allow rules, deferring entirely to the general permission system
- **Default**: returns `allow` -- tools that do not override are auto-allowed

#### 6.3 Hook Pattern Matching

For hook `if` conditions (patterns like `Bash(git *)`), tools can provide a custom matcher. This is called once per hook-input pair; expensive parsing happens here. The returned closure is called per pattern.

For shell commands, the matcher parses the command into an AST, extracts subcommands (stripping variable prefixes), and matches each against the pattern. For compound commands (e.g., `ls && git push`), any subcommand matching triggers the hook -- this prevents security hooks from being bypassed by chaining.

#### 6.4 Input Backfilling

Before hooks and permission checks see the input, a backfill method can mutate a cloned copy to add legacy/derived fields. The canonical use is expanding `~` and relative paths so hook allowlists cannot be bypassed via path tricks. The original API-bound input is never mutated (preserves prompt cache). Hook-returned updated input objects are not re-backfilled.

#### 6.5 Permission Modes

The system supports several permission modes:

| Mode | Behavior |
|---|---|
| `default` | Writes prompt the user; reads auto-allowed |
| `acceptEdits` | File edits auto-allowed; other writes prompt |
| `bypassPermissions` | Everything auto-allowed (dangerous) |
| `plan` | All writes require plan approval |
| `dontAsk` | Auto-deny anything that would prompt |
| `auto` | Classifier decides allow/deny without user prompting |

#### 6.6 Security Classifier Input

In `auto` mode, a security classifier evaluates tool calls. Each tool provides a compact representation of its input:
- Shell tool: returns the raw command string
- File edit tool: returns `"path: new_content"`
- Default: returns empty string (skips classifier -- only security-relevant tools override)

#### 6.7 Permission Decision Flow

```
tool.checkPermissions(input, context)
        |
        v
general permission system
        |
        +-- behavior: 'allow' -> resolve immediately
        |
        +-- behavior: 'deny' -> resolve denied
        |
        +-- behavior: 'ask'
              |
              +-- coordinator mode? -> coordinator permission handling
              +-- swarm worker? -> swarm worker permission handling
              +-- classifier available? -> speculative check + race
              +-- fallback -> interactive permission dialog
```

---

### 7. MCP Tool Composition

#### 7.1 Template Pattern

A base MCP tool template defines shared rendering logic (progress display, result display, error display) while allowing per-tool identity and behavior via override. The template always returns `passthrough` for permissions, deferring to the general system.

#### 7.2 Instance Creation

When an MCP server reports its tools, per-tool instances are created via object spread from the template. Each instance overrides:
- Identity: fully qualified name, server/tool info, search hint
- Schema: raw JSON Schema (not Zod)
- Behavioral metadata: derived from MCP annotations (`readOnlyHint`, `destructiveHint`, `openWorldHint`)
- Permissions: passthrough with suggested allow rules
- Execution: MCP protocol `callTool` invocation with retry logic

MCP tools use raw JSON Schema rather than Zod schemas. The API layer handles translation.

#### 7.3 Name Prefixing

By default, MCP tool names are prefixed: `mcp__<normalized_server>__<tool>`. An environment variable can disable prefixing, allowing MCP tools to override built-in tools by name.

---

### 8. Deferred Loading

#### 8.1 The Problem

When many MCP servers are connected, the tool count can grow to hundreds. Including full schemas for all tools in the system prompt consumes significant context window space and prompt cache capacity.

#### 8.2 The Solution: ToolSearchTool

When tool search is enabled (total tool count exceeds a threshold), most tools are deferred. Their names appear in the prompt, but their full schemas are withheld until the model explicitly loads them via ToolSearchTool.

#### 8.3 Deferral Rules

A tool is deferred or not based on this priority:
1. If the tool declares `alwaysLoad` -- never deferred
2. If the tool is from MCP -- always deferred (workflow-specific)
3. If the tool is ToolSearchTool itself -- never deferred (bootstrapping)
4. If fork-subagent is enabled and the tool is the subagent tool -- never deferred (must be available turn 1)
5. If the tool is a primary communication channel (KAIROS) -- never deferred
6. Otherwise: deferred if the tool's `shouldDefer` flag is true

#### 8.4 Query Forms

The model can load deferred tools three ways:
- **Exact selection**: `"select:Read,Edit,Grep"` -- fetch exact tools by name (comma-separated)
- **Keyword search**: `"notebook jupyter"` -- scored against tool name + description + search hint
- **Required-name search**: `"+slack send"` -- require "slack" in the name, rank by remaining terms

Results are returned as schema blocks in the same format as the initial tool list. Once loaded, a tool is callable exactly like any initially-present tool.

#### 8.5 Cache Invalidation

Tool descriptions are memoized by name. When the set of deferred tools changes (e.g., MCP server connects mid-session), the cache key (sorted deferred tool names) changes and the cache is cleared.

---

### 9. Tool Execution Flow

#### 9.1 Execution Pipeline

When the model emits a tool use content block, the execution pipeline proceeds through:

1. **Tool Lookup**: find the tool by name in available tools; fall back to alias lookup for deprecated names
2. **Abort Check**: if aborted, yield cancel message and return
3. **Schema Validation**: parse input against the tool's schema; failure yields a validation error (with a hint to call ToolSearch if the tool is deferred and undiscovered)
4. **Semantic Validation**: tool-specific input validation; failure yields a validation error
5. **Input Backfill**: clone input and backfill legacy/derived fields (path expansion, etc.)
6. **Speculative Classifier Check**: for shell commands, start classifier in parallel with hooks/permissions
7. **PreToolUse Hooks**: run registered hooks (may modify input, add messages, stop execution); collect hook permission decisions; emit progress and timing
8. **Permission Resolution**: merge hook + tool permission decisions; apply mode-based, classifier, or interactive permission checks; deny yields error + PermissionDenied hooks
9. **Tool Execution**: invoke `call()` with progress streaming via callback
10. **Result Processing**: map result to API format, check size (persist to disk if too large), track git commit attribution, run PostToolUse hooks, yield result message + any injected messages

#### 9.2 Error Recovery

- Shell errors or abort errors: handled specifically with appropriate messages
- MCP authentication errors: trigger MCP authentication flow
- Generic errors: wrapped in an error XML block and returned to the model

#### 9.3 Result Size Management

Each tool declares a maximum result character count. When exceeded, the full output is persisted to disk and replaced with a truncated preview containing the file path. File reading has infinite limit (to avoid circular read loops).

---

### 10. Tool Progress Reporting

#### 10.1 Progress Types

Tools report progress via a callback passed to `call()`. Each tool defines its own progress data type:
- Shell: stdout/stderr chunks, exit code, timing
- Subagent: status, partial results
- MCP: call status (started/completed/error)
- Web search: search result streaming
- Skill: execution progress
- Background task: task output
- Shell forwarding: shell execution progress from subagents

#### 10.2 Progress Flow

Progress events flow: tool `call()` -> callback -> stream -> query loop -> UI renderer. Each tool determines how its progress is displayed (live stdout for shell, status for subagent, elapsed time for MCP).

#### 10.3 Background Hints

Long-running tools show a background hint after a threshold (2 seconds), indicating the operation can be sent to background. In assistant mode, blocking shell commands auto-background after 15 seconds.

---

### 11. Design Principles

1. **Uniform Abstraction**: every capability is a `Tool`. The model, permission system, rendering, and telemetry operate on this single type. New capabilities integrate without core engine changes.

2. **Fail-Closed Defaults**: conservative defaults (not concurrency-safe, not read-only, invisible to classifier). Security-relevant tools must explicitly opt in.

3. **Separation of Validation, Permission, and Execution**: three distinct phases that can evolve independently. Validation errors never reach the permission dialog; permission denials never reach execution.

4. **Progressive Schema Loading**: deferred loading solves the tension between capability breadth and prompt efficiency. The model starts lean and loads schemas on demand.

5. **Composition Over Inheritance**: MCP tools are created via object spread, not class inheritance. This keeps everything as plain objects, avoids prototype chain complexity, and allows selective property override.

6. **Cache-Aware Assembly**: built-in and MCP tools are sorted into separate contiguous partitions. Adding/removing an MCP tool only invalidates the MCP suffix -- the built-in prefix cache is preserved.

7. **Defense in Depth for Shell Execution**: AST parsing, subcommand-level permission matching, sed edit interception, sandbox mode, field stripping from model-facing schema with double defense at execution time, speculative classifier checks in parallel with hooks, read-only constraint checking.

8. **Transparent Extensibility**: adding a new tool requires creating a directory, defining a buildTool export, registering it in the exhaustive list, and implementing rendering. No execution engine, permission system, or query loop changes needed. MCP tools require zero source code changes.
