# Claude Code: CLAUDE.md Hierarchy — Design Specification

This document analyzes the multi-level instruction file system of Claude Code — Anthropic's
agentic CLI tool — focusing on how CLAUDE.md files are discovered, prioritized, composed,
filtered, and injected into the system prompt to customize agent behavior.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Four-Level Priority Hierarchy](#2-four-level-priority-hierarchy)
- [3. File Discovery](#3-file-discovery)
- [4. @path Syntax — Composing Instructions from Multiple Sources](#4-path-syntax--composing-instructions-from-multiple-sources)
- [5. Circular Reference Prevention](#5-circular-reference-prevention)
- [6. Injection Filtering and Content Sanitization](#6-injection-filtering-and-content-sanitization)
- [7. Enterprise Managed Instructions](#7-enterprise-managed-instructions)
- [8. Project vs Local](#8-project-vs-local)
- [9. Rules Directory](#9-rules-directory)
- [10. Integration with Context Assembly](#10-integration-with-context-assembly)
- [11. Security Considerations](#11-security-considerations)
- [12. Design Principles](#12-design-principles)

---

## 1. Overview

CLAUDE.md is the primary mechanism through which users, teams, and enterprises customize
Claude Code's behavior. It is a plain-text markdown file (or set of files) that contains
natural-language instructions the model must follow. The system is analogous to `.editorconfig`
or `.eslintrc` for AI behavior: it tells the agent what conventions to follow, what tools
to prefer, what patterns to avoid, and how to interact with a particular codebase.

> **Source:** `src/utils/claudemd.ts`

The instructions are loaded once at session start (via memoized `getMemoryFiles()` at line
790), assembled into a single prompt string by `getClaudeMds()` (line 1153), and injected
into the dynamic region of the system prompt. The model is told explicitly:

```
Codebase and user instructions are shown below. Be sure to adhere to these instructions.
IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them
exactly as written.
```

This framing gives CLAUDE.md content higher behavioral authority than the system prompt's
default guidelines — a deliberate design choice that lets project owners shape agent
behavior without modifying Claude Code itself.

The system supports six memory types, defined in `src/utils/memory/types.ts`:

| Type | Description |
|---|---|
| `Managed` | Enterprise/admin-controlled instructions (policy) |
| `User` | Personal global instructions across all projects |
| `Project` | Repository-level instructions (committed to VCS) |
| `Local` | Private project-specific overrides (gitignored) |
| `AutoMem` | Auto-extracted persistent memory (cross-session) |
| `TeamMem` | Shared team memory (feature-gated behind `TEAMMEM`) |

This document focuses on the first four — the "instruction hierarchy" proper. AutoMem and
TeamMem are covered in the companion context management design spec.

---

## 2. Four-Level Priority Hierarchy

> **Source:** `src/utils/claudemd.ts:1-26` (module-level docstring)

Files are loaded in a strict four-level priority order, from lowest to highest:

```
Priority (lowest → highest, model pays more attention to later entries):

  Level 1: Managed    /etc/claude-code/CLAUDE.md              ← enterprise policy
  Level 2: User       ~/.claude/CLAUDE.md                     ← personal global preferences
  Level 3: Project    CLAUDE.md + .claude/CLAUDE.md           ← repo-level instructions
                      + .claude/rules/*.md
  Level 4: Local      CLAUDE.local.md                         ← private project overrides
```

The ordering is enforced by `getMemoryFiles()` which processes sources in exactly this
sequence: Managed first, then User, then Project (from root toward CWD), then Local.
Because the assembled prompt preserves insertion order and the model tends to weight later
content more heavily, higher-priority files effectively override lower-priority ones when
instructions conflict.

### 2.1 Level 1: Managed (Enterprise Policy)

**Path:** Platform-dependent managed directory + `CLAUDE.md`

| Platform | Base Path |
|---|---|
| macOS | `/Library/Application Support/ClaudeCode/` |
| Linux | `/etc/claude-code/` |
| Windows | `C:\Program Files\ClaudeCode\` |

> **Source:** `src/utils/settings/managedPath.ts`

Managed instructions are controlled by system administrators and apply globally to all
users on a machine. They are loaded unconditionally — the `policySettings` source is
always enabled regardless of user or project configuration (`src/utils/settings/constants.ts:164`).
This level also supports a rules directory at `<managed-path>/.claude/rules/*.md`.

### 2.2 Level 2: User (Personal Global)

**Path:** `~/.claude/CLAUDE.md` (plus `~/.claude/rules/*.md`)

User instructions express personal preferences that apply across all projects: preferred
coding style, tool preferences, language settings. This level is gated by the
`userSettings` source — it can be disabled via `--setting-sources` but is enabled by
default.

User memory files have a special privilege: they are **always allowed to include external
files** via `@path` syntax (`src/utils/claudemd.ts:835`, `includeExternal: true`). This
is because the user explicitly authored these files and should be trusted to reference
their own filesystem.

### 2.3 Level 3: Project (Repository-Level)

**Paths:** `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md` — at every directory
from root down to CWD.

Project instructions are the most common level. They are checked into version control and
shared with the entire team. The system looks for three types of files at each directory:

1. `CLAUDE.md` — top-level project instructions
2. `.claude/CLAUDE.md` — nested under the `.claude` directory (equivalent)
3. `.claude/rules/*.md` — modular rule files (see section 9)

Gated by the `projectSettings` source. For external `@path` includes, project files
require explicit user approval via the external includes dialog
(`src/components/ClaudeMdExternalIncludesDialog.tsx`).

### 2.4 Level 4: Local (Private Project Overrides)

**Path:** `CLAUDE.local.md` at each directory from root to CWD.

Local instructions override everything else. They are intended for private, per-developer
customization that should not be committed to version control. A typical `.gitignore`
would include `CLAUDE.local.md`.

Gated by the `localSettings` source.

---

## 3. File Discovery

> **Source:** `src/utils/claudemd.ts:850-934`

### 3.1 The Directory Walk

Project and Local files are discovered by walking the directory hierarchy from the
current working directory (`getOriginalCwd()`) upward to the filesystem root:

```typescript
let currentDir = originalCwd
while (currentDir !== parse(currentDir).root) {
  dirs.push(currentDir)
  currentDir = dirname(currentDir)
}
```

The collected directories are then reversed so processing happens **from root downward
toward CWD**. This means files closer to the CWD are loaded later, giving them higher
effective priority:

```
/home/user/projects/                   ← loaded first (lowest priority)
/home/user/projects/myapp/             ← loaded second
/home/user/projects/myapp/packages/    ← loaded third (CWD, highest priority)
```

At each directory, the system attempts to read (in order):
1. `<dir>/CLAUDE.md` (Project)
2. `<dir>/.claude/CLAUDE.md` (Project)
3. `<dir>/.claude/rules/*.md` (Project, recursive into subdirectories)
4. `<dir>/CLAUDE.local.md` (Local)

All file reads are attempted via `safelyReadMemoryFileAsync()` which catches `ENOENT`
(file not found) and `EISDIR` (directory, not file) silently — the walk is speculative.

### 3.2 Git Worktree Deduplication

> **Source:** `src/utils/claudemd.ts:860-884`

When Claude Code runs inside a git worktree nested within its main repository (as with
`claude -w` which creates worktrees under `.claude/worktrees/<name>/`), the upward walk
passes through both the worktree root and the main repo root. Both directories contain
the same checked-in files (CLAUDE.md, .claude/rules/), which would cause duplicate
loading.

The system detects this situation by comparing `findGitRoot()` (the worktree root) with
`findCanonicalGitRoot()` (the main repo root). If they differ and the worktree is nested
inside the main repo, Project-type files from directories above the worktree but within
the main repo are skipped. `CLAUDE.local.md` is still loaded from those directories
because it is gitignored and only exists in the main repo.

### 3.3 Additional Directories (--add-dir)

> **Source:** `src/utils/claudemd.ts:936-977`

When `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` is enabled and `--add-dir` was used,
CLAUDE.md files from the additional directories are also loaded. These follow the same
three-file pattern (CLAUDE.md, .claude/CLAUDE.md, .claude/rules/) but are scoped to the
single specified directory rather than walking upward.

This is gated behind an environment variable because it is an explicit user action — the
SDK defaults `settingSources` to `[]` when not specified, which would normally suppress
project settings, but `--add-dir` should still be honored since the user explicitly
asked for it.

### 3.4 Bare Mode and Disable Flags

> **Source:** `src/context.ts:162-167`

Two mechanisms suppress CLAUDE.md loading entirely:

- **`CLAUDE_CODE_DISABLE_CLAUDE_MDS`**: Hard off. All CLAUDE.md content is suppressed.
- **`--bare` mode**: Skips auto-discovery (the directory walk), but honors explicit
  `--add-dir` directories. The rationale: `--bare` means "skip what I didn't ask for",
  not "ignore what I explicitly asked for."

---

## 4. @path Syntax — Composing Instructions from Multiple Sources

> **Source:** `src/utils/claudemd.ts:451-535`

CLAUDE.md files can reference other files using the `@path` directive. This allows
modular composition of instructions from multiple sources without duplicating content.

### 4.1 Syntax

```markdown
@path/to/file.md
@./relative/path.md
@~/home-relative/path.md
@/absolute/path.md
@path/with\ spaces/file.md
@path/to/file.md#heading        (fragment stripped, file included whole)
```

The `@` sigil must appear at the start of a word boundary (preceded by whitespace or
start-of-line). Paths without a prefix (`@path`) are treated as relative to the
including file's directory — equivalent to `@./path`.

### 4.2 Path Resolution

Paths are resolved via `expandPath()` using the including file's directory as the base
for relative paths. Three forms are recognized:

| Prefix | Resolution |
|---|---|
| `./` or bare | Relative to the including file's directory |
| `~/` | Relative to the user's home directory |
| `/` | Absolute path |

Fragment identifiers (`#heading`) are stripped before resolution — the entire referenced
file is included regardless of any fragment.

### 4.3 Parsing

Include paths are extracted during markdown lexing (`extractIncludePathsFromTokens()`).
The parser uses the `marked` lexer with `gfm: false` to prevent `~/path` from being
tokenized as strikethrough markup. The extraction:

- **Processes only text nodes** — `@` references inside code blocks (`code`, `codespan`)
  and raw HTML tags are ignored
- **Handles HTML comments specially** — strips comment spans and checks residual text for
  `@` references (so `<!-- note --> @./file.md` correctly includes `file.md`)
- **Recurses into list items** — `items` arrays on list tokens are walked
- **Supports escaped spaces** — `\\ ` in paths is unescaped to a literal space

### 4.4 Text-File Allowlist

> **Source:** `src/utils/claudemd.ts:96-227`

To prevent binary files (images, PDFs, compiled artifacts) from being loaded into the
prompt, included files are checked against a comprehensive allowlist of ~100 text file
extensions. The `TEXT_FILE_EXTENSIONS` set covers:

- Markdown, text, and documentation formats (`.md`, `.txt`, `.rst`, `.adoc`)
- Data formats (`.json`, `.yaml`, `.toml`, `.xml`, `.csv`)
- Source code in ~30 languages (`.js`, `.ts`, `.py`, `.go`, `.rs`, `.java`, etc.)
- Configuration files (`.env`, `.ini`, `.conf`, `.properties`)
- Build files (`.cmake`, `.gradle`, `.sbt`)

Files with extensions not in this set are silently skipped with a debug log message.

### 4.5 External Include Approval

> **Source:** `src/utils/claudemd.ts:1399-1430`, `src/components/ClaudeMdExternalIncludesDialog.tsx`

When a Project-level CLAUDE.md includes files outside the current working directory,
the system triggers an approval dialog. This prevents a cloned repository from silently
pulling in files from arbitrary locations on the user's machine.

The approval state is tracked in the project config:
- `hasClaudeMdExternalIncludesApproved` — user granted permission
- `hasClaudeMdExternalIncludesWarningShown` — dialog was already presented

User-level files (`~/.claude/CLAUDE.md`) bypass this check entirely — they always have
external include permission.

---

## 5. Circular Reference Prevention

> **Source:** `src/utils/claudemd.ts:618-685`

The `processMemoryFile()` function maintains a `processedPaths: Set<string>` that
accumulates every file path encountered during the recursive inclusion walk. Before
processing a file, its path is checked against this set; if already present, the file
is skipped.

```typescript
const normalizedPath = normalizePathForComparison(filePath)
if (processedPaths.has(normalizedPath) || depth >= MAX_INCLUDE_DEPTH) {
  return []
}
processedPaths.add(normalizedPath)
```

Two complementary guards prevent unbounded recursion:

1. **Path deduplication** — the `processedPaths` set. Paths are normalized for
   comparison to handle Windows drive letter casing differences (`C:\Users` vs
   `c:\Users`). Symlinks are resolved via `safeResolvePath()` and both the original
   and resolved paths are added to the set, preventing cycles through symlinked
   directories.

2. **Depth limit** — `MAX_INCLUDE_DEPTH = 5`. Even without cycles, deeply nested
   includes are capped. This bounds the total I/O cost of the inclusion walk.

The `processedPaths` set is shared across all files at a given level — if a User-level
file includes `@./shared.md`, and a Project-level file also includes `@./shared.md`
resolving to the same absolute path, the file is loaded only once.

### 5.1 Symlink Cycle Detection in Rules Directories

> **Source:** `src/utils/claudemd.ts:697-788`

The `processMdRules()` function for `.claude/rules/` directories maintains a separate
`visitedDirs: Set<string>` to detect directory-level cycles via symlinks. Before
descending into a subdirectory, its path is checked (both original and resolved) against
this set. This prevents infinite directory traversal when a rules directory contains
a symlink that points back to an ancestor.

---

## 6. Injection Filtering and Content Sanitization

Content undergoes multiple transformations between disk and prompt injection.

### 6.1 HTML Comment Stripping

> **Source:** `src/utils/claudemd.ts:292-334`

Block-level HTML comments (`<!-- ... -->`) are stripped from CLAUDE.md content before
injection. This allows file authors to include private notes that are not sent to the
model.

The stripping uses the `marked` lexer to identify `html`-type tokens at the block level
only:

- Comments inside code blocks and inline code spans are preserved
- Comments inside paragraphs (inline HTML) are left intact
- Unclosed comments (`<!--` without `-->`) are left in place to avoid silently swallowing
  content after a typo

When a comment is stripped, any residual content on the same line is preserved. For
example, `<!-- note --> Use bun` keeps `Use bun`.

### 6.2 Frontmatter Stripping

> **Source:** `src/utils/frontmatterParser.ts`, `src/utils/claudemd.ts:254-279`

YAML frontmatter (delimited by `---`) is parsed and removed from content before prompt
injection. The frontmatter can contain metadata such as:

- `paths` — glob patterns controlling when a rule applies (see section 9)
- `description` — human-readable description of the rule
- `allowed-tools` — tool access declarations

The frontmatter is parsed via a YAML parser with a fallback: if initial parsing fails
(common with glob patterns containing special YAML characters like `{`, `*`), the system
retries after auto-quoting problematic values.

### 6.3 Path-Based Exclusion

> **Source:** `src/utils/claudemd.ts:540-612`

The `claudeMdExcludes` setting allows users to specify glob patterns for CLAUDE.md files
that should be skipped entirely. This applies to User, Project, and Local types —
Managed and AutoMem/TeamMem files are never excluded.

Exclusion patterns support picomatch glob syntax with dot-file matching. To handle macOS
symlinks (where `/tmp` resolves to `/private/tmp`), the system expands absolute patterns
by resolving symlinks in their static path prefixes and matching against both the original
and resolved forms.

### 6.4 Feature-Gated Content Filtering

> **Source:** `src/utils/claudemd.ts:1142-1151`

`filterInjectedMemoryFiles()` provides an additional filtering layer. When the
`tengu_moth_copse` feature flag is enabled, AutoMem and TeamMem entries are excluded from
the system prompt injection (they are instead surfaced via attachments by the prefetch
system). This function is called by `getUserContext()` before assembling the prompt string.

### 6.5 Project-Level Skipping

> **Source:** `src/utils/claudemd.ts:1158-1161`

When the `tengu_paper_halyard` feature flag is enabled, Project and Local type files are
skipped entirely during prompt assembly in `getClaudeMds()`. This is a runtime kill
switch for the project instruction system.

---

## 7. Enterprise Managed Instructions

> **Source:** `src/utils/settings/managedPath.ts`, `src/utils/settings/constants.ts`,
> `src/utils/settings/settings.ts`

### 7.1 policySettings Source

The `policySettings` source is architecturally privileged:

- **Always enabled** — `getEnabledSettingSources()` unconditionally adds `policySettings`
  to the enabled set, regardless of the `--setting-sources` flag
  (`src/utils/settings/constants.ts:164`)
- **Read-only** — excluded from `EditableSettingSource` type
  (`src/utils/settings/constants.ts:182-184`)
- **First-source-wins** — when multiple policy sources exist (remote API, HKLM/plist,
  file, HKCU), the highest-priority non-null source wins
  (`src/utils/settings/settings.ts:322-323`)

### 7.2 Policy Source Cascade

For `policySettings`, the system checks four sources in priority order:

1. **Remote managed settings** — fetched from the API via
   `getRemoteManagedSettingsSyncFromCache()`. This allows organizations to push
   configuration updates without touching each machine.
2. **MDM/system policy** — read from OS-level management frameworks (HKLM on Windows,
   managed preferences on macOS)
3. **Managed settings file** — `<managed-path>/managed-settings.json` with drop-in
   directory `<managed-path>/managed-settings.d/*.json`
4. **HKCU (Windows)** — current-user registry settings

The first non-null source in this cascade is used as the policy settings.

### 7.3 Managed CLAUDE.md and Rules

Enterprise administrators can deploy:

- `<managed-path>/CLAUDE.md` — global instructions for all users on the machine
- `<managed-path>/.claude/rules/*.md` — modular rule files

These follow the same format and processing pipeline as user and project files but
cannot be overridden by users — they are loaded first and the model is instructed to
follow all levels of instructions, with later (higher-priority) content taking precedence
when there are conflicts.

### 7.4 Disabling Instruction Sources

Managed settings can restrict which instruction sources are loaded. The
`settingSources` setting (typically set via policy) controls whether `userSettings`,
`projectSettings`, and `localSettings` are enabled. This allows enterprises to lock
down the instruction pipeline — for example, disabling `projectSettings` to prevent
cloned repositories from influencing agent behavior.

---

## 8. Project vs Local

The distinction between Project (`.claude/CLAUDE.md`) and Local (`CLAUDE.local.md`)
files is a core architectural decision that mirrors the split between committed and
gitignored configuration in tools like `.env` vs `.env.local`.

### 8.1 Project Instructions (.claude/CLAUDE.md)

- **Committed to version control** — shared with all developers working on the repository
- **Typed as `'Project'`** in the memory system
- **Description in prompt**: "(project instructions, checked into the codebase)"
- **Multiple file locations**: both `CLAUDE.md` at the project root and `.claude/CLAUDE.md`
  are supported; both are loaded as Project type
- **External includes require approval** — a dialog is shown on first use
- **Subject to `projectSettings` enable/disable** — can be suppressed by
  `--setting-sources` or managed policy

### 8.2 Local Instructions (CLAUDE.local.md)

- **Not committed** — should be listed in `.gitignore`
- **Typed as `'Local'`** in the memory system
- **Description in prompt**: "(user's private project instructions, not checked in)"
- **Highest priority** — loaded last at each directory level, after Project files
- **Subject to `localSettings` enable/disable**
- **Use cases**: developer-specific tool paths, personal API endpoints, experimental
  instructions that should not affect the team

### 8.3 Prompt Differentiation

When CLAUDE.md content is assembled into the prompt by `getClaudeMds()`, each file is
annotated with its source type and a human-readable description:

```
Contents of /path/to/.claude/CLAUDE.md (project instructions, checked into the codebase):

<content>

Contents of /path/to/CLAUDE.local.md (user's private project instructions, not checked in):

<content>
```

This labeling helps the model understand the provenance and relative authority of
conflicting instructions.

---

## 9. Rules Directory

> **Source:** `src/utils/claudemd.ts:697-788`, `src/utils/claudemd.ts:1344-1397`

### 9.1 Structure

The `.claude/rules/` directory enables modular, file-per-concern instruction organization:

```
.claude/
├── CLAUDE.md                        ← monolithic project instructions
└── rules/
    ├── coding-style.md              ← unconditional rule
    ├── testing.md                   ← unconditional rule
    ├── typescript-conventions.md    ← conditional rule (paths: **/*.ts)
    └── api/
        └── rest-guidelines.md       ← unconditional rule (subdirectories supported)
```

Rules directories exist at three levels:
- `<managed-path>/.claude/rules/` — enterprise rules
- `~/.claude/rules/` — user-global rules
- `<project-dir>/.claude/rules/` — project rules (at each directory in the CWD walk)

### 9.2 Unconditional vs Conditional Rules

Rules are divided into two categories based on frontmatter:

**Unconditional rules** — no `paths` frontmatter, or `paths: **` (match-all). These are
always loaded, regardless of what files the model is working with. They are loaded eagerly
at session start as part of the standard `getMemoryFiles()` flow.

**Conditional rules** — have `paths` frontmatter specifying glob patterns:

```yaml
---
paths: src/**/*.ts, src/**/*.tsx
---
Always use `interface` instead of `type` for object shapes.
```

Conditional rules are loaded **lazily** — only when the model touches a file matching
the glob patterns. The matching logic (`processConditionedMdRules()` at line 1354) uses
the `ignore` library for gitignore-style pattern matching:

- For Project rules: glob patterns are relative to the directory containing `.claude/`
- For Managed/User rules: glob patterns are relative to the original CWD

### 9.3 Frontmatter Path Parsing

> **Source:** `src/utils/frontmatterParser.ts:189-266`

The `paths` field supports:
- Comma-separated strings: `paths: src/*.ts, lib/*.ts`
- YAML lists: `paths: ["src/*.ts", "lib/*.ts"]`
- Brace expansion: `paths: src/*.{ts,tsx}` expands to `src/*.ts` and `src/*.tsx`
- Nested brace expansion: `{a,b}/{c,d}` expands to `a/c`, `a/d`, `b/c`, `b/d`
- Commas inside braces are not treated as separators

The `/**` suffix is automatically stripped because the `ignore` library already treats
a path as matching both itself and everything inside it.

### 9.4 Recursive Directory Processing

`processMdRules()` recursively descends into subdirectories within the rules directory.
Symlinks are resolved and tracked via `visitedDirs` to prevent cycles. Only files with
`.md` extensions are processed. Non-directory, non-`.md` entries are silently skipped.

---

## 10. Integration with Context Assembly

> **Source:** `src/context.ts`, `src/constants/prompts.ts`

### 10.1 The Assembly Pipeline

CLAUDE.md content enters the system prompt through a pipeline of memoized functions:

```
getMemoryFiles()                    → MemoryFileInfo[]
  ↓
filterInjectedMemoryFiles()         → MemoryFileInfo[] (feature-gated filtering)
  ↓
getClaudeMds()                      → string (formatted prompt text)
  ↓
getUserContext()                     → { claudeMd: string, currentDate: string }
  ↓
getSystemPrompt() dynamic region    → system prompt array element
```

### 10.2 Placement in the System Prompt

The system prompt has a critical architectural split at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`
(`src/constants/prompts.ts:114`). CLAUDE.md content is placed in the **dynamic region**
(after the boundary), meaning it is NOT shared across organizations for prompt caching.
This is correct — CLAUDE.md content is user/project-specific.

The assembled CLAUDE.md content appears in the system prompt as a `claudeMd` key within
a `<system-reminder>` tag. The surrounding framing is:

```xml
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
Codebase and user instructions are shown below. Be sure to adhere to these
instructions. IMPORTANT: These instructions OVERRIDE any default behavior and
you MUST follow them exactly as written.

Contents of ~/.claude/CLAUDE.md (user's private global instructions for all projects):
...

Contents of /project/.claude/CLAUDE.md (project instructions, checked into the codebase):
...
</system-reminder>
```

### 10.3 Memoization and Cache Invalidation

`getMemoryFiles()` is memoized via lodash's `memoize()` — it computes once per session
and returns the cached result for subsequent calls. The cache can be invalidated through
two paths:

- **`clearMemoryFileCaches()`** — clears the memoize cache without firing the
  `InstructionsLoaded` hook. Used for correctness-only invalidation (worktree
  enter/exit, settings sync, `/memory` dialog).
- **`resetGetMemoryFilesCache(reason)`** — clears the cache AND arms the
  `InstructionsLoaded` hook to fire on the next load. Used when instructions are
  genuinely being reloaded into context (e.g., after compaction).

### 10.4 Classifier Cache

> **Source:** `src/context.ts:176`

The assembled CLAUDE.md string is cached separately via `setCachedClaudeMdContent()` for
the auto-mode classifier (`yoloClassifier.ts`). This avoids an import cycle:
`permissions/filesystem → permissions → yoloClassifier → claudemd.ts` would create a
circular dependency. The cached string lets the classifier read CLAUDE.md content without
importing the claudemd module.

### 10.5 InstructionsLoaded Hook

> **Source:** `src/utils/claudemd.ts:1042-1071`

When files are loaded (and the hook is armed), the system fires `InstructionsLoaded`
hook events for each instruction file. These hooks support audit and observability use
cases — enterprise deployments can monitor which instructions are being loaded, from
where, and why. The hook receives:

- File path
- Memory type (User, Project, Local, Managed)
- Load reason (`session_start`, `compact`, `include`)
- Metadata (glob patterns, parent file path for includes)

---

## 11. Security Considerations

### 11.1 Threat Model

The primary threat is a **malicious repository** shipping adversarial CLAUDE.md or
`.claude/rules/*.md` files designed to manipulate the agent. Attack vectors include:

1. **Prompt injection** — instructions that override safety guidelines or manipulate
   the model into taking harmful actions
2. **Data exfiltration** — `@path` includes that reference sensitive files outside the
   project (`~/.ssh/id_rsa`, `~/.aws/credentials`)
3. **Symlink attacks** — symlinks in `.claude/rules/` that point to sensitive directories,
   causing the recursive walk to read unintended content
4. **Path traversal** — `@../../etc/passwd` style includes
5. **Binary injection** — including large binary files to overflow or confuse the context

### 11.2 Defenses

**External include approval** (`src/components/ClaudeMdExternalIncludesDialog.tsx`):
Project-level `@path` directives that resolve outside the CWD trigger an explicit user
approval dialog. The dialog warns: "Never allow this for third-party repositories."
Approval state is per-project and persists in the project config.

**Text-file allowlist** (`TEXT_FILE_EXTENSIONS`): Only files with recognized text
extensions can be included. Binary files (images, PDFs, compiled output) are silently
rejected. The allowlist covers ~100 extensions and is intentionally conservative.

**Depth limit** (`MAX_INCLUDE_DEPTH = 5`): Prevents unbounded recursion even in the
absence of cycles. Bounds the total I/O cost of inclusion.

**Symlink resolution and cycle detection**: Both file-level (`processedPaths`) and
directory-level (`visitedDirs`) cycle detection use resolved symlink paths.
`safeResolvePath()` handles the resolution, and both original and resolved paths are
tracked.

**Path normalization**: Paths are normalized for comparison via
`normalizePathForComparison()` to handle platform-specific case sensitivity (Windows
drive letters) and prevent duplicate loading through casing tricks.

**Setting source control**: Enterprise administrators can disable `projectSettings`
entirely via managed policy, preventing any repository from influencing agent behavior
through committed files.

**Exclusion patterns** (`claudeMdExcludes`): Users can configure glob patterns to skip
specific CLAUDE.md files. This is a user-controlled safety valve.

**Content size warnings**: Files exceeding `MAX_MEMORY_CHARACTER_COUNT` (40,000
characters) trigger a context warning in the `/doctor` diagnostic
(`src/utils/doctorContextWarnings.ts`), alerting users to oversized instruction files
that may degrade performance.

### 11.3 Trust Boundaries

The system recognizes three trust levels:

| Level | Files | Include Policy | Can Disable? |
|---|---|---|---|
| Administrator | Managed (policy) | Always external | No |
| User | User (~/.claude/) | Always external | Via --setting-sources |
| Repository | Project + Local | External requires approval | Via --setting-sources or policy |

The critical security invariant: **untrusted repository content never automatically
reaches outside the project directory**. External includes from Project files always
require explicit user approval, and that approval is never implicitly granted.

---

## 12. Design Principles

### 12.1 Layered Override

The four-level hierarchy follows the principle of increasing specificity: enterprise
policy provides guardrails, user preferences set defaults, project instructions
customize for the codebase, and local overrides handle developer-specific needs. Each
layer can refine or override the ones below it, but cannot suppress the ones above it
(managed instructions are always loaded).

### 12.2 Convention over Configuration

The system discovers instruction files automatically through naming conventions and
directory structure. Users do not need to register files or configure paths — placing a
`CLAUDE.md` in the right location is sufficient. The directory walk, rules directory
scanning, and frontmatter parsing all follow predictable conventions.

### 12.3 Security by Default, Flexibility by Opt-In

The default posture is restrictive: external includes are blocked, binary files are
rejected, depth is limited, and project instructions can be entirely suppressed. Each
relaxation (external include approval, additional directories, broader setting sources)
requires an explicit user action.

### 12.4 Composition over Monoliths

The `@path` syntax and `.claude/rules/` directory enable instruction composition from
multiple focused files rather than requiring a single monolithic CLAUDE.md. Conditional
rules with `paths` frontmatter take this further — instructions can be scoped to specific
file types or directories, keeping the effective prompt lean and relevant.

### 12.5 Transparent Provenance

Every instruction file injected into the prompt is annotated with its full path and type
description. The model — and through the `/context` command, the user — can see exactly
which instructions are active and where they came from. This traceability is essential
for debugging unexpected agent behavior.

### 12.6 Graceful Degradation

Missing files produce no errors — `ENOENT` is silently handled. Malformed frontmatter
falls back to no-frontmatter parsing. Invalid YAML retries with auto-quoting. Symlink
resolution failures are caught. The system is designed to work in the common case
(files exist and are well-formed) and degrade quietly in edge cases, rather than
failing loudly on optional configuration.

### 12.7 Enterprise Controllability

The managed instruction level, combined with the `policySettings` source that cannot be
disabled by users, gives enterprises a reliable enforcement point. Administrators can
deploy instructions that are guaranteed to be loaded on every session, and can restrict
which other instruction sources are active. The remote managed settings sync enables
fleet-wide updates without per-machine access.

### 12.8 Cache Awareness

CLAUDE.md content is deliberately placed after the `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` in
the system prompt, ensuring that instruction changes do not invalidate the cross-org
prompt cache for static content (tool descriptions, safety guidelines). The memoization
of `getMemoryFiles()` and the classifier cache avoid redundant computation within a
session. The `InstructionsLoaded` hook fires at controlled points (session start,
post-compaction) rather than on every access.
