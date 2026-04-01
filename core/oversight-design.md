# Oversight — Design Document

## Part A: Design Document

*(All code-level details removed. This is purely the design, architecture, strategy, and conceptual content.)*

---

# Claude Code: Oversight & Human-in-the-Loop System Design

This document describes the oversight architecture of Claude Code -- Anthropic's agentic CLI tool -- focusing on how it implements a **Swiss cheese model** of layered safety defenses, how auto mode works, and how human oversight is maintained throughout.

---

## 1. Design Philosophy: The Swiss Cheese Model

The [Swiss cheese model](https://en.wikipedia.org/wiki/Swiss_cheese_model) originates from risk management in aviation and medicine. Multiple independent defensive layers are stacked in sequence. Each layer has "holes" -- weaknesses or blind spots -- but the holes are at different positions across layers, so a hazard must pass through aligned holes in *every* layer simultaneously to cause harm.

Claude Code applies this to agentic AI oversight with 14 layers. The core design principle: **no two adjacent layers share the same weakness**. An injection attack that fools the model still hits a deterministic security scanner. A scanner bypass still hits an AI classifier reviewing the full transcript. A classifier hallucination still forces human review after repeated denials. A careless human approval is still constrained by an OS-level sandbox. A sandbox escape is still logged for post-hoc detection.

```
     Hazard (dangerous tool invocation)
        |
        v
+----------------------------+
| L1  System Prompt          |  Model-level safety instructions
+----------------------------+
| L2  Input Validation       |  Schema + semantic checks, fail-closed
+----------------------------+
| L3  Permission Rules       |  User-defined deny/allow/ask rules
+----------------------------+
| L4  Tool-Specific Checks   |  Domain logic (paths, protected files)
+----------------------------+
| L5  Bash Security Scanner  |  23 deterministic injection checks
+----------------------------+
| L6  Pattern Stripping      |  Removes overly broad rules at auto entry
+----------------------------+
| L7  Sandbox                |  OS-level filesystem/network isolation
+----------------------------+
| L8  Mode Transformation    |  dontAsk->deny, plan->prompt, auto->classifier
+----------------------------+
| L9  Hooks                  |  Extensible external allow/deny injection
+----------------------------+
| L10 AI Classifier          |  2-stage LLM with fast+thinking pipeline
+----------------------------+
| L11 Denial Limits          |  3 consecutive / 20 total -> force human
+----------------------------+
| L12 Human Approval         |  5-way interactive race
+----------------------------+
| L13 Audit Log              |  Tagged decisions + 3 telemetry sinks
+----------------------------+
| L14 Query Pipeline Safety  |  Error recovery, budget enforcement
+----------------------------+
        |
        v
     Action executes (or is blocked)
```

## 2. Permission Modes

The outermost design choice is the user's **permission mode** -- a coarse trust posture that governs how all downstream layers behave:

| Mode | Behavior | Use Case |
|------|----------|----------|
| `default` | Prompt user for any sensitive operation | Normal interactive use |
| `plan` | Suspend execution, show intent first | Architecture exploration |
| `acceptEdits` | Auto-allow file edits in working directory | Trusted editing sessions |
| `dontAsk` | Convert all `ask` decisions to `deny` | Read-only exploration |
| `auto` | Replace user prompts with AI classifier | Unattended operation |
| `bypassPermissions` | Skip most checks (kill-switch gated) | Development/debugging |

The `auto` mode is feature-gated and only available in builds with the transcript classifier feature flag. Mode transitions trigger side effects: entering auto mode strips dangerous allow rules; exiting restores them.

## 3. The Decision Pipeline

Every tool invocation passes through a central permission function that produces one of four outcomes:

- **Allow** -- proceed with (optionally modified) input
- **Ask** -- show interactive dialog, optionally with a pending classifier check
- **Deny** -- block with a message
- **Passthrough** -- internal intermediate state; continue checking

Every decision carries a typed **reason tag** (rule, mode, hook, classifier, safetyCheck, sandboxOverride, asyncAgent, workingDir, other) so the system maintains a complete audit trail of *why* each decision was made.

## 4. The Layers

### L1: System Prompt Safety Instructions

Before any tool runs, the model receives explicit safety instructions embedded in the system prompt. These cover:

- **Prompt injection awareness** -- flag suspicious tool results to the user
- **OWASP top 10** -- avoid generating code with command injection, XSS, SQL injection
- **Reversibility heuristic** -- consider blast radius; confirm destructive, hard-to-reverse, or externally-visible actions
- **Explicit risky-action taxonomy** -- destructive (rm -rf, drop table), hard-to-reverse (force-push, reset --hard), externally-visible (push, send message, post to API)
- **Permission mode awareness** -- don't re-attempt denied tool calls

The system prompt is split into static (cacheable) and dynamic (session-specific) halves by a boundary marker, so safety instructions remain in the cached prefix and are never evicted.

Context is assembled from CLAUDE.md files loaded in priority order (managed, user, project, local, auto-memory, team-memory), with `@include` directive support and circular reference prevention (max depth 5, path deduplication).

**Design rationale:** Probabilistic but broad coverage. The model might not always follow instructions, but the instructions establish the expected behavior baseline and catch the majority of "obvious" risks.

### L2: Tool Input Validation

A two-level validation pipeline runs before any permission logic:

1. **Structural validation** (schema-based) -- catches type errors, missing/extra fields
2. **Semantic validation** (per-tool) -- catches domain-specific errors like file-not-found, UNC path NTLM credential leaks, read-before-edit violations, encoding issues, and file size limits

Both levels are **fail-closed**: any validation failure returns an error to the model. The tool never executes.

**Design rationale:** Catches malformed or nonsensical tool calls before they enter the permission system. Prevents the model from accidentally crafting inputs that exploit downstream assumptions.

### L3: Rule-Based Permission Hierarchy

Users configure deny/allow/ask rules from 8 sources, loaded in priority order:

```
policySettings -> flagSettings -> projectSettings -> userSettings
     -> localSettings -> cliArg -> command -> session
```

Evaluation order: **deny rules first** (immediate block), then **ask rules** (prompt), then **tool's own permission check**, then **allow rules** (approve). This ensures that explicit denials always win.

Rules support three pattern types: exact match, legacy prefix, and wildcard. Tools provide pattern matchers for domain-specific matching (file paths, command prefixes).

When a rule triggers an `ask` decision, the system generates **suggestions** -- pre-built permission updates the user can accept to avoid being asked again.

**Design rationale:** Gives users explicit, persistent control. Deny rules can block entire tool categories. The priority ordering means organizational policies always win over personal settings.

### L4: Tool-Specific Permission Checks

Each tool implements domain-specific safety logic:

- **File tools**: Check paths against working directory, resolve symlinks, normalize case, block `..` traversal, guard sensitive paths (`.git/`, `.claude/`, `.vscode/`, SSH keys) with **safetyCheck** decisions that are immune to bypass mode
- **Bash tool**: AST-based security parse, sandbox auto-allow logic, classifier for command description matching, pipe/redirect validation
- **Agent tool**: Auto-allows in non-auto modes, defers to classifier in auto mode
- **MCP tool**: Always defers to the general permission framework

The `safetyCheck` decision type has a `classifierApprovable` boolean: `false` means the check is **immune to all auto-approve paths** including the AI classifier (e.g., Windows path bypass attempts). `true` means the classifier can evaluate it (e.g., sensitive file paths where context matters).

**Design rationale:** Domain experts know their risks best. File tools understand path traversal; the bash tool understands shell injection. The general permission system can't replicate this domain knowledge.

### L5: Bash Security Scanner

A dedicated injection detection engine with **23 numbered check categories** covering:

- Command substitution (subshell, backtick, parameter expansion, process substitution, zsh equals substitution)
- Shell metacharacter injection (`;`, `|`, `&` in arguments)
- Obfuscated flags (ANSI-C quoting, empty-quote tricks)
- Parser differentials (shell-quote vs bash tokenization mismatches)
- Variable injection in redirections/pipes
- IFS injection, brace expansion, Unicode whitespace
- Zsh-specific: module loading attacks, eval equivalents, file/network I/O bypass
- Comment-quote desynchronization, quoted newline hiding
- `/proc/environ` access, control characters

The scanner uses AST-based analysis when available, falling back to regex-based parsing. It detects attacks that exploit **differential behavior** between the command parser used for permission checking and bash's actual evaluation.

**Design rationale:** Deterministic and not susceptible to model manipulation. Covers a large surface area of known injection techniques. The AST/regex dual path provides defense in depth within the layer itself.

### L6: Dangerous Pattern Stripping

When entering auto mode, the system proactively **strips overly broad allow rules** that would let the classifier be bypassed entirely. Rules matching code-execution entry points (python, node, bash, ssh, curl, etc.) in any of five pattern shapes (exact, prefix, trailing/space/flag wildcards) are removed and stashed for restoration on exit.

This also covers PowerShell cmdlets (invoke-expression, start-process, add-type, etc.) and Agent tool rules (preventing delegation attacks).

**Design rationale:** Users who set broad bash permissions for convenience in default mode should not have those rules silently bypass the classifier in auto mode. The stripping is transparent -- rules are logged to debug output and restored on exit.

### L7: Sandbox Enforcement

OS-level filesystem and network isolation wraps command execution:

- **Filesystem**: Allowlists/denylists derived from permission rules, with hardcoded protections for settings files, skill files, and git bare repo files
- **Network**: Domain allowlists extracted from web fetch rules
- **Git security**: Detects and scrubs planted bare repository files (HEAD, objects, refs, hooks, config) to prevent `core.fsmonitor` escape attacks

Platform support: macOS (seatbelt), Linux (bubblewrap/seccomp), WSL2+. The sandbox is explicitly **not** treated as a security boundary for excluded commands -- the permission prompt is the actual control.

**Design rationale:** Even if all software-level checks are bypassed, the sandbox constrains what the process can actually do at the OS level. This is the only layer that operates below the application.

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

An extensible hook system allows external code to inject decisions at every stage of the tool lifecycle:

- **Permission hooks**: Can allow (with modified input), deny, or defer
- **Pre/post tool hooks**: Run before/after execution
- **Session lifecycle hooks**: Start, end, compact, config change
- **User prompt hooks**: Can block or modify user input before processing

Hooks support multiple implementation types (shell command, LLM prompt, agent, HTTP, callback, function) and are loaded from multiple sources. All hooks require **workspace trust** in interactive mode -- a centralized RCE prevention gate.

**Design rationale:** Enables enterprise integrations, CI/CD pipelines, and custom safety policies without modifying core code. The workspace trust requirement prevents untrusted projects from injecting executable hooks.

### L10: AI Classifier (Auto Mode)

A 2-stage LLM pipeline replaces human approval in auto mode:

- **Stage 1 (fast)**: Conservative, low-token-budget evaluation. If it allows, return immediately. If it blocks, escalate.
- **Stage 2 (thinking)**: Full chain-of-thought reasoning with larger token budget. Can override stage 1 blocks (reducing false positives).

Three fast paths skip the classifier entirely: non-classifier-approvable safety checks (deny), acceptEdits-equivalent operations (allow), and a safe tool allowlist (allow).

The classifier sees the full conversation transcript but **excludes assistant text** to prevent self-influence. User auto-mode rules (allow/soft_deny/environment) are merged into the classifier's system prompt.

**Failure modes**: API error with fail-closed default (deny, controllable via feature gate), transcript too long (fall back to user prompt), circuit breaker (prevents re-entry after remote disable).

**Design rationale:** Replaces human judgment with AI judgment for unattended operation, but with multiple guardrails: fast paths for obvious cases, conservative stage 1, deliberate stage 2, and explicit failure mode handling.

### L11: Denial Limit Tracking

If the classifier denies too many actions, the system forces fallback to human review:

- **3 consecutive denials** -> fall back to interactive prompting
- **20 total denials per session** -> fall back to interactive prompting

Consecutive denials reset on any successful tool use. In headless mode, exceeding limits throws an abort error rather than silently continuing.

Main agents and subagents maintain **isolated** denial state to prevent cross-agent interference.

**Design rationale:** Prevents the classifier from being exploited by repeatedly probing with variations of a dangerous command. The fixed thresholds are a circuit breaker, not a complete solution -- the value is in forcing a human checkpoint.

### L12: Human-in-the-Loop Dialog

When the system determines a human must approve, it runs a **5-way race** where the first responder wins:

1. **Local terminal input** -- user types in the CLI
2. **IDE bridge** -- approval from VS Code / JetBrains
3. **Channel relay** -- approval via Telegram / iMessage / etc.
4. **Permission hooks** -- external system decision
5. **Async classifier** -- background auto-approval

An atomic claim guard prevents race conditions. A 200ms grace period prevents accidental keypresses from canceling the classifier. If the classifier wins, a checkmark is briefly displayed (3s focused, 1s unfocused) that the user can dismiss.

Three handler variants serve different contexts: interactive (parallel 5-way race), coordinator (sequential hooks then classifier then dialog), and swarm worker (classifier locally then forward to leader).

**Design rationale:** Meeting users where they are. The race model means the fastest available decision source wins, but the human always has override capability.

### L13: Decision Audit Log

Every permission decision flows through centralized logging to three sinks:

1. **Analytics events** -- 9 distinct event types covering grants, denials, classifier decisions, and limit breaches
2. **OpenTelemetry** -- structured tool decision events with code-edit language metrics
3. **In-session store** -- a decision map for deduplication and span attribution

The auto mode classifier logs extensive overhead telemetry: token usage, latency, cost, stage breakdown, prompt component lengths, and session-total ratios for computing classifier overhead percentage.

**Design rationale:** Detective control. Doesn't prevent harm but enables post-hoc analysis of classifier accuracy, user override frequency, failure mode distribution, and cost overhead.

### L14: Query Pipeline Safety

The query loop itself contains safety mechanisms:

- **Withheld error recovery**: Prompt-too-long, media size, and max-output-token errors are withheld from the stream and routed through a recovery cascade (context collapse drain, reactive compact, surface error).
- **Tool result budget**: Per-message aggregate size limits prevent token overflow; large results are persisted to disk as file references.
- **Infinite retry prevention**: 6 mechanisms including max recovery limits, one-shot guards, stop-hook blocking preservation, and abort signal propagation.

**Design rationale:** Prevents the system from entering unbounded error-recovery loops that could burn resources or produce incoherent output. Each recovery path has an explicit termination condition.

## 5. Auto Mode in Detail

Auto mode is the most architecturally complex layer because it replaces human judgment with AI judgment while preserving safety guarantees.

**Entry**: Feature flag enabled + user selects `auto` mode + tool check returns `ask`.

**Decision cascade**:
1. Non-classifier-approvable safety checks -> DENY (always)
2. acceptEdits equivalence test -> ALLOW (skip classifier for safe edits)
3. Safe tool allowlist -> ALLOW (skip classifier for read-only tools)
4. 2-stage classifier -> ALLOW or DENY (with full reasoning)
5. Denial limit check -> fall back to human if limits exceeded
6. Headless agents -> auto-deny if no hook provides a decision

**Classifier pipeline**:
- System prompt: base rules + permissions template + user allow/deny/environment rules
- Transcript: user messages + assistant tool_use blocks (no assistant text)
- Stage 1: 64 tokens, conservative bias
- Stage 2: 4096 tokens, chain-of-thought, can reverse stage 1 blocks
- Same model as the main agent loop

**Failure handling**: API errors default to fail-closed (deny), controllable via feature gate. Transcript overflow falls back to user prompt. Circuit breaker prevents re-entry after remote disable.

## 6. The Interactive Permission Race

When a human must decide, 5 decision sources race in parallel:

```
+--------------------------------------------+
|       Permission Prompt Dialog             |
+--------------------------------------------+
| 1. Terminal input (direct user interaction)|
| 2. IDE bridge (VS Code / JetBrains)       |
| 3. Channel relay (Telegram / iMessage)    |
| 4. Permission hooks (external systems)    |
| 5. Async classifier (auto-approve)        |
+--------------------------------------------+
              | first wins
              v
       PermissionDecision
```

An atomic claim primitive ensures exactly one source resolves the decision. The 200ms grace period prevents accidental classifier cancellation. Cleanup handlers on abort signals prevent resource leaks.

Three handler variants adapt to context: interactive agents get the full 5-way race, coordinator workers get sequential automation, and swarm workers delegate to their leader via mailbox IPC.

## 7. Audit Trail and Telemetry

Every decision fans out to analytics, OpenTelemetry, and an in-session store. The classifier logs approximately 40 telemetry fields per decision including per-stage token usage, latency, cost, prompt component lengths, and session-total ratios.

Metadata types are restricted to boolean and number to prevent accidental code/filepath logging, enforced by a designated marker type.

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

The architectural bet: the model might be tricked, but the scanner isn't. The scanner might miss a novel pattern, but the classifier sees the full transcript. The classifier might hallucinate, but the denial limits force human review. The human might approve carelessly, but the sandbox constrains execution. The sandbox might be bypassed, but the audit trail enables post-hoc detection.

Defense in depth applied to agentic AI -- not a single perfect gate, but many imperfect gates where aligned holes are exponentially unlikely.
