# Plugin & Skill System — Code Reference

### Source Files

| Component | Source File |
|---|---|
| Skill directory loading | `src/skills/loadSkillsDir.ts` |
| Bundled skills registry | `src/skills/bundledSkills.ts`, `src/skills/bundled/index.ts` |
| MCP skill builders | `src/skills/mcpSkillBuilders.ts` |
| Plugin loader | `src/utils/plugins/pluginLoader.ts` |
| Plugin schemas | `src/utils/plugins/schemas.ts` |
| Plugin reconciler | `src/utils/plugins/reconciler.ts` |
| Plugin installation manager | `src/services/plugins/PluginInstallationManager.ts` |
| Plugin operations | `src/services/plugins/pluginOperations.ts` |
| Plugin CLI commands | `src/services/plugins/pluginCliCommands.ts` |
| Plugin validation | `src/utils/plugins/validatePlugin.ts` |
| Plugin policy | `src/utils/plugins/pluginPolicy.ts` |
| Command registry | `src/commands.ts` |
| SkillTool | `src/tools/SkillTool/SkillTool.ts` |
| SkillTool prompt | `src/tools/SkillTool/prompt.ts` |

### Key Functions and Identifiers

**Skill Loading (`src/skills/loadSkillsDir.ts`)**:
- `loadSkillsFromDir()` -- loads skills from a single directory
- `discoverSkillDirsForPaths()` -- walks parent directories to find skill dirs
- `activateConditionalSkillsForPaths()` -- activates skills matching `paths` globs
- `parseSkillFrontmatterFields()` -- parses YAML frontmatter from SKILL.md
- `createSkillCommand()` -- creates a `Command` object from a skill definition
- `getProjectDirsUpToHome()` -- returns project directory walk results
- Session-scoped state: `dynamicSkillDirs` set, `dynamicSkills` map, `conditionalSkills` map
- Signal: `skillsLoaded` with subscriber registration via `onDynamicSkillsLoaded()`
- Gitignore check: `isPathGitignored()`

**Bundled Skills (`src/skills/bundledSkills.ts`, `src/skills/bundled/index.ts`)**:
- `registerBundledSkill()` -- pushes a `Command` into the bundled skills array
- `getBundledSkillsRoot()` -- returns nonce-bearing temp directory path
- `resolveSkillFilePath()` -- validates and resolves relative file paths (rejects `..` and absolute paths)
- Properties: `source: 'bundled'`, `loadedFrom: 'bundled'`, `contentLength: 0`
- `files: Record<string, string>` -- auxiliary reference files
- File write flags: `O_CREAT | O_EXCL | O_NOFOLLOW`, permissions `0o600` (files), `0o700` (dirs)
- Feature gates: `KAIROS`, `AGENT_TRIGGERS`
- Bundled skill names: `/verify`, `/debug`, `/lorem-ipsum`, `/skillify`, `/remember`, `/simplify`, `/batch`, `/stuck`, `/update-config`, `/keybindings`, `/dream`, `/hunter`, `/loop`, `/schedule-remote-agents`, `/claude-api`, `/claude-in-chrome`

**MCP Skills (`src/skills/mcpSkillBuilders.ts`)**:
- Write-once registry pattern for `createSkillCommand` and `parseSkillFrontmatterFields`
- Property: `loadedFrom: 'mcp'`
- Security: `getPromptForCommand()` skips `executeShellCommandsInPrompt()` when `loadedFrom === 'mcp'`

**Plugin Schemas (`src/utils/plugins/schemas.ts`)**:
- `PluginManifestSchema` -- Zod schema for `plugin.json`
- `PluginIdSchema` -- enforces `plugin@marketplace` format with alphanumeric/hyphen/dot/underscore
- Top-level unknown fields: silently stripped (`.passthrough()` or equivalent)
- Nested objects (`userConfig`, `channels`, `lspServers`): `.strict()` validation

**Plugin Loader (`src/utils/plugins/pluginLoader.ts`)**:
- `loadAllPlugins()` -- full load with marketplace access
- `loadAllPluginsCacheOnly()` -- cache-only, non-blocking startup
- `getBuiltinPlugins()` -- returns built-in plugins with reserved marketplace `builtin`
- Caching: via `memoize`
- Plugin resolution: from `plugin@marketplace` entries in settings, `--plugin-dir` CLI flag, SDK `plugins` option
- `verifyAndDemote()` -- checks dependency graphs and demotes plugins with missing dependencies

**Plugin Operations (`src/services/plugins/pluginOperations.ts`)**:
- `installPluginOp()` -- settings-first install
- `uninstallPluginOp()` -- remove from settings, clear caches, remove V2 record
- `setPluginEnabledOp()` -- resolve ID and scope, check policy, write settings
- `updatePluginOp()` -- non-inplace update, versioned cache, V2 record update
- Scope precedence: `local > project > user`

**Plugin Installation Manager (`src/services/plugins/PluginInstallationManager.ts`)**:
- `performBackgroundPluginInstallations()` -- async background install
- `reconcileMarketplaces()` -- called with progress callbacks
- `refreshActivePlugins()` -- auto-refresh on new installs
- AppState integration: pending status, progress callbacks, `needsRefresh` flag

**Plugin Reconciler (`src/utils/plugins/reconciler.ts`)**:
- Computes diff between declared and materialized marketplaces
- Cache directory: `~/.claude/plugins/`
- Settings key: `extraKnownMarketplaces`

**Plugin CLI Commands (`src/services/plugins/pluginCliCommands.ts`)**:
- Wraps `installPluginOp()`, `uninstallPluginOp()`, `enablePluginOp()`, `disablePluginOp()`, `disableAllPluginsOp()`, `updatePluginOp()`
- Error handler: `handlePluginCommandError()` -- log, format, telemetry, `process.exit(1)`
- Telemetry events: `tengu_plugin_installed_cli`, `tengu_plugin_uninstalled_cli`, `tengu_plugin_command_failed`
- PII-tagged column: `_PROTO_plugin_name`

**Plugin Validation (`src/utils/plugins/validatePlugin.ts`)**:
- `checkPathTraversal()` -- scans for `..` segments in paths
- `isBlockedOfficialName()` -- regex + non-ASCII homograph detection
- `validateOfficialNameSource()` -- reserved names restricted to `anthropics` GitHub org
- Reserved marketplace names: `claude-code-marketplace`, `anthropic-plugins` (and others)

**Plugin Policy (`src/utils/plugins/pluginPolicy.ts`)**:
- `isPluginBlockedByPolicy()` -- checks `policySettings.enabledPlugins[pluginId] === false`
- `isSourceAllowedByPolicy()` -- checks `strictKnownMarketplaces` (hostPattern/pathPattern regexes)
- `isSourceInBlocklist()` -- checks `blockedMarketplaces`

**Command Registry (`src/commands.ts`)**:
- `getCommands(cwd)` -- single entry point for complete command list
- `loadAllCommands()` -- memoized base command loading
- `clearCommandsCache()` -- clears all cache layers
- `clearCommandMemoizationCaches()` -- clears only command-list caches
- `COMMANDS()` -- returns built-in CLI commands
- `builtInCommandNames` -- protected set of built-in names
- `meetsAvailabilityRequirement()` -- auth/provider checks
- `isCommandEnabled()` -- feature gates, `isEnabled()` callbacks
- Command types: `'prompt'`, `'local'`, `'local-jsx'`

**SkillTool (`src/tools/SkillTool/SkillTool.ts`)**:
- `validateInput()` -- checks name, existence, invocation permission, type
- `checkPermissions()` -- rule-based: deny rules, allow rules, safe-property auto-allow, user prompt
- `SAFE_SKILL_PROPERTIES` -- allowlist of properties that auto-allow without user approval
- `call()` returns: `newMessages`, `contextModifier` (for inline), or `ToolResult` with `status: 'forked'`
- `processPromptSlashCommand()` -- expands skill markdown
- `prepareForkedCommandContext()` -- context for forked execution
- `runAgent()` -- sub-agent execution for forked skills
- `addInvokedSkill()` -- registers for compaction preservation
- `recordSkillUsage()` -- records for ranking
- `loadRemoteSkill()` -- loads from AKI/GCS (feature-gated: `EXPERIMENTAL_SKILL_SEARCH`)
- Model override: preserves `[1m]` suffix when replacing `mainLoopModel`
- Effort override: replaces `effortValue`

**SkillTool Prompt (`src/tools/SkillTool/prompt.ts`)**:
- `getSkillToolCommands()` -- memoized filter for model-invocable skills
- `estimateSkillFrontmatterTokens()` -- quick estimate from frontmatter fields
- Budget: 1% of context window, default 8,000 characters
- Per-entry description cap: 250 characters
- Truncation modes: full description, truncated description, name-only
- Injected as `system-reminder` attachment

**Installed Plugins File**:
- Path: `~/.claude/plugins/installed_plugins.json`
- Schema version: `2`
- Structure: `{ version: 2, plugins: { [pluginId]: InstallEntry[] } }`
- `InstallEntry`: `{ scope, installPath, version, projectPath? }`

### TypeScript Code Snippets

**`getCommands()` parallel loading:**
```typescript
const [
  { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
  pluginCommands,
  workflowCommands,
] = await Promise.all([
  getSkills(cwd),
  getPluginCommands(),
  getWorkflowCommands(cwd),
])
```

**`loadAllCommands()` concatenation order:**
```typescript
return [
  ...bundledSkills,
  ...builtinPluginSkills,
  ...skillDirCommands,
  ...workflowCommands,
  ...pluginCommands,
  ...pluginSkills,
  ...COMMANDS(),
]
```

**Dynamic skill insertion:**
```typescript
const insertIndex = baseCommands.findIndex(c => builtInNames.has(c.name))
return [
  ...baseCommands.slice(0, insertIndex),
  ...uniqueDynamicSkills,
  ...baseCommands.slice(insertIndex),
]
```

**Dynamic skill deduplication:**
```typescript
const baseCommandNames = new Set(baseCommands.map(c => c.name))
const uniqueDynamicSkills = dynamicSkills.filter(
  s => !baseCommandNames.has(s.name) && ...
)
```

**Conditional skill activation:**
```typescript
activateConditionalSkillsForPaths(['src/App.tsx'], '/project')
// activates skills whose paths include "src/**" or "*.tsx"
```

### Variable Substitution Tokens

- `${CLAUDE_PLUGIN_ROOT}` -- plugin root directory
- `${CLAUDE_PLUGIN_DATA}` -- plugin data directory
- `${CLAUDE_SKILL_DIR}` -- skill directory
- `${CLAUDE_SESSION_ID}` -- session identifier
- `${user_config.KEY}` -- user config value (sensitive values replaced with placeholder)

### Telemetry Events

| Event | Description |
|---|---|
| `tengu_skill_tool_invocation` | Model invoked a skill (name, source, execution context, depth) |
| `tengu_skill_tool_slash_prefix` | Model included leading `/` in skill name |
| `tengu_dynamic_skills_changed` | Dynamic skills added (source, count, directory count) |
| `tengu_skill_descriptions_truncated` | Skill listing exceeded budget (truncation mode, counts) |
| `tengu_marketplace_background_install` | Background marketplace reconciliation results |
| `tengu_plugin_installed_cli` | Plugin installed via CLI |
| `tengu_plugin_uninstalled_cli` | Plugin uninstalled via CLI |
| `tengu_plugin_command_failed` | Plugin CLI command failed (error category) |

### V2 Installed Plugins JSON Structure

```json
{
  "version": 2,
  "plugins": {
    "code-formatter@anthropic-tools": [
      { "scope": "user", "installPath": "...", "version": "1.0.0" },
      { "scope": "project", "projectPath": "/path/to/project", "version": "1.1.0" }
    ]
  }
}
```

### Compaction Preservation Constants

- Per-skill truncation limit: 5,000 characters
- Total skill token budget for post-compaction restoration: 25,000 tokens
- Function: `createSkillAttachmentIfNeeded()`
