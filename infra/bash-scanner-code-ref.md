# Bash Scanner — Code Reference

## Source Files

| File | Path | Role |
|------|------|------|
| Bash parser (pure-TS) | `/src/utils/bash/bashParser.ts` | Produces tree-sitter-bash-compatible ASTs; defines `PARSE_TIMEOUT_MS` (50ms), `MAX_NODES` (50,000); exports `TsNode` type |
| Parser gateway | `/src/utils/bash/parser.ts` | Defines `MAX_COMMAND_LENGTH` (10,000), `PARSE_ABORTED` sentinel; `parseCommandRaw()` function |
| AST analysis | `/src/utils/bash/ast.ts` | `parseForSecurity()` (line 381), `parseForSecurityFromAst()` (line 400), `checkSemantics()` (line 2213); extracts `SimpleCommand` objects |
| Tree-sitter analysis | `/src/utils/bash/treeSitterAnalysis.ts` | `TreeSitterAnalysis` type (line 58), `analyzeCommand()` (line 496), `extractQuoteContext()` (line 224), `extractCompoundStructure()` (line 296), `hasActualOperatorNodes()` (line 421), `extractDangerousPatterns()` (line 448), `collectQuoteSpans()` (line 88) |
| Security validators | `/src/tools/BashTool/bashSecurity.ts` | `BASH_SECURITY_CHECK_IDS` (line 77), `bashCommandIsSafe_DEPRECATED()` (line 2257), `bashCommandIsSafeAsync_DEPRECATED()` (line 2426), `ValidationContext` type (line 103), all 23 validator functions |
| Permission integration | `/src/tools/BashTool/bashPermissions.ts` | `bashToolHasPermission()`, `stripSafeWrappers()` (line 524), `checkCommandOperatorPermissions()`, `MAX_SUBCOMMANDS_FOR_SECURITY_CHECK` (50, line 103) |
| Classifier | `/src/utils/permissions/bashClassifier.ts` | `classifyBashCommand()`, `ClassifierResult` type, feature-gated by `BASH_CLASSIFIER` flag |
| Sandbox | `/src/tools/BashTool/shouldUseSandbox.ts` | `shouldUseSandbox()`, `areUnsandboxedCommandsAllowed()`, `dangerouslyDisableSandbox` flag handling |
| Read-only validation | `/src/tools/BashTool/readOnlyValidation.ts` | Read-only command detection for auto-allow |

## Constants and IDs

```typescript
const BASH_SECURITY_CHECK_IDS = {
  INCOMPLETE_COMMANDS: 1,
  JQ_SYSTEM_FUNCTION: 2,
  JQ_FILE_ARGUMENTS: 3,
  OBFUSCATED_FLAGS: 4,
  SHELL_METACHARACTERS: 5,
  DANGEROUS_VARIABLES: 6,
  NEWLINES: 7,
  DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION: 8,
  DANGEROUS_PATTERNS_INPUT_REDIRECTION: 9,
  DANGEROUS_PATTERNS_OUTPUT_REDIRECTION: 10,
  IFS_INJECTION: 11,
  GIT_COMMIT_SUBSTITUTION: 12,
  PROC_ENVIRON_ACCESS: 13,
  MALFORMED_TOKEN_INJECTION: 14,
  BACKSLASH_ESCAPED_WHITESPACE: 15,
  BRACE_EXPANSION: 16,
  CONTROL_CHARACTERS: 17,
  UNICODE_WHITESPACE: 18,
  MID_WORD_HASH: 19,
  ZSH_DANGEROUS_COMMANDS: 20,
  BACKSLASH_ESCAPED_OPERATORS: 21,
  COMMENT_QUOTE_DESYNC: 22,
  QUOTED_NEWLINE: 23,
}
```

## Key Types

- **`TsNode`** (`bashParser.ts`): `{ type: string, text: string, startIndex: number, endIndex: number, children: TsNode[] }` -- startIndex/endIndex are UTF-8 byte offsets.
- **`TreeSitterAnalysis`** (`treeSitterAnalysis.ts`, line 58): `{ quoteContext: QuoteContext, compoundStructure: CompoundStructure, hasActualOperatorNodes: boolean, dangerousPatterns: DangerousPatterns }`
- **`ValidationContext`** (`bashSecurity.ts`, line 103): `{ originalCommand, baseCommand, unquotedContent, fullyUnquotedContent, fullyUnquotedPreStrip, unquotedKeepQuoteChars, treeSitter?: TreeSitterAnalysis }`
- **`SimpleCommand`** (`ast.ts`): `{ argv[], envVars[], redirects[], text }`
- **`PARSE_ABORTED`** (`parser.ts`, line 93): `Symbol('parse-aborted')` -- distinct from `null`.

## Safety Parameters

| Parameter | Value | Defined In |
|-----------|-------|------------|
| `PARSE_TIMEOUT_MS` | 50 | `bashParser.ts` line 29 |
| `MAX_NODES` | 50,000 | `bashParser.ts` line 32 |
| `MAX_COMMAND_LENGTH` | 10,000 | `parser.ts` line 19 |
| `MAX_SUBCOMMANDS_FOR_SECURITY_CHECK` | 50 | `bashPermissions.ts` line 103 |

## Validator Functions (in execution order)

**Early validators** (lines 2309-2312 of `bashSecurity.ts`):
1. `validateEmpty` (line 233)
2. `validateIncompleteCommands` (line 244)
3. `validateSafeCommandSubstitution` (line 585)
4. `validateGitCommit` (line 612)

**Main validators** (lines 2348-2377 of `bashSecurity.ts`):
1. `validateJqCommand`
2. `validateObfuscatedFlags`
3. `validateShellMetacharacters`
4. `validateDangerousVariables`
5. `validateCommentQuoteDesync`
6. `validateQuotedNewline`
7. `validateCarriageReturn`
8. `validateNewlines`
9. `validateIFSInjection`
10. `validateProcEnvironAccess`
11. `validateDangerousPatterns`
12. `validateRedirections`
13. `validateBackslashEscapedWhitespace`
14. `validateBackslashEscapedOperators`
15. `validateUnicodeWhitespace`
16. `validateMidWordHash`
17. `validateBraceExpansion`
18. `validateZshDangerousCommands`
19. `validateMalformedTokenInjection`

**Non-misparsing validators** (line 2343): `validateNewlines`, `validateRedirections`

## Key Flags and Sentinels

- `isBashSecurityCheckForMisparsing: true` -- attached to ask results from misparsing validators and all fallback-path results; causes early block in permission pipeline.
- `PARSE_ABORTED` symbol -- returned by `parseCommandRaw()` on timeout/budget exhaustion; must be treated as fail-closed.
- `__CMDSUB_OUTPUT__` placeholder -- inserted in argv for recursive command substitution extraction.
- `__TRACKED_VAR__` placeholder -- inserted for variable references to variables assigned in the same command.

## Analytics

Every triggered check logs `tengu_bash_security_check_triggered` with `checkId` (numeric from `BASH_SECURITY_CHECK_IDS`) and optional `subId` (numeric sub-check within a category).

## AST Node Type Classification (in `parseForSecurityFromAst`)

- **Structural** (recursed): `program`, `list`, `pipeline`, `redirected_statement`
- **Separator** (skipped): `&&`, `||`, `|`, `;`, `&`, `|&`, `\n`
- **Dangerous** (trigger too-complex): `command_substitution`, `process_substitution`, `expansion`, `simple_expansion`, `brace_expression`, `subshell`, `compound_statement`, control flow statements, `function_definition`, `test_command`, `ansi_c_string`, `translated_string`, heredoc types

## Quote Context Extraction Spans (`collectQuoteSpans`)

- `raw`: `raw_string` (single-quoted)
- `ansiC`: `ansi_c_string` (`$'...'`)
- `double`: `string` (double-quoted)
- `heredoc`: `heredoc_redirect` (quoted heredocs only)

## Known Errata in Spec vs. Code

1. `MAX_COMMAND_LENGTH` is in `parser.ts`, not `bashParser.ts` as originally stated in Section 6.1. **[Fixed in bash-scanner.md]**
2. The async scanner function is `bashCommandIsSafeAsync_DEPRECATED()` (with `_DEPRECATED` suffix), not `bashCommandIsSafeAsync()` as originally stated in Section 10.1. **[Fixed in bash-scanner.md]**
3. `checkSemantics()` export declaration is at line 2213, not "2209+" (2209 is the doc comment). **[Fixed in bash-scanner.md]**
