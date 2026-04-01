# Bash Scanner — Design Document

## 1. Overview

Claude Code executes bash commands on behalf of an AI model. The model generates command strings passed to the user's default shell (bash or zsh). This creates an adversarial boundary: if the model is manipulated via prompt injection, the command string itself becomes the attack vector. The Bash Security Scanner is the deterministic gatekeeper that sits between the model's tool call and the shell execution, analyzing the raw command text for structural properties that indicate injection, obfuscation, or privilege escalation.

The scanner operates in the permission pipeline after user-defined rules and tool-specific domain checks (e.g., path validation), but before the sandbox and any AI classifier. Its key design property is that it is **purely deterministic** -- no model calls, no probabilistic reasoning. It applies structural analysis and pattern matching to the command text and produces one of three outcomes:

- **Passthrough** -- the command passed all 23 checks; continue to downstream layers.
- **Ask** -- at least one check flagged a concern; prompt the user for approval.
- **Allow** -- an early validator proved the command is structurally safe (e.g., a simple `git commit -m "message"` with no metacharacters).

The scanner never produces a **deny** outcome on its own. Its purpose is to force human review when deterministic analysis detects structural ambiguity, not to block commands outright. Deterministic scanners are good at detecting suspicious structure but poor at judging intent -- the human (or AI classifier in auto mode) makes the final call.

## 2. Threat Model

The scanner defends against two primary threat classes:

**Parser differential attacks.** Claude Code uses multiple parsers internally (tree-sitter, shell-quote, regex) and the actual shell (bash/zsh) uses yet another parser. When these parsers disagree on the structure of a command, an attacker can craft input that one parser considers safe but the shell executes differently. The canonical example: `echo\ test/../../../usr/bin/touch /tmp/file` -- shell-quote splits this into two tokens, but bash treats `echo\ test` as a single token (a command named "echo test"), resolving the path traversal differently.

**Obfuscation attacks.** Even when parsers agree on structure, shell quoting rules allow the same command to be expressed in many equivalent forms. An attacker can use ANSI-C quoting (`$'\x2d'exec`), empty quote concatenation (`"""-f"`), brace expansion (`{--upload-pack="evil",test}`), or Unicode whitespace to disguise dangerous flags or command names so they evade pattern-matching permission rules while bash still executes the intended payload.

The scanner also provides defense-in-depth against:

- Zsh-specific attack surface (module loading, socket primitives, eval equivalents)
- /proc/environ access (environment variable exfiltration)
- IFS injection (field separator manipulation to split arguments)
- Heredoc abuse (hiding commands in here-document bodies)
- Comment-quote desynchronization (using `#` comments to confuse quote tracking)

## 3. Architecture: Dual-Path Analysis

The scanner uses a **dual-path architecture** with a tree-sitter AST parser as the primary analysis engine and a regex/shell-quote fallback for environments where tree-sitter is unavailable.

```
              +---------------+
              |  Raw Command  |
              +-------+-------+
                      |
              +-------v-------+
              |  Pre-checks   |  Control chars, Unicode WS, backslash-WS
              |  (both paths) |  These run BEFORE any parser
              +-------+-------+
                      |
          +-----------+-----------+
          |                       |
   +------v-------+       +------v-------+
   |  tree-sitter  |       |  Regex/shell-|
   |  AST path     |       |  quote path  |
   |  (primary)    |       |  (fallback)  |
   +------+-------+       +------+-------+
          |                       |
          |  AST walk:            |  23 regex validators
          |  structural + semantic|  character-by-character
          |  analysis             |  state machine
          |                       |
          +-----------+-----------+
                      |
              +-------v-------+
              |  Permission   |
              |  Result       |
              | (allow/ask/   |
              |  passthrough) |
              +--------------+
```

### 3.1 Tree-Sitter AST Path (Primary)

The primary path uses a pure-TypeScript bash parser producing tree-sitter-bash-compatible ASTs. The parser was validated against a 3449-input golden corpus generated from the WASM parser. It operates with a 50ms wall-clock timeout and a 50,000-node budget to prevent adversarial inputs from causing denial-of-service. A hard command length limit also applies before parsing is attempted.

The AST path provides two levels of analysis:

1. **Structural analysis** -- walks the AST with an explicit allowlist of node types. Any node type not in the allowlist causes the entire command to be classified as `too-complex`, which means the command goes through the normal permission prompt flow. This is a **fail-closed** design: the system never interprets structure it does not understand.

2. **Semantic analysis** -- runs after the structural analysis returns `simple`. Performs semantic checks on extracted `argv[]` arrays: detects eval-like builtins, dangerous flags (e.g., `-exec` on `find`), Zsh dangerous commands, and `/proc/environ` access.

The AST path also provides enriched context to the regex validators via a typed analysis object, which includes:

- **QuoteContext** -- extracted quoted/unquoted content computed from AST node spans rather than character-by-character regex tracking. This eliminates an entire class of quote-tracking desync bugs.
- **CompoundStructure** -- whether the command has compound operators (`&&`, `||`, `;`), pipelines, subshells, or command groups, extracted from AST node types.
- **Operator node detection** -- whether real operator nodes exist in the AST. This is the key mechanism for eliminating the `find -exec \;` false positive: tree-sitter parses `\;` as part of a `word` node (an argument), NOT as a `;` operator.
- **DangerousPatterns** -- presence of command substitution, process substitution, parameter expansion, heredocs, and comments.

### 3.2 Regex/Shell-Quote Fallback Path

When tree-sitter is unavailable (external builds where the feature flag is off, or when the parser times out), the scanner falls back to a legacy function. This function runs the same 23 validator checks but uses:

- A character-by-character state machine that tracks single/double quote state and backslash escaping to produce three derived strings: one with single-quoted content removed, one with all quoted content removed, and one with content removed but delimiter characters preserved.
- Heredoc body stripping for quoted/escaped delimiters (where the body is literal text).
- The shell-quote library for tokenization, used to detect unbalanced delimiters.

The fallback path is known to have parser differential vulnerabilities that the AST path does not. For this reason, ask results from the fallback path carry a misparsing flag, which causes them to be blocked early in the permission pipeline (before any rule matching that might use the same vulnerable parser).

### 3.3 Parse Abort Handling

If the tree-sitter parser times out or exceeds its node budget, a sentinel value is returned that is distinct from "module not loaded." Callers **must** treat this as fail-closed (`too-complex`), NOT route it to the legacy regex path. This is critical because adversarial input can intentionally trigger parser abort -- e.g., `(( a[0][0]... ))` with approximately 2800 subscripts hits the timeout -- and the legacy path lacks protections for eval-like builtins that the AST path catches.

## 4. The 23 Injection Check Categories

Each check is identified by a numeric constant for analytics logging. The checks are organized into early validators (that can short-circuit with allow), main validators (that flag concerns), and deferred non-misparsing validators.

### Check 1: Incomplete Commands
Detects command fragments that look like continuation lines: starts with tab, starts with dash, or starts with a shell operator (`&&`, `||`, `;`, `>>`, `<`). These suggest the model generated a fragment rather than a full command.

### Check 2: jq system() Function
Detects `system()` calls inside jq expressions (an eval equivalent) and dangerous jq flags (`-f`, `--from-file`, `--rawfile`, `--slurpfile`, `-L`, `--library-path`) that could read arbitrary files.

### Check 3: jq File Arguments
Flags jq invocations with flags that can load external files or libraries, which could be used to exfiltrate data or execute code from attacker-controlled file paths.

### Check 4: Obfuscated Flags
A comprehensive multi-layer detection system for flag obfuscation via shell quoting. Eleven sub-checks detect ANSI-C quoting, locale quoting, empty quote + dash patterns, quote sequences before dashes, homogeneous empty pairs adjacent to quoted dashes, three or more consecutive quotes at word start, quoted characters inside flag names, split-quote flags, chained quotes, and flag continuation characters after quotes. The check skips `echo` commands without shell operators since echoing obfuscated text is harmless.

### Check 5: Shell Metacharacters
Detects shell metacharacters (`;`, `|`, `&`) inside quoted arguments in unquoted content, catching cases where metacharacters are embedded in arguments to commands like `find -name` or `-path`.

### Check 6: Dangerous Variables in Redirections/Pipes
Detects shell variables (`$VAR`) adjacent to pipe (`|`) or redirection (`<`, `>`) operators, where variable values could control file targets or command names.

### Check 7: Newlines
Detects newlines in unquoted content that could separate multiple commands. Allows backslash-newline continuations at word boundaries but flags mid-word continuations. A separate sub-check detects carriage return (`\r`) outside double quotes, which exploits a parser differential between shell-quote (which treats CR as whitespace) and bash (which does not).

### Check 8: Command Substitution and Dangerous Patterns
Detects 13 forms of command/process/parameter substitution: process substitution (`<()`, `>()`), zsh process substitution (`=()`), zsh equals expansion (`=cmd`), command substitution (`$()`, backticks), parameter substitution (`${}`), legacy arithmetic expansion (`$[]`), zsh parameter expansion (`~[`), zsh glob qualifiers with execution (`(e:`, `(+`), zsh always blocks (`} always {`), and PowerShell comment syntax (`<#`).

### Check 9: Input Redirection
Detects `<` in fully unquoted content.

### Check 10: Output Redirection
Detects `>` in fully unquoted content. Safe redirections (`2>&1`, `>/dev/null`, `</dev/null`) are stripped before this check, with trailing boundary assertions to prevent prefix matching attacks.

### Check 11: IFS Injection
Detects any usage of the `$IFS` variable or `${...IFS...}` parameter expansions, which can cause argument splitting at unexpected positions.

### Check 12: Git Commit Substitution
Special-case validation for `git commit -m "message"` commands. Detects command substitution patterns inside double-quoted commit messages and validates that the command remainder does not contain shell operators.

### Check 13: /proc/environ Access
Detects paths matching `/proc/*/environ` which expose environment variables including API keys and secrets.

### Check 14: Malformed Token Injection
Detects commands with malformed tokens (unbalanced delimiters) combined with command separators. Discovered via HackerOne review.

### Check 15: Backslash-Escaped Whitespace
Detects `\ ` (backslash-space) or `\<tab>` outside quotes. In bash, `echo\ test` is a single token, but shell-quote produces two tokens -- a parser differential enabling path traversal attacks.

### Check 16: Brace Expansion
Detects unquoted brace expansion syntax (`{a,b}` or `{1..5}`) that bash expands but other parsers treat as literal. Three sub-checks: comma/sequence patterns at outermost nesting, mismatched brace counts after quote stripping, and quoted brace characters inside unquoted brace contexts.

### Check 17: Control Characters
Detects non-printable control characters (`0x00-0x08`, `0x0B-0x0C`, `0x0E-0x1F`, `0x7F`). Runs first, before any other processing, because control characters can confuse all downstream validators.

### Check 18: Unicode Whitespace
Detects Unicode whitespace characters (NBSP, Ogham space, various Unicode spaces, line/paragraph separators, BOM/ZWNBSP). Shell-quote treats these as word separators but bash treats them as literal word content.

### Check 19: Mid-Word Hash
Detects `#` preceded by a non-whitespace character. Shell-quote treats mid-word `#` as a comment start but bash treats it as a literal character, creating a parser differential.

### Check 20: Zsh Dangerous Commands
Detects zsh-specific commands: `zmodload` (module loading), `emulate -c` (eval equivalent), module builtins (`sysopen`, `sysread`, `syswrite`, `ztcp`, `zsocket`, `mapfile`, etc.), and `fc -e` (editor execution). Strips environment variable assignments and zsh precommand modifiers before extracting the base command name.

### Check 21: Backslash-Escaped Operators
Detects a backslash immediately preceding a shell operator (`;`, `|`, `&`, `<`, `>`) outside quotes. A normalization function converts `\;` to bare `;`, which on re-parse becomes an operator, enabling attacks where one command is seen by the validator but two commands execute. When tree-sitter confirms no actual operator nodes exist, this check is skipped (eliminating the `find -exec \;` false positive).

### Check 22: Comment-Quote Desynchronization
Detects quote characters inside `#` comments. In bash, everything after unquoted `#` is a comment, but regex quote-tracking does not handle comments, so quotes after `#` desynchronize the tracker. When tree-sitter provides quote context, this check is bypassed.

### Check 23: Quoted Newline Hiding
Detects newlines inside quoted strings where the next line would be stripped by a line-based comment stripper. This allows attackers to hide sensitive paths from path validation.

## 5. Validator Execution Order and Short-Circuit Logic

The validators are organized into three groups:

**Early Validators (can short-circuit with allow):**
Empty check -> Incomplete commands -> Safe command substitution -> Git commit

If any early validator returns `allow`, the scanner returns `passthrough`. If any returns `ask`, it is tagged as a misparsing concern and returned immediately.

**Main Validators (sequential, with deferred non-misparsing logic):**
jq -> Obfuscated flags -> Shell metacharacters -> Dangerous variables -> Comment-quote desync -> Quoted newline -> Carriage return -> Newlines -> IFS injection -> /proc/environ access -> Dangerous patterns -> Redirections -> Backslash-escaped whitespace -> Backslash-escaped operators -> Unicode whitespace -> Mid-word hash -> Brace expansion -> Zsh dangerous commands -> Malformed token injection

Two validators (Newlines and Redirections) are classified as **non-misparsing** -- their ask results go through the standard permission flow rather than being blocked early.

**Deferred result mechanism:** The scanner does not short-circuit when a non-misparsing validator returns `ask` if there are still misparsing validators later in the list. The non-misparsing result is deferred. If any subsequent misparsing validator fires, its result (with the misparsing flag) takes priority. Only if no misparsing validator fires does the deferred non-misparsing result get returned. This prevents payloads that combine a non-misparsing concern (e.g., `>`) with a misparsing concern (e.g., `\;`) from slipping through.

## 6. Auto-Allow Logic

The scanner includes several early-allow paths:

- **Safe heredoc substitution** -- allows `$(cat <<'DELIM'...DELIM)` patterns where the delimiter is single-quoted or backslash-escaped, the closing delimiter is the first matching line, the heredoc is in argument position, the remaining text passes all validators, and the remaining text contains only safe ASCII characters.

- **Git commit early allow** -- simple `git commit -m "message"` commands with messages containing no command substitution, no shell operators in the remainder, no leading dashes, no backslashes, and no metacharacters before `-m`.

- **Read-only commands** -- commands like `ls`, `cat`, `grep`, `find` (without `-exec`) that only read files are auto-allowed when all extracted paths are within permitted directories.

- **Sandbox auto-allow** -- when the OS-level sandbox is enabled, commands that would normally require user approval can be auto-allowed because the sandbox constrains their effects.

## 7. Classifier Integration

The permission system supports a classifier (a small model) that can match commands against natural language descriptions for allow/deny/ask decisions. The classifier is feature-gated and in the open-source build is stubbed out. When enabled, permission rules can contain `prompt:` prefixed descriptions, the classifier evaluates commands against these descriptions, and results include confidence levels and match reasons.

## 8. Integration with Permission System

The scanner's results feed into the permission system through multiple integration points:

- **Pre-split security gate** -- before splitting compound commands, the full command runs through the scanner. If a misparsing flag is set, the command is immediately prompted to the user without further rule matching.

- **Per-subcommand security checks** -- after splitting compound commands, each subcommand runs through the scanner independently, with a cap of 50 subcommands to prevent event loop starvation.

- **AST-based permission checking** -- uses the tree-sitter AST to detect unsafe compound commands, split pipe segments, and detect cross-segment patterns that could exploit repository hooks.

- **Wrapper and environment variable stripping** -- removes safe environment variable prefixes (from a curated allowlist) and safe wrapper commands (`timeout`, `time`, `nice`, `stdbuf`, `nohup`) using horizontal-only whitespace matching to prevent cross-newline attacks.

## 9. Platform Considerations

**Bash vs Zsh:** The scanner accounts for zsh-specific attack surface including module loading (`zmodload`), extended process substitution (`=()`), equals expansion (`=cmd`), glob qualifiers with execution (`(e:code:)`), always blocks, precommand modifiers, and dynamic named directories (`~[name]`).

**PowerShell:** The scanner blocks PowerShell comment syntax (`<#`) as defense-in-depth.

## 10. Sandbox Enforcement

The OS-level sandbox complements the scanner by providing containment when deterministic analysis cannot prove safety. The sandbox and scanner are independent layers: the scanner flags structural concerns in command text; the sandbox constrains runtime effects regardless of command structure. A command that passes all 23 scanner checks still runs in the sandbox. A command that fails scanner checks and gets user approval still runs in the sandbox.

## 11. False Positive Management

Key strategies:

- Tree-sitter eliminates structural false positives (especially `find . -exec cmd {} \;`).
- AST-derived quote context eliminates regex quote tracker confusion.
- Non-misparsing classification allows commands with legitimate newlines and redirections to proceed to rule matching.
- Divergence logging between tree-sitter and regex paths feeds continuous improvement.
- Command-specific exceptions (e.g., `echo` commands skip obfuscated flag checks).

## 12. Design Principles

1. **Fail closed.** Unknown structures default to prompting. Parser timeouts route to ask, not to the legacy path.
2. **No shared weaknesses between layers.** Deterministic analysis is immune to prompt injection; the sandbox is immune to parser differential attacks.
3. **Defense in depth.** Multiple checks cover overlapping attack surfaces.
4. **Parser differential awareness.** The scanner flags ambiguity rather than trying to replicate bash's parser perfectly.
5. **Minimal false positive surface.** Each check flags the smallest possible set of commands containing the attack pattern.
6. **Audit trail.** Every check logs an event with numeric check ID and optional sub-ID.
7. **Determinism.** Identical results for identical inputs with no randomness, no model calls, and no external state dependencies.
