# Claude Code: Oversight & Human-in-the-Loop System Design

This document analyzes the oversight architecture of Claude Code — Anthropic's agentic
CLI tool — focusing on how it implements a **Swiss cheese model** of layered safety
defenses, how auto mode works, and how human oversight is maintained throughout.

The document is split into two parts: **Part I** describes the system design at an
architectural level, and **Part II** provides implementation details with source code
references.

---

# Part I: System Design

## 1. Design Philosophy: The Swiss Cheese Model

The [Swiss cheese model](https://en.wikipedia.org/wiki/Swiss_cheese_model) originates
from risk management in aviation and medicine. Multiple independent defensive layers
are stacked in sequence. Each layer has "holes" — weaknesses or blind spots — but the
holes are at different positions across layers, so a hazard must pass through aligned
holes in *every* layer simultaneously to cause harm.

Claude Code applies this to agentic AI oversight with 14 layers. The core design
principle: **no two adjacent layers share the same weakness**. An injection attack that
fools the model still hits a deterministic security scanner. A scanner bypass still
hits an AI classifier reviewing the full transcript. A classifier hallucination still
forces human review after repeated denials. A careless human approval is still
constrained by an OS-level sandbox. A sandbox escape is still logged for post-hoc
detection.

```
     Hazard (dangerous tool invocation)
        │
        ▼
┌──────────────────────────┐
│ L1  System Prompt         │  Model-level safety instructions
├───────○──────────────────┤
│ L2  Input Validation      │  Schema + semantic checks, fail-closed
├──────────────○───────────┤
│ L3  Permission Rules      │  User-defined deny/allow/ask rules
├───○──────────────────────┤
│ L4  Tool-Specific Checks  │  Domain logic (paths, protected files)
├──────────○───────────────┤
│ L5  Bash Security Scanner │  23 deterministic injection checks
├─────────────────○────────┤
│ L6  Pattern Stripping     │  Removes overly broad rules at auto entry
├──○───────────────────────┤
│ L7  Sandbox               │  OS-level filesystem/network isolation
├────────────────○─────────┤
│ L8  Mode Transformation   │  dontAsk→deny, plan→prompt, auto→classifier
├──────○───────────────────┤
│ L9  Hooks                 │  Extensible external allow/deny injection
├────────────────○─────────┤
│ L10 AI Classifier         │  2-stage LLM with fast+thinking pipeline
├───────○──────────────────┤
│ L11 Denial Limits         │  3 consecutive / 20 total → force human
├──────────────────○───────┤
│ L12 Human Approval        │  5-way interactive race
├─○────────────────────────┤
│ L13 Audit Log             │  Tagged decisions + 3 telemetry sinks
├────────────────────○─────┤
│ L14 Query Pipeline Safety │  Error recovery, budget enforcement
└──────────────────────────┘
        │
        ▼
     Action executes (or is blocked)
```

## 2. Permission Modes

The outermost design choice is the user's **permission mode** — a coarse trust posture
that governs how all downstream layers behave:

| Mode | Behavior | Use Case |
|------|----------|----------|
| `default` | Prompt user for any sensitive operation | Normal interactive use |
| `plan` | Suspend execution, show intent first | Architecture exploration |
| `acceptEdits` | Auto-allow file edits in working directory | Trusted editing sessions |
| `dontAsk` | Convert all `ask` decisions to `deny` | Read-only exploration |
| `auto` | Replace user prompts with AI classifier | Unattended operation |
| `bypassPermissions` | Skip most checks (kill-switch gated) | Development/debugging |

The `auto` mode is feature-gated and only available in builds with the
`TRANSCRIPT_CLASSIFIER` flag. Mode transitions trigger side effects: entering auto mode
strips dangerous allow rules; exiting restores them.

## 3. The Decision Pipeline

Every tool invocation passes through a central permission function that produces one
of four outcomes:

- **Allow** — proceed with (optionally modified) input
- **Ask** — show interactive dialog, optionally with a pending classifier check
- **Deny** — block with a message
- **Passthrough** — internal intermediate state; continue checking

Every decision carries a typed **reason tag** (rule, mode, hook, classifier,
safetyCheck, sandboxOverride, asyncAgent, workingDir, other) so the system maintains
a complete audit trail of *why* each decision was made.

## 4. The Layers

### L1: System Prompt Safety Instructions

Before any tool runs, the model receives explicit safety instructions embedded in the
system prompt. These cover:

- **Prompt injection awareness** — flag suspicious tool results to the user
- **OWASP top 10** — avoid generating code with command injection, XSS, SQL injection
- **Reversibility heuristic** — consider blast radius; confirm destructive, hard-to-reverse, or externally-visible actions
- **Explicit risky-action taxonomy** — destructive (rm -rf, drop table), hard-to-reverse (force-push, reset --hard), externally-visible (push, send message, post to API)
- **Permission mode awareness** — don't re-attempt denied tool calls

The system prompt is split into static (cacheable) and dynamic (session-specific)
halves by a boundary marker, so safety instructions remain in the cached prefix and
are never evicted.

Context is assembled from CLAUDE.md files loaded in priority order (managed → user →
project → local → auto-memory → team-memory), with `@include` directive support and
circular reference prevention (max depth 5, path deduplication).

**Design rationale:** Probabilistic but broad coverage. The model might not always
follow instructions, but the instructions establish the expected behavior baseline
and catch the majority of "obvious" risks.

### L2: Tool Input Validation

A two-level validation pipeline runs before any permission logic:

1. **Structural validation** (Zod schema) — catches type errors, missing/extra fields
2. **Semantic validation** (per-tool `validateInput()`) — catches domain-specific errors
   like file-not-found, UNC path NTLM credential leaks, read-before-edit violations,
   encoding issues, and file size limits

Both levels are **fail-closed**: any validation failure returns an error to the model
wrapped in `<tool_use_error>` tags. The tool never executes.

**Design rationale:** Catches malformed or nonsensical tool calls before they enter
the permission system. Prevents the model from accidentally crafting inputs that
exploit downstream assumptions.

### L3: Rule-Based Permission Hierarchy

Users configure deny/allow/ask rules from 8 sources, loaded in priority order:

```
policySettings  →  flagSettings  →  projectSettings  →  userSettings
     →  localSettings  →  cliArg  →  command  →  session
```

Evaluation order: **deny rules first** (immediate block), then **ask rules** (prompt),
then **tool's own checkPermissions**, then **allow rules** (approve). This ensures
that explicit denials always win.

Rules support three pattern types: exact match, legacy prefix (`:*`), and wildcard
(`*`). Tools provide pattern matchers via `preparePermissionMatcher()` for domain-specific
matching (file paths, command prefixes).

When a rule triggers an `ask` decision, the system generates **suggestions** —
pre-built permission updates the user can accept to avoid being asked again.

**Design rationale:** Gives users explicit, persistent control. Deny rules can block
entire tool categories. The priority ordering means organizational policies always win
over personal settings.

### L4: Tool-Specific Permission Checks

Each tool implements domain-specific safety logic:

- **File tools**: Check paths against working directory, resolve symlinks, normalize
  case, block `..` traversal, guard sensitive paths (`.git/`, `.claude/`, `.vscode/`,
  SSH keys) with **safetyCheck** decisions that are immune to bypass mode
- **Bash tool**: AST-based security parse via tree-sitter, sandbox auto-allow logic,
  Haiku classifier for command description matching, pipe/redirect validation
- **Agent tool**: Auto-allows in non-auto modes, defers to classifier in auto mode
- **MCP tool**: Always defers to the general permission framework

The `safetyCheck` decision type has a `classifierApprovable` boolean: `false` means
the check is **immune to all auto-approve paths** including the AI classifier (e.g.,
Windows path bypass attempts). `true` means the classifier can evaluate it (e.g.,
sensitive file paths where context matters).

**Design rationale:** Domain experts know their risks best. File tools understand path
traversal; the bash tool understands shell injection. The general permission system
can't replicate this domain knowledge.

### L5: Bash Security Scanner

A dedicated injection detection engine with **23 numbered check categories** covering:

- Command substitution (`$()`, backticks, `${}`, `<()`, `>()`, zsh `=()`)
- Shell metacharacter injection (`;`, `|`, `&` in arguments)
- Obfuscated flags (ANSI-C quoting, empty-quote tricks)
- Parser differentials (shell-quote vs bash tokenization mismatches)
- Variable injection in redirections/pipes
- IFS injection, brace expansion, Unicode whitespace
- Zsh-specific: `zmodload` (module attacks), `emulate` (eval equivalent),
  `sysopen`/`ztcp`/`zsocket` (file/network I/O bypass)
- Comment-quote desynchronization, quoted newline hiding
- `/proc/environ` access, control characters

The scanner uses tree-sitter WASM for AST-based analysis when available, falling back
to regex-based parsing. It detects attacks that exploit **differential behavior**
between the command parser used for permission checking and bash's actual evaluation.

**Design rationale:** Deterministic and not susceptible to model manipulation. Covers
a large surface area of known injection techniques. The tree-sitter/regex dual path
provides defense in depth within the layer itself.

### L6: Dangerous Pattern Stripping

When entering auto mode, the system proactively **strips overly broad allow rules**
that would let the classifier be bypassed entirely. Rules matching code-execution
entry points (python, node, bash, ssh, curl, etc.) in any of five pattern shapes
(exact, prefix, trailing/space/flag wildcards) are removed and stashed for restoration
on exit.

This also covers PowerShell cmdlets (`iex`, `invoke-expression`, `start-process`,
`add-type`, etc.) and Agent tool rules (preventing delegation attacks).

**Design rationale:** Users who set `Bash(python:*)` for convenience in default mode
should not have that rule silently bypass the classifier in auto mode. The stripping
is transparent — rules are logged to debug output and restored on exit.

### L7: Sandbox Enforcement

OS-level filesystem and network isolation wraps command execution:

- **Filesystem**: Allowlists/denylists derived from permission rules, with hardcoded
  protections for settings files, `.claude/skills`, and git bare repo files
- **Network**: Domain allowlists extracted from WebFetch rules
- **Git security**: Detects and scrubs planted bare repository files (`HEAD`, `objects`,
  `refs`, `hooks`, `config`) to prevent `core.fsmonitor` escape attacks

Platform support: macOS (seatbelt), Linux (bubblewrap/seccomp), WSL2+. The sandbox is
explicitly **not** treated as a security boundary for excluded commands — the
permission prompt is the actual control.

**Design rationale:** Even if all software-level checks are bypassed, the sandbox
constrains what the process can actually do at the OS level. This is the only layer
that operates below the application.

### L8: Mode Transformation

After all tool-specific checks, the permission mode applies a final transformation:

| Mode | `ask` becomes | Effect |
|------|--------------|--------|
| `dontAsk` | `deny` | Silent rejection |
| `auto` | Classifier evaluation | AI decides |
| `acceptEdits` | `allow` (for edits in CWD) | File operations pass through |
| `plan` | Prompt | Always show intent |
| `default` | Prompt | Interactive dialog |
| `bypassPermissions` | `allow` | Skip (except safety checks) |

Non-classifier-approvable safety checks remain immune across all modes.

### L9: Hooks

An extensible hook system with **26 hook types** allows external code to inject
decisions at every stage of the tool lifecycle:

- **Permission hooks**: Can allow (with modified input), deny, or defer
- **Pre/post tool hooks**: Run before/after execution
- **Session lifecycle hooks**: Start, end, compact, config change
- **User prompt hooks**: Can block or modify user input before processing

Hooks support 6 implementation types (shell command, LLM prompt, agent, HTTP, callback,
function) and are loaded from 7 sources. All hooks require **workspace trust** in
interactive mode — a centralized RCE prevention gate.

**Design rationale:** Enables enterprise integrations, CI/CD pipelines, and custom
safety policies without modifying core code. The workspace trust requirement prevents
untrusted projects from injecting executable hooks.

### L10: AI Classifier (Auto Mode)

A 2-stage LLM pipeline replaces human approval in auto mode:

- **Stage 1 (fast)**: Conservative, low-token-budget evaluation. If it allows, return
  immediately. If it blocks, escalate.
- **Stage 2 (thinking)**: Full chain-of-thought reasoning with 4096 token budget.
  Can override stage 1 blocks (reducing false positives).

Three fast paths skip the classifier entirely: non-classifier-approvable safety checks
(deny), acceptEdits-equivalent operations (allow), and a safe tool allowlist (allow).

The classifier sees the full conversation transcript but **excludes assistant text**
to prevent self-influence. User auto-mode rules (allow/soft_deny/environment) are
merged into the classifier's system prompt.

**Failure modes**: API error with fail-closed default (deny, controlled by
`iron_gate_closed` gate), transcript too long (fall back to user prompt),
circuit breaker (prevents re-entry after remote disable).

**Design rationale:** Replaces human judgment with AI judgment for unattended operation,
but with multiple guardrails: fast paths for obvious cases, conservative stage 1,
deliberate stage 2, and explicit failure mode handling.

### L11: Denial Limit Tracking

If the classifier denies too many actions, the system forces fallback to human review:

- **3 consecutive denials** → fall back to interactive prompting
- **20 total denials per session** → fall back to interactive prompting

Consecutive denials reset on any successful tool use. In headless mode, exceeding
limits throws an abort error rather than silently continuing.

Main agents and subagents maintain **isolated** denial state to prevent cross-agent
interference.

**Design rationale:** Prevents the classifier from being exploited by repeatedly
probing with variations of a dangerous command. The fixed thresholds are a circuit
breaker, not a complete solution — the value is in forcing a human checkpoint.

### L12: Human-in-the-Loop Dialog

When the system determines a human must approve, it runs a **5-way race** where the
first responder wins:

1. **Local terminal input** — user types in the CLI
2. **IDE bridge** — approval from VS Code / JetBrains
3. **Channel relay** — approval via Telegram / iMessage / etc.
4. **Permission hooks** — external system decision
5. **Async classifier** — background auto-approval

An atomic `claim()` guard prevents race conditions. A 200ms grace period prevents
accidental keypresses from canceling the classifier. If the classifier wins, a
checkmark is briefly displayed (3s focused, 1s unfocused) that the user can dismiss.

Three handler variants serve different contexts: interactive (parallel 5-way race),
coordinator (sequential hooks → classifier → dialog), and swarm worker (classifier
locally → forward to leader).

**Design rationale:** Meeting users where they are. The race model means the fastest
available decision source wins, but the human always has override capability.

### L13: Decision Audit Log

Every permission decision flows through centralized logging to three sinks:

1. **Analytics events** (Statsig/Datadog) — 9 distinct event types covering grants,
   denials, classifier decisions, and limit breaches
2. **OpenTelemetry** — structured `tool_decision` events with code-edit language metrics
3. **In-session store** — `toolDecisions` Map for deduplication and OTel span attribution

The auto mode classifier logs extensive overhead telemetry: token usage, latency, cost,
stage breakdown, prompt component lengths, and session-total ratios for computing
classifier overhead percentage.

**Design rationale:** Detective control. Doesn't prevent harm but enables post-hoc
analysis of classifier accuracy, user override frequency, failure mode distribution,
and cost overhead.

### L14: Query Pipeline Safety

The query loop itself contains safety mechanisms:

- **Withheld error recovery**: Prompt-too-long (413), media size, and max-output-token
  errors are withheld from the stream and routed through a recovery cascade (context
  collapse drain → reactive compact → surface error).
- **Tool result budget**: Per-message aggregate size limits prevent token overflow;
  large results are persisted to disk as file references.
- **Infinite retry prevention**: 6 mechanisms including max recovery limits, one-shot
  guards, stop-hook blocking preservation, and abort signal propagation.

**Design rationale:** Prevents the system from entering unbounded error-recovery loops
that could burn resources or produce incoherent output. Each recovery path has an
explicit termination condition.

## 5. Auto Mode in Detail

Auto mode is the most architecturally complex layer because it replaces human judgment
with AI judgment while preserving safety guarantees.

**Entry**: Feature flag enabled + user selects `auto` mode + tool check returns `ask`.

**Decision cascade**:
1. Non-classifier-approvable safety checks → DENY (always)
2. acceptEdits equivalence test → ALLOW (skip classifier for safe edits)
3. Safe tool allowlist → ALLOW (skip classifier for read-only tools)
4. 2-stage classifier → ALLOW or DENY (with full reasoning)
5. Denial limit check → fall back to human if limits exceeded
6. Headless agents → auto-deny if no hook provides a decision

**Classifier pipeline**:
- System prompt: base rules + permissions template + user allow/deny/environment rules
- Transcript: user messages + assistant tool_use blocks (no assistant text)
- Stage 1: 64 tokens, stop on `</block>`, conservative bias
- Stage 2: 4096 tokens, chain-of-thought, can reverse stage 1 blocks
- Same model as the main agent loop

**Failure handling**: API errors default to fail-closed (deny), controllable via
feature gate. Transcript overflow falls back to user prompt. Circuit breaker prevents
re-entry after remote disable.

## 6. The Interactive Permission Race

When a human must decide, 5 decision sources race in parallel:

```
┌────────────────────────────────────────────┐
│       Permission Prompt Dialog              │
├────────────────────────────────────────────┤
│ 1. Terminal input (direct user interaction) │
│ 2. IDE bridge (VS Code / JetBrains)        │
│ 3. Channel relay (Telegram / iMessage)     │
│ 4. Permission hooks (external systems)     │
│ 5. Async classifier (auto-approve)         │
└────────────────────────────────────────────┘
              │ first wins
              ▼
       PermissionDecision
```

An atomic `claim()` primitive ensures exactly one source resolves the decision. The
200ms grace period prevents accidental classifier cancellation. Cleanup handlers on
abort signals prevent resource leaks.

Three handler variants adapt to context: interactive agents get the full 5-way race,
coordinator workers get sequential automation, and swarm workers delegate to their
leader via mailbox IPC.

## 7. Audit Trail and Telemetry

Every decision fans out to analytics, OpenTelemetry, and an in-session store. The
classifier logs ~40 telemetry fields per decision including per-stage token usage,
latency, cost, prompt component lengths, and session-total ratios.

Metadata types are restricted to `boolean | number` to prevent accidental code/filepath
logging, enforced by a marker type (`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`).

## 8. Summary: How the Layers Compose

| Layer | Catches | Weakness |
|-------|---------|----------|
| System prompt | Model generating obviously dangerous actions | Model can be manipulated |
| Input validation | Malformed inputs, UNC path attacks | Only structural + known semantic errors |
| Permission rules | Known-bad patterns (user-defined) | Requires user anticipation |
| Tool-specific checks | Domain violations (paths, symlinks, protected files) | Only covers known domains |
| Bash security scanner | Injection attacks (23 categories, parser differentials) | Novel attack patterns |
| Pattern stripping | Overly broad rules bypassing classifier | Only applies at mode entry |
| Sandbox | Filesystem/network access outside allowed scope | Platform-dependent, excludable |
| Mode transformation | Wrong trust posture for task | User must choose correctly |
| Hooks | External system veto/approval | Requires configuration |
| AI classifier | Contextual risk with full transcript | Hallucination, context limits |
| Denial limits | Repeated classifier exploitation | Fixed thresholds |
| Human approval | Everything automated layers miss | Fatigue, approval blindness |
| Audit logging | Nothing (detective, not preventive) | Only useful after the fact |
| Query pipeline | Error spirals, token overflow, runaway retries | Only known recovery patterns |

The architectural bet: the model might be tricked, but the scanner isn't. The scanner
might miss a novel pattern, but the classifier sees the full transcript. The classifier
might hallucinate, but the denial limits force human review. The human might approve
carelessly, but the sandbox constrains execution. The sandbox might be bypassed, but
the audit trail enables post-hoc detection.

Defense in depth applied to agentic AI — not a single perfect gate, but many imperfect
gates where aligned holes are exponentially unlikely.

---

# Part II: Implementation Reference

This section maps each design concept to its exact location in the source code.
All paths are relative to `src/`.

## Permission Decision Types

| Type | Definition | Location |
|------|-----------|----------|
| `PermissionAllowDecision` | Allow with optional modified input | `types/permissions.ts:174` |
| `PermissionAskDecision` | Prompt with pending classifier check | `types/permissions.ts:199` |
| `PermissionDenyDecision` | Block with message and reason | `types/permissions.ts:231` |
| `PermissionDecisionReason` | Tagged union of 10 reason types | `types/permissions.ts:271-324` |
| `PermissionMode` | 6 modes (5 external + auto) | `types/permissions.ts:16-29` |
| `PermissionRuleSource` | 8 priority sources | `types/permissions.ts:54-62` |
| `PermissionRule` | Source + behavior + value | `types/permissions.ts:75-79` |
| `ToolPermissionContext` | Immutable context carrying all rules | `types/permissions.ts:427-441` |
| `YoloClassifierResult` | Classifier output with stage info | `types/permissions.ts:346-397` |
| `ClassifierResult` | Bash classifier output | `types/permissions.ts:330-335` |

## L1: System Prompt — Source Locations

| Component | Location |
|-----------|----------|
| Main entry: `getSystemPrompt()` | `constants/prompts.ts:444` |
| Prompt injection defense | `constants/prompts.ts:191` |
| OWASP top 10 warning | `constants/prompts.ts:234` |
| Reversibility heuristic | `constants/prompts.ts:258` |
| Destructive operations list | `constants/prompts.ts:261-264` |
| Permission mode awareness | `constants/prompts.ts:189` |
| Dynamic boundary marker | `constants/prompts.ts:114-115` (inserted at 573) |
| `getSystemContext()` | `context.ts:116` |
| `getUserContext()` | `context.ts:155` |
| `getMemoryFiles()` | `utils/claudemd.ts:790` |
| `filterInjectedMemoryFiles()` | `utils/claudemd.ts:1142` |
| `getClaudeMds()` | `utils/claudemd.ts:1153` |
| `@include` directive parsing | `utils/claudemd.ts:451-535` |
| Circular ref prevention | `utils/claudemd.ts:630` (max depth 5, path Set) |
| `appendSystemContext()` | `utils/api.ts:437` |
| `prependUserContext()` | `utils/api.ts:449` |

## L2: Input Validation — Source Locations

| Component | Location |
|-----------|----------|
| `validateInput()` interface | `Tool.ts:489-492` |
| `ValidationResult` type | `Tool.ts:95-101` |
| Zod schema validation | `services/tools/toolExecution.ts:615` |
| `validateInput()` invocation | `services/tools/toolExecution.ts:683` |
| `formatZodValidationError()` | `utils/toolErrors.ts:66-132` |
| Error wrapping in `<tool_use_error>` | `services/tools/toolExecution.ts:664` |
| Tool interface: `isReadOnly()` | `Tool.ts:404` |
| Tool interface: `isDestructive()` | `Tool.ts:406` |
| Tool interface: `isOpenWorld()` | `Tool.ts:434` |
| Tool interface: `requiresUserInteraction()` | `Tool.ts:435` |
| Tool interface: `checkPermissions()` | `Tool.ts:500` |
| Tool interface: `toAutoClassifierInput()` | `Tool.ts:556` |
| Default implementations | `Tool.ts:757-769` |
| FileEditTool `validateInput` | `tools/FileEditTool/FileEditTool.ts:137-250` |
| BashTool `validateInput` | `tools/BashTool/BashTool.tsx:524-537` |
| NotebookEditTool `validateInput` | `tools/NotebookEditTool/NotebookEditTool.ts:176-290` |

## L3: Permission Rules — Source Locations

| Component | Location |
|-----------|----------|
| `checkRuleBasedPermissions()` | `utils/permissions/permissions.ts:1071` |
| `loadAllPermissionRulesFromDisk()` | `utils/permissions/permissionsLoader.ts:120` |
| `settingsJsonToRules()` | `utils/permissions/permissionsLoader.ts:91` |
| `parsePermissionRule()` | `utils/permissions/shellRuleMatching.ts:159` |
| `matchWildcardPattern()` | `utils/permissions/shellRuleMatching.ts:90` |
| `applyPermissionUpdate()` | `utils/permissions/PermissionUpdate.ts:55` |
| `persistPermissionUpdate()` | `utils/permissions/PermissionUpdate.ts:222` |
| `generateSuggestions()` | `utils/permissions/filesystem.ts:1414` |
| `suggestionForExactCommand()` | `utils/permissions/shellRuleMatching.ts:189` |

## L4: Tool-Specific Checks — Source Locations

| Tool | `checkPermissions` | Key Logic |
|------|--------------------|-----------|
| BashTool | `tools/BashTool/BashTool.tsx:539` → `tools/BashTool/bashPermissions.ts:1663` | AST parse, sandbox auto-allow, exact/prefix rules, Haiku classifier, pipe validation |
| FileEditTool | `tools/FileEditTool/FileEditTool.ts:125` → `utils/permissions/filesystem.ts:1205` | Deny rules (symlink-aware), path safety, `.claude/` session scoping |
| FileWriteTool | `tools/FileWriteTool/FileWriteTool.ts:135` → `utils/permissions/filesystem.ts:1205` | Same shared `checkWritePermissionForTool` |
| AgentTool | `tools/AgentTool/AgentTool.tsx:1281` | Allow in non-auto modes, passthrough in auto |
| MCPTool | `tools/MCPTool/MCPTool.ts:56` | Always passthrough |
| Path safety checks | `utils/permissions/filesystem.ts:620-665` | Windows patterns, Claude config, dangerous files |

## L5: Bash Security Scanner — Source Locations

| Component | Location |
|-----------|----------|
| Check ID enum (23 checks) | `tools/BashTool/bashSecurity.ts:77-101` |
| Command substitution patterns | `tools/BashTool/bashSecurity.ts:16-41` |
| Zsh dangerous commands | `tools/BashTool/bashSecurity.ts:45-74` |
| `bashCommandIsSafe_DEPRECATED()` (sync) | `tools/BashTool/bashSecurity.ts:2257` |
| `bashCommandIsSafeAsync_DEPRECATED()` (async) | `tools/BashTool/bashSecurity.ts:2426` |
| `validateGitCommit()` | `tools/BashTool/bashSecurity.ts:612` |
| `validateObfuscatedFlags()` | `tools/BashTool/bashSecurity.ts:1130` |
| `validateBraceExpansion()` | `tools/BashTool/bashSecurity.ts:1751` |
| `validateZshDangerousCommands()` | `tools/BashTool/bashSecurity.ts:2186` |
| `validateCommentQuoteDesync()` | `tools/BashTool/bashSecurity.ts:1990` |
| `validateQuotedNewline()` | `tools/BashTool/bashSecurity.ts:2109` |
| Tree-sitter analysis | `utils/bash/treeSitterAnalysis.ts` |
| `hasMalformedTokens()` | `utils/bash/shellQuote.ts:117` |
| `hasShellQuoteSingleQuoteBug()` | `utils/bash/shellQuote.ts:190` |
| Heredoc extraction | `utils/bash/heredoc.ts:113` |

## L6: Dangerous Pattern Stripping — Source Locations

| Component | Location |
|-----------|----------|
| `CROSS_PLATFORM_CODE_EXEC` | `utils/permissions/dangerousPatterns.ts:18-42` |
| `DANGEROUS_BASH_PATTERNS` | `utils/permissions/dangerousPatterns.ts:44-80` |
| `isDangerousBashPermission()` | `utils/permissions/permissionSetup.ts:94-147` |
| `isDangerousPowerShellPermission()` | `utils/permissions/permissionSetup.ts:157-233` |
| `isDangerousTaskPermission()` | `utils/permissions/permissionSetup.ts:240-245` |
| `stripDangerousPermissionsForAutoMode()` | `utils/permissions/permissionSetup.ts:510-553` |
| `restoreDangerousPermissions()` | `utils/permissions/permissionSetup.ts:561-579` |
| `transitionPermissionMode()` | `utils/permissions/permissionSetup.ts:597-646` |

## L7: Sandbox — Source Locations

| Component | Location |
|-----------|----------|
| `shouldUseSandbox()` | `tools/BashTool/shouldUseSandbox.ts:130-153` |
| `convertToSandboxRuntimeConfig()` | `utils/sandbox/sandbox-adapter.ts:172-381` |
| Settings file write protection | `utils/sandbox/sandbox-adapter.ts:230-236` |
| Git bare repo scrubbing | `utils/sandbox/sandbox-adapter.ts:257-280, 404-414` |
| Platform detection | `utils/sandbox/sandbox-adapter.ts:491-526` |
| `canSandboxAutoAllow` | `utils/permissions/permissions.ts:1186-1195` |
| SandboxPermissionRequest UI | `components/permissions/SandboxPermissionRequest.tsx` |
| Excluded commands (NOT a security boundary) | `tools/BashTool/shouldUseSandbox.ts:18-128` |

## L8: Mode Transformation — Source Locations

| Transformation | Location |
|---------------|----------|
| `hasPermissionsToUseTool()` outer | `utils/permissions/permissions.ts:473` |
| `hasPermissionsToUseToolInner()` | `utils/permissions/permissions.ts:1158` |
| `dontAsk` → `deny` | `utils/permissions/permissions.ts:503-517` |
| Auto mode entry | `utils/permissions/permissions.ts:519-927` |
| `acceptEdits` fast path | `utils/permissions/permissions.ts:600-656` |
| Safety check immunity | `utils/permissions/permissions.ts:532-548` |
| Headless agent auto-deny | `utils/permissions/permissions.ts:929-952` |

## L9: Hooks — Source Locations

| Component | Location |
|-----------|----------|
| `executePermissionRequestHooks()` | `utils/hooks.ts:4157` |
| `executePreToolHooks()` | `utils/hooks.ts:3394` |
| `executePostToolHooks()` | `utils/hooks.ts:3450` |
| `executeStopHooks()` | `utils/hooks.ts:3639` |
| `executeUserPromptSubmitHooks()` | `utils/hooks.ts:3826` |
| `executeSessionStartHooks()` | `utils/hooks.ts:3867` |
| Hook schema definitions | `schemas/hooks.ts` |
| Workspace trust security gate | `utils/hooks.ts:1992-1999` |
| Permission hook capability (allow/deny/modify input) | `utils/hooks.ts:617-622` |
| Hook result processing | `hooks/toolPermission/PermissionContext.ts:216-263` |

## L10: AI Classifier — Source Locations

| Component | Location |
|-----------|----------|
| `classifyYoloAction()` entry | `utils/permissions/yoloClassifier.ts:1012` |
| `classifyYoloActionXml()` 2-stage | `utils/permissions/yoloClassifier.ts:711-996` |
| `buildYoloSystemPrompt()` | `utils/permissions/yoloClassifier.ts:484-540` |
| `buildTranscriptEntries()` | `utils/permissions/yoloClassifier.ts:302-360` |
| `formatActionForClassifier()` | `utils/permissions/yoloClassifier.ts:1487` |
| Stage 1 suffix | `utils/permissions/yoloClassifier.ts:550` |
| Stage 2 suffix | `utils/permissions/yoloClassifier.ts:560-561` |
| Stage 1 logic | `utils/permissions/yoloClassifier.ts:771-857` |
| Stage 2 logic | `utils/permissions/yoloClassifier.ts:860-940` |
| Safe tool allowlist | `utils/permissions/classifierDecision.ts:56-94` |
| `isAutoModeAllowlistedTool()` | `utils/permissions/classifierDecision.ts:96` |
| Auto mode state / circuit breaker | `utils/permissions/autoModeState.ts:4-33` |
| Classifier invocation in pipeline | `utils/permissions/permissions.ts:693` |
| `iron_gate_closed` check | `utils/permissions/permissions.ts:848` |

## L11: Denial Limits — Source Locations

| Component | Location |
|-----------|----------|
| `DENIAL_LIMITS` | `utils/permissions/denialTracking.ts:12-15` |
| `DenialTrackingState` type | `utils/permissions/denialTracking.ts:7-10` |
| `recordDenial()` | `utils/permissions/denialTracking.ts:24` |
| `recordSuccess()` | `utils/permissions/denialTracking.ts:32` |
| `shouldFallbackToPrompting()` | `utils/permissions/denialTracking.ts:40` |
| `handleDenialLimitExceeded()` | `utils/permissions/permissions.ts:984-1058` |
| `persistDenialState()` | `utils/permissions/permissions.ts:963-978` |
| Headless mode abort | `utils/permissions/permissions.ts:1023-1027` |
| `recordSuccess` call sites | `utils/permissions/permissions.ts:496, 621, 661, 915` |
| `recordDenial` call site | `utils/permissions/permissions.ts:879` |

## L12: Human-in-the-Loop Dialog — Source Locations

| Component | Location |
|-----------|----------|
| `handleInteractivePermission()` | `hooks/toolPermission/handlers/interactiveHandler.ts:57` |
| `handleCoordinatorPermission()` | `hooks/toolPermission/handlers/coordinatorHandler.ts:26` |
| `handleSwarmWorkerPermission()` | `hooks/toolPermission/handlers/swarmWorkerHandler.ts:40` |
| `createResolveOnce()` | `hooks/toolPermission/PermissionContext.ts:75-94` |
| Bridge flow | `hooks/toolPermission/handlers/interactiveHandler.ts:244-298` |
| Channel relay flow | `hooks/toolPermission/handlers/interactiveHandler.ts:300-408` |
| Permission hooks race | `hooks/toolPermission/handlers/interactiveHandler.ts:411-431` |
| Classifier async race | `hooks/toolPermission/handlers/interactiveHandler.ts:434-530` |
| 200ms grace period | `hooks/toolPermission/handlers/interactiveHandler.ts:108-122` |
| Checkmark timer (3s/1s) | `hooks/toolPermission/handlers/interactiveHandler.ts:495-520` |
| BashPermissionRequest | `components/permissions/BashPermissionRequest/BashPermissionRequest.tsx` |
| FileEditPermissionRequest | `components/permissions/FileEditPermissionRequest/FileEditPermissionRequest.tsx` |
| PermissionDialog | `components/permissions/PermissionDialog.tsx` |
| PermissionPrompt | `components/permissions/PermissionPrompt.tsx` |
| PermissionExplanation (risk levels) | `components/permissions/PermissionExplanation.tsx:41-59` |
| SandboxPermissionRequest | `components/permissions/SandboxPermissionRequest.tsx` |
| Rule management | `components/permissions/rules/PermissionRuleList.tsx` |
| Recent denials | `components/permissions/rules/RecentDenialsTab.tsx` |
| Feedback collection | `components/permissions/useShellPermissionFeedback.ts:41-149` |

## L13: Decision Audit — Source Locations

| Component | Location |
|-----------|----------|
| `logPermissionDecision()` | `hooks/toolPermission/permissionLogging.ts:181` |
| `logApprovalEvent()` | `hooks/toolPermission/permissionLogging.ts:107` |
| `logRejectionEvent()` | `hooks/toolPermission/permissionLogging.ts:152` |
| `sourceToString()` | `hooks/toolPermission/permissionLogging.ts:68` |
| `isCodeEditingTool()` | `hooks/toolPermission/permissionLogging.ts:35` |
| `tengu_tool_use_granted_in_config` | `hooks/toolPermission/permissionLogging.ts:116` |
| `tengu_tool_use_granted_by_classifier` | `hooks/toolPermission/permissionLogging.ts:126` |
| `tengu_tool_use_granted_in_prompt_*` | `hooks/toolPermission/permissionLogging.ts:135-136` |
| `tengu_tool_use_granted_by_permission_hook` | `hooks/toolPermission/permissionLogging.ts:141` |
| `tengu_tool_use_denied_in_config` | `hooks/toolPermission/permissionLogging.ts:161` |
| `tengu_tool_use_rejected_in_prompt` | `hooks/toolPermission/permissionLogging.ts:166` |
| `tengu_auto_mode_decision` | `utils/permissions/permissions.ts:626, 666, 733` |
| `tengu_auto_mode_denial_limit_exceeded` | `utils/permissions/permissions.ts:1009` |
| `logEvent()` analytics sink | `services/analytics/index.ts:133` |
| `logOTelEvent()` | `utils/telemetry/events.ts:21` |
| In-session `toolDecisions` Map | `Tool.ts:258-265` |

## L14: Query Pipeline Safety — Source Locations

| Component | Location |
|-----------|----------|
| Main query loop | `query.ts:307-1728` |
| Withheld prompt-too-long | `query.ts:801-809` |
| Withheld media size errors | `query.ts:815-818` |
| Withheld max output tokens | `query.ts:820-822` |
| Context collapse drain recovery | `query.ts:1085-1117` |
| Reactive compact recovery | `query.ts:1119-1166` |
| `applyToolResultBudget()` | `query.ts:379-394` |
| Max output tokens recovery limit (3) | `query.ts:164, 1223` |
| Reactive compact guard | `query.ts:1121, 1157` |
| Stop hook blocking guard | `query.ts:1292-1296` |
| Stop failure hooks (error spiral prevention) | `query.ts:1263` |
| Abort signal breaks | `query.ts:1015-1052` |
| Post-sampling hooks | `query.ts:999-1009` |
