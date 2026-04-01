# Claude Code: Plugin & Skill System — Design Specification

This document analyzes the extensibility architecture of Claude Code — Anthropic's
agentic CLI tool — focusing on how plugins and skills extend the system's capabilities,
how they are discovered and loaded, how they integrate with the command registry, and how
the model invokes them at runtime.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Skill Architecture](#2-skill-architecture)
- [3. Plugin Architecture](#3-plugin-architecture)
- [4. Skill Properties & Frontmatter](#4-skill-properties--frontmatter)
- [5. Dynamic Skill Detection](#5-dynamic-skill-detection)
- [6. Command Integration](#6-command-integration)
- [7. Source Priority & Deduplication](#7-source-priority--deduplication)
- [8. SkillTool — Model Invocation](#8-skilltool--model-invocation)
- [9. Plugin CLI Commands](#9-plugin-cli-commands)
- [10. Security](#10-security)
- [11. Design Principles](#11-design-principles)

---

## 1. Overview

Claude Code's extensibility system has two complementary halves:

1. **Skills** — markdown-based prompt fragments that teach the model specialized
   behaviors. A skill is a directory containing a `SKILL.md` file with YAML
   frontmatter. When the model (or the user) invokes a skill, its markdown content
   is injected into the conversation as a user message, effectively giving the model
   new instructions, workflows, and constraints mid-session.

2. **Plugins** — distributable packages that bundle skills, slash commands, agent
   definitions, hooks, MCP servers, LSP servers, and output styles into a single
   installable unit. Plugins are discovered through **marketplaces** — curated JSON
   manifests that list available plugins and their sources (git repos, npm packages,
   local paths).

The two layers compose cleanly: a plugin's `skills/` directory is loaded with the
same machinery that loads user skills from `~/.claude/skills/`, and a plugin's
`commands/` directory produces the same `Command` objects that the built-in command
registry uses. From the model's perspective, there is no distinction between a
bundled skill, a user skill, and a plugin skill — they are all prompt-type commands
invokable via the SkillTool.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Command Registry                             │
│  (getCommands → unified Command[] for slash commands & model tools)  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌────────────────────┐ │
│  │ Bundled   │  │ User     │  │ Plugin    │  │ MCP               │ │
│  │ Skills    │  │ Skills   │  │ Commands  │  │ Skills            │ │
│  │           │  │          │  │ & Skills  │  │                   │ │
│  │ Compiled  │  │ ~/.claude│  │ Market-   │  │ Remote MCP server │ │
│  │ into CLI  │  │ /skills/ │  │ place-    │  │ skill_listing     │ │
│  │ binary    │  │          │  │ sourced   │  │ resources         │ │
│  └──────────┘  └──────────┘  └───────────┘  └────────────────────┘ │
│       ▲              ▲             ▲               ▲               │
│       │              │             │               │               │
│  registerBundled  loadSkills   loadPlugin      mcpSkills.ts       │
│  Skill()          FromDir()   Commands()                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Skill Architecture

### 2.1 Skill Directory Layout

> **Source:** `src/skills/loadSkillsDir.ts`

Skills use a directory-based format. Each skill is a subdirectory containing a
`SKILL.md` file:

```
~/.claude/skills/
├── commit/
│   └── SKILL.md        ← markdown with YAML frontmatter
├── review-pr/
│   ├── SKILL.md
│   └── templates/      ← auxiliary files the skill can reference
│       └── checklist.md
└── deploy/
    └── SKILL.md
```

The skill name is the directory name. Single `.md` files directly in the `skills/`
directory are **not** supported (only the directory format). This constraint enables
skills to bundle auxiliary files that the model can `Read` or `Grep` during execution.

### 2.2 Skill Loading Hierarchy

Skills are loaded from three tiers of directories, paralleling the CLAUDE.md hierarchy:

| Tier | Path | Setting Source |
|---|---|---|
| **Managed** (enterprise) | `/etc/claude-code/.claude/skills/` | `policySettings` |
| **User** (personal) | `~/.claude/skills/` | `userSettings` |
| **Project** (per-repo) | `.claude/skills/` (walked upward to home) | `projectSettings` |

Additionally:
- `--add-dir` paths inject project-scoped skills from arbitrary directories
- Legacy `~/.claude/commands/` and `.claude/commands/` directories are still loaded
  (with `loadedFrom: 'commands_DEPRECATED'`)

All tiers are loaded in parallel via `Promise.all()`. The resulting skills are
flattened into a single array, deduplicated by resolved file path (symlinks resolved
via `realpath`), and cached by `memoize`.

### 2.3 Bundled Skills

> **Source:** `src/skills/bundledSkills.ts`, `src/skills/bundled/index.ts`

Bundled skills ship compiled into the CLI binary. They are registered imperatively
at startup via `registerBundledSkill()`, which pushes a `Command` object into an
internal array. Unlike file-based skills, bundled skills:

- Have `source: 'bundled'` and `loadedFrom: 'bundled'`
- May be feature-gated (e.g., `/dream` requires `KAIROS`, `/loop` requires `AGENT_TRIGGERS`)
- Can declare `files: Record<string, string>` — auxiliary reference files extracted
  lazily to `~/.claude/bundled-skills/<name>/` on first invocation
- Have `contentLength: 0` (not applicable; content is generated programmatically)

The bundled skill registry includes: `/verify`, `/debug`, `/lorem-ipsum`, `/skillify`,
`/remember`, `/simplify`, `/batch`, `/stuck`, `/update-config`, `/keybindings`, and
several feature-gated skills (`/dream`, `/hunter`, `/loop`, `/schedule-remote-agents`,
`/claude-api`, `/claude-in-chrome`).

**File extraction security**: Bundled skill files are written with `O_CREAT | O_EXCL |
O_NOFOLLOW` flags and `0o600` permissions into a nonce-bearing directory
(`getBundledSkillsRoot()`). Path traversal is blocked by `resolveSkillFilePath()`,
which rejects `..` segments and absolute paths.

### 2.4 MCP Skills

> **Source:** `src/skills/mcpSkillBuilders.ts`

MCP servers can expose skills via skill resources. These are loaded through a
write-once registry pattern: `loadSkillsDir.ts` registers its `createSkillCommand`
and `parseSkillFrontmatterFields` functions into a dependency-graph leaf module
(`mcpSkillBuilders.ts`), which MCP skill loading code can import without creating
circular dependencies.

MCP skills receive `loadedFrom: 'mcp'` and are subject to a critical security
constraint: **inline shell commands (`!`...`) in MCP skill markdown are never
executed**. The `getPromptForCommand()` handler skips `executeShellCommandsInPrompt()`
when `loadedFrom === 'mcp'`, because MCP skills are remote and untrusted.

### 2.5 The `--bare` Mode

When Claude Code runs with `--bare`, automatic skill discovery from managed, user,
and project directories is skipped entirely. Only explicit `--add-dir` paths and
bundled skills are loaded. This provides a minimal baseline for CI/scripting contexts
where ambient skill loading would be unpredictable.

---

## 3. Plugin Architecture

### 3.1 Plugin Directory Structure

> **Source:** `src/utils/plugins/pluginLoader.ts`, `src/utils/plugins/schemas.ts`

A plugin is a directory containing:

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json       ← Manifest (name, version, author, components)
├── commands/             ← Slash commands (markdown files)
│   ├── build.md
│   └── deploy.md
├── skills/               ← Skills (SKILL.md in subdirectories)
│   └── code-review/
│       └── SKILL.md
├── agents/               ← Agent definitions (markdown)
│   └── test-runner.md
├── hooks/                ← Hook configurations
│   └── hooks.json
├── output-styles/        ← Output formatting styles
│   └── concise.md
├── .mcp.json             ← MCP server configurations
└── .lsp.json             ← LSP server configurations
```

### 3.2 Plugin Manifest (`plugin.json`)

> **Source:** `src/utils/plugins/schemas.ts`, `PluginManifestSchema`

The manifest declares plugin metadata and component paths. It is validated at load
time by `PluginManifestSchema`, a Zod schema that enforces:

- **Required**: `name` (kebab-case, no spaces)
- **Optional metadata**: `version` (semver), `description`, `author` (name/email/url),
  `homepage`, `repository`, `license`, `keywords`, `dependencies`
- **Component declarations**: `commands`, `skills`, `agents`, `hooks`, `mcpServers`,
  `lspServers`, `outputStyles`, `channels`, `settings`, `userConfig`

Unknown top-level fields are **silently stripped** (not rejected) for forward
compatibility. Nested config objects (`userConfig`, `channels`, `lspServers`) use
`.strict()` — typos there are likely author mistakes, not vendor extensions.

Plugin commands are namespaced by the plugin name: a `deploy.md` in `my-plugin`
becomes the slash command `/my-plugin:deploy`.

### 3.3 Marketplace System

Plugins are distributed through marketplaces — JSON manifests listing available plugins.
A marketplace has its own source configuration:

| Source Type | Description |
|---|---|
| `github` | GitHub repo (owner/repo format) |
| `git` | Arbitrary git URL (HTTPS or SSH) |
| `url` | Direct URL to `marketplace.json` |
| `npm` | NPM package containing marketplace manifest |
| `file` | Local file path |
| `directory` | Local directory with `.claude-plugin/marketplace.json` |
| `settings` | Inline plugins declared directly in settings.json |

Marketplaces are declared in settings (`extraKnownMarketplaces`) and materialized
to a local cache at `~/.claude/plugins/`. The reconciliation process
(`src/utils/plugins/reconciler.ts`) computes a diff between declared and materialized
marketplaces and installs/updates as needed.

### 3.4 Plugin Installation Manager

> **Source:** `src/services/plugins/PluginInstallationManager.ts`

Background plugin installation runs asynchronously after startup. The
`performBackgroundPluginInstallations()` function:

1. Computes a diff between declared and materialized marketplaces
2. Initializes AppState with pending status for UI spinners
3. Calls `reconcileMarketplaces()` with progress callbacks that update AppState
4. On new installs: auto-refreshes plugins via `refreshActivePlugins()`
5. On updates only: sets `needsRefresh` flag, notifying the user to run `/reload-plugins`

This is a **thin wrapper** that maps reconciliation progress events to React state
updates. The heavy lifting (git clone, version calculation, cache management) happens
in the reconciler and plugin loader.

### 3.5 Plugin Loader

> **Source:** `src/utils/plugins/pluginLoader.ts`

The plugin loader discovers, loads, and validates plugins from multiple sources:

1. **Marketplace plugins**: Resolved from `plugin@marketplace` entries in settings.
   The loader searches materialized marketplaces for matching entries, then resolves
   the plugin to a cached directory.

2. **Session-only plugins**: From `--plugin-dir` CLI flag or SDK `plugins` option.
   These bypass the marketplace system entirely.

3. **Built-in plugins**: Registered via `getBuiltinPlugins()`, using the reserved
   marketplace name `builtin`.

The loader handles manifest validation, hooks configuration loading (with variable
resolution), duplicate name detection, enable/disable state management, and error
collection. Results are cached via `memoize` and exposed through
`loadAllPlugins()` (full load with marketplace access) and
`loadAllPluginsCacheOnly()` (cache-only, for non-blocking startup).

### 3.6 Plugin Operations

> **Source:** `src/services/plugins/pluginOperations.ts`

Core plugin lifecycle operations are implemented as pure library functions that
return result objects (no `console.log`, no `process.exit`):

- **`installPluginOp()`**: Settings-first install. Searches marketplaces, writes
  settings, caches plugin, records version hint. Checks for policy blocks and
  dependency policy blocks.

- **`uninstallPluginOp()`**: Removes from settings, clears caches, removes V2
  installation record. Marks orphaned versions for cleanup. Warns (but does not
  block) on reverse dependencies.

- **`setPluginEnabledOp()`**: Resolves plugin ID and scope from settings, checks
  policy, checks idempotency, writes settings. Supports cross-scope overrides
  (e.g., `disable --scope local` to override a project-enabled plugin).

- **`updatePluginOp()`**: Non-inplace update. Downloads new version to temp dir,
  calculates version hash, copies to versioned cache, updates V2 installation
  record. Old versions are marked orphaned if no longer referenced.

**Scope precedence** for finding plugins in settings: `local > project > user`
(most specific wins).

### 3.7 Installed Plugins File (V2)

Plugin installations are tracked in `~/.claude/plugins/installed_plugins.json`.
The V2 schema maps each `plugin@marketplace` ID to an **array** of installation
entries (one per scope), enabling multi-scope installation of the same plugin at
different versions:

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

---

## 4. Skill Properties & Frontmatter

### 4.1 Frontmatter Fields

> **Source:** `src/skills/loadSkillsDir.ts`, `parseSkillFrontmatterFields()`

Skills declare their properties in YAML frontmatter:

| Field | Type | Default | Purpose |
|---|---|---|---|
| `name` | string | directory name | Display name (model sees the directory name) |
| `description` | string | first markdown line | What the skill does |
| `when_to_use` | string | — | Guidance for when the model should invoke this skill |
| `allowed-tools` | string[] | `[]` | Tools auto-allowed when skill is active |
| `argument-hint` | string | — | Hint for arguments (e.g., `"[file]"`) |
| `arguments` | string[] | — | Named argument placeholders for `$ARGUMENTS` substitution |
| `user-invocable` | boolean | `true` | Whether users can type `/skill-name` |
| `disable-model-invocation` | boolean | `false` | If true, model cannot invoke via SkillTool |
| `model` | string | (inherit) | Model override (e.g., `haiku`, `sonnet`, `opus`) |
| `effort` | int/string | — | Reasoning effort level override |
| `context` | `fork` | `inline` | Execution mode: inline (same context) or forked sub-agent |
| `agent` | string | — | Agent type for forked execution |
| `paths` | string[] | — | Glob patterns for conditional activation |
| `hooks` | object | — | Per-skill hook definitions |
| `version` | string | — | Skill version |
| `shell` | `bash`/`powershell` | — | Shell for `!command` blocks |

### 4.2 `paths` — Conditional Activation Globs

The `paths` frontmatter field declares glob patterns (gitignore-style) that control
when a skill becomes visible to the model. Skills with `paths` are **not** loaded
into the active skill set at startup. Instead, they are stored in a `conditionalSkills`
map and activated only when the user or model touches a file matching one of the
patterns.

```yaml
---
paths: ["src/components/**", "*.tsx"]
description: React component guidelines
---
```

This keeps the model's context budget lean — React guidelines don't consume tokens
in a Python project, even if the skill is defined at the user level.

### 4.3 `contentLength` — Token Budget Estimation

Every skill carries a `contentLength` field (the character count of its markdown body).
This is used by the SkillTool prompt generator to estimate token cost and truncate
skill listing descriptions when the total exceeds 1% of the context window. The
`estimateSkillFrontmatterTokens()` function provides a quick estimate based on
frontmatter fields alone (name, description, whenToUse).

### 4.4 `allowedTools` — Permission Escalation

When a skill declares `allowed-tools`, those tool patterns are injected into the
`alwaysAllowRules.command` array for the duration of the skill's execution. This
means the model can use those tools without user approval prompts. The injection
happens in two places:

1. **Inline execution**: via `contextModifier` in SkillTool's `call()` return value
2. **Shell commands in skill content**: via a modified `getAppState()` in
   `getPromptForCommand()`

### 4.5 `context: fork` — Forked Execution

Skills with `context: fork` run in an isolated sub-agent via `runAgent()`. The
forked agent has its own token budget and message history, receives the skill prompt
as its initial context, and returns a text result that is injected back into the
parent conversation. This is used for skills that need extensive tool use (e.g.,
multi-file review workflows) without polluting the parent context.

---

## 5. Dynamic Skill Detection

> **Source:** `src/skills/loadSkillsDir.ts`, `discoverSkillDirsForPaths()`,
> `activateConditionalSkillsForPaths()`

### 5.1 Directory Discovery

When the model reads or writes a file, the system walks up from the file's parent
directory to `cwd`, checking for `.claude/skills/` directories at each level. The
cwd-level itself is excluded (those skills are already loaded at startup).

```
/project/src/components/Button.tsx  ← file touched
         ↑
         /project/src/components/.claude/skills/  ← check
         /project/src/.claude/skills/             ← check
         /project/.claude/skills/                 ← already loaded (cwd)
```

Discovered directories are tracked in a session-scoped `dynamicSkillDirs` set to
avoid redundant `stat` calls. Directories inside gitignored paths (e.g.,
`node_modules/`) are rejected.

### 5.2 Conditional Skill Activation

Separately from directory discovery, skills with `paths` frontmatter are matched
against touched file paths using the `ignore` library (gitignore-style matching).
When a match is found, the skill moves from `conditionalSkills` to `dynamicSkills`,
making it visible to the model.

```typescript
activateConditionalSkillsForPaths(['src/App.tsx'], '/project')
// → activates skills whose paths include "src/**" or "*.tsx"
```

### 5.3 Signal Propagation

When dynamic skills are added (either from directory discovery or conditional
activation), a `skillsLoaded` signal fires. Subscribers (registered via
`onDynamicSkillsLoaded()`) clear cached command lists so the next
`getCommands()` call returns the updated set.

---

## 6. Command Integration

> **Source:** `src/commands.ts`

### 6.1 The `getCommands()` Function

All command sources converge in `getCommands(cwd)`, the single entry point for the
complete command list. It loads from five sources in parallel:

```typescript
const [
  { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
  pluginCommands,
  workflowCommands,
] = await Promise.all([
  getSkills(cwd),           // skills/ dirs + bundled + built-in plugins
  getPluginCommands(),       // plugin commands/ dirs
  getWorkflowCommands(cwd), // workflow scripts (feature-gated)
])
```

The result is flattened in a specific order (see section 7), then filtered by
`meetsAvailabilityRequirement()` (auth/provider checks) and `isCommandEnabled()`
(feature gates, `isEnabled()` callbacks).

Dynamic skills are appended at query time — they are not part of the memoized
`loadAllCommands()` cache but are merged in every `getCommands()` call.

### 6.2 Command Types

Commands have a discriminated union type:

| Type | Purpose |
|---|---|
| `prompt` | Skills and prompt-based commands — expand to text for the model |
| `local` | Local operations returning text (e.g., `/compact`, `/cost`) |
| `local-jsx` | Local operations rendering React/Ink UI (e.g., `/config`, `/mcp`) |

Only `prompt` commands can be invoked by the model via SkillTool.

### 6.3 Plugin Command Namespacing

Plugin commands are automatically namespaced with the plugin name:

```
my-plugin/
├── commands/
│   ├── build.md      → /my-plugin:build
│   └── ops/
│       └── deploy.md → /my-plugin:ops:deploy
└── skills/
    └── review/
        └── SKILL.md  → /my-plugin:review
```

The colon-separated namespace prevents collisions between plugins and between
plugins and built-in commands.

---

## 7. Source Priority & Deduplication

### 7.1 Load Order

The `loadAllCommands()` function concatenates sources in this order:

```typescript
return [
  ...bundledSkills,           // 1. Bundled (lowest priority for name collisions)
  ...builtinPluginSkills,     // 2. Built-in plugin skills
  ...skillDirCommands,        // 3. User/project skills
  ...workflowCommands,        // 4. Workflow scripts
  ...pluginCommands,          // 5. Plugin commands
  ...pluginSkills,            // 6. Plugin skills
  ...COMMANDS(),              // 7. Built-in CLI commands (highest priority)
]
```

Dynamic skills are inserted **before** built-in commands but after plugin skills:

```typescript
const insertIndex = baseCommands.findIndex(c => builtInNames.has(c.name))
return [
  ...baseCommands.slice(0, insertIndex),
  ...uniqueDynamicSkills,
  ...baseCommands.slice(insertIndex),
]
```

### 7.2 Skill File Deduplication

Skills loaded from overlapping parent directory walks are deduplicated by **resolved
file identity** (via `realpath`). This handles:

- Symlinks pointing to the same file from different skill directories
- Overlapping `getProjectDirsUpToHome()` results when `--add-dir` paths overlap
  with project directory walks

The first-loaded copy wins. Duplicates are logged and skipped.

### 7.3 Dynamic Skill Deduplication

Dynamic skills (discovered during file operations) are deduplicated against the base
command set by name:

```typescript
const baseCommandNames = new Set(baseCommands.map(c => c.name))
const uniqueDynamicSkills = dynamicSkills.filter(
  s => !baseCommandNames.has(s.name) && ...
)
```

Built-in command names are protected — no skill or plugin can shadow `/help`,
`/clear`, `/compact`, etc.

### 7.4 Plugin Name Collision Detection

Within the plugin loader, duplicate plugin names across different marketplaces are
detected and reported as errors. The `verifyAndDemote()` function checks dependency
graphs and demotes plugins with missing dependencies.

---

## 8. SkillTool — Model Invocation

> **Source:** `src/tools/SkillTool/SkillTool.ts`, `src/tools/SkillTool/prompt.ts`

### 8.1 Overview

The SkillTool is the bridge between the model's tool-calling capability and the skill
system. When the model decides to invoke a skill, it calls the Skill tool with
`{ skill: "name", args: "optional args" }`.

### 8.2 The Skill Listing Prompt

The model discovers available skills through a listing injected into the system prompt
as a `system-reminder` attachment. The listing is budget-constrained:

- **Budget**: 1% of the context window (in characters), defaulting to 8,000 characters
- **Per-entry cap**: 250 characters per description (truncated with ellipsis)
- **Bundled skills** are never truncated — only user/plugin skill descriptions shrink
- **Extreme case**: When even truncated descriptions exceed the budget, non-bundled
  skills fall back to name-only entries (`- skill-name`)

The listing includes all commands that pass the `getSkillToolCommands()` filter:
prompt-type, non-`disableModelInvocation`, non-built-in, and either bundled, from
`/skills/`, from legacy `/commands/`, or having an explicit description/`whenToUse`.

### 8.3 Validation & Permission Checking

**`validateInput()`** confirms:
1. The skill name is non-empty
2. The skill exists in `getAllCommands()` (local + MCP)
3. `disableModelInvocation` is not set
4. The command is `type: 'prompt'`

**`checkPermissions()`** implements a rule-based permission system:
1. Check deny rules (exact match or prefix with `:*` wildcard)
2. Check allow rules (same matching)
3. Auto-allow skills with only "safe properties" (no `allowedTools`, no `hooks`,
   no `shell` — see `SAFE_SKILL_PROPERTIES` allowlist)
4. Default: ask user for permission, offering exact-match and prefix suggestions

### 8.4 Execution Modes

**Inline execution** (default):
1. Call `processPromptSlashCommand()` to expand the skill's markdown
2. Extract `allowedTools` and `model` overrides
3. Return `newMessages` (the skill content as user messages) and a `contextModifier`
   that injects allowed-tool rules and model overrides

**Forked execution** (`context: fork`):
1. Prepare forked context via `prepareForkedCommandContext()`
2. Run `runAgent()` with the skill prompt as initial messages
3. Stream progress (tool uses) back to the parent
4. Extract result text from agent messages
5. Return as a `ToolResult` with `status: 'forked'`

**Remote execution** (experimental, `EXPERIMENTAL_SKILL_SEARCH` feature gate):
1. Load `SKILL.md` from AKI/GCS via `loadRemoteSkill()`
2. Strip frontmatter, inject base directory header
3. Register with compaction-preservation state
4. Inject as a user message (no shell command expansion — remote skills are
   declarative only)

### 8.5 Post-Invocation Effects

After a skill executes inline:
- The skill's `allowedTools` are merged into `alwaysAllowRules.command`
- The skill's `model` override replaces `mainLoopModel` (preserving `[1m]` suffix)
- The skill's `effort` level overrides `effortValue`
- The skill content is registered with `addInvokedSkill()` for compaction preservation
- Skill usage is recorded for ranking via `recordSkillUsage()`

---

## 9. Plugin CLI Commands

> **Source:** `src/services/plugins/pluginCliCommands.ts`

The plugin CLI commands are thin wrappers around the core operations
(`pluginOperations.ts`) that add console output and `process.exit()`:

| Command | Operation | Description |
|---|---|---|
| `claude plugin install <name> [--scope]` | `installPluginOp()` | Install from marketplace |
| `claude plugin uninstall <name> [--scope]` | `uninstallPluginOp()` | Remove plugin |
| `claude plugin enable <name> [--scope]` | `enablePluginOp()` | Enable disabled plugin |
| `claude plugin disable <name> [--scope]` | `disablePluginOp()` | Disable plugin |
| `claude plugin disable-all` | `disableAllPluginsOp()` | Disable all enabled plugins |
| `claude plugin update <name> --scope` | `updatePluginOp()` | Update to latest version |

**Valid installable scopes**: `user`, `project`, `local` (not `managed`).
**Valid update scopes**: `user`, `project`, `local`, `managed`.

Every command emits structured telemetry via `logEvent()`:
- Success events: `tengu_plugin_installed_cli`, `tengu_plugin_uninstalled_cli`, etc.
- Failure events: `tengu_plugin_command_failed` with `error_category` classification
- Plugin names are routed to PII-tagged BQ columns (`_PROTO_plugin_name`)

The `handlePluginCommandError()` function provides uniform error handling: log the
error, format a user-visible message, emit telemetry with error classification, and
`process.exit(1)`.

---

## 10. Security

### 10.1 Manifest Validation

> **Source:** `src/utils/plugins/validatePlugin.ts`, `src/utils/plugins/schemas.ts`

Plugin and marketplace manifests are validated by Zod schemas with security-focused
refinements:

- **Path traversal**: `checkPathTraversal()` scans `commands`, `agents`, `skills`,
  and `source` paths for `..` segments before schema validation
- **Official name protection**: `isBlockedOfficialName()` blocks marketplace names
  impersonating Anthropic/Claude official marketplaces using regex detection and
  non-ASCII (homograph) character blocking
- **Source org verification**: `validateOfficialNameSource()` ensures reserved
  marketplace names (`claude-code-marketplace`, `anthropic-plugins`, etc.) can only
  be used by repos from the `anthropics` GitHub organization
- **Plugin ID format**: `PluginIdSchema` enforces `plugin@marketplace` format with
  alphanumeric/hyphen/dot/underscore characters

### 10.2 Plugin Policy

> **Source:** `src/utils/plugins/pluginPolicy.ts`

Enterprise administrators can force-disable plugins via managed settings
(`policySettings`). `isPluginBlockedByPolicy()` checks if
`policySettings.enabledPlugins[pluginId] === false`. Policy-blocked plugins:

- Cannot be installed (`installPluginOp` checks)
- Cannot be enabled (`setPluginEnabledOp` checks)
- Cannot have dependencies that are policy-blocked (`dependency-blocked-by-policy`)

### 10.3 Marketplace Source Restrictions

The `strictKnownMarketplaces` setting in managed settings restricts which marketplace
sources are allowed. Supports `hostPattern` (regex against domain) and `pathPattern`
(regex against filesystem paths). The `blockedMarketplaces` setting explicitly denies
specific sources. `isSourceAllowedByPolicy()` and `isSourceInBlocklist()` enforce
these restrictions.

### 10.4 Bundled Skill File Security

Bundled skill reference files are extracted to a nonce-bearing temporary directory
(`getBundledSkillsRoot()`) with defense-in-depth:

- Per-process nonce prevents symlink pre-creation attacks
- `O_CREAT | O_EXCL | O_NOFOLLOW` flags prevent overwriting existing files
- `0o700` (directories) and `0o600` (files) permissions restrict access to the owner
- `resolveSkillFilePath()` rejects `..` segments and absolute paths in relative file
  entries

### 10.5 MCP Skill Shell Restriction

MCP-sourced skills (`loadedFrom: 'mcp'`) are treated as untrusted remote content.
The `getPromptForCommand()` handler in `createSkillCommand()` explicitly skips
`executeShellCommandsInPrompt()` for MCP skills, preventing `!command` and
````!...```` blocks from executing arbitrary shell commands.

### 10.6 SkillTool Permission Model

The SkillTool `checkPermissions()` method uses a property-based allowlist
(`SAFE_SKILL_PROPERTIES`) to determine which skills can auto-execute without user
approval. If a skill has any property NOT in the safe set with a meaningful value
(non-null, non-empty), it requires explicit permission. This ensures newly added
properties default to requiring permission until reviewed.

### 10.7 Gitignore Filtering for Dynamic Skills

Dynamically discovered skill directories are checked against `.gitignore` via
`isPathGitignored()`. This prevents skills from gitignored paths (e.g.,
`node_modules/some-package/.claude/skills/`) from loading silently. The invocation-time
trust dialog is the actual security boundary; gitignore filtering is an early guard.

### 10.8 Plugin Variable Substitution

Plugin commands support `${CLAUDE_PLUGIN_ROOT}`, `${CLAUDE_PLUGIN_DATA}`,
`${CLAUDE_SKILL_DIR}`, `${CLAUDE_SESSION_ID}`, and `${user_config.KEY}` substitution.
Sensitive `userConfig` values resolve to a descriptive placeholder in skill content
(which goes to the model prompt) — secrets are never injected into the conversation.

---

## 11. Design Principles

### 11.1 Markdown as the Universal Extension Format

Skills are markdown files with YAML frontmatter. This is a deliberate choice:
markdown is human-readable, version-controllable, and trivially composable. A skill
author needs no build step, no SDK, no runtime — just a text file. The model reads
the same markdown that the author wrote, closing the gap between authoring and
execution.

### 11.2 Settings-First Architecture

Plugin install, enable, and disable operations write to settings **first**, then
materialize the change (cache, version hint, etc.). This means the declaration of
intent (settings.json) is the source of truth. If materialization fails, the next
startup will retry. If a marketplace is not yet cloned, the background reconciler
will handle it. The system converges toward the declared state.

### 11.3 Namespace Isolation

Plugin commands are prefixed with the plugin name (`/plugin-name:command`). This
makes it impossible for two plugins to collide with each other or with built-in
commands. The built-in command set (`builtInCommandNames`) is explicitly protected
from shadowing.

### 11.4 Graduated Trust

The system applies different trust levels to different skill sources:

| Source | Trust Level | Shell Execution | Auto-Allow |
|---|---|---|---|
| Bundled | Full | Yes | Yes (safe properties) |
| User (`~/.claude/skills/`) | High | Yes | Per permission rules |
| Project (`.claude/skills/`) | Medium | Yes | Per permission rules |
| Plugin (marketplace) | Medium | Yes | Per permission rules + policy |
| MCP (remote server) | Low | **No** | Per permission rules |

### 11.5 Lazy Loading and Caching

Skills and plugins are loaded lazily and cached aggressively:

- `getSkillDirCommands()`: memoized by `cwd`
- `getPluginCommands()`: memoized (no arguments — global state)
- `loadAllCommands()`: memoized by `cwd`
- `getSkillToolCommands()`: memoized by `cwd`
- Dynamic skills: session-scoped `Map` (not reloaded on `getCommands()` calls)
- Bundled skill file extraction: closure-local memoized promise (extract once per process)

Cache invalidation is explicit: `clearCommandsCache()` clears all layers, and
`clearCommandMemoizationCaches()` clears only the command-list caches without
reloading skills from disk.

### 11.6 Graceful Degradation

Every skill and plugin loading path catches errors and continues:

- Skills that fail to parse produce `null` and are filtered out
- Plugin loading errors are collected into an `errors[]` array, not thrown
- `getSkills()` wraps both `getSkillDirCommands()` and `getPluginSkills()` in
  `.catch()` handlers that log and return empty arrays
- The SkillTool prompt is built even when skill loading partially fails

The system never crashes because an extension is broken. It logs, skips, and moves on.

### 11.7 Compaction Awareness

Skills are registered with `addInvokedSkill()` so their content survives context
compaction. After auto-compact, the post-compact restoration phase re-injects active
skills (truncated to 5,000 characters each, up to 25,000 tokens total) via
`createSkillAttachmentIfNeeded()`. This ensures that a compacted session retains
awareness of active skill instructions.

### 11.8 Observability

Every significant extensibility event is instrumented:

| Event | Meaning |
|---|---|
| `tengu_skill_tool_invocation` | Model invoked a skill (name, source, execution context, depth) |
| `tengu_skill_tool_slash_prefix` | Model included leading `/` in skill name |
| `tengu_dynamic_skills_changed` | Dynamic skills added (source, count, directory count) |
| `tengu_skill_descriptions_truncated` | Skill listing exceeded budget (truncation mode, counts) |
| `tengu_marketplace_background_install` | Background marketplace reconciliation results |
| `tengu_plugin_installed_cli` | Plugin installed via CLI |
| `tengu_plugin_command_failed` | Plugin CLI command failed (error category) |
