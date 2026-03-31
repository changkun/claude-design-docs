# Claude Code: Command System — Design Specification

This document analyzes the command and skill architecture of Claude Code — Anthropic's
agentic CLI tool — focusing on how slash commands are defined, discovered, loaded,
resolved, and executed across a multi-layer source hierarchy.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Command Types](#2-command-types)
- [3. Command Interface](#3-command-interface)
- [4. Registry and Loading](#4-registry-and-loading)
- [5. Command Resolution](#5-command-resolution)
- [6. Execution Models](#6-execution-models)
- [7. Availability and Enablement](#7-availability-and-enablement)
- [8. The Skill System](#8-the-skill-system)
- [9. Feature Gating](#9-feature-gating)
- [10. Tool Permission Constraints](#10-tool-permission-constraints)
- [11. Key Commands by Category](#11-key-commands-by-category)
- [12. Design Principles](#12-design-principles)

---

## 1. Overview

Commands are Claude Code's primary user-facing extension surface. When a user types
`/commit`, `/review`, `/help`, or `/my-custom-skill`, the system matches the input
against a unified registry of commands drawn from six distinct sources, dispatches
the match to one of three execution models, and integrates the result back into the
conversation or terminal UI.

The command system serves three distinct audiences simultaneously:

1. **End users** — who type `/command` in the REPL to trigger actions, toggle settings,
   or invoke skills
2. **The model** — which invokes prompt-type commands (skills) via the Skill tool,
   expanding specialized instructions into the conversation
3. **Extension authors** — who publish skills as markdown files in `~/.claude/skills/`,
   project `.claude/skills/` directories, or plugin packages

The architecture unifies these audiences through a single `Command` type that carries
enough metadata for the REPL typeahead, the model's skill index, the help screen,
and the execution engine to all operate from one source of truth.

> **Source:** `src/commands.ts` (registry, loading, resolution), `src/types/command.ts`
> (type definitions), `src/skills/loadSkillsDir.ts` (skill loading),
> `src/skills/bundledSkills.ts` (bundled skill registration),
> `src/utils/plugins/loadPluginCommands.ts` (plugin command loading)

---

## 2. Command Types

Every command is one of three discriminated union variants, distinguished by
the `type` field:

### 2.1 Prompt Commands (`type: 'prompt'`)

Prompt commands expand into content blocks (`ContentBlockParam[]`) that are injected
into the conversation as a user message and sent to the LLM. The model then processes
the injected instructions and responds — possibly calling tools.

This is the execution model for all skills (user-authored, plugin-provided, bundled,
and MCP-sourced). The expansion function `getPromptForCommand(args, context)` is
async and receives both the user-supplied arguments and the full `ToolUseContext`.

**Examples:** `/commit`, `/review`, `/security-review`, all user-defined skills

```typescript
type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  contentLength: number
  argNames?: string[]
  allowedTools?: string[]
  model?: string
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  getPromptForCommand(
    args: string,
    context: ToolUseContext,
  ): Promise<ContentBlockParam[]>
  // ... additional fields (see §3)
}
```

### 2.2 Local Commands (`type: 'local'`)

Local commands execute entirely on the client side, producing a `LocalCommandResult` —
either a text string, a compaction result, or a skip sentinel. They never touch the
LLM. The implementation is lazy-loaded via `load()` to defer heavy imports until
invocation.

**Examples:** `/compact`, `/clear`, `/advisor`

```typescript
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<LocalCommandModule>
}

type LocalCommandResult =
  | { type: 'text'; value: string }
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
  | { type: 'skip' }
```

The `supportsNonInteractive` flag indicates whether the command can run in headless
(SDK/non-interactive) mode. Commands that render Ink UI or require terminal input
set this to `false`.

### 2.3 Local JSX Commands (`type: 'local-jsx'`)

Local JSX commands render interactive React/Ink UI components within the terminal.
Like local commands, they are lazy-loaded, but their `call()` signature returns a
`React.ReactNode` and communicates completion via an `onDone` callback rather than
a return value.

**Examples:** `/help`, `/config`, `/model`, `/login`, `/exit`, `/session`

```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}

type LocalJSXCommandCall = (
  onDone: LocalJSXCommandOnDone,
  context: ToolUseContext & LocalJSXCommandContext,
  args: string,
) => Promise<React.ReactNode>
```

The `onDone` callback supports rich completion signaling:

```typescript
type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: 'skip' | 'system' | 'user'
    shouldQuery?: boolean          // Send messages to model after completion
    metaMessages?: string[]        // Additional model-visible hidden messages
    nextInput?: string             // Pre-fill next user input
    submitNextInput?: boolean      // Auto-submit the pre-filled input
  },
) => void
```

This allows JSX commands to chain into model queries (e.g., `/model` changing the
model then optionally re-sending a pending message) or inject hidden context.

---

## 3. Command Interface

> **Source:** `src/types/command.ts`

All three command types share a `CommandBase` that carries metadata used by the
REPL typeahead, help system, skill index, analytics, and security gates:

```typescript
type CommandBase = {
  // Identity
  name: string                          // Primary name (e.g., 'commit', 'config')
  aliases?: string[]                    // Alternative names (e.g., 'settings' for config)
  userFacingName?: () => string         // Display name override (e.g., stripped plugin prefix)

  // Documentation
  description: string                   // One-line description for typeahead and help
  hasUserSpecifiedDescription?: boolean // Whether description came from frontmatter (vs auto-derived)
  argumentHint?: string                 // Gray hint text after command name (e.g., '[model]')
  whenToUse?: string                    // Extended usage guidance for the model's skill index

  // Visibility and access control
  availability?: CommandAvailability[]  // Auth/provider requirements ('claude-ai' | 'console')
  isEnabled?: () => boolean             // Dynamic enablement (feature flags, env checks)
  isHidden?: boolean                    // Hidden from typeahead/help but still invocable

  // Skill metadata
  version?: string                      // Skill version string
  disableModelInvocation?: boolean      // Block model from invoking via Skill tool
  userInvocable?: boolean               // Whether users can type /skill-name
  loadedFrom?: LoadedFrom               // Provenance tracking for filtering
  kind?: 'workflow'                     // Badge in autocomplete for workflow commands
  isMcp?: boolean                       // Loaded from an MCP server

  // Execution behavior
  immediate?: boolean                   // Execute immediately, bypass queue
  isSensitive?: boolean                 // Redact args from conversation history
}

type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

### Key Metadata Fields

**`loadedFrom`** tracks provenance and is critical for filtering commands into the
right views:

| Value | Meaning |
|---|---|
| `'commands_DEPRECATED'` | Legacy `~/.claude/commands/` directory |
| `'skills'` | Modern `~/.claude/skills/` or `.claude/skills/` |
| `'plugin'` | Installed plugin package |
| `'managed'` | Enterprise-managed policy skills |
| `'bundled'` | Ships compiled into the CLI binary |
| `'mcp'` | Discovered from an MCP server |

**`source`** on prompt commands records the setting source hierarchy level:

| Value | Meaning |
|---|---|
| `'policySettings'` | Enterprise policy (`/etc/claude-code/`) |
| `'userSettings'` | User home (`~/.claude/`) |
| `'projectSettings'` | Project repo (`.claude/`) |
| `'builtin'` | Hardcoded in the CLI source |
| `'plugin'` | Plugin package |
| `'bundled'` | Bundled skill |
| `'mcp'` | MCP server |

**`immediate`** controls execution timing. Most commands wait for the current model
turn to finish before executing. Commands with `immediate: true` (like `/exit`) bypass
the queue and execute instantly. Some commands use a dynamic getter:

```typescript
// src/commands/model/index.ts
get immediate() {
  return shouldInferenceConfigCommandBeImmediate()
}
```

**`description`** can be a dynamic getter that re-evaluates on access:

```typescript
// src/commands/model/index.ts
get description() {
  return `Set the AI model for Claude Code (currently ${renderModelName(getMainLoopModel())})`
}
```

---

## 4. Registry and Loading

> **Source:** `src/commands.ts`

### 4.1 The Multi-Layer Source Hierarchy

Commands are drawn from six distinct sources, loaded in parallel, and merged into
a single ordered array. The ordering determines precedence — earlier entries shadow
later ones when names collide:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    loadAllCommands(cwd)                              │
│                                                                     │
│  1. Bundled skills        ← getBundledSkills()                      │
│  2. Built-in plugin skills ← getBuiltinPluginSkillCommands()        │
│  3. User/project skills   ← getSkillDirCommands(cwd)                │
│  4. Workflow commands      ← getWorkflowCommands(cwd)               │
│  5. Plugin commands        ← getPluginCommands()                    │
│  6. Plugin skills          ← getPluginSkills()                      │
│  7. Built-in commands      ← COMMANDS()                             │
│                                                                     │
│  + Dynamic skills         ← getDynamicSkills() (inserted at step 7) │
└─────────────────────────────────────────────────────────────────────┘
```

This ordering means:
- Bundled skills can be shadowed by nothing (highest precedence)
- User/project skills override built-in commands of the same name
- Plugin commands sit between user skills and built-ins
- Built-in commands are the fallback layer

### 4.2 Built-In Commands (`COMMANDS()`)

The built-in command list is a memoized function (not a module-level constant) because
underlying functions read from config that is unavailable at module initialization time:

```typescript
const COMMANDS = memoize((): Command[] => [
  addDir,
  advisor,
  agents,
  branch,
  clear,
  compact,
  config,
  // ... ~80 more commands
  ...(bridge ? [bridge] : []),
  ...(voiceCommand ? [voiceCommand] : []),
  // ... conditionally included commands
  ...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO
    ? INTERNAL_ONLY_COMMANDS
    : []),
])
```

The array spreads are how feature-gated commands are included only when their
compile-time flag is active (see [Section 9](#9-feature-gating)).

### 4.3 Internal-Only Commands

A separate `INTERNAL_ONLY_COMMANDS` array contains commands restricted to Anthropic
internal users (`process.env.USER_TYPE === 'ant'`). These are compiled into the
binary but filtered out at runtime for external users:

```typescript
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions,
  breakCache,
  bughunter,
  commit,
  commitPushPr,
  ctx_viz,
  goodClaude,
  issue,
  // ... ~25 more
].filter(Boolean)
```

### 4.4 Skill Directory Loading

> **Source:** `src/skills/loadSkillsDir.ts`

`getSkillDirCommands(cwd)` loads skills from the filesystem hierarchy. It searches
four locations in parallel:

1. **Managed skills** — `/etc/claude-code/.claude/skills/` (enterprise policy)
2. **User skills** — `~/.claude/skills/` (personal)
3. **Project skills** — `.claude/skills/` at every directory from cwd up to `$HOME`
4. **Additional directory skills** — from `--add-dir` CLI flags
5. **Legacy commands** — `~/.claude/commands/` and `.claude/commands/` (deprecated path)

Each skill is a directory containing a `SKILL.md` file:

```
~/.claude/skills/
├── my-skill/
│   └── SKILL.md           ← Frontmatter + markdown prompt content
├── another-skill/
│   ├── SKILL.md
│   └── helper-script.sh   ← Ancillary files the skill can reference
```

**Deduplication**: Skills loaded from overlapping paths (symlinks, parent directories)
are deduplicated by resolving to canonical paths via `realpath()`. The first-loaded
copy wins.

**Bare mode**: When `--bare` is active, skill directory discovery is skipped entirely
except for explicit `--add-dir` paths. This is a startup optimization for headless/SDK
usage.

**Policy lockout**: When `isRestrictedToPluginOnly('skills')` is true, only
enterprise-managed skills load. User and project skills are suppressed.

### 4.5 Bundled Skills

> **Source:** `src/skills/bundledSkills.ts`, `src/skills/bundled/index.ts`

Bundled skills are compiled into the CLI binary and registered programmatically at
startup via `registerBundledSkill()`. They bypass filesystem loading entirely:

```typescript
export function registerBundledSkill(definition: BundledSkillDefinition): void {
  const command: Command = {
    type: 'prompt',
    name: definition.name,
    source: 'bundled',
    loadedFrom: 'bundled',
    // ...
  }
  bundledSkills.push(command)
}
```

`initBundledSkills()` is called at startup and registers all bundled skills:

```typescript
export function initBundledSkills(): void {
  registerUpdateConfigSkill()
  registerKeybindingsSkill()
  registerVerifySkill()
  registerDebugSkill()
  registerLoremIpsumSkill()
  registerSkillifySkill()
  registerRememberSkill()
  registerSimplifySkill()
  registerBatchSkill()
  registerStuckSkill()
  // ... feature-gated registrations
}
```

Bundled skills that include ancillary files (reference scripts, templates) use a
lazy-extraction mechanism: files are embedded in the binary and extracted to a
per-process temp directory (`getBundledSkillExtractDir()`) on first invocation.
The extraction uses `O_NOFOLLOW | O_EXCL` flags and per-process nonces to defend
against symlink attacks.

### 4.6 Plugin Commands and Skills

> **Source:** `src/utils/plugins/loadPluginCommands.ts`

Plugins provide two command namespaces:

- **Plugin commands** — from `commands/` directory in the plugin package
- **Plugin skills** — from `skills/` directory in the plugin package

Both are namespaced with the plugin name: a skill `deploy` in plugin `my-plugin`
becomes `/my-plugin:deploy`. This prevents cross-plugin name collisions.

Plugin commands support:
- YAML frontmatter metadata (description, allowed-tools, model, arguments)
- `${CLAUDE_PLUGIN_ROOT}` variable substitution in content and tool rules
- `${user_config.X}` substitution from plugin option storage
- `${CLAUDE_SESSION_ID}` for session tracking

### 4.7 Built-In Plugin Skills

> **Source:** `src/plugins/builtinPlugins.ts`

Built-in plugins are a hybrid between bundled skills and marketplace plugins. They
ship with the CLI but appear in the `/plugin` UI where users can enable/disable them.
Their skills use `source: 'bundled'` (not `'builtin'`) to ensure they appear in
the Skill tool listing and analytics.

```typescript
export function getBuiltinPluginSkillCommands(): Command[] {
  const { enabled } = getBuiltinPlugins()
  const commands: Command[] = []
  for (const plugin of enabled) {
    const definition = BUILTIN_PLUGINS.get(plugin.name)
    if (!definition?.skills) continue
    for (const skill of definition.skills) {
      commands.push(skillDefinitionToCommand(skill))
    }
  }
  return commands
}
```

### 4.8 MCP Skills

MCP-connected servers can expose skills that are loaded into the command registry.
These use the same `createSkillCommand()` infrastructure as filesystem skills but
are tagged with `loadedFrom: 'mcp'`. A critical security boundary: MCP skills never
execute inline shell commands (`!`...`` ` `` syntax) from their markdown body, because
MCP content is remote and untrusted:

```typescript
// In createSkillCommand():
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(finalContent, ...)
}
```

MCP skills are filtered separately via `getMcpSkillCommands()` because they live
in `AppState.mcp.commands` rather than in the main `getCommands()` pipeline.

### 4.9 Workflow Commands

Workflow-backed commands (gated behind `WORKFLOW_SCRIPTS`) are generated from
user-defined workflow YAML files. They are prompt-type commands with `kind: 'workflow'`
set, which causes them to display a "workflow" badge in the autocomplete UI.

---

## 5. Command Resolution

> **Source:** `src/commands.ts`

### 5.1 `getCommands(cwd)` — The Master Query

The primary entry point for obtaining the filtered, available command list:

```typescript
export async function getCommands(cwd: string): Promise<Command[]> {
  const allCommands = await loadAllCommands(cwd)
  const dynamicSkills = getDynamicSkills()

  const baseCommands = allCommands.filter(
    _ => meetsAvailabilityRequirement(_) && isCommandEnabled(_),
  )

  // Dedupe and insert dynamic skills before built-in commands
  // ...
  return mergedCommands
}
```

Key properties:
- `loadAllCommands` is memoized by `cwd` — the expensive disk I/O and dynamic
  imports happen once per working directory
- Availability and enablement checks run **fresh on every call** — auth state can
  change mid-session (e.g., after `/login`)
- Dynamic skills are merged in after the base list, positioned before built-in
  commands but after plugin commands

### 5.2 `findCommand()` — Name Lookup

```typescript
export function findCommand(
  commandName: string,
  commands: Command[],
): Command | undefined {
  return commands.find(
    _ =>
      _.name === commandName ||
      getCommandName(_) === commandName ||
      _.aliases?.includes(commandName),
  )
}
```

Resolution checks three identifiers in order:
1. Internal `name` (e.g., `'config'`)
2. User-facing name via `getCommandName()` (e.g., a display-name override)
3. Any alias (e.g., `'settings'` for `/config`, `'quit'` for `/exit`)

`getCommand()` wraps `findCommand()` with a throwing variant that produces a
diagnostic error listing all available commands and their aliases.

### 5.3 Memoization and Cache Invalidation

The loading pipeline uses multiple memoization layers:

| Cache | Scope | Invalidation |
|---|---|---|
| `loadAllCommands` | Per-cwd | `clearCommandMemoizationCaches()` |
| `COMMANDS` | Process-wide | Never (built-in commands are static) |
| `getSkillDirCommands` | Per-cwd | `clearSkillCaches()` |
| `getSkillToolCommands` | Per-cwd | `clearCommandMemoizationCaches()` |
| `getSlashCommandToolSkills` | Per-cwd | `clearCommandMemoizationCaches()` |
| `getPluginCommands` | Process-wide | `clearPluginCommandCache()` |
| `getPluginSkills` | Process-wide | `clearPluginSkillsCache()` |

Two invalidation entry points:

- **`clearCommandMemoizationCaches()`** — clears only the memoization layers,
  used when dynamic skills are added mid-session
- **`clearCommandsCache()`** — full reset of all caches including skill directory
  caches, plugin caches, and conditional skill state

Dynamic skill loading fires `skillsLoaded.emit()`, a signal that downstream caches
(like the skill search index) subscribe to for invalidation.

---

## 6. Execution Models

### 6.1 Prompt Command Execution

When a prompt command is invoked, the flow is:

```
User types /commit → findCommand('commit', commands) → cmd.getPromptForCommand(args, context)
    → ContentBlockParam[] → injected as user message → sent to LLM → model responds with tool calls
```

The `getPromptForCommand()` function can perform substantial work before returning:

1. **Argument substitution** — `${1}`, `${2}`, named `${arg_name}` placeholders
   replaced with user-provided arguments
2. **Variable substitution** — `${CLAUDE_SKILL_DIR}`, `${CLAUDE_SESSION_ID}`,
   `${CLAUDE_PLUGIN_ROOT}` replaced with runtime values
3. **Shell command execution** — Inline `` !`command` `` and ``` ```! ... ``` ```
   blocks are executed and their output spliced into the prompt text
4. **Base directory injection** — Skills from directories get a
   `"Base directory for this skill: <path>"` prefix

For built-in prompt commands like `/commit`, the prompt is constructed programmatically:

```typescript
const command = {
  type: 'prompt',
  name: 'commit',
  allowedTools: [
    'Bash(git add:*)',
    'Bash(git status:*)',
    'Bash(git commit:*)',
  ],
  async getPromptForCommand(_args, context) {
    const promptContent = getPromptContent()  // Template with !`git status` etc.
    const finalContent = await executeShellCommandsInPrompt(promptContent, ...)
    return [{ type: 'text', text: finalContent }]
  },
} satisfies Command
```

### 6.2 Local Command Execution

Local commands are loaded lazily and called directly:

```
User types /compact → findCommand('compact', commands) → cmd.load()
    → { call } → call(args, context) → LocalCommandResult
```

The result is processed based on its `type`:
- `'text'` — displayed as system message
- `'compact'` — triggers compaction UI with the `CompactionResult`
- `'skip'` — no visible output

Some local commands define their `call` inline and return it from `load()` immediately:

```typescript
// src/commands/advisor.ts
const advisor = {
  type: 'local',
  name: 'advisor',
  supportsNonInteractive: true,
  load: () => Promise.resolve({ call }),  // No dynamic import needed
} satisfies Command
```

### 6.3 Local JSX Command Execution

JSX commands render interactive Ink UI:

```
User types /config → findCommand('config', commands) → cmd.load()
    → { call } → call(onDone, context, args) → React.ReactNode
```

The returned React node is rendered in the terminal. When the UI interaction completes,
`onDone()` is called with optional result text and display options. The `onDone`
callback controls what happens next:

- `display: 'skip'` — suppress the result message
- `display: 'system'` — show as a system notification
- `display: 'user'` — show as if the user typed it
- `shouldQuery: true` — after the command completes, send messages to the model
- `metaMessages` — inject hidden messages visible to the model but not the user

### 6.4 Lazy Loading

All three command types use lazy loading to minimize startup time. The `load()`
function returns a `Promise` of the implementation module:

```typescript
// Metadata is always available (for typeahead, help)
const compact = {
  type: 'local',
  name: 'compact',
  description: 'Clear conversation history but keep a summary in context.',
  load: () => import('./compact.js'),  // Heavy implementation loaded on demand
} satisfies Command
```

For the `insights` command (113KB, 3200 lines), even the prompt command uses a
lazy shim:

```typescript
const usageReport: Command = {
  type: 'prompt',
  name: 'insights',
  async getPromptForCommand(args, context) {
    const real = (await import('./commands/insights.js')).default
    if (real.type !== 'prompt') throw new Error('unreachable')
    return real.getPromptForCommand(args, context)
  },
}
```

### 6.5 Execution Context

Prompt commands that specify `context: 'fork'` in their frontmatter run as sub-agents
with separate context and token budgets, rather than expanding inline into the current
conversation. The `agent` field can further specify which agent type to use for the fork.

---

## 7. Availability and Enablement

The system has two independent gates that control whether a command appears in the
UI and can be invoked:

### 7.1 Availability — Auth/Provider Requirements

```typescript
type CommandAvailability = 'claude-ai' | 'console'
```

`meetsAvailabilityRequirement()` checks whether the current auth context matches
at least one of the command's declared availability values:

```typescript
export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true   // No restriction = universal
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai':
        if (isClaudeAISubscriber()) return true
        break
      case 'console':
        if (!isClaudeAISubscriber() && !isUsing3PServices()
            && isFirstPartyAnthropicBaseUrl()) return true
        break
    }
  }
  return false
}
```

Commands without `availability` are available to everyone. Commands with
`availability: ['claude-ai', 'console']` are visible to claude.ai subscribers
and direct Console API key users, but hidden from Bedrock/Vertex/Foundry users.

This check is deliberately **not memoized** — auth state can change mid-session
(e.g., after `/login`).

### 7.2 Enablement — Dynamic Feature Gates

`isEnabled()` is an optional callback that returns `false` to disable the command
at runtime. Default is `true` when not specified:

```typescript
export function isCommandEnabled(cmd: CommandBase): boolean {
  return cmd.isEnabled?.() ?? true
}
```

Enablement reasons include:
- **Feature flags**: `isEnabled: () => isFastModeEnabled()`
- **Environment variables**: `isEnabled: () => !isEnvTruthy(process.env.DISABLE_COMPACT)`
- **Runtime conditions**: `isEnabled: () => canUserConfigureAdvisor()`
- **User type gates**: checking `process.env.USER_TYPE === 'ant'` inside the callback

### 7.3 Remote/Bridge Safety Classification

Two allowlists gate which commands can execute in constrained environments:

**`REMOTE_SAFE_COMMANDS`** — commands safe for `--remote` mode (TUI state only,
no local filesystem/git/shell dependency):

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim, cost, usage,
  copy, btw, feedback, plan, keybindings, statusline, stickers, mobile,
])
```

**`BRIDGE_SAFE_COMMANDS`** — local commands safe to execute when input arrives
over the Remote Control bridge (mobile/web client):

```typescript
export const BRIDGE_SAFE_COMMANDS: Set<Command> = new Set([
  compact, clear, cost, summary, releaseNotes, files,
])
```

The `isBridgeSafeCommand()` predicate implements a type-based policy:
- `'local-jsx'` commands are always blocked (they render Ink UI)
- `'prompt'` commands are always allowed (they expand to text)
- `'local'` commands need explicit opt-in via the allowlist

### 7.4 Visibility vs. Invocability

Three separate mechanisms control visibility:

| Field | Effect |
|---|---|
| `isHidden: true` | Hidden from typeahead and help, but still invocable by name |
| `isEnabled: () => false` | Fully disabled — invisible and non-functional |
| `availability` mismatch | Fully filtered out — invisible and non-functional |
| `disableModelInvocation: true` | Hidden from the model's Skill tool, but user can still type it |
| `userInvocable: false` | Model can invoke it, but hidden from user typeahead |

---

## 8. The Skill System

### 8.1 Skill File Format

A skill is a directory containing a `SKILL.md` file with optional YAML frontmatter:

```markdown
---
description: Deploy the current branch to staging
allowed-tools: Bash(deploy:*), Bash(kubectl:*)
argument-hint: <environment>
arguments: environment
when_to_use: When the user wants to deploy code to a staging or production environment
model: sonnet
context: fork
paths: src/deploy/**, infrastructure/**
---

You are a deployment assistant. Deploy the code to the ${1} environment.

Current git status: !`git status --short`
Current branch: !`git branch --show-current`

## Steps
1. Run the deployment script
2. Verify the deployment succeeded
3. Report the results
```

### 8.2 Frontmatter Fields

`parseSkillFrontmatterFields()` extracts the following from frontmatter:

| Field | Type | Default | Purpose |
|---|---|---|---|
| `description` | string | Auto-derived from first line | One-line description |
| `name` | string | Directory name | Display name override |
| `allowed-tools` | string/string[] | `[]` | Tool permission constraints |
| `argument-hint` | string | — | Gray hint text in typeahead |
| `arguments` | string/string[] | — | Named argument placeholders |
| `when_to_use` | string | — | Extended guidance for model's skill index |
| `version` | string | — | Skill version |
| `model` | string | inherit | Override model (e.g., `'sonnet'`, `'haiku'`) |
| `disable-model-invocation` | boolean | `false` | Block model from invoking |
| `user-invocable` | boolean | `true` | Whether user can type `/skill-name` |
| `hooks` | HooksSettings | — | Lifecycle hooks to register on invocation |
| `context` | `'fork'` | `'inline'` | Run as sub-agent vs. inline expansion |
| `agent` | string | — | Agent type for forked execution |
| `effort` | EffortValue | — | Reasoning effort override |
| `paths` | string[] | — | Glob patterns for conditional activation |
| `shell` | string | — | Shell to use for inline command execution |

### 8.3 Skill Directory Hierarchy

Skills are discovered from multiple filesystem locations, searched in priority order:

```
/etc/claude-code/.claude/skills/     ← Enterprise policy (policySettings)
~/.claude/skills/                    ← User personal (userSettings)
<cwd>/.claude/skills/                ← Project root (projectSettings)
<cwd>/subdir/.claude/skills/         ← Nested project dirs (dynamic discovery)
<--add-dir>/.claude/skills/          ← Explicit additional directories
~/.claude/commands/                  ← Legacy path (commands_DEPRECATED)
.claude/commands/                    ← Legacy project path (commands_DEPRECATED)
```

### 8.4 Dynamic Skill Discovery

> **Source:** `src/skills/loadSkillsDir.ts`, `discoverSkillDirsForPaths()`

Skills are not only loaded at startup. When the model reads or writes files in
subdirectories below cwd, the system walks up from the file path looking for
`.claude/skills/` directories. Newly discovered directories are loaded and their
skills merged into the registry:

```typescript
export async function discoverSkillDirsForPaths(
  filePaths: string[],
  cwd: string,
): Promise<string[]> {
  // For each file path, walk up to cwd (exclusive)
  // Check for .claude/skills/ at each level
  // Skip gitignored directories
  // Return newly discovered directories, sorted deepest first
}
```

Discovered skills are tracked in a module-scoped `dynamicSkills` map. On discovery,
`skillsLoaded.emit()` fires to invalidate downstream caches.

### 8.5 Conditional Skills (Path-Filtered)

Skills with a `paths` frontmatter field are held in a pending state until the model
touches files matching the glob patterns. This prevents skills specific to a subdirectory
(e.g., `paths: src/deploy/**`) from cluttering the skill index when the user is
working in unrelated code:

```typescript
export function activateConditionalSkillsForPaths(
  filePaths: string[],
  cwd: string,
): string[] {
  for (const [name, skill] of conditionalSkills) {
    const skillIgnore = ignore().add(skill.paths)
    for (const filePath of filePaths) {
      const relativePath = relative(cwd, filePath)
      if (skillIgnore.ignores(relativePath)) {
        dynamicSkills.set(name, skill)
        conditionalSkills.delete(name)
        activatedConditionalSkillNames.add(name)
        // ...
      }
    }
  }
}
```

Once activated, a conditional skill remains active for the rest of the session
(tracked in `activatedConditionalSkillNames` which survives cache clears).

### 8.6 Skill Tool Filtering

Two filtered views of the command list serve the model:

**`getSkillToolCommands(cwd)`** — all prompt-based commands the model can invoke
via the Skill tool:

```typescript
export const getSkillToolCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const allCommands = await getCommands(cwd)
  return allCommands.filter(cmd =>
    cmd.type === 'prompt' &&
    !cmd.disableModelInvocation &&
    cmd.source !== 'builtin' &&
    (cmd.loadedFrom === 'bundled' ||
     cmd.loadedFrom === 'skills' ||
     cmd.loadedFrom === 'commands_DEPRECATED' ||
     cmd.hasUserSpecifiedDescription ||
     cmd.whenToUse)
  )
})
```

**`getSlashCommandToolSkills(cwd)`** — the subset exposed in the `/skills` listing:

```typescript
export const getSlashCommandToolSkills = memoize(async (cwd: string): Promise<Command[]> => {
  const allCommands = await getCommands(cwd)
  return allCommands.filter(cmd =>
    cmd.type === 'prompt' &&
    cmd.source !== 'builtin' &&
    (cmd.hasUserSpecifiedDescription || cmd.whenToUse) &&
    (cmd.loadedFrom === 'skills' ||
     cmd.loadedFrom === 'plugin' ||
     cmd.loadedFrom === 'bundled' ||
     cmd.disableModelInvocation)
  )
})
```

---

## 9. Feature Gating

### 9.1 Compile-Time Gating via `bun:bundle`

Claude Code uses Bun's `feature()` function for compile-time dead code elimination.
Commands gated behind features are either included in or excluded from the binary
at build time:

```typescript
import { feature } from 'bun:bundle'

const bridge = feature('BRIDGE_MODE')
  ? require('./commands/bridge/index.js').default
  : null

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

When a feature flag is `false`, the `require()` call is eliminated entirely — the
command module is not included in the compiled binary, saving bundle size.

The `COMMANDS()` array spreads these nullable values:

```typescript
const COMMANDS = memoize((): Command[] => [
  // ... always-present commands
  ...(bridge ? [bridge] : []),
  ...(voiceCommand ? [voiceCommand] : []),
  ...(proactive ? [proactive] : []),
])
```

### 9.2 Feature Flags in Use

| Feature Flag | Commands/Skills Gated |
|---|---|
| `BRIDGE_MODE` | `/bridge`, `/remoteControlServer` |
| `VOICE_MODE` | `/voice` |
| `PROACTIVE` or `KAIROS` | `/proactive` |
| `KAIROS` | `/assistant`, `/brief` |
| `KAIROS_BRIEF` | `/brief` |
| `DAEMON` + `BRIDGE_MODE` | `/remoteControlServer` |
| `HISTORY_SNIP` | `/force-snip` |
| `WORKFLOW_SCRIPTS` | `/workflows`, workflow commands |
| `CCR_REMOTE_SETUP` | `/remote-setup` |
| `EXPERIMENTAL_SKILL_SEARCH` | Skill search index cache clearing |
| `KAIROS_GITHUB_WEBHOOKS` | `/subscribe-pr` |
| `ULTRAPLAN` | `/ultraplan` |
| `TORCH` | `/torch` |
| `UDS_INBOX` | `/peers` |
| `FORK_SUBAGENT` | `/fork` |
| `BUDDY` | `/buddy` |
| `MCP_SKILLS` | MCP skill filtering |

### 9.3 Bundled Skill Feature Gating

Bundled skills use the same `feature()` mechanism but at registration time:

```typescript
export function initBundledSkills(): void {
  registerUpdateConfigSkill()   // Always registered
  registerVerifySkill()         // Always registered
  // ...
  if (feature('KAIROS') || feature('KAIROS_DREAM')) {
    const { registerDreamSkill } = require('./dream.js')
    registerDreamSkill()
  }
  if (feature('REVIEW_ARTIFACT')) {
    const { registerHunterSkill } = require('./hunter.js')
    registerHunterSkill()
  }
  // ...
}
```

### 9.4 Internal-Only Gating

Beyond compile-time flags, some commands use runtime environment checks:

```typescript
// In bundled skill registration:
export function registerStuckSkill(): void {
  if (process.env.USER_TYPE !== 'ant') {
    return   // Skip registration entirely for external users
  }
  registerBundledSkill({ name: 'stuck', ... })
}
```

---

## 10. Tool Permission Constraints

### 10.1 The `allowedTools` Array

Prompt commands can declare an `allowedTools` array that constrains which tools the
model is permitted to use when executing the command's instructions. This is a
security and correctness mechanism — `/commit` should only use git commands, not
arbitrary bash:

```typescript
const ALLOWED_TOOLS = [
  'Bash(git add:*)',
  'Bash(git status:*)',
  'Bash(git commit:*)',
]

const command = {
  type: 'prompt',
  name: 'commit',
  allowedTools: ALLOWED_TOOLS,
  // ...
}
```

### 10.2 Permission Propagation

When a skill executes, its `allowedTools` are merged into the `toolPermissionContext`
passed to `executeShellCommandsInPrompt()`:

```typescript
async getPromptForCommand(args, context) {
  const finalContent = await executeShellCommandsInPrompt(
    promptContent,
    {
      ...context,
      getAppState() {
        const appState = context.getAppState()
        return {
          ...appState,
          toolPermissionContext: {
            ...appState.toolPermissionContext,
            alwaysAllowRules: {
              ...appState.toolPermissionContext.alwaysAllowRules,
              command: allowedTools,
            },
          },
        }
      },
    },
    '/commit',
  )
  return [{ type: 'text', text: finalContent }]
}
```

This means that tools declared in `allowedTools` are auto-approved for the inline
shell commands (`` !`...` ``) that fire during prompt expansion, without requiring
user confirmation.

### 10.3 Plugin Variable Substitution

Plugin commands support `${CLAUDE_PLUGIN_ROOT}` in their `allowed-tools` values,
allowing tool rules relative to the plugin's installation directory:

```typescript
const substitutedAllowedTools =
  typeof rawAllowedTools === 'string'
    ? substitutePluginVariables(rawAllowedTools, { path: pluginPath, source: sourceName })
    : rawAllowedTools
```

---

## 11. Key Commands by Category

### 11.1 Session Management

| Command | Type | Description |
|---|---|---|
| `/clear` (aliases: `/reset`, `/new`) | local | Clear conversation history |
| `/compact` | local | Summarize old context to free tokens |
| `/exit` (alias: `/quit`) | local-jsx | Exit the REPL (immediate) |
| `/resume` | local-jsx | Resume a previous session |
| `/session` | local-jsx | Show session QR code/URL |

### 11.2 Git and Code Review

| Command | Type | Description |
|---|---|---|
| `/commit` | prompt | Create a git commit (internal) |
| `/review` | prompt | Review a pull request locally |
| `/ultrareview` | local-jsx | Remote bug-finding review |
| `/diff` | local-jsx | Show git diff |
| `/pr_comments` | local-jsx | View PR comments |

### 11.3 Configuration

| Command | Type | Description |
|---|---|---|
| `/config` (alias: `/settings`) | local-jsx | Configuration panel |
| `/model` | local-jsx | Set the AI model |
| `/fast` | local-jsx | Toggle fast mode |
| `/theme` | local-jsx | Change terminal theme |
| `/vim` | local-jsx | Toggle vim keybindings |
| `/permissions` | local-jsx | Manage tool permissions |
| `/effort` | local-jsx | Set reasoning effort level |
| `/advisor` | local | Configure advisor model |

### 11.4 Context and Memory

| Command | Type | Description |
|---|---|---|
| `/memory` | local-jsx | Manage auto-memory |
| `/context` | local-jsx | View context window usage |
| `/cost` | local-jsx | Show session cost |
| `/files` | local | List tracked files |

### 11.5 Extension Points

| Command | Type | Description |
|---|---|---|
| `/skills` | local-jsx | List available skills |
| `/mcp` | local-jsx | Manage MCP server connections |
| `/plugin` | local-jsx | Manage plugins |

### 11.6 Information and Help

| Command | Type | Description |
|---|---|---|
| `/help` | local-jsx | Show help screen |
| `/status` | local-jsx | Show system status |
| `/doctor` | local-jsx | Diagnose installation issues |
| `/usage` | local-jsx | Show API usage statistics |
| `/insights` | prompt | Generate session analysis report |

### 11.7 Authentication

| Command | Type | Description |
|---|---|---|
| `/login` | local-jsx | Sign in with Anthropic account |
| `/logout` | local-jsx | Sign out |

### 11.8 Bundled Skills (Model-Invocable)

| Skill | Description |
|---|---|
| `update-config` | Update Claude Code configuration |
| `keybindings` | Manage keyboard shortcuts |
| `verify` | Verify code changes |
| `debug` | Debug issues |
| `lorem-ipsum` | Generate placeholder text |
| `skillify` | Create a new skill from conversation |
| `remember` | Save information to memory |
| `simplify` | Simplify code |
| `batch` | Run batch operations |
| `stuck` | Diagnose frozen sessions (internal) |

---

## 12. Design Principles

### 12.1 Unified Abstraction

A single `Command` type serves all audiences: the REPL typeahead, the model's skill
index, the help system, analytics, and the execution engine. Adding a new command
means defining one object that satisfies the `Command` type — no registration in
multiple places.

### 12.2 Lazy Everything

Command implementations are never loaded at startup. Every command exports only its
metadata (name, description, type) at the top level. The actual implementation is
behind a `load()` function or `getPromptForCommand()` that triggers dynamic imports
only when the command is invoked. The `insights` command takes this further with a
lazy shim around a 113KB module.

### 12.3 Source Hierarchy with Deterministic Precedence

The six-layer source hierarchy (bundled > builtin-plugin > user/project > workflow >
plugin > built-in) ensures predictable shadowing. A user skill named `commit` will
override the built-in `/commit`. Enterprise policy skills take the highest precedence.

### 12.4 Separation of Visibility from Capability

Three independent mechanisms (`availability`, `isEnabled`, `isHidden`) allow
fine-grained control over who sees what. A command can be invocable but hidden from
typeahead (`isHidden`), visible but disabled (`isEnabled: false`), or fully filtered
by auth context (`availability`). The model has its own visibility dimension via
`disableModelInvocation` and `userInvocable`.

### 12.5 Security-Conscious Extension

Skills from untrusted sources (MCP servers) are sandboxed: no inline shell execution,
explicit tool permission boundaries. Plugin commands have namespace isolation
(`plugin-name:command-name`). Enterprise policy can lock out user and project skills
entirely. Bundled skill file extraction uses TOCTOU-safe file creation with
`O_NOFOLLOW | O_EXCL` and per-process nonces.

### 12.6 Progressive Discovery

Conditional skills (with `paths` frontmatter) and dynamic skill discovery mean the
model's skill index grows organically as the user works deeper into a codebase.
A monorepo with specialized skills in each package only shows relevant skills when
files in that package are touched.

### 12.7 Graceful Degradation

Every layer of skill loading is wrapped in try/catch with fallback to empty arrays:

```typescript
const [skillDirCommands, pluginSkills] = await Promise.all([
  getSkillDirCommands(cwd).catch(err => {
    logError(toError(err))
    return []
  }),
  getPluginSkills().catch(err => {
    logError(toError(err))
    return []
  }),
])
```

A broken skill file, inaccessible directory, or failing plugin never crashes the
system. Built-in commands remain available regardless of what happens in the skill
loading pipeline.

### 12.8 Cache-Conscious Architecture

Loading is expensive (filesystem walks, dynamic imports, frontmatter parsing), so
results are memoized aggressively. But availability and enablement checks run fresh
on every `getCommands()` call because auth state is mutable. The two-tier design
(memoized loading + fresh filtering) balances startup cost against runtime correctness.

### 12.9 Extensibility Without Framework Lock-In

Skills are markdown files with optional YAML frontmatter — no TypeScript, no build
step, no SDK dependency. A user can create a skill by writing a `SKILL.md` file in
a directory. This makes the extension surface accessible to non-programmers and
compatible with any editor or version control workflow.
