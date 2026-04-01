# Plugin & Skill System — Design Document

## Part A: Design Document

### 1. Overview

Claude Code's extensibility system consists of two complementary halves:

**Skills** are markdown-based prompt fragments that teach the model specialized behaviors. A skill is a directory containing a `SKILL.md` file with YAML frontmatter. When invoked, its markdown content is injected into the conversation as a user message, giving the model new instructions, workflows, and constraints mid-session.

**Plugins** are distributable packages that bundle skills, slash commands, agent definitions, hooks, MCP servers, LSP servers, and output styles into a single installable unit. Plugins are discovered through **marketplaces** -- curated JSON manifests that list available plugins and their sources.

The two layers compose cleanly: a plugin's `skills/` directory is loaded with the same machinery that loads user skills, and a plugin's `commands/` directory produces the same `Command` objects that the built-in command registry uses. From the model's perspective, there is no distinction between a bundled skill, a user skill, and a plugin skill -- they are all prompt-type commands invokable via the SkillTool.

**Architecture Diagram:**

```
+---------------------------------------------------------------------+
|                        Command Registry                              |
|  (getCommands -> unified Command[] for slash commands & model tools)  |
+---------------------------------------------------------------------+
|                                                                      |
|  +----------+  +----------+  +-----------+  +--------------------+   |
|  | Bundled  |  | User     |  | Plugin    |  | MCP               |   |
|  | Skills   |  | Skills   |  | Commands  |  | Skills            |   |
|  |          |  |          |  | & Skills  |  |                   |   |
|  | Compiled |  | ~/.claude|  | Market-   |  | Remote MCP server |   |
|  | into CLI |  | /skills/ |  | place-    |  | skill_listing     |   |
|  | binary   |  |          |  | sourced   |  | resources         |   |
|  +----------+  +----------+  +-----------+  +--------------------+   |
+---------------------------------------------------------------------+
```

### 2. Skill Architecture

#### 2.1 Skill Directory Layout

Skills use a directory-based format. Each skill is a subdirectory containing a `SKILL.md` file. The skill name is the directory name. Single `.md` files directly in the `skills/` directory are not supported (only the directory format). This constraint enables skills to bundle auxiliary files that the model can read during execution.

```
~/.claude/skills/
+-- commit/
|   +-- SKILL.md
+-- review-pr/
|   +-- SKILL.md
|   +-- templates/
|       +-- checklist.md
+-- deploy/
    +-- SKILL.md
```

#### 2.2 Skill Loading Hierarchy

Skills are loaded from three tiers of directories, paralleling the CLAUDE.md hierarchy:

| Tier | Path | Setting Source |
|---|---|---|
| **Managed** (enterprise) | `/etc/claude-code/.claude/skills/` | `policySettings` |
| **User** (personal) | `~/.claude/skills/` | `userSettings` |
| **Project** (per-repo) | `.claude/skills/` (walked upward to home) | `projectSettings` |

Additionally:
- `--add-dir` paths inject project-scoped skills from arbitrary directories.
- Legacy `commands/` directories are still loaded with a deprecated marker.

All tiers are loaded in parallel. The resulting skills are flattened into a single array, deduplicated by resolved file path (symlinks resolved), and cached.

#### 2.3 Bundled Skills

Bundled skills ship compiled into the CLI binary. They are registered imperatively at startup. Unlike file-based skills:

- They may be feature-gated (requiring specific feature flags).
- They can declare auxiliary reference files extracted lazily to a local directory on first invocation.
- They have zero content length (content is generated programmatically).

**File extraction security**: Bundled skill files are written with exclusive-create and no-follow flags with owner-only permissions into a nonce-bearing directory. Path traversal is blocked by rejecting `..` segments and absolute paths.

#### 2.4 MCP Skills

MCP servers can expose skills via skill resources. These are loaded through a write-once registry pattern that avoids circular dependencies.

MCP skills are subject to a critical security constraint: **inline shell commands in MCP skill markdown are never executed**, because MCP skills are remote and untrusted.

#### 2.5 The `--bare` Mode

When running with `--bare`, automatic skill discovery from managed, user, and project directories is skipped entirely. Only explicit `--add-dir` paths and bundled skills are loaded. This provides a minimal baseline for CI/scripting contexts.

### 3. Plugin Architecture

#### 3.1 Plugin Directory Structure

A plugin is a directory containing:

- `.claude-plugin/plugin.json` -- Manifest (name, version, author, components)
- `commands/` -- Slash commands (markdown files)
- `skills/` -- Skills (SKILL.md in subdirectories)
- `agents/` -- Agent definitions (markdown)
- `hooks/` -- Hook configurations
- `output-styles/` -- Output formatting styles
- `.mcp.json` -- MCP server configurations
- `.lsp.json` -- LSP server configurations

#### 3.2 Plugin Manifest

The manifest declares plugin metadata and component paths. Validation enforces:

- **Required**: `name` (kebab-case, no spaces)
- **Optional metadata**: `version` (semver), `description`, `author` (name/email/url), `homepage`, `repository`, `license`, `keywords`, `dependencies`
- **Component declarations**: `commands`, `skills`, `agents`, `hooks`, `mcpServers`, `lspServers`, `outputStyles`, `channels`, `settings`, `userConfig`

Unknown top-level fields are silently stripped (not rejected) for forward compatibility. Nested config objects use strict validation -- typos there are treated as likely author mistakes, not vendor extensions.

Plugin commands are namespaced by the plugin name: a `deploy.md` in `my-plugin` becomes `/my-plugin:deploy`.

#### 3.3 Marketplace System

Plugins are distributed through marketplaces -- JSON manifests listing available plugins. Marketplace source types:

| Source Type | Description |
|---|---|
| `github` | GitHub repo (owner/repo format) |
| `git` | Arbitrary git URL (HTTPS or SSH) |
| `url` | Direct URL to `marketplace.json` |
| `npm` | NPM package containing marketplace manifest |
| `file` | Local file path |
| `directory` | Local directory with marketplace JSON |
| `settings` | Inline plugins declared directly in settings |

Marketplaces are declared in settings and materialized to a local cache. A reconciliation process computes a diff between declared and materialized marketplaces and installs/updates as needed.

#### 3.4 Plugin Installation Manager

Background plugin installation runs asynchronously after startup:

1. Computes a diff between declared and materialized marketplaces
2. Initializes UI state with pending status for spinners
3. Calls the reconciler with progress callbacks that update UI state
4. On new installs: auto-refreshes plugins
5. On updates only: sets `needsRefresh` flag, notifying the user to run `/reload-plugins`

This is a thin wrapper that maps reconciliation progress events to UI state updates. The heavy lifting (git clone, version calculation, cache management) happens in the reconciler and plugin loader.

#### 3.5 Plugin Loader

The plugin loader discovers, loads, and validates plugins from multiple sources:

1. **Marketplace plugins**: Resolved from `plugin@marketplace` entries in settings.
2. **Session-only plugins**: From CLI flags or SDK options. These bypass the marketplace system.
3. **Built-in plugins**: Registered using a reserved marketplace name `builtin`.

The loader handles manifest validation, hooks configuration loading, duplicate name detection, enable/disable state management, and error collection. Results are cached and exposed through both full-load and cache-only interfaces.

#### 3.6 Plugin Operations

Core plugin lifecycle operations are implemented as pure library functions that return result objects (no side effects to console or process):

- **Install**: Settings-first install. Searches marketplaces, writes settings, caches plugin, records version hint. Checks for policy blocks and dependency policy blocks.
- **Uninstall**: Removes from settings, clears caches, removes installation record. Marks orphaned versions for cleanup. Warns (but does not block) on reverse dependencies.
- **Enable/Disable**: Resolves plugin ID and scope from settings, checks policy, checks idempotency, writes settings. Supports cross-scope overrides.
- **Update**: Non-inplace update. Downloads new version to temp dir, calculates version hash, copies to versioned cache, updates installation record. Old versions are marked orphaned if no longer referenced.

**Scope precedence** for finding plugins in settings: `local > project > user` (most specific wins).

#### 3.7 Installed Plugins File (V2)

Plugin installations are tracked in a JSON file. The V2 schema maps each `plugin@marketplace` ID to an array of installation entries (one per scope), enabling multi-scope installation of the same plugin at different versions.

### 4. Skill Properties and Frontmatter

#### 4.1 Frontmatter Fields

Skills declare their properties in YAML frontmatter:

| Field | Type | Default | Purpose |
|---|---|---|---|
| `name` | string | directory name | Display name |
| `description` | string | first markdown line | What the skill does |
| `when_to_use` | string | -- | Guidance for when the model should invoke |
| `allowed-tools` | string[] | `[]` | Tools auto-allowed when skill is active |
| `argument-hint` | string | -- | Hint for arguments |
| `arguments` | string[] | -- | Named argument placeholders for substitution |
| `user-invocable` | boolean | `true` | Whether users can type the slash command |
| `disable-model-invocation` | boolean | `false` | If true, model cannot invoke via SkillTool |
| `model` | string | (inherit) | Model override |
| `effort` | int/string | -- | Reasoning effort level override |
| `context` | `fork` or `inline` | `inline` | Execution mode |
| `agent` | string | -- | Agent type for forked execution |
| `paths` | string[] | -- | Glob patterns for conditional activation |
| `hooks` | object | -- | Per-skill hook definitions |
| `version` | string | -- | Skill version |
| `shell` | `bash`/`powershell` | -- | Shell for command blocks |

#### 4.2 Conditional Activation via `paths`

The `paths` frontmatter field declares glob patterns (gitignore-style) that control when a skill becomes visible to the model. Skills with `paths` are not loaded into the active skill set at startup. Instead, they are stored in a conditional skills map and activated only when the user or model touches a file matching one of the patterns.

This keeps the model's context budget lean -- e.g., React guidelines don't consume tokens in a Python project, even if the skill is defined at the user level.

#### 4.3 Token Budget Estimation

Every skill carries a `contentLength` field (character count of its markdown body). This is used by the SkillTool prompt generator to estimate token cost and truncate skill listing descriptions when the total exceeds 1% of the context window.

#### 4.4 Permission Escalation via `allowedTools`

When a skill declares `allowed-tools`, those tool patterns are injected into the always-allow rules for the duration of the skill's execution. The model can use those tools without user approval prompts. The injection happens during both inline execution and shell command expansion in skill content.

#### 4.5 Forked Execution

Skills with `context: fork` run in an isolated sub-agent with its own token budget and message history, receiving the skill prompt as initial context and returning a text result injected back into the parent conversation. This is used for skills needing extensive tool use without polluting the parent context.

### 5. Dynamic Skill Detection

#### 5.1 Directory Discovery

When the model reads or writes a file, the system walks up from the file's parent directory to cwd, checking for skill directories at each level. The cwd-level itself is excluded (already loaded at startup).

```
/project/src/components/Button.tsx  <-- file touched
         |
         /project/src/components/.claude/skills/  <-- check
         /project/src/.claude/skills/             <-- check
         /project/.claude/skills/                 <-- already loaded (cwd)
```

Discovered directories are tracked in a session-scoped set to avoid redundant filesystem calls. Directories inside gitignored paths are rejected.

#### 5.2 Conditional Skill Activation

Skills with `paths` frontmatter are matched against touched file paths using gitignore-style matching. When a match is found, the skill moves from the conditional set to the dynamic set, making it visible to the model.

#### 5.3 Signal Propagation

When dynamic skills are added, a signal fires. Subscribers clear cached command lists so the next query returns the updated set.

### 6. Command Integration

#### 6.1 Unified Command Entry Point

All command sources converge in a single entry point for the complete command list. It loads from five sources in parallel: skills directories + bundled + built-in plugins, plugin commands, and workflow commands.

The result is flattened in a specific order, then filtered by availability requirements (auth/provider checks) and enabled state (feature gates, callback checks).

Dynamic skills are appended at query time -- not part of the cached base set but merged on every call.

#### 6.2 Command Types

Commands have a discriminated union type:

| Type | Purpose |
|---|---|
| `prompt` | Skills and prompt-based commands -- expand to text for the model |
| `local` | Local operations returning text |
| `local-jsx` | Local operations rendering React/Ink UI |

Only `prompt` commands can be invoked by the model via SkillTool.

#### 6.3 Plugin Command Namespacing

Plugin commands are automatically namespaced with the plugin name using colon separation. Nested directories produce multi-level namespaces (e.g., `/my-plugin:ops:deploy`). This prevents collisions between plugins and between plugins and built-in commands.

### 7. Source Priority and Deduplication

#### 7.1 Load Order

Commands are concatenated in this order:

1. Bundled skills (lowest priority for name collisions)
2. Built-in plugin skills
3. User/project skills
4. Workflow scripts
5. Plugin commands
6. Plugin skills
7. Built-in CLI commands (highest priority)

Dynamic skills are inserted before built-in commands but after plugin skills.

#### 7.2 Skill File Deduplication

Skills loaded from overlapping parent directory walks are deduplicated by resolved file identity (via realpath). This handles symlinks and overlapping directory paths. The first-loaded copy wins; duplicates are logged and skipped.

#### 7.3 Dynamic Skill Deduplication

Dynamic skills are deduplicated against the base command set by name. Built-in command names are protected -- no skill or plugin can shadow them.

#### 7.4 Plugin Name Collision Detection

Within the plugin loader, duplicate plugin names across different marketplaces are detected and reported as errors. A verification function checks dependency graphs and demotes plugins with missing dependencies.

### 8. SkillTool -- Model Invocation

#### 8.1 Overview

The SkillTool bridges the model's tool-calling capability and the skill system. When the model decides to invoke a skill, it calls the Skill tool with a skill name and optional arguments.

#### 8.2 The Skill Listing Prompt

The model discovers available skills through a listing injected into the system prompt as an attachment. The listing is budget-constrained:

- **Budget**: 1% of the context window (in characters), defaulting to 8,000 characters
- **Per-entry cap**: 250 characters per description (truncated with ellipsis)
- **Bundled skills** are never truncated -- only user/plugin skill descriptions shrink
- **Extreme case**: When even truncated descriptions exceed the budget, non-bundled skills fall back to name-only entries

#### 8.3 Validation and Permission Checking

**Validation** confirms: the skill name is non-empty, the skill exists, model invocation is not disabled, and the command is prompt-type.

**Permission checking** implements a rule-based system:
1. Check deny rules (exact match or prefix with wildcard)
2. Check allow rules (same matching)
3. Auto-allow skills with only "safe properties" (no allowed tools, no hooks, no shell)
4. Default: ask user for permission, offering exact-match and prefix suggestions

#### 8.4 Execution Modes

**Inline execution** (default):
1. Expand the skill's markdown
2. Extract allowed tools and model overrides
3. Return the skill content as new messages and a context modifier for rule/model injection

**Forked execution** (`context: fork`):
1. Prepare forked context
2. Run a sub-agent with the skill prompt as initial messages
3. Stream progress back to the parent
4. Extract result text
5. Return as a tool result with forked status

**Remote execution** (experimental, feature-gated):
1. Load skill from remote storage
2. Strip frontmatter, inject base directory header
3. Register with compaction-preservation state
4. Inject as a user message (no shell command expansion -- remote skills are declarative only)

#### 8.5 Post-Invocation Effects

After inline execution:
- The skill's `allowedTools` are merged into always-allow rules
- The skill's `model` override replaces the main loop model
- The skill's `effort` level overrides the effort value
- The skill content is registered for compaction preservation
- Skill usage is recorded for ranking

### 9. Plugin CLI Commands

The plugin CLI commands are thin wrappers around the core operations that add console output and process exit:

| Command | Description |
|---|---|
| `claude plugin install <name> [--scope]` | Install from marketplace |
| `claude plugin uninstall <name> [--scope]` | Remove plugin |
| `claude plugin enable <name> [--scope]` | Enable disabled plugin |
| `claude plugin disable <name> [--scope]` | Disable plugin |
| `claude plugin disable-all` | Disable all enabled plugins |
| `claude plugin update <name> --scope` | Update to latest version |

**Valid installable scopes**: `user`, `project`, `local` (not `managed`).
**Valid update scopes**: `user`, `project`, `local`, `managed`.

Every command emits structured telemetry with success/failure event classification.

### 10. Security Model

#### 10.1 Manifest Validation

Plugin and marketplace manifests are validated with security-focused refinements:
- **Path traversal**: Scans commands, agents, skills, and source paths for `..` segments before schema validation.
- **Official name protection**: Blocks marketplace names impersonating official Anthropic/Claude marketplaces using regex detection and non-ASCII (homograph) character blocking.
- **Source org verification**: Ensures reserved marketplace names can only be used by repos from the authorized GitHub organization.
- **Plugin ID format**: Enforces `plugin@marketplace` format with restricted characters.

#### 10.2 Plugin Policy

Enterprise administrators can force-disable plugins via managed settings. Policy-blocked plugins cannot be installed, enabled, or have policy-blocked dependencies.

#### 10.3 Marketplace Source Restrictions

Managed settings can restrict which marketplace sources are allowed (via host and path pattern regexes) and explicitly deny specific sources via a blocklist.

#### 10.4 Bundled Skill File Security

Bundled skill reference files use defense-in-depth extraction:
- Per-process nonce prevents symlink pre-creation attacks
- Exclusive-create and no-follow flags prevent overwriting existing files
- Owner-only permissions (`0o700` directories, `0o600` files)
- Path traversal rejection for `..` segments and absolute paths

#### 10.5 MCP Skill Shell Restriction

MCP-sourced skills are treated as untrusted remote content. Shell command expansion is explicitly skipped for MCP skills.

#### 10.6 SkillTool Permission Model

A property-based allowlist determines which skills can auto-execute without user approval. If a skill has any property not in the safe set with a meaningful value, it requires explicit permission. Newly added properties default to requiring permission until reviewed.

#### 10.7 Gitignore Filtering for Dynamic Skills

Dynamically discovered skill directories are checked against `.gitignore`. This prevents skills from gitignored paths (e.g., `node_modules/`) from loading silently. The invocation-time trust dialog is the actual security boundary; gitignore filtering is an early guard.

#### 10.8 Plugin Variable Substitution

Plugin commands support variable substitution. Sensitive `userConfig` values resolve to a descriptive placeholder in skill content -- secrets are never injected into the conversation.

### 11. Design Principles

1. **Markdown as the Universal Extension Format**: Skills are markdown files with YAML frontmatter. No build step, no SDK, no runtime -- just a text file. The model reads the same markdown the author wrote.

2. **Settings-First Architecture**: Plugin operations write to settings first, then materialize. The settings file is the source of truth. If materialization fails, the next startup retries. The system converges toward the declared state.

3. **Namespace Isolation**: Plugin commands are prefixed with the plugin name. Built-in commands are explicitly protected from shadowing.

4. **Graduated Trust**: Different trust levels for different skill sources:

   | Source | Trust Level | Shell Execution | Auto-Allow |
   |---|---|---|---|
   | Bundled | Full | Yes | Yes (safe properties) |
   | User | High | Yes | Per permission rules |
   | Project | Medium | Yes | Per permission rules |
   | Plugin | Medium | Yes | Per permission rules + policy |
   | MCP | Low | No | Per permission rules |

5. **Lazy Loading and Caching**: Skills and plugins are loaded lazily and cached aggressively. Cache invalidation is explicit.

6. **Graceful Degradation**: Every loading path catches errors and continues. The system never crashes because an extension is broken.

7. **Compaction Awareness**: Skills are registered so their content survives context compaction. After auto-compact, active skills are re-injected (truncated to 5,000 characters each, up to 25,000 tokens total).

8. **Observability**: Every significant extensibility event is instrumented with telemetry events.
