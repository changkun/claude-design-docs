# Command System — Design Document

## Part A: Design Document

### 1. Overview

Commands are Claude Code's primary user-facing extension surface. When a user types a slash command (e.g., `/commit`, `/help`, `/my-custom-skill`), the system matches the input against a unified registry drawn from multiple distinct sources, dispatches the match to one of three execution models, and integrates the result back into the conversation or terminal UI.

The command system serves three audiences simultaneously:

1. **End users** -- who type `/command` in the REPL to trigger actions, toggle settings, or invoke skills
2. **The model** -- which invokes prompt-type commands (skills) via the Skill tool, expanding specialized instructions into the conversation
3. **Extension authors** -- who publish skills as markdown files in user, project, or plugin directories

The architecture unifies these audiences through a single Command type that carries enough metadata for the REPL typeahead, the model's skill index, the help screen, and the execution engine to all operate from one source of truth.

---

### 2. Command Types

Every command is one of three discriminated union variants, distinguished by a `type` field.

#### 2.1 Prompt Commands (`type: 'prompt'`)

Prompt commands expand into content blocks that are injected into the conversation as a user message and sent to the LLM. The model then processes the injected instructions and responds, possibly calling tools. The expansion function is async and receives both user-supplied arguments and the full tool-use context.

This is the execution model for all skills (user-authored, plugin-provided, bundled, and MCP-sourced).

**Examples:** `/commit`, `/review`, `/security-review`, all user-defined skills

#### 2.2 Local Commands (`type: 'local'`)

Local commands execute entirely on the client side, producing a result that is either a text string, a compaction result, or a skip sentinel. They never touch the LLM. Implementation is lazy-loaded to defer heavy imports until invocation.

A `supportsNonInteractive` flag indicates whether the command can run in headless (SDK/non-interactive) mode. Commands that render terminal UI or require terminal input set this to `false`.

**Result variants:**
- `text` -- displayed as a system message
- `compact` -- triggers compaction UI
- `skip` -- no visible output

**Examples:** `/compact`, `/clear`, `/advisor`

#### 2.3 Local JSX Commands (`type: 'local-jsx'`)

Local JSX commands render interactive React/Ink UI components within the terminal. Like local commands, they are lazy-loaded, but their invocation returns a React node and communicates completion via a callback rather than a return value.

The completion callback supports rich signaling:
- Control display mode (skip, system message, user message)
- Chain into model queries after completion
- Inject hidden messages visible to the model
- Pre-fill or auto-submit the next user input

**Examples:** `/help`, `/config`, `/model`, `/login`, `/exit`, `/session`

---

### 3. Command Metadata (Common Interface)

All three command types share a base that carries metadata used by the REPL typeahead, help system, skill index, analytics, and security gates:

**Identity:** Primary name, optional aliases, optional display-name override.

**Documentation:** One-line description, optional argument hint text, optional extended usage guidance for the model.

**Visibility and access control:** Auth/provider requirements, dynamic enablement callback, hidden flag.

**Skill metadata:** Version, model-invocation blocking, user-invocability, provenance tracking, workflow badge, MCP flag.

**Execution behavior:** Immediate execution flag (bypass queue), sensitive-args redaction flag.

#### Key Metadata Semantics

**Provenance tracking (`loadedFrom`)** records where a command was loaded from, which is critical for filtering commands into the right views:

| Value | Meaning |
|---|---|
| `'commands_DEPRECATED'` | Legacy commands directory |
| `'skills'` | Modern skills directory |
| `'plugin'` | Installed plugin package |
| `'managed'` | Enterprise-managed policy skills |
| `'bundled'` | Ships compiled into the CLI binary |
| `'mcp'` | Discovered from an MCP server |

**Source hierarchy level (`source`)** on prompt commands records which settings tier the command belongs to:

| Value | Meaning |
|---|---|
| `'policySettings'` | Enterprise policy |
| `'userSettings'` | User home |
| `'projectSettings'` | Project repo |
| `'builtin'` | Hardcoded in the CLI |
| `'plugin'` | Plugin package |
| `'bundled'` | Bundled skill |
| `'mcp'` | MCP server |

**Immediate execution** controls timing. Most commands wait for the current model turn to finish before executing. Commands marked immediate bypass the queue and execute instantly. Some commands compute this dynamically.

**Description** can be a dynamic getter that re-evaluates on access (e.g., showing the current model name).

---

### 4. Registry and Loading Architecture

#### 4.1 Multi-Layer Source Hierarchy

Commands are drawn from seven distinct sources, loaded in parallel, and merged into a single ordered array. The ordering determines precedence -- earlier entries shadow later ones when names collide:

```
1. Bundled skills          (highest precedence)
2. Built-in plugin skills
3. User/project skills
4. Workflow commands
5. Plugin commands
6. Plugin skills
7. Built-in commands       (lowest precedence, fallback layer)
+ Dynamic skills           (inserted between steps 6 and 7)
```

This ordering means:
- Bundled skills have the highest precedence
- User/project skills override built-in commands of the same name
- Plugin commands sit between user skills and built-ins
- Built-in commands are the fallback layer

#### 4.2 Built-In Commands

The built-in command list is a memoized function (not a module-level constant) because underlying functions read from config that is unavailable at module initialization time. The array conditionally includes feature-gated commands and internal-only commands.

#### 4.3 Internal-Only Commands

A separate set of commands is restricted to internal (Anthropic) users via a runtime environment variable check. These are compiled into the binary but filtered out at runtime for external users.

#### 4.4 Skill Directory Loading

Skill directory commands are loaded from the filesystem hierarchy. The system searches multiple locations in parallel:

1. **Managed skills** -- enterprise policy directory
2. **User skills** -- personal home directory
3. **Project skills** -- project repo, walking up from cwd to home
4. **Additional directory skills** -- from explicit CLI flags
5. **Legacy commands** -- deprecated command directories

Each skill is a directory containing a `SKILL.md` file (frontmatter + markdown prompt content).

**Key behaviors:**
- **Deduplication**: Skills from overlapping paths (symlinks, parent directories) are deduplicated by resolving to canonical paths. First-loaded copy wins.
- **Bare mode**: When active, skill directory discovery is skipped except for explicit additional paths. This is a startup optimization for headless/SDK usage.
- **Policy lockout**: Enterprise policy can restrict loading to only managed skills, suppressing user and project skills entirely.

#### 4.5 Bundled Skills

Bundled skills are compiled into the CLI binary and registered programmatically at startup. They bypass filesystem loading entirely. Bundled skills that include ancillary files use a lazy-extraction mechanism: files are embedded in the binary and extracted to a per-process temp directory on first invocation. The extraction uses secure file creation flags and per-process nonces to defend against symlink attacks.

#### 4.6 Plugin Commands and Skills

Plugins provide two command namespaces: commands and skills. Both are namespaced with the plugin name (e.g., `my-plugin:deploy`) to prevent cross-plugin name collisions.

Plugin commands support:
- YAML frontmatter metadata
- Variable substitution for plugin root paths, user config, and session IDs

#### 4.7 Built-In Plugin Skills

Built-in plugins are a hybrid between bundled skills and marketplace plugins. They ship with the CLI but appear in the plugin UI where users can enable/disable them.

#### 4.8 MCP Skills

MCP-connected servers can expose skills loaded into the command registry. A critical security boundary: MCP skills never execute inline shell commands from their markdown body, because MCP content is remote and untrusted.

#### 4.9 Workflow Commands

Workflow-backed commands are generated from user-defined workflow YAML files. They are prompt-type commands with a special `kind` that causes them to display a "workflow" badge in the autocomplete UI.

---

### 5. Command Resolution

#### 5.1 Master Query

The primary entry point for obtaining the filtered, available command list:
1. Load all commands (memoized by working directory)
2. Filter by availability and enablement (fresh on every call, since auth state can change)
3. Merge in dynamic skills before built-in commands

Key properties:
- Expensive disk I/O and dynamic imports happen once per working directory
- Availability and enablement checks run fresh every call
- Dynamic skills are positioned before built-in commands but after plugin commands

#### 5.2 Name Lookup

Resolution checks three identifiers in order:
1. Internal name
2. User-facing name (display-name override)
3. Any alias

A throwing variant produces a diagnostic error listing all available commands.

#### 5.3 Memoization and Cache Invalidation

The loading pipeline uses multiple memoization layers with different scopes (per-cwd vs. process-wide) and different invalidation strategies.

Two invalidation entry points:
- **Partial reset** -- clears only memoization layers (used when dynamic skills are added mid-session)
- **Full reset** -- clears all caches including skill directories, plugins, and conditional skill state

Dynamic skill loading fires a signal that downstream caches (like the skill search index) subscribe to for invalidation.

---

### 6. Execution Models

#### 6.1 Prompt Command Execution Flow

```
User types /command --> find command --> expand prompt (getPromptForCommand)
    --> content blocks --> inject as user message --> send to LLM --> model responds
```

Prompt expansion can perform substantial work:
1. Argument substitution (positional and named placeholders)
2. Variable substitution (skill directory, session ID, plugin root)
3. Shell command execution (inline commands in the markdown body, output spliced into prompt)
4. Base directory injection (skills from directories get a path prefix)

#### 6.2 Local Command Execution Flow

```
User types /command --> find command --> lazy load --> call(args, context) --> result
```

Result is processed based on type (text display, compaction, or skip).

#### 6.3 Local JSX Command Execution Flow

```
User types /command --> find command --> lazy load --> call(onDone, context, args) --> React node
```

The returned React node is rendered in the terminal. Completion callback controls what happens next.

#### 6.4 Lazy Loading Strategy

All three command types use lazy loading to minimize startup time. Only metadata (name, description, type) is available at the top level. Implementation is behind a load function or prompt-generation function that triggers dynamic imports only on invocation.

#### 6.5 Forked Execution Context

Prompt commands can specify `context: 'fork'` to run as sub-agents with separate context and token budgets, rather than expanding inline into the current conversation.

---

### 7. Availability and Enablement

The system has two independent gates controlling command visibility and invocability.

#### 7.1 Availability (Auth/Provider Requirements)

Commands can declare which auth contexts they require (`claude-ai` and/or `console`). Commands without availability restrictions are universal. This check is deliberately not memoized since auth state can change mid-session.

#### 7.2 Enablement (Dynamic Feature Gates)

An optional callback returns false to disable a command at runtime. Reasons include feature flags, environment variables, runtime conditions, and user type checks.

#### 7.3 Remote/Bridge Safety Classification

Two allowlists gate which commands can execute in constrained environments:
- **Remote-safe commands** -- safe for remote mode (TUI state only, no local filesystem dependency)
- **Bridge-safe commands** -- safe when input arrives over the Remote Control bridge

Bridge safety follows a type-based policy:
- `local-jsx` commands always blocked (they render terminal UI)
- `prompt` commands always allowed (they expand to text)
- `local` commands need explicit opt-in via allowlist

#### 7.4 Visibility vs. Invocability Matrix

| Mechanism | Effect |
|---|---|
| `isHidden: true` | Hidden from typeahead and help, but still invocable by name |
| `isEnabled: () => false` | Fully disabled -- invisible and non-functional |
| `availability` mismatch | Fully filtered out -- invisible and non-functional |
| `disableModelInvocation: true` | Hidden from the model's Skill tool, but user can still type it |
| `userInvocable: false` | Model can invoke it, but hidden from user typeahead |

---

### 8. The Skill System

#### 8.1 Skill File Format

A skill is a directory containing a `SKILL.md` file with optional YAML frontmatter followed by markdown prompt content.

#### 8.2 Frontmatter Fields

| Field | Purpose |
|---|---|
| `description` | One-line description (auto-derived from first line if absent) |
| `name` | Display name override (defaults to directory name) |
| `allowed-tools` | Tool permission constraints |
| `argument-hint` | Gray hint text in typeahead |
| `arguments` | Named argument placeholders |
| `when_to_use` | Extended guidance for model's skill index |
| `version` | Skill version |
| `model` | Override model for execution |
| `disable-model-invocation` | Block model from invoking |
| `user-invocable` | Whether user can type the command |
| `hooks` | Lifecycle hooks to register on invocation |
| `context` | Run as sub-agent (`fork`) vs. inline expansion (`inline`, the default) |
| `agent` | Agent type for forked execution |
| `effort` | Reasoning effort override |
| `paths` | Glob patterns for conditional activation |
| `shell` | Shell to use for inline command execution |

#### 8.3 Skill Directory Hierarchy (Priority Order)

```
Enterprise policy directory    (highest priority)
User personal directory
Project root directory
Nested project directories     (dynamic discovery)
Explicit additional directories
Legacy command directories     (lowest priority)
```

#### 8.4 Dynamic Skill Discovery

Skills are not only loaded at startup. When the model reads or writes files in subdirectories, the system walks up from the file path looking for skill directories. Newly discovered directories are loaded and merged into the registry.

Discovered skills are tracked in a module-scoped map. On discovery, a signal fires to invalidate downstream caches.

#### 8.5 Conditional Skills (Path-Filtered)

Skills with a `paths` frontmatter field are held in a pending state until the model touches files matching the glob patterns. This prevents irrelevant skills from cluttering the index.

Once activated, a conditional skill remains active for the rest of the session (survives cache clears).

#### 8.6 Skill Tool Filtering

Two filtered views of the command list serve the model:
- **Skill tool commands** -- all prompt-based commands the model can invoke via the Skill tool (excludes `disableModelInvocation`, excludes `builtin` source, requires certain provenance or description)
- **Slash command skills listing** -- the subset exposed in the `/skills` UI

---

### 9. Feature Gating Strategy

#### 9.1 Compile-Time Dead Code Elimination

Commands gated behind feature flags are either included in or excluded from the binary at build time. When a flag is false, the command module is not included in the compiled binary, saving bundle size.

#### 9.2 Feature Flags in Use

Feature flags gate commands for: bridge mode, voice mode, proactive mode, daemon mode, history snipping, workflow scripts, remote setup, experimental skill search, GitHub webhooks, ultraplan, torch, inbox, fork subagent, buddy, MCP skills.

#### 9.3 Bundled Skill Feature Gating

Bundled skills use the same compile-time mechanism but at registration time.

#### 9.4 Internal-Only Runtime Gating

Some commands use runtime environment checks to skip registration entirely for external users.

---

### 10. Tool Permission Constraints

#### 10.1 Allowed Tools

Prompt commands can declare an `allowedTools` array constraining which tools the model may use during execution. This is a security and correctness mechanism (e.g., `/commit` should only use git commands).

#### 10.2 Permission Propagation

When a skill executes, its allowed tools are merged into the tool permission context. Tools declared in `allowedTools` are auto-approved for inline shell commands during prompt expansion, without requiring user confirmation.

#### 10.3 Plugin Variable Substitution

Plugin commands support variable substitution in their allowed-tools values, enabling tool rules relative to the plugin's installation directory.

---

### 11. Design Principles

1. **Unified Abstraction** -- A single Command type serves all audiences. Adding a command means defining one object.
2. **Lazy Everything** -- Implementations never loaded at startup; only metadata is eagerly available.
3. **Source Hierarchy with Deterministic Precedence** -- Predictable shadowing across all seven source layers.
4. **Separation of Visibility from Capability** -- Independent mechanisms for auth filtering, enablement, hiding, and model/user visibility. Note that `isEnabled: false` fully removes the command (invisible and non-functional), while `isHidden: true` keeps it invocable but hidden from typeahead.
5. **Security-Conscious Extension** -- MCP skills sandboxed (no shell execution), plugin namespace isolation, enterprise policy lockout, secure file extraction for bundled skills.
6. **Progressive Discovery** -- The skill index grows organically as the user works deeper into a codebase.
7. **Graceful Degradation** -- Every layer of skill loading is wrapped in error handling with fallback to empty arrays.
8. **Cache-Conscious Architecture** -- Memoized loading with fresh filtering balances startup cost against runtime correctness.
9. **Extensibility Without Framework Lock-In** -- Skills are markdown files; no TypeScript, build step, or SDK dependency required.

---

### 12. Key Commands by Category

#### Session Management
`/clear`, `/compact`, `/exit`, `/resume`, `/session`

#### Git and Code Review
`/commit`, `/review`, `/ultrareview`, `/diff`, `/pr_comments`

#### Configuration
`/config`, `/model`, `/fast`, `/theme`, `/vim`, `/permissions`, `/effort`, `/advisor`

#### Context and Memory
`/memory`, `/context`, `/cost`, `/files`

#### Extension Points
`/skills`, `/mcp`, `/plugin`

#### Information and Help
`/help`, `/status`, `/doctor`, `/usage`, `/insights`

#### Authentication
`/login`, `/logout`

#### Bundled Skills (Model-Invocable)
`update-config`, `keybindings`, `verify`, `debug`, `lorem-ipsum`, `skillify`, `remember`, `simplify`, `batch`, `stuck`
