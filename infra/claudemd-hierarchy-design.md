# CLAUDE.md Hierarchy — Design Document

## Part A: Design Document

### 1. Overview

CLAUDE.md is the primary mechanism through which users, teams, and enterprises customize Claude Code's behavior. It is a plain-text markdown file (or set of files) that contains natural-language instructions the model must follow. The system is analogous to `.editorconfig` or `.eslintrc` for AI behavior: it tells the agent what conventions to follow, what tools to prefer, what patterns to avoid, and how to interact with a particular codebase.

The instructions are loaded once at session start (via a memoized loader function), assembled into a single prompt string, and injected into the dynamic region of the system prompt. The model is told explicitly:

```
Codebase and user instructions are shown below. Be sure to adhere to these instructions.
IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them
exactly as written.
```

This framing gives CLAUDE.md content higher behavioral authority than the system prompt's default guidelines -- a deliberate design choice that lets project owners shape agent behavior without modifying Claude Code itself.

The system supports six memory types:

| Type | Description |
|---|---|
| `Managed` | Enterprise/admin-controlled instructions (policy) |
| `User` | Personal global instructions across all projects |
| `Project` | Repository-level instructions (committed to VCS) |
| `Local` | Private project-specific overrides (gitignored) |
| `AutoMem` | Auto-extracted persistent memory (cross-session) |
| `TeamMem` | Shared team memory (feature-gated) |

This document focuses on the first four -- the "instruction hierarchy" proper. AutoMem and TeamMem are covered in a companion context management design spec.

---

### 2. Four-Level Priority Hierarchy

Files are loaded in a strict four-level priority order, from lowest to highest:

```
Priority (lowest -> highest, model pays more attention to later entries):

  Level 1: Managed    System-wide managed directory + CLAUDE.md      <- enterprise policy
  Level 2: User       ~/.claude/CLAUDE.md + ~/.claude/rules/*.md     <- personal global preferences
  Level 3: Project    CLAUDE.md + .claude/CLAUDE.md                  <- repo-level instructions
                      + .claude/rules/*.md
  Level 4: Local      CLAUDE.local.md                                <- private project overrides
```

The ordering is enforced by the memory file loader which processes sources in exactly this sequence: Managed first, then User, then Project (from root toward CWD), then Local. Because the assembled prompt preserves insertion order and the model tends to weight later content more heavily, higher-priority files effectively override lower-priority ones when instructions conflict.

#### 2.1 Level 1: Managed (Enterprise Policy)

**Path:** Platform-dependent managed directory + `CLAUDE.md`

| Platform | Base Path |
|---|---|
| macOS | `/Library/Application Support/ClaudeCode/` |
| Linux | `/etc/claude-code/` |
| Windows | `C:\Program Files\ClaudeCode\` |

Managed instructions are controlled by system administrators and apply globally to all users on a machine. They are loaded unconditionally -- the policy settings source is always enabled regardless of user or project configuration. This level also supports a rules directory at `<managed-path>/.claude/rules/*.md`.

#### 2.2 Level 2: User (Personal Global)

**Path:** `~/.claude/CLAUDE.md` (plus `~/.claude/rules/*.md`)

User instructions express personal preferences that apply across all projects: preferred coding style, tool preferences, language settings. This level is gated by a user settings source -- it can be disabled but is enabled by default.

User memory files have a special privilege: they are **always allowed to include external files** via `@path` syntax. This is because the user explicitly authored these files and should be trusted to reference their own filesystem.

#### 2.3 Level 3: Project (Repository-Level)

**Paths:** `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md` -- at every directory from root down to CWD.

Project instructions are the most common level. They are checked into version control and shared with the entire team. The system looks for three types of files at each directory:

1. `CLAUDE.md` -- top-level project instructions
2. `.claude/CLAUDE.md` -- nested under the `.claude` directory (equivalent)
3. `.claude/rules/*.md` -- modular rule files (see Rules Directory section)

Gated by the project settings source. For external `@path` includes, project files require explicit user approval via an external includes dialog.

#### 2.4 Level 4: Local (Private Project Overrides)

**Path:** `CLAUDE.local.md` at each directory from root to CWD.

Local instructions override everything else. They are intended for private, per-developer customization that should not be committed to version control. A typical `.gitignore` would include `CLAUDE.local.md`.

Gated by the local settings source.

---

### 3. File Discovery

#### 3.1 The Directory Walk

Project and Local files are discovered by walking the directory hierarchy from the current working directory upward to the filesystem root. The collected directories are then reversed so processing happens **from root downward toward CWD**. This means files closer to the CWD are loaded later, giving them higher effective priority:

```
/home/user/projects/                   <- loaded first (lowest priority)
/home/user/projects/myapp/             <- loaded second
/home/user/projects/myapp/packages/    <- loaded third (CWD, highest priority)
```

At each directory, the system attempts to read (in order):
1. `<dir>/CLAUDE.md` (Project)
2. `<dir>/.claude/CLAUDE.md` (Project)
3. `<dir>/.claude/rules/*.md` (Project, recursive into subdirectories)
4. `<dir>/CLAUDE.local.md` (Local)

All file reads are speculative -- file-not-found and directory-instead-of-file errors are handled silently.

#### 3.2 Git Worktree Deduplication

When Claude Code runs inside a git worktree nested within its main repository (as with the `-w` flag which creates worktrees under `.claude/worktrees/<name>/`), the upward walk passes through both the worktree root and the main repo root. Both directories contain the same checked-in files (CLAUDE.md, .claude/rules/), which would cause duplicate loading.

The system detects this situation by comparing the worktree root with the canonical (main repo) root. If they differ and the worktree is nested inside the main repo, Project-type files from directories above the worktree but within the main repo are skipped. `CLAUDE.local.md` is still loaded from those directories because it is gitignored and only exists in the main repo.

#### 3.3 Additional Directories (--add-dir)

When an environment flag is enabled and `--add-dir` was used, CLAUDE.md files from the additional directories are also loaded. These follow the same three-file pattern (CLAUDE.md, .claude/CLAUDE.md, .claude/rules/) but are scoped to the single specified directory rather than walking upward.

This is gated behind an environment variable because it is an explicit user action -- the SDK defaults setting sources to an empty array when not specified, which would normally suppress project settings, but `--add-dir` should still be honored since the user explicitly asked for it.

#### 3.4 Bare Mode and Disable Flags

Two mechanisms suppress CLAUDE.md loading entirely:

- **Hard disable**: All CLAUDE.md content is suppressed via an environment variable.
- **`--bare` mode**: Skips auto-discovery (the directory walk), but honors explicit `--add-dir` directories. The rationale: `--bare` means "skip what I didn't ask for", not "ignore what I explicitly asked for."

---

### 4. @path Syntax -- Composing Instructions from Multiple Sources

CLAUDE.md files can reference other files using the `@path` directive. This allows modular composition of instructions from multiple sources without duplicating content.

#### 4.1 Syntax

```markdown
@path/to/file.md
@./relative/path.md
@~/home-relative/path.md
@/absolute/path.md
@path/with\ spaces/file.md
@path/to/file.md#heading        (fragment stripped, file included whole)
```

The `@` sigil must appear at the start of a word boundary (preceded by whitespace or start-of-line). Paths without a prefix (`@path`) are treated as relative to the including file's directory -- equivalent to `@./path`.

#### 4.2 Path Resolution

Paths are resolved using the including file's directory as the base for relative paths. Three forms are recognized:

| Prefix | Resolution |
|---|---|
| `./` or bare | Relative to the including file's directory |
| `~/` | Relative to the user's home directory |
| `/` | Absolute path |

Fragment identifiers (`#heading`) are stripped before resolution -- the entire referenced file is included regardless of any fragment.

#### 4.3 Parsing

Include paths are extracted during markdown lexing. The parser uses a markdown lexer with GFM disabled to prevent `~/path` from being tokenized as strikethrough markup. The extraction:

- **Processes only text nodes** -- `@` references inside code blocks and inline code spans are ignored
- **Handles HTML comments specially** -- strips comment spans and checks residual text for `@` references
- **Recurses into list items** -- item arrays on list tokens are walked
- **Supports escaped spaces** -- `\ ` in paths is unescaped to a literal space

#### 4.4 Text-File Allowlist

To prevent binary files (images, PDFs, compiled artifacts) from being loaded into the prompt, included files are checked against a comprehensive allowlist of approximately 100 text file extensions. The set covers:

- Markdown, text, and documentation formats (`.md`, `.txt`, `.rst`, `.adoc`)
- Data formats (`.json`, `.yaml`, `.toml`, `.xml`, `.csv`)
- Source code in approximately 30 languages (`.js`, `.ts`, `.py`, `.go`, `.rs`, `.java`, etc.)
- Configuration files (`.env`, `.ini`, `.conf`, `.properties`)
- Build files (`.cmake`, `.gradle`, `.sbt`)

Files with extensions not in this set are silently skipped.

#### 4.5 External Include Approval

When a Project-level CLAUDE.md includes files outside the current working directory, the system triggers an approval dialog. This prevents a cloned repository from silently pulling in files from arbitrary locations on the user's machine.

The approval state is tracked per-project in the project config. User-level files (`~/.claude/CLAUDE.md`) bypass this check entirely -- they always have external include permission.

---

### 5. Circular Reference Prevention

The recursive inclusion walker maintains a set of processed paths that accumulates every file path encountered during the walk. Before processing a file, its path is checked against this set; if already present, the file is skipped.

Two complementary guards prevent unbounded recursion:

1. **Path deduplication** -- Paths are normalized for comparison to handle platform-specific case sensitivity (e.g., Windows drive letter casing). Symlinks are resolved and both the original and resolved paths are added to the set, preventing cycles through symlinked directories.

2. **Depth limit** (`MAX_INCLUDE_DEPTH = 5`) -- Even without cycles, deeply nested includes are capped. This bounds the total I/O cost of the inclusion walk.

The processed paths set is shared across all files at a given level -- if a User-level file includes `@./shared.md`, and a Project-level file also includes `@./shared.md` resolving to the same absolute path, the file is loaded only once.

#### 5.1 Symlink Cycle Detection in Rules Directories

The rules directory processor maintains a separate visited-directories set to detect directory-level cycles via symlinks. Before descending into a subdirectory, its path is checked (both original and resolved) against this set. This prevents infinite directory traversal when a rules directory contains a symlink that points back to an ancestor.

---

### 6. Injection Filtering and Content Sanitization

Content undergoes multiple transformations between disk and prompt injection.

#### 6.1 HTML Comment Stripping

Block-level HTML comments (`<!-- ... -->`) are stripped from CLAUDE.md content before injection. This allows file authors to include private notes that are not sent to the model.

The stripping uses a markdown lexer to identify HTML-type tokens at the block level only:

- Comments inside code blocks and inline code spans are preserved
- Comments inside paragraphs (inline HTML) are left intact
- Unclosed comments (`<!--` without `-->`) are left in place to avoid silently swallowing content after a typo

When a comment is stripped, any residual content on the same line is preserved. For example, `<!-- note --> Use bun` keeps `Use bun`.

#### 6.2 Frontmatter Stripping

YAML frontmatter (delimited by `---`) is parsed and removed from content before prompt injection. The frontmatter can contain metadata such as:

- `paths` -- glob patterns controlling when a rule applies
- `description` -- human-readable description of the rule
- `allowed-tools` -- tool access declarations

The frontmatter is parsed via a YAML parser with a fallback: if initial parsing fails (common with glob patterns containing special YAML characters like `{`, `*`), the system retries after auto-quoting problematic values.

#### 6.3 Path-Based Exclusion

A `claudeMdExcludes` setting allows users to specify glob patterns for CLAUDE.md files that should be skipped entirely. This applies to User, Project, and Local types -- Managed and AutoMem/TeamMem files are never excluded.

Exclusion patterns support picomatch glob syntax with dot-file matching. To handle macOS symlinks (where `/tmp` resolves to `/private/tmp`), the system expands absolute patterns by resolving symlinks in their static path prefixes and matching against both the original and resolved forms.

#### 6.4 Feature-Gated Content Filtering

An additional filtering layer exists: when a specific feature flag is enabled, AutoMem and TeamMem entries are excluded from the system prompt injection (they are instead surfaced via attachments by the prefetch system). This function is called before assembling the prompt string.

#### 6.5 Project-Level Skipping

A separate feature flag can cause Project and Local type files to be skipped entirely during prompt assembly. This is a runtime kill switch for the project instruction system.

---

### 7. Enterprise Managed Instructions

#### 7.1 Policy Settings Source

The policy settings source is architecturally privileged:

- **Always enabled** -- unconditionally added to the enabled set, regardless of user flags
- **Read-only** -- excluded from the set of editable setting sources
- **First-source-wins** -- when multiple policy sources exist (remote API, MDM, file, registry), the highest-priority non-null source wins

#### 7.2 Policy Source Cascade

For policy settings, the system checks four sources in priority order:

1. **Remote managed settings** -- fetched from an API. This allows organizations to push configuration updates without touching each machine.
2. **MDM/system policy** -- read from OS-level management frameworks (HKLM on Windows, managed preferences on macOS)
3. **Managed settings file** -- a JSON settings file with drop-in directory support
4. **HKCU (Windows)** -- current-user registry settings

The first non-null source in this cascade is used as the policy settings.

#### 7.3 Managed CLAUDE.md and Rules

Enterprise administrators can deploy:

- `<managed-path>/CLAUDE.md` -- global instructions for all users on the machine
- `<managed-path>/.claude/rules/*.md` -- modular rule files

These follow the same format and processing pipeline as user and project files but cannot be overridden by users -- they are loaded first and the model is instructed to follow all levels of instructions, with later (higher-priority) content taking precedence when there are conflicts.

#### 7.4 Disabling Instruction Sources

Managed settings can restrict which instruction sources are loaded. The setting sources configuration (typically set via policy) controls whether user, project, and local settings are enabled. This allows enterprises to lock down the instruction pipeline -- for example, disabling project settings to prevent cloned repositories from influencing agent behavior.

---

### 8. Project vs Local

The distinction between Project (`.claude/CLAUDE.md`) and Local (`CLAUDE.local.md`) files is a core architectural decision that mirrors the split between committed and gitignored configuration in tools like `.env` vs `.env.local`.

#### 8.1 Project Instructions

- **Committed to version control** -- shared with all developers
- **Multiple file locations**: both `CLAUDE.md` at the project root and `.claude/CLAUDE.md` are supported
- **External includes require approval** -- a dialog is shown on first use
- **Subject to enable/disable** -- can be suppressed by flags or managed policy

#### 8.2 Local Instructions

- **Not committed** -- should be listed in `.gitignore`
- **Highest priority** -- loaded last at each directory level, after Project files
- **Use cases**: developer-specific tool paths, personal API endpoints, experimental instructions

#### 8.3 Prompt Differentiation

When CLAUDE.md content is assembled into the prompt, each file is annotated with its source type and a human-readable description:

```
Contents of /path/to/.claude/CLAUDE.md (project instructions, checked into the codebase):

<content>

Contents of /path/to/CLAUDE.local.md (user's private project instructions, not checked in):

<content>
```

This labeling helps the model understand the provenance and relative authority of conflicting instructions.

---

### 9. Rules Directory

#### 9.1 Structure

The `.claude/rules/` directory enables modular, file-per-concern instruction organization:

```
.claude/
|-- CLAUDE.md                        <- monolithic project instructions
+-- rules/
    |-- coding-style.md              <- unconditional rule
    |-- testing.md                   <- unconditional rule
    |-- typescript-conventions.md    <- conditional rule (paths: **/*.ts)
    +-- api/
        +-- rest-guidelines.md       <- unconditional rule (subdirectories supported)
```

Rules directories exist at three levels:
- `<managed-path>/.claude/rules/` -- enterprise rules
- `~/.claude/rules/` -- user-global rules
- `<project-dir>/.claude/rules/` -- project rules (at each directory in the CWD walk)

#### 9.2 Unconditional vs Conditional Rules

**Unconditional rules** -- no `paths` frontmatter, or `paths: **` (match-all). These are always loaded, regardless of what files the model is working with. They are loaded eagerly at session start.

**Conditional rules** -- have `paths` frontmatter specifying glob patterns:

```yaml
---
paths: src/**/*.ts, src/**/*.tsx
---
Always use `interface` instead of `type` for object shapes.
```

Conditional rules are loaded **lazily** -- only when the model touches a file matching the glob patterns. The matching logic uses gitignore-style pattern matching:

- For Project rules: glob patterns are relative to the directory containing `.claude/`
- For Managed/User rules: glob patterns are relative to the original CWD

#### 9.3 Frontmatter Path Parsing

The `paths` field supports:
- Comma-separated strings: `paths: src/*.ts, lib/*.ts`
- YAML lists: `paths: ["src/*.ts", "lib/*.ts"]`
- Brace expansion: `paths: src/*.{ts,tsx}` expands to `src/*.ts` and `src/*.tsx`
- Nested brace expansion: `{a,b}/{c,d}` expands to `a/c`, `a/d`, `b/c`, `b/d`
- Commas inside braces are not treated as separators

The `/**` suffix is automatically stripped because the glob library already treats a path as matching both itself and everything inside it.

#### 9.4 Recursive Directory Processing

Rules directories are recursively descended into subdirectories. Symlinks are resolved and tracked to prevent cycles. Only files with `.md` extensions are processed. Non-directory, non-`.md` entries are silently skipped.

---

### 10. Integration with Context Assembly

#### 10.1 The Assembly Pipeline

CLAUDE.md content enters the system prompt through a pipeline of memoized functions:

```
Load memory files                   -> MemoryFileInfo[]
  |
Filter (feature-gated)              -> MemoryFileInfo[]
  |
Assemble prompt text                -> string
  |
Build user context                  -> { claudeMd, currentDate }
  |
Inject into system prompt           -> system prompt array element
```

#### 10.2 Placement in the System Prompt

The system prompt has a critical architectural split at a dynamic boundary marker. CLAUDE.md content is placed in the **dynamic region** (after the boundary), meaning it is NOT shared across organizations for prompt caching. This is correct -- CLAUDE.md content is user/project-specific.

The assembled CLAUDE.md content appears in the system prompt within a `<system-reminder>` tag with framing that establishes the override authority of the instructions.

#### 10.3 Memoization and Cache Invalidation

The memory file loader is memoized -- it computes once per session and returns the cached result for subsequent calls. The cache can be invalidated through two paths:

- **Correctness-only invalidation** -- clears the memoize cache without firing hooks. Used for worktree enter/exit, settings sync, and memory dialogs.
- **Full reload invalidation** -- clears the cache AND arms an instructions-loaded hook to fire on the next load. Used when instructions are genuinely being reloaded into context (e.g., after compaction).

#### 10.4 Classifier Cache

The assembled CLAUDE.md string is cached separately for the auto-mode classifier. This avoids a circular dependency in the import graph: the permissions system depends on the classifier which would otherwise need to import the CLAUDE.md module. The cached string lets the classifier read CLAUDE.md content without creating the cycle.

#### 10.5 InstructionsLoaded Hook

When files are loaded (and the hook is armed), the system fires hook events for each instruction file. These hooks support audit and observability use cases -- enterprise deployments can monitor which instructions are being loaded, from where, and why. The hook receives:

- File path
- Memory type (User, Project, Local, Managed)
- Load reason (session start, compact, include)
- Metadata (glob patterns, parent file path for includes)

---

### 11. Security Considerations

#### 11.1 Threat Model

The primary threat is a **malicious repository** shipping adversarial CLAUDE.md or rules files designed to manipulate the agent. Attack vectors include:

1. **Prompt injection** -- instructions that override safety guidelines or manipulate the model
2. **Data exfiltration** -- `@path` includes that reference sensitive files outside the project
3. **Symlink attacks** -- symlinks in rules directories that point to sensitive directories
4. **Path traversal** -- `@../../etc/passwd` style includes
5. **Binary injection** -- including large binary files to overflow or confuse the context

#### 11.2 Defenses

- **External include approval**: Project-level `@path` directives that resolve outside the CWD trigger an explicit user approval dialog with a warning about third-party repositories. Approval state is per-project and persists in config.
- **Text-file allowlist**: Only files with recognized text extensions can be included. Approximately 100 extensions, intentionally conservative.
- **Depth limit** (MAX_INCLUDE_DEPTH = 5): Prevents unbounded recursion. Bounds total I/O cost.
- **Symlink resolution and cycle detection**: Both file-level and directory-level cycle detection use resolved symlink paths. Both original and resolved paths are tracked.
- **Path normalization**: Handles platform-specific case sensitivity to prevent duplicate loading through casing tricks.
- **Setting source control**: Enterprises can disable project settings entirely via managed policy.
- **Exclusion patterns**: Users can configure glob patterns to skip specific CLAUDE.md files.
- **Content size warnings**: Files exceeding 40,000 characters trigger a diagnostic warning.

#### 11.3 Trust Boundaries

| Level | Files | Include Policy | Can Disable? |
|---|---|---|---|
| Administrator | Managed (policy) | Always external | No |
| User | User (~/.claude/) | Always external | Via flags |
| Repository | Project + Local | External requires approval | Via flags or policy |

The critical security invariant: **untrusted repository content never automatically reaches outside the project directory**. External includes from Project files always require explicit user approval, and that approval is never implicitly granted.

---

### 12. Design Principles

1. **Layered Override** -- Enterprise policy provides guardrails, user preferences set defaults, project instructions customize for the codebase, local overrides handle developer-specific needs.

2. **Convention over Configuration** -- Instruction files are discovered automatically through naming conventions and directory structure. No registration or path configuration needed.

3. **Security by Default, Flexibility by Opt-In** -- The default posture is restrictive. Each relaxation requires an explicit user action.

4. **Composition over Monoliths** -- The `@path` syntax and rules directories enable instruction composition from multiple focused files rather than a single monolithic file.

5. **Transparent Provenance** -- Every instruction file is annotated with its full path and type. Traceability is essential for debugging unexpected agent behavior.

6. **Graceful Degradation** -- Missing files produce no errors. Malformed frontmatter falls back. Invalid YAML retries. The system works in the common case and degrades quietly in edge cases.

7. **Enterprise Controllability** -- The managed instruction level, combined with the always-enabled policy source, gives enterprises a reliable enforcement point.

8. **Cache Awareness** -- CLAUDE.md content is placed after the dynamic boundary to avoid invalidating the cross-org prompt cache. Memoization avoids redundant computation. Hooks fire at controlled points rather than on every access.
