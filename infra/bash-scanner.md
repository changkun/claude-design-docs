# Claude Code: Bash Security Scanner Design

This document describes the **Bash Security Scanner** -- the deterministic injection
detection system that forms Layer 5 of Claude Code's Swiss cheese oversight
architecture. It analyzes every shell command before execution, detects 23 categories
of injection and obfuscation attacks, and feeds results into the permission pipeline
to decide whether a command should be auto-allowed, prompted to the user, or denied.

The document is split into two parts: **Part I** describes the architectural design
and threat model, and **Part II** provides implementation details with source code
references.

---

# Part I: System Design

## 1. Overview

Claude Code executes bash commands on behalf of an AI model. The model generates
command strings that are passed to the user's default shell (bash or zsh). This
creates an adversarial boundary: if the model is manipulated via prompt injection,
the command string itself becomes the attack vector. The Bash Security Scanner is
the deterministic gatekeeper that sits between the model's tool call and the shell
execution, analyzing the raw command text for structural properties that indicate
injection, obfuscation, or privilege escalation.

The scanner operates in the permission pipeline after user-defined rules (Layer 3)
and tool-specific domain checks (Layer 4, e.g. path validation), but before the
sandbox (Layer 7) and any AI classifier (Layer 10). Its key design property is that
it is **purely deterministic** -- no model calls, no probabilistic reasoning. It
applies structural analysis and pattern matching to the command text and produces one
of three outcomes:

- **Passthrough** -- the command passed all 23 checks; continue to downstream layers.
- **Ask** -- at least one check flagged a concern; prompt the user for approval.
- **Allow** -- an early validator proved the command is structurally safe (e.g. a
  simple `git commit -m "message"` with no metacharacters).

The scanner never produces a **deny** outcome on its own. Its purpose is to force
human review when deterministic analysis detects structural ambiguity, not to block
commands outright. This design choice reflects the principle that deterministic
scanners are good at detecting suspicious structure but poor at judging intent --
the human (or AI classifier in auto mode) makes the final call.

## 2. Threat Model

The scanner defends against two primary threat classes:

**Parser differential attacks.** Claude Code uses multiple parsers internally
(tree-sitter, shell-quote, regex) and the actual shell (bash/zsh) uses yet another
parser. When these parsers disagree on the structure of a command, an attacker can
craft input that one parser considers safe but the shell executes differently. The
canonical example: `echo\ test/../../../usr/bin/touch /tmp/file` -- shell-quote
splits this into two tokens (`echo` and `test/...`), but bash treats `echo\ test`
as a single token (a command named "echo test"), resolving the path traversal to
execute `touch`.

**Obfuscation attacks.** Even when parsers agree on structure, shell quoting rules
allow the same command to be expressed in many equivalent forms. An attacker can use
ANSI-C quoting (`$'\x2d'exec`), empty quote concatenation (`"""-f"`), brace
expansion (`{--upload-pack="evil",test}`), or Unicode whitespace to disguise
dangerous flags or command names so they evade pattern-matching permission rules
while bash still executes the intended payload.

The scanner also provides defense-in-depth against:

- **Zsh-specific attack surface** (module loading, socket primitives, eval equivalents)
- **/proc/environ access** (environment variable exfiltration)
- **IFS injection** (field separator manipulation to split arguments)
- **Heredoc abuse** (hiding commands in here-document bodies)
- **Comment-quote desynchronization** (using `#` comments to confuse quote tracking)

## 3. Architecture: Dual-Path Analysis

The scanner uses a **dual-path architecture** with a tree-sitter AST parser as the
primary analysis engine and a regex/shell-quote fallback for environments where
tree-sitter is unavailable.

```
              ┌─────────────┐
              │ Raw Command  │
              └──────┬───────┘
                     │
              ┌──────▼───────┐
              │ Pre-checks   │  Control chars, Unicode WS, backslash-WS
              │ (both paths) │  These run BEFORE any parser
              └──────┬───────┘
                     │
         ┌───────────┴───────────┐
         │                       │
  ┌──────▼───────┐       ┌──────▼───────┐
  │ tree-sitter  │       │ Regex/shell- │
  │ AST path     │       │ quote path   │
  │ (primary)    │       │ (fallback)   │
  └──────┬───────┘       └──────┬───────┘
         │                       │
         │  AST walk:            │  23 regex validators
         │  parseForSecurity()   │  bashCommandIsSafe_DEPRECATED()
         │  checkSemantics()     │
         │                       │
         └───────────┬───────────┘
                     │
              ┌──────▼───────┐
              │ Permission   │
              │ Result       │
              │ (allow/ask/  │
              │  passthrough)│
              └──────────────┘
```

### 3.1 Tree-Sitter AST Path (Primary)

The primary path uses a pure-TypeScript bash parser (`bashParser.ts`) that produces
tree-sitter-bash-compatible ASTs. This parser was validated against a 3449-input
golden corpus generated from the WASM parser. It operates with a 50ms wall-clock
timeout and a 50,000-node budget to prevent adversarial inputs from causing
denial-of-service.

The AST path provides two levels of analysis:

1. **`parseForSecurity()`** (`ast.ts`) -- walks the AST with an explicit allowlist
   of node types. Any node type not in the allowlist causes the entire command to be
   classified as `too-complex`, which means the command goes through the normal
   permission prompt flow. This is a **fail-closed** design: the system never
   interprets structure it does not understand.

2. **`checkSemantics()`** (`ast.ts`) -- runs after `parseForSecurity()` returns
   `simple`. Performs semantic checks on the extracted `argv[]` arrays: detects
   eval-like builtins, dangerous flags (e.g. `-exec` on `find`), Zsh dangerous
   commands, and `/proc/environ` access.

The AST path also provides enriched context to the regex validators via the
`TreeSitterAnalysis` type, which includes:

- **QuoteContext** -- extracted quoted/unquoted content computed from AST node spans
  rather than character-by-character regex tracking. This eliminates an entire class
  of quote-tracking desync bugs.
- **CompoundStructure** -- whether the command has compound operators (`&&`, `||`, `;`),
  pipelines, subshells, or command groups, extracted from AST node types rather than
  regex pattern matching.
- **hasActualOperatorNodes** -- whether real operator nodes exist in the AST. This is
  the key function for eliminating the `find -exec \;` false positive: tree-sitter
  parses `\;` as part of a `word` node (an argument), NOT as a `;` operator.
- **DangerousPatterns** -- presence of command substitution, process substitution,
  parameter expansion, heredocs, and comments.

### 3.2 Regex/Shell-Quote Fallback Path

When tree-sitter is unavailable (external builds where the feature flag is off, or
when the parser times out), the scanner falls back to `bashCommandIsSafe_DEPRECATED()`.
This function runs the same 23 validator checks but uses:

- **`extractQuotedContent()`** -- a character-by-character state machine that tracks
  single/double quote state and backslash escaping to produce three derived strings:
  `withDoubleQuotes` (single-quoted content removed), `fullyUnquoted` (all quoted
  content removed), and `unquotedKeepQuoteChars` (content removed but delimiter
  characters preserved).

- **`extractHeredocs()`** -- strips heredoc bodies for quoted/escaped delimiters
  (where the body is literal text) before running validators.

- **`tryParseShellCommand()`** -- the shell-quote library for tokenization, used by
  `validateMalformedTokenInjection` to detect unbalanced delimiters.

The fallback path is known to have parser differential vulnerabilities that the AST
path does not. For this reason, ask results from the fallback path carry an
`isBashSecurityCheckForMisparsing: true` flag, which causes them to be blocked early
in the permission pipeline (before any rule matching that might use the same
vulnerable parser).

### 3.3 Parse Abort Handling

If the tree-sitter parser times out or exceeds its node budget, `parseCommandRaw()`
returns a `PARSE_ABORTED` sentinel (distinct from `null`, which means the module is
not loaded). Callers **must** treat this as fail-closed (`too-complex`), NOT route
it to the legacy regex path. This is critical because adversarial input can
intentionally trigger parser abort -- e.g. `(( a[0][0]... ))` with approximately
2800 subscripts hits the timeout -- and the legacy path lacks protections for
eval-like builtins that the AST path catches.

## 4. The 23 Injection Check Categories

Each check is identified by a numeric constant in `BASH_SECURITY_CHECK_IDS` for
analytics logging. The checks are organized into early validators (that can
short-circuit with allow), main validators (that flag concerns), and deferred
non-misparsing validators. The following is a comprehensive enumeration.

### Check 1: Incomplete Commands

**ID:** `INCOMPLETE_COMMANDS` (1)

Detects command fragments that look like continuation lines rather than complete
commands. Three sub-checks:

- Starts with a tab (likely an indented heredoc body or copy-paste artifact)
- Starts with a dash (looks like flags without a command name)
- Starts with a shell operator (`&&`, `||`, `;`, `>>`, `<`)

These patterns suggest the model generated a fragment rather than a full command,
which could be concatenated with attacker-controlled context.

### Check 2: jq system() Function

**ID:** `JQ_SYSTEM_FUNCTION` (2)

Detects `system()` calls inside jq expressions. The jq `system()` function
executes arbitrary shell commands, making it an eval equivalent. Also flags
dangerous jq flags (`-f`, `--from-file`, `--rawfile`, `--slurpfile`, `-L`,
`--library-path`) that could read arbitrary files.

### Check 3: jq File Arguments

**ID:** `JQ_FILE_ARGUMENTS` (3)

Flags jq invocations with flags that can load external files or libraries,
which could be used to exfiltrate data or execute code from attacker-controlled
file paths.

### Check 4: Obfuscated Flags

**ID:** `OBFUSCATED_FLAGS` (4)

A comprehensive multi-layer detection system for flag obfuscation via shell quoting.
Eleven sub-checks detect:

- **ANSI-C quoting** (`$'\x2d'exec`) -- can encode any character via escape sequences
- **Locale quoting** (`$"..."`) -- similar encoding capabilities
- **Empty quote + dash** (`$''-exec`, `$""-exec`, `''-flag`, `""-flag`)
- **Quote sequences before dash** (`''""- `) -- any sequence of empty quote pairs
  followed by a dash
- **Homogeneous empty pairs adjacent to quoted dash** (`"""-f"`) -- empty `""` plus
  quoted `"-f"` concatenates to `-f` in bash but evades all other checks
- **Three or more consecutive quotes at word start** (`"""x"`) -- safety net for
  multi-quote obfuscation not covered by specific patterns
- **Quoted characters inside flag names** (`"-"exec`, `'-'-output`) -- any quote
  character interrupting a flag name
- **Split-quote flags** (`"-"exec`) -- dashes inside quotes, letters continue after
- **Chained quotes** (`"-""exec"`) -- dashes in first quote, letters in second
- **Flag continuation characters** after quotes -- detects when unquoted characters
  after a closing quote complete a flag pattern

The check tracks quote state character-by-character with correct single-quote
backslash handling (backslash is literal inside single quotes) and skips `echo`
commands without shell operators, since echoing obfuscated text is harmless.

### Check 5: Shell Metacharacters

**ID:** `SHELL_METACHARACTERS` (5)

Detects shell metacharacters (`;`, `|`, `&`) inside quoted arguments in
unquoted content. This catches cases where metacharacters are embedded in
arguments to commands like `find -name`, `-path`, `-iname`, or `-regex`, where
they might be interpreted as operators if quote handling is inconsistent.

### Check 6: Dangerous Variables in Redirections/Pipes

**ID:** `DANGEROUS_VARIABLES` (6)

Detects shell variables (`$VAR`) adjacent to pipe (`|`) or redirection (`<`, `>`)
operators in fully unquoted content. Patterns like `> $FILE` or `$CMD | sink` allow
variable values to control file targets or command names, creating injection vectors
when variable values are attacker-influenced.

### Check 7: Newlines

**ID:** `NEWLINES` (7)

Detects newlines in unquoted content that could separate multiple commands. A
newline in bash is a command separator equivalent to `;`. The check uses the
pre-strip content (before `stripSafeRedirections`) to prevent bypasses where
stripping `>/dev/null` creates a phantom backslash-newline continuation.

Backslash-newline continuations at word boundaries (`cmd \<newline>--flag`) are
allowed because they are standard bash line continuation. Mid-word continuations
(`tr\<newline>aceroute`) are flagged because they can hide dangerous command
names from allowlist checks.

A separate sub-check detects carriage return (`\r`) outside double quotes. This
is a parser differential concern: shell-quote's `\s` regex includes `\r` (splitting
tokens at CR boundaries) while bash's IFS does not include CR (treating it as part
of the current word). This allows attacks like `TZ=UTC\recho curl evil.com` where
the validator sees `echo` but bash executes `curl`.

### Check 8: Command Substitution and Dangerous Patterns

**ID:** `DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION` (8)

Detects 13 forms of command/process/parameter substitution in unquoted content:

- Process substitution: `<()`, `>()`
- Zsh process substitution: `=()`
- Zsh equals expansion: `=cmd` at word start (expands to `$(which cmd)`)
- Command substitution: `$()`, backticks (with unescaped detection)
- Parameter substitution: `${}`
- Legacy arithmetic expansion: `$[]`
- Zsh parameter expansion: `~[`
- Zsh glob qualifiers with execution: `(e:`, `(+`
- Zsh always blocks: `} always {`
- PowerShell comment syntax: `<#` (defense-in-depth against future changes)

Backtick detection uses `hasUnescapedChar()` which correctly distinguishes escaped
backticks (`\`safe\``) from unescaped ones by tracking backslash parity.

### Check 9: Input Redirection

**ID:** `DANGEROUS_PATTERNS_INPUT_REDIRECTION` (9)

Detects `<` in fully unquoted content, which could read sensitive files.

### Check 10: Output Redirection

**ID:** `DANGEROUS_PATTERNS_OUTPUT_REDIRECTION` (10)

Detects `>` in fully unquoted content, which could write to arbitrary files.
The scanner strips safe redirections (`2>&1`, `>/dev/null`, `</dev/null`) before
this check, with trailing boundary assertions (`(?=\s|$)`) to prevent prefix
matching attacks (e.g. `> /dev/nullo` must not match `/dev/null`).

### Check 11: IFS Injection

**ID:** `IFS_INJECTION` (11)

Detects any usage of the `$IFS` variable or `${...IFS...}` parameter expansions.
IFS (Internal Field Separator) controls how bash splits words; manipulating it
can cause argument splitting at unexpected positions, bypassing regex validation
that assumes standard whitespace splitting.

### Check 12: Git Commit Substitution

**ID:** `GIT_COMMIT_SUBSTITUTION` (12)

Special-case validation for `git commit -m "message"` commands. Detects command
substitution patterns (`$()`, backticks, `${}`) inside double-quoted commit
messages, which would execute during shell expansion. Also validates that the
command remainder after the message does not contain shell operators that could
chain additional commands.

### Check 13: /proc/environ Access

**ID:** `PROC_ENVIRON_ACCESS` (13)

Detects paths matching `/proc/*/environ` in the raw command. The `/proc/<pid>/environ`
file exposes all environment variables of a process, potentially including API keys,
secrets, and tokens. While path validation typically blocks `/proc` access, this
provides defense-in-depth.

### Check 14: Malformed Token Injection

**ID:** `MALFORMED_TOKEN_INJECTION` (14)

Detects commands with malformed tokens (unbalanced delimiters) combined with
command separators. Discovered via HackerOne review: when shell-quote parses
ambiguous patterns like `echo {"hi":"hi;evil"}`, it may produce unbalanced tokens
(e.g. `{hi:"hi`). Combined with command separators, this can lead to unintended
command execution via eval re-parsing.

### Check 15: Backslash-Escaped Whitespace

**ID:** `BACKSLASH_ESCAPED_WHITESPACE` (15)

Detects `\ ` (backslash-space) or `\<tab>` outside of quotes. In bash, `echo\ test`
is a single token (command named "echo test"), but shell-quote decodes the escape and
produces two tokens (`echo`, `test`). This parser differential enables path traversal
attacks where the validator sees one command but bash resolves a different path.

### Check 16: Brace Expansion

**ID:** `BRACE_EXPANSION` (16)

Detects unquoted brace expansion syntax (`{a,b}` or `{1..5}`) that bash expands but
tree-sitter/shell-quote treat as literal strings. Three sub-checks:

- **Comma or sequence patterns** at the outermost nesting level inside `{...}`
- **Mismatched brace counts** after quote stripping (more closing than opening braces
  indicates a quoted `{` was stripped, enabling obfuscation like
  `git diff {@'{'0},--output=/tmp/pwned}`)
- **Quoted brace characters** (`'{'`, `"{"`) inside an unquoted brace context --
  the specific attack primitive for brace expansion obfuscation

The brace expansion check uses `isEscapedAtPosition()` to correctly handle
backslash-escaped braces and tracks nesting depth to find commas/sequences only
at the outer level.

### Check 17: Control Characters

**ID:** `CONTROL_CHARACTERS` (17)

Detects non-printable control characters (`0x00-0x08`, `0x0B-0x0C`, `0x0E-0x1F`,
`0x7F`) in the raw command. Bash silently drops null bytes and ignores most control
characters, so an attacker can use them to slip metacharacters past validators while
bash still executes the intended command (e.g. `echo safe\x00; rm -rf /`). This
check runs **first**, before any other processing, because control characters can
confuse all downstream validators.

### Check 18: Unicode Whitespace

**ID:** `UNICODE_WHITESPACE` (18)

Detects Unicode whitespace characters (`U+00A0` NBSP, `U+1680` Ogham space,
`U+2000-U+200A` various spaces, `U+2028` line separator, `U+2029` paragraph
separator, `U+202F` narrow NBSP, `U+205F` medium math space, `U+3000` ideographic
space, `U+FEFF` BOM/ZWNBSP) in the raw command. Shell-quote treats these as word
separators (JavaScript `\s` matches them) but bash treats them as literal word
content. While this differential is currently defense-favorable (shell-quote
over-splits), blocking these proactively prevents future edge cases.

### Check 19: Mid-Word Hash

**ID:** `MID_WORD_HASH` (19)

Detects `#` preceded by a non-whitespace character (mid-word hash) in unquoted
content with quote characters preserved. Shell-quote treats mid-word `#` as a
comment start but bash treats it as a literal character. This creates a parser
differential where shell-quote drops content after `#` but bash preserves it.

The check also processes continuation-joined content (joining `\<newline>` sequences)
because shell-quote operates on post-join text. Excludes `${#` which is bash
string-length syntax.

### Check 20: Zsh Dangerous Commands

**ID:** `ZSH_DANGEROUS_COMMANDS` (20)

Detects Zsh-specific commands that can bypass security checks. The dangerous
command set includes:

- **`zmodload`** -- the gateway to dangerous module-based attacks: `zsh/mapfile`
  (invisible file I/O), `zsh/system` (sysopen/syswrite), `zsh/zpty` (pseudo-terminal
  execution), `zsh/net/tcp` (network exfiltration), `zsh/files` (builtin rm/mv/ln/chmod)
- **`emulate`** -- with `-c` flag is an eval-equivalent
- **Module builtins** (defense-in-depth): `sysopen`, `sysread`, `syswrite`, `sysseek`,
  `zpty`, `ztcp`, `zsocket`, `mapfile`, `zf_rm`, `zf_mv`, `zf_ln`, `zf_chmod`,
  `zf_chown`, `zf_mkdir`, `zf_rmdir`, `zf_chgrp`
- **`fc -e`** -- executes arbitrary editors on command history (eval equivalent)

The check strips leading environment variable assignments and Zsh precommand
modifiers (`command`, `builtin`, `noglob`, `nocorrect`) before extracting the
base command name, preventing bypasses like `builtin zmodload zsh/system`.

### Check 21: Backslash-Escaped Operators

**ID:** `BACKSLASH_ESCAPED_OPERATORS` (21)

Detects a backslash immediately preceding a shell operator (`;`, `|`, `&`, `<`, `>`)
outside of quotes. The `splitCommand` normalization function converts `\;` to a bare
`;` in its output. When downstream code re-parses that normalized string, the bare
`;` becomes an operator, causing a false split. This enables attacks like:

```
cat safe.txt \; echo ~/.ssh/id_rsa
```

Bash executes ONE `cat` command reading all arguments as files. After `splitCommand`
normalizes: `"cat safe.txt ; echo ~/.ssh/id_rsa"`. Nested re-parse sees two segments
-- both pass `isCommandReadOnly`, and the sensitive path in the `echo` segment is
never validated by path constraints. Auto-allowed. Private key leaked.

When tree-sitter is available and confirms no actual operator nodes exist in the AST,
this check is skipped entirely -- this eliminates the `find . -exec cmd {} \;` false
positive.

### Check 22: Comment-Quote Desynchronization

**ID:** `COMMENT_QUOTE_DESYNC` (22)

Detects quote characters (`'`, `"`) inside `#` comments. In bash, everything after
an unquoted `#` is a comment -- quote characters are literal text. But the regex
quote-tracking functions do not handle comments, so a `'` or `"` after `#` toggles
their quote state. Attackers can craft `# ' "` sequences that desynchronize the
tracker, causing subsequent content (on following lines) to appear "inside quotes"
when it is actually unquoted in bash.

Example attack:
```
echo "it's" # ' " <<'MARKER'
rm -rf /
MARKER
```

When tree-sitter provides the quote context, this check is bypassed entirely --
the AST is authoritative regardless of comment content.

### Check 23: Quoted Newline Hiding

**ID:** `QUOTED_NEWLINE` (23)

Detects a newline inside a quoted string where the next line would be stripped by
`stripCommentLines` (trimmed line starts with `#`). The `stripCommentLines` function
processes commands line-by-line without tracking quote state. A quoted newline lets
an attacker position the next line to start with `#`, causing it to be dropped
entirely -- hiding sensitive paths from path validation and permission rule matching.

Example attack (auto-allowed in `acceptEdits` mode without any Bash rules):
```
mv ./decoy '<\n>#' ~/.ssh/id_rsa ./exfil_dir
```

Bash moves `./decoy` AND `~/.ssh/id_rsa` into `./exfil_dir/`, but
`stripCommentLines` drops line 2 (starts with `#`), and path constraints only see
`./decoy` (in cwd) -- passthrough. Zero clicks, no warning.

## 5. Validator Execution Order and Short-Circuit Logic

The validators are organized into three groups with specific execution semantics:

### Early Validators (can short-circuit with allow)

```
validateEmpty → validateIncompleteCommands → validateSafeCommandSubstitution → validateGitCommit
```

If any early validator returns `allow`, the scanner returns `passthrough` (the command
is structurally proven safe). If any returns `ask`, it is tagged with
`isBashSecurityCheckForMisparsing: true` and returned immediately.

### Main Validators (deferred non-misparsing logic)

The remaining validators run in order:

```
validateJqCommand → validateObfuscatedFlags → validateShellMetacharacters →
validateDangerousVariables → validateCommentQuoteDesync → validateQuotedNewline →
validateCarriageReturn → validateNewlines → validateIFSInjection →
validateProcEnvironAccess → validateDangerousPatterns → validateRedirections →
validateBackslashEscapedWhitespace → validateBackslashEscapedOperators →
validateUnicodeWhitespace → validateMidWordHash → validateBraceExpansion →
validateZshDangerousCommands → validateMalformedTokenInjection
```

Two validators (`validateNewlines` and `validateRedirections`) are classified as
**non-misparsing** -- their ask results go through the standard permission flow
rather than being blocked early, because LF newlines and redirections are normal
patterns that `splitCommand` handles correctly.

The scanner **does not short-circuit** when a non-misparsing validator returns `ask`
if there are still misparsing validators later in the list. Instead, the
non-misparsing ask result is **deferred**. If any subsequent misparsing validator
fires, its result (with the `isBashSecurityCheckForMisparsing` flag) takes priority.
Only if no misparsing validator fires does the deferred non-misparsing result get
returned. This prevents a payload like `cat safe.txt \; echo /etc/passwd > ./out`
from slipping through when `validateRedirections` (non-misparsing) fires on `>` but
`validateBackslashEscapedOperators` (misparsing) would have caught `\;`.

---

# Part II: Implementation Details

## 6. AST Analysis Pipeline

### 6.1 The Pure-TypeScript Parser

The parser (`bashParser.ts`) is a pure-TypeScript implementation producing
tree-sitter-bash-compatible ASTs. It has no native dependencies and requires no
async initialization. Key safety parameters:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `PARSE_TIMEOUT_MS` | 50 | Wall-clock cap; bails on adversarial input |
| `MAX_NODES` | 50,000 | Node budget cap; prevents OOM on deep nesting |
| `MAX_COMMAND_LENGTH` | 10,000 | Hard length limit before parse is attempted |

The parser produces `TsNode` objects with `type`, `text`, `startIndex`, `endIndex`,
and `children` fields. The `startIndex`/`endIndex` are UTF-8 byte offsets (not JS
string indices).

Source: `/src/utils/bash/bashParser.ts`

### 6.2 AST Security Walker

The `parseForSecurityFromAst()` function in `ast.ts` runs pre-checks for known
tree-sitter/bash differentials, then walks the AST with an explicit allowlist of
node types.

**Structural types** (recursed through): `program`, `list`, `pipeline`,
`redirected_statement`.

**Separator types** (skipped): `&&`, `||`, `|`, `;`, `&`, `|&`, `\n`.

**Dangerous types** (trigger too-complex): `command_substitution`,
`process_substitution`, `expansion`, `simple_expansion`, `brace_expression`,
`subshell`, `compound_statement`, control flow statements, `function_definition`,
`test_command`, `ansi_c_string`, `translated_string`, heredoc types.

The walker extracts `SimpleCommand` objects containing:
- `argv[]` -- command name and arguments with quotes resolved
- `envVars[]` -- leading `VAR=val` assignments
- `redirects[]` -- input/output redirections with operator and target
- `text` -- original source span

Command substitutions (`$(...)`) are recursively extracted: inner commands become
separate `SimpleCommand` entries, and the outer argv receives a
`__CMDSUB_OUTPUT__` placeholder. Variable references (`$VAR`) to variables assigned
earlier in the same command receive a `__TRACKED_VAR__` placeholder.

Source: `/src/utils/bash/ast.ts`

### 6.3 Tree-Sitter Analysis for Regex Validators

The `analyzeCommand()` function in `treeSitterAnalysis.ts` extracts a
`TreeSitterAnalysis` object that enriches the regex validator context:

- **`extractQuoteContext()`** -- uses a single fused tree walk (`collectQuoteSpans`)
  that collects all quote span types simultaneously (previously 5 separate walks).
  Handles `raw_string` (single-quoted), `string` (double-quoted), `ansi_c_string`
  (`$'...'`), and `heredoc_redirect` (quoted heredocs only). Produces three output
  strings by removing/replacing spans with correct handling of nested quotes
  (e.g. `"$(echo 'hi')"` -- the inner `raw_string` inside the outer `string`).

- **`extractCompoundStructure()`** -- walks top-level children of the program node
  to detect `list` nodes (compound operators), `pipeline` nodes, `subshell` nodes,
  and `compound_statement` nodes. Correctly recurses through `redirected_statement`
  wrappers and `negated_command` nodes.

- **`hasActualOperatorNodes()`** -- checks whether the AST contains real `;`,
  `&&`, or `||` operator nodes (as opposed to `\;` which appears as a `word` node).

- **`extractDangerousPatterns()`** -- walks the tree to detect `command_substitution`,
  `process_substitution`, `expansion`, `heredoc_redirect`, and `comment` nodes.

Source: `/src/utils/bash/treeSitterAnalysis.ts`

### 6.4 Semantic Checks

After `parseForSecurity()` returns `simple`, `checkSemantics()` runs post-argv
semantic checks on the extracted command list:

- **Wrapper stripping** -- strips `time`, `nohup`, `timeout`, `nice`, `stdbuf` to
  analyze the wrapped command. Each wrapper's flags are parsed with explicit allowlists
  and the function fails closed on unrecognized flags.
- **Eval-like builtins** -- blocks `eval`, `exec`, `source`, `.`, `trap`, `enable`,
  `hash`, `set`, `shopt`, and others that can execute arbitrary code.
- **Zsh dangerous builtins** -- blocks the same set as `ZSH_DANGEROUS_COMMANDS`.
- **Dangerous flags** -- per-command flag blocklists (e.g. `find -exec`, `jq -f`,
  `awk -f`, `git --upload-pack`, `git --exec-path`).
- **/proc/environ access** -- scans argv for `/proc/*/environ` paths.
- **jq system()** -- scans argv for `system(` calls.

Source: `/src/utils/bash/ast.ts` (lines 2209+)

## 7. Regex Fallback Details

The sync fallback function `bashCommandIsSafe_DEPRECATED()` processes the command
through these stages:

1. **Control character pre-check** -- blocks before any other processing.
2. **Shell-quote single-quote bug detection** -- detects `'\'` patterns that exploit
   shell-quote's incorrect backslash handling inside single quotes.
3. **Heredoc body stripping** -- strips bodies for quoted/escaped delimiters only.
4. **Quote content extraction** -- character-by-character state machine.
5. **Safe redirection stripping** -- removes `2>&1`, `>/dev/null`, `</dev/null` with
   trailing boundary assertions.
6. **Early validators** -- empty, incomplete, safe heredoc substitution, git commit.
7. **Main validators** -- the 23 checks described in Section 4.

The context object passed to validators contains:
- `originalCommand` -- the raw command string
- `baseCommand` -- first word of the command
- `unquotedContent` -- single-quoted content removed, double-quoted preserved
- `fullyUnquotedContent` -- all quoted content removed, safe redirections stripped
- `fullyUnquotedPreStrip` -- all quoted content removed, before redirection stripping
- `unquotedKeepQuoteChars` -- content removed but quote delimiters preserved
- `treeSitter` -- optional `TreeSitterAnalysis` when AST is available

Source: `/src/tools/BashTool/bashSecurity.ts`

## 8. Auto-Allow Logic

The scanner includes several early-allow paths for commands that can be
structurally proven safe without user interaction:

### 8.1 Safe Heredoc Substitution

The `isSafeHeredoc()` function allows `$(cat <<'DELIM'...DELIM)` patterns where:
- The delimiter is single-quoted or backslash-escaped (literal body, no expansion)
- The closing delimiter is the FIRST matching line (line-based matching, not regex)
- The `$(` is in argument position (not command-name position)
- The remaining text (with heredoc stripped) passes all validators
- The remaining text contains only safe ASCII characters

This is used by tools like the Edit tool which generates heredoc-based commands.

### 8.2 Git Commit Early Allow

Simple `git commit -m "message"` commands with single or double-quoted messages are
auto-allowed when:
- The message (if double-quoted) contains no `$()`, backticks, or `${}`
- The remainder after the message contains no shell operators
- The message does not start with a dash
- The command contains no backslashes
- Metacharacter classes before `-m` exclude shell operators

### 8.3 Safe Command Detection in Permission Pipeline

Beyond the scanner itself, the permission pipeline in `bashPermissions.ts` performs
additional safe-command detection:

- **Read-only commands** (`readOnlyValidation.ts`) -- commands like `ls`, `cat`,
  `grep`, `find` (without `-exec`) that only read files are auto-allowed when all
  extracted paths are within permitted directories.
- **Sandbox auto-allow** -- when the sandbox is enabled, commands that would normally
  require user approval can be auto-allowed because the OS-level sandbox constrains
  their effects.

## 9. Haiku Classifier Integration

The Bash permission system supports a **classifier** -- a small model (referred to
as "Haiku" in the codebase) that can match commands against natural language
descriptions for allow/deny/ask decisions. The classifier is feature-gated
(`BASH_CLASSIFIER` flag) and in the open-source build is stubbed out as always
returning `matches: false`.

When enabled, the classifier operates as follows:

1. Permission rules can contain `prompt:` prefixed descriptions (e.g.
   `prompt: commands that modify git history`).
2. When a command does not match any static rule, the classifier evaluates it
   against all prompt descriptions for each behavior (deny, ask, allow).
3. The classifier returns a `ClassifierResult` with `matches`, `confidence`
   (`high`/`medium`/`low`), `matchedDescription`, and `reason`.
4. Classifier results are attached as `pendingClassifierCheck` objects to the
   permission result, allowing the downstream pipeline to incorporate them.

The classifier is invoked via `classifyBashCommand()` which takes the command,
current working directory, descriptions, target behavior, and an abort signal.

Source: `/src/utils/permissions/bashClassifier.ts`

## 10. Integration with Permission System

The scanner's results feed into the `bashToolHasPermission()` function in
`bashPermissions.ts` through multiple integration points:

### 10.1 Pre-Split Security Gate

Before splitting compound commands, the full command runs through
`bashCommandIsSafeAsync()`. If the result has `isBashSecurityCheckForMisparsing: true`,
the command is immediately prompted to the user (behavior `ask`) without any
further rule matching. This prevents the compromised parser from incorrectly matching
rules against a misparsed command structure.

If the command has a safe heredoc substitution, the heredoc is stripped and the
remainder is re-checked through the security gate.

### 10.2 Per-Subcommand Security Checks

After splitting compound commands (e.g. `cmd1 && cmd2`), each subcommand runs
through the security scanner independently. If any subcommand triggers a misparsing
check, the entire compound command is prompted. The scanner enforces a cap of 50
subcommands (`MAX_SUBCOMMANDS_FOR_SECURITY_CHECK`) to prevent adversarial inputs
from starving the event loop.

### 10.3 AST-Based Permission Checking

The `checkCommandOperatorPermissions()` function uses the tree-sitter AST (when
available) or `ParsedCommand.parse()` to:
- Detect unsafe compound commands (subshells, command groups)
- Split pipe segments and check each through the full permission system
- Detect cross-segment `cd + git` patterns that could exploit bare repository
  `fsmonitor` hooks

### 10.4 Wrapper and Environment Variable Stripping

Before rule matching, `stripSafeWrappers()` removes:
- Safe environment variable prefixes (from a curated allowlist of ~50 variables)
- Safe wrapper commands (`timeout`, `time`, `nice`, `stdbuf`, `nohup`)

The stripping uses horizontal-only whitespace (`[ \t]+` not `\s+`) to prevent
cross-newline matching that could join two separate commands, and allowlists flag
values with `[A-Za-z0-9_.+-]` to prevent shell expansion inside flag arguments.

## 11. Platform Considerations

### 11.1 Bash vs Zsh Differences

Claude Code executes commands through the user's default shell, which is often zsh
on macOS. The scanner accounts for zsh-specific attack surface:

- **Module system** -- zsh's `zmodload` can load modules that provide raw file I/O
  (`zsh/system`), network access (`zsh/net/tcp`), pseudo-terminal execution
  (`zsh/zpty`), and filesystem primitives (`zsh/files`) that bypass normal command-level
  permission checks.
- **Process substitution** -- zsh supports `=()` in addition to bash's `<()` and `>()`.
- **Equals expansion** -- `=cmd` at word start expands to the absolute path of `cmd`.
- **Glob qualifiers** -- `(e:code:)` and `(+cmd)` execute arbitrary code during
  filename expansion.
- **Always blocks** -- `} always {` provides try/always semantics.
- **Precommand modifiers** -- `command`, `builtin`, `noglob`, `nocorrect` can prefix
  commands without changing execution, which the scanner strips before checking.
- **Dynamic named directories** -- `~[name]` invokes a hook that can run code.

### 11.2 PowerShell Defense-in-Depth

The scanner blocks PowerShell comment syntax (`<#`) as defense-in-depth against
future changes that might introduce PowerShell execution paths.

## 12. Sandbox Enforcement

The OS-level sandbox (Layer 7) complements the scanner by providing containment
when deterministic analysis cannot prove safety. The sandbox integration is
managed by `shouldUseSandbox()` in `shouldUseSandbox.ts`:

- **Sandbox enabled** -- commands run inside the OS sandbox (macOS seatbelt / Linux
  bubblewrap) which restricts filesystem access and network connectivity.
- **Sandbox disabled** -- commands run unsandboxed (user opted out or platform does
  not support sandboxing).
- **Explicit disable** -- `dangerouslyDisableSandbox` flag allows bypassing the
  sandbox, but only when `areUnsandboxedCommandsAllowed()` is true.
- **Excluded commands** -- user-configurable patterns that skip the sandbox (for
  commands like Docker that need system access). This is explicitly documented as
  NOT a security boundary.

The sandbox and scanner are independent layers. The scanner flags structural
concerns in the command text; the sandbox constrains runtime effects regardless of
command structure. A command that passes all 23 scanner checks still runs in the
sandbox. A command that fails scanner checks and gets user approval still runs in
the sandbox.

Source: `/src/tools/BashTool/shouldUseSandbox.ts`

## 13. False Positive Management

The scanner is tuned to minimize false positives while maintaining strict security
guarantees. Key strategies:

### 13.1 Tree-Sitter Eliminates Structural False Positives

The single largest source of false positives in the regex path is the
`find . -exec cmd {} \;` pattern, where `\;` triggers the backslash-escaped operator
check. Tree-sitter eliminates this by confirming that no actual `;` operator node
exists in the AST -- the `\;` is parsed as a word argument.

### 13.2 Quote Context from AST

Using tree-sitter's AST to extract quote context eliminates false positives from
the regex quote tracker, which can be confused by comments containing quote
characters, nested quotes inside command substitutions, and ANSI-C quoting.

### 13.3 Non-Misparsing Classification

Validators for LF newlines and redirections are classified as non-misparsing,
meaning their ask results do not block commands at the early security gate. This
allows commands with legitimate newlines (multiline scripts) and redirections
(output capture) to proceed to rule matching, where user-defined allow rules can
approve them.

### 13.4 Divergence Logging

The async path logs divergences between tree-sitter and regex quote extraction
(skipping heredoc commands where divergence is expected). This telemetry feeds
into continuous improvement of both paths and helps identify cases where the
regex fallback would produce different security decisions.

### 13.5 Command-Specific Exceptions

The obfuscated flags check skips `echo` commands (without shell operators) since
echoing obfuscated text is harmless. The jq check allows file arguments (validated
by path validation downstream) but blocks dangerous file-loading flags. The git
commit check allows the common `-m "message"` pattern without prompting.

## 14. Design Principles

The Bash Security Scanner embodies several core design principles:

1. **Fail closed.** When the scanner cannot determine whether a command is safe,
   it defaults to prompting the user. Unknown AST node types trigger `too-complex`.
   Parser timeouts trigger `PARSE_ABORTED` which routes to ask, not to the legacy
   path. Unrecognized wrapper flags prevent wrapper stripping.

2. **No shared weaknesses between layers.** The scanner uses deterministic structural
   analysis, which is immune to the prompt injection attacks that target the model
   (Layer 1) and the social engineering that targets human approval (Layer 12). The
   sandbox provides runtime containment that is immune to the parser differential
   attacks that target the scanner.

3. **Defense in depth.** Multiple checks cover overlapping attack surfaces. Brace
   expansion is checked by both the AST walker (node type `brace_expression`) and the
   regex validator (pattern matching). Zsh commands are checked by both
   `checkSemantics()` and `validateZshDangerousCommands()`. Control characters are
   checked by both the AST pre-checks and the regex validator.

4. **Parser differential awareness.** The scanner explicitly identifies and blocks
   patterns where its own parsers disagree with bash/zsh. Rather than trying to
   replicate bash's parser perfectly, it flags any ambiguity for human review.

5. **Minimal false positive surface.** Each check is designed to flag the smallest
   possible set of commands that includes the attack pattern. Checks use the most
   precise content representation available (e.g. `fullyUnquotedPreStrip` for brace
   expansion to avoid false negatives from redirection stripping, but
   `unquotedKeepQuoteChars` for mid-word hash to catch quote-adjacent patterns).

6. **Audit trail.** Every check that fires logs a `tengu_bash_security_check_triggered`
   event with the numeric `checkId` and optional `subId`, enabling post-hoc analysis
   of which checks are firing, how often, and whether they correlate with actual
   attacks or false positives.

7. **Determinism.** The scanner produces identical results for identical inputs with
   no randomness, no model calls, and no external state dependencies. This makes it
   testable, debuggable, and resistant to the non-determinism that plagues
   model-based safety systems.
