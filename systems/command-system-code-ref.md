# Command System — Code Reference

### Source Files

| Area | File Path |
|---|---|
| Registry, loading, resolution | `src/commands.ts` |
| Type definitions | `src/types/command.ts` |
| Skill directory loading | `src/skills/loadSkillsDir.ts` |
| Bundled skill registration | `src/skills/bundledSkills.ts` |
| Bundled skill modules | `src/skills/bundled/index.ts` |
| Plugin command loading | `src/utils/plugins/loadPluginCommands.ts` |
| Built-in plugins | `src/plugins/builtinPlugins.ts` |

### Type Definitions (`src/types/command.ts`)

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
}

type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<LocalCommandModule>
}

type LocalCommandResult =
  | { type: 'text'; value: string }
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
  | { type: 'skip' }

type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}

type LocalJSXCommandCall = (
  onDone: LocalJSXCommandOnDone,
  context: ToolUseContext & LocalJSXCommandContext,
  args: string,
) => Promise<React.ReactNode>

type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: 'skip' | 'system' | 'user'
    shouldQuery?: boolean
    metaMessages?: string[]
    nextInput?: string
    submitNextInput?: boolean
  },
) => void

type CommandAvailability = 'claude-ai' | 'console'

type CommandBase = {
  name: string
  aliases?: string[]
  userFacingName?: () => string
  description: string
  hasUserSpecifiedDescription?: boolean
  argumentHint?: string
  whenToUse?: string
  availability?: CommandAvailability[]
  isEnabled?: () => boolean
  isHidden?: boolean
  version?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  loadedFrom?: LoadedFrom
  kind?: 'workflow'
  isMcp?: boolean
  immediate?: boolean
  isSensitive?: boolean
}

type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

### Registry Functions (`src/commands.ts`)

**`loadAllCommands(cwd)`** -- Memoized per-cwd. Loads from all seven sources in parallel and merges into ordered array.

**`COMMANDS()`** -- Memoized process-wide. Returns the built-in command list:
```typescript
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, clear, compact, config,
  // ... ~80 more commands
  ...(bridge ? [bridge] : []),
  ...(voiceCommand ? [voiceCommand] : []),
  ...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO
    ? INTERNAL_ONLY_COMMANDS : []),
])
```

**`INTERNAL_ONLY_COMMANDS`** -- Exported array, filtered by `.filter(Boolean)`:
```typescript
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, // ... ~25 more
].filter(Boolean)
```

**`getCommands(cwd)`** -- Main entry point. Calls `loadAllCommands`, filters by `meetsAvailabilityRequirement()` and `isCommandEnabled()`, merges dynamic skills.

**`findCommand(commandName, commands)`** -- Checks `_.name`, `getCommandName(_)`, `_.aliases?.includes(commandName)` in order.

**`getCommand()`** -- Throwing wrapper around `findCommand()`.

### Cache Functions (`src/commands.ts`)

| Function | What It Clears |
|---|---|
| `clearCommandMemoizationCaches()` | `loadAllCommands`, `getSkillToolCommands`, `getSlashCommandToolSkills` |
| `clearCommandsCache()` | All of the above plus `clearSkillCaches()`, `clearPluginCommandCache()`, `clearPluginSkillsCache()`, conditional skill state |

Signal: `skillsLoaded.emit()` fires on dynamic skill discovery.

### Availability Check (`src/commands.ts`)

```typescript
export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true
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

export function isCommandEnabled(cmd: CommandBase): boolean {
  return cmd.isEnabled?.() ?? true
}
```

### Skill Loading (`src/skills/loadSkillsDir.ts`)

**`getSkillDirCommands(cwd)`** -- Loads skills from filesystem. Searches:
- `/etc/claude-code/.claude/skills/`
- `~/.claude/skills/`
- `.claude/skills/` at every directory from cwd up to `$HOME`
- `--add-dir` paths
- `~/.claude/commands/` and `.claude/commands/` (legacy)

**`discoverSkillDirsForPaths(filePaths, cwd)`** -- Walks up from file paths to cwd, checks for `.claude/skills/` directories, skips gitignored directories, returns newly discovered directories sorted deepest first.

**`activateConditionalSkillsForPaths(filePaths, cwd)`** -- Uses `ignore().add(skill.paths)` to match file paths against glob patterns. Moves matching skills from `conditionalSkills` map to `dynamicSkills` map.

### Bundled Skills (`src/skills/bundledSkills.ts`, `src/skills/bundled/index.ts`)

**`registerBundledSkill(definition: BundledSkillDefinition)`** -- Creates Command object with `source: 'bundled'`, `loadedFrom: 'bundled'`, pushes to module-scoped `bundledSkills` array.

**`initBundledSkills()`** -- Called at startup. Registers all bundled skills, some gated:
```typescript
if (feature('KAIROS') || feature('KAIROS_DREAM')) {
  const { registerDreamSkill } = require('./dream.js')
  registerDreamSkill()
}
if (feature('REVIEW_ARTIFACT')) {
  const { registerHunterSkill } = require('./hunter.js')
  registerHunterSkill()
}
```

**`getBundledSkillExtractDir()`** -- Returns per-process temp directory for extracted ancillary files. Uses `O_NOFOLLOW | O_EXCL` flags.

### Built-In Plugin Skills (`src/plugins/builtinPlugins.ts`)

```typescript
export function getBuiltinPluginSkillCommands(): Command[] {
  const { enabled } = getBuiltinPlugins()
  // Iterates enabled plugins, calls skillDefinitionToCommand() for each skill
}
```

### Plugin Commands (`src/utils/plugins/loadPluginCommands.ts`)

Plugin commands support variable substitution:
- `${CLAUDE_PLUGIN_ROOT}` -- plugin installation directory
- `${user_config.X}` -- plugin option storage
- `${CLAUDE_SESSION_ID}` -- session tracking

Namespacing: `plugin-name:command-name`.

### Skill Tool Filtering (`src/commands.ts`)

**`getSkillToolCommands(cwd)`** -- Memoized. Filters: `type === 'prompt'`, `!disableModelInvocation`, `source !== 'builtin'`, provenance or description checks.

**`getSlashCommandToolSkills(cwd)`** -- Memoized. Filters: `type === 'prompt'`, `source !== 'builtin'`, description or whenToUse present, specific `loadedFrom` values or `disableModelInvocation`.

### Feature Gating Mechanism

Compile-time via `bun:bundle`:
```typescript
import { feature } from 'bun:bundle'
const bridge = feature('BRIDGE_MODE') ? require('./commands/bridge/index.js').default : null
const voiceCommand = feature('VOICE_MODE') ? require('./commands/voice/index.js').default : null
```

Runtime internal-only check pattern:
```typescript
export function registerStuckSkill(): void {
  if (process.env.USER_TYPE !== 'ant') return
  registerBundledSkill({ name: 'stuck', ... })
}
```

### Remote/Bridge Safety Sets (`src/commands.ts`)

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim, cost, usage,
  copy, btw, feedback, plan, keybindings, statusline, stickers, mobile,
])

export const BRIDGE_SAFE_COMMANDS: Set<Command> = new Set([
  compact, clear, cost, summary, releaseNotes, files,
])
```

Bridge safety type policy: `local-jsx` always blocked, `prompt` always allowed, `local` requires explicit allowlist opt-in.

### MCP Skill Security Boundary

```typescript
// In createSkillCommand():
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(finalContent, ...)
}
```

### Prompt Command Permission Propagation

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
}
```

### Lazy Loading Patterns

Standard lazy import:
```typescript
load: () => import('./compact.js')
```

Inline call return (no dynamic import needed):
```typescript
// src/commands/advisor.ts
load: () => Promise.resolve({ call })
```

Lazy shim for large modules:
```typescript
// src/commands.ts -- insights command (113KB)
async getPromptForCommand(args, context) {
  const real = (await import('./commands/insights.js')).default
  if (real.type !== 'prompt') throw new Error('unreachable')
  return real.getPromptForCommand(args, context)
}
```

### Dynamic Description/Immediate Getters

```typescript
// src/commands/model/index.ts
get immediate() {
  return shouldInferenceConfigCommandBeImmediate()
}

get description() {
  return `Set the AI model for Claude Code (currently ${renderModelName(getMainLoopModel())})`
}
```

### Frontmatter Parsing

`parseSkillFrontmatterFields()` extracts all fields listed in the design document from YAML frontmatter in `SKILL.md` files. The `paths` field uses the `ignore` library for glob matching. The `context` field accepts `'fork'` or `'inline'` (default: `'inline'`) to control execution context. The `shell` field overrides the shell used for inline `!`...`` ` `` command execution.

### Graceful Degradation Pattern

```typescript
const [skillDirCommands, pluginSkills] = await Promise.all([
  getSkillDirCommands(cwd).catch(err => { logError(toError(err)); return [] }),
  getPluginSkills().catch(err => { logError(toError(err)); return [] }),
])
```

### Feature Flag Reference Table

| Flag | Gated Items |
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
| `KAIROS_DREAM` | Dream skill (bundled) |
| `REVIEW_ARTIFACT` | Hunter skill (bundled) |
