# Permission Modes — Design Document

## 1. Overview

The permission mode system is a coarse-grained trust dial that shapes the entire permission decision pipeline. When a user starts Claude Code, they select a **permission mode** that expresses their intent: "I want to approve everything manually," "I trust file edits," "let the AI decide for me," or "skip checks entirely." The mode does not replace the permission rules, tool-specific safety checks, or the security scanner -- it sits atop all of them as a **transformation layer** that converts intermediate `ask` decisions into final outcomes.

The system exists to solve three fundamental tensions:

1. **Safety vs. productivity.** Developers want to work fast, but agentic tools that can execute shell commands, edit files, and spawn subagents must not silently perform destructive or irreversible operations.

2. **Interactive vs. unattended operation.** Claude Code runs both in terminal sessions (human at keyboard) and in CI/CD pipelines, background agents, and headless SDK contexts where no human can answer a permission prompt.

3. **Configuration vs. simplicity.** Power users want granular rule-based control. Casual users want a single toggle. The mode system provides the toggle; the rule hierarchy provides the granularity; and they compose cleanly.

The mode is the outermost ring of the permission system. Inside it, priority-ordered rule sources, tool-specific domain checks, a bash security scanner, an AI classifier, denial limit tracking, and an interactive multi-source approval dialog all participate. The mode determines which of these inner layers actually fire for a given decision.

## 2. The Six Modes

Claude Code defines six permission modes. Five are externally addressable (exposed in settings, CLI flags, and the IDE carousel). The sixth -- `auto` -- is feature-gated behind a build flag.

### 2.1 `default` -- Interactive Approval

| Property | Value |
|----------|-------|
| Trust level | Low |
| `ask` becomes | Interactive prompt |
| Use case | Normal development sessions |
| Feature gate | None |

In default mode, every tool invocation that is not explicitly allowed by a rule or tool-specific logic requires human approval. The user sees an interactive permission dialog showing the tool name, the proposed action, a risk-level explanation, and suggested rule updates they can accept to avoid future prompts for the same pattern.

This is the baseline posture. Every safety layer is active: deny rules block, ask rules prompt, tool-specific checks run, and unapproved operations require explicit human consent.

### 2.2 `plan` -- Intent Display

| Property | Value |
|----------|-------|
| Trust level | Low (read-only by default) |
| `ask` becomes | Interactive prompt |
| Use case | Architecture exploration, code review |
| Feature gate | None |

Plan mode suspends destructive execution and shows the model's intended actions before running them. The model can still use read-only tools freely. When the user exits plan mode, the system restores the previous mode (stashed as a "pre-plan mode" reference on the context).

Plan mode has a special interaction with auto mode: when the user has opted in to auto mode and enabled "use auto mode during plan" in settings, the classifier runs during plan mode. In this configuration, plan mode functions as auto mode with an intent-display overlay -- the classifier evaluates actions, but the user sees the plan first.

When transitioning to plan mode from bypass mode, auto-mode semantics are explicitly suppressed to prevent the classifier from silently approving actions that bypass mode would have auto-allowed.

### 2.3 `acceptEdits` -- Trusted File Editing

| Property | Value |
|----------|-------|
| Trust level | Medium |
| `ask` becomes | `allow` for file edits in working directory; prompt for everything else |
| Use case | Focused coding sessions with trusted edit scope |
| Feature gate | None |

In acceptEdits mode, file write and edit operations within the current working directory (and any additional working directories) are auto-approved. Shell commands, agent spawns, MCP tool calls, and file operations outside the working directory still require interactive approval.

The mode is implemented not as a special case in the mode transformation, but by re-running each tool's permission check method with a synthetic acceptEdits-mode context. This means tool-specific safety checks (such as protected path detection for `.git/`, `.claude/`, and shell configuration files) still fire even in acceptEdits mode.

### 2.4 `dontAsk` -- Silent Denial

| Property | Value |
|----------|-------|
| Trust level | Read-only |
| `ask` becomes | `deny` |
| Use case | Exploration, read-only auditing |
| Feature gate | None |

The `dontAsk` mode converts every `ask` decision into a `deny`. The model receives a rejection message and must find an alternative approach that uses only already-allowed tools. This is useful when the user wants to explore a codebase without granting any new permissions and without being interrupted by prompts.

The transformation is applied as the very last step in the permission pipeline, after all rule-based and tool-specific checks have run. This ensures that deny rules still fire with their proper reason tags (rather than being masked by the mode transformation), and that allow rules still permit explicitly trusted operations.

### 2.5 `auto` -- AI Classifier Approval

| Property | Value |
|----------|-------|
| Trust level | High (with AI oversight) |
| `ask` becomes | Classifier evaluation |
| Use case | Unattended operation, CI/CD, long-running tasks |
| Feature gate | Build flag + dynamic config |

Auto mode replaces human approval with an AI classifier that evaluates each proposed action against the full conversation transcript. This is the most architecturally complex mode, with its own 2-stage classifier pipeline, three fast paths, dangerous pattern stripping, denial limit tracking, and multiple failure modes.

Auto mode is gated by a build flag and additionally controlled by a dynamic config that can enable, disable (circuit breaker), or set the mode to opt-in. The mode is not available on all models -- a compatibility check runs at initialization. Auto mode can also be disabled via settings. A three-state availability system (`enabled`, `disabled`, `opt-in`) controls whether the mode appears in the UI carousel and whether explicit entry is allowed.

### 2.6 `bypassPermissions` -- Developer Kill Switch

| Property | Value |
|----------|-------|
| Trust level | Maximum |
| `ask` becomes | `allow` (with exceptions) |
| Use case | Development, debugging, trusted environments |
| Feature gate | Kill switch gated |

Bypass mode auto-approves nearly all tool invocations. However, three categories of checks remain immune:

1. **Deny rules** -- Explicit deny rules from any source still block.
2. **Content-specific ask rules** -- Rules like `Bash(npm publish:*)` that the user explicitly configured as `ask` are respected.
3. **Safety checks** -- Tool-specific safety checks marked as non-classifier-approvable are immune to bypass mode. These cover protected paths (`.git/`, `.claude/`, `.vscode/`, SSH keys), Windows path bypass attempts, and cross-machine bridge messages.

Bypass mode can be disabled by a feature gate, a settings field, or remote environment restrictions. When bypass mode is disabled after the session starts (async check), the system falls back to default mode. In extreme cases, the process performs a graceful shutdown.

When plan mode is entered from bypass mode, the system remembers that bypass was the origin mode. In this state, the bypass transformation still fires even though the current mode is `plan`.

## 3. Mode Transformation

The mode transformation is the final step in the permission decision pipeline. After all tool-specific checks, rule evaluations, and security scans have run, the system examines the result. If the result is `allow` or `deny`, it passes through unchanged -- the mode cannot override explicit decisions. If the result is `ask` (meaning no rule or check made a definitive decision), the mode determines the outcome:

```
Tool invocation
    |
    v
[1] Deny rules -> if match -> DENY (immediate, mode-independent)
[2] Ask rules -> if match -> provisional ASK
[3] Tool checkPermissions -> DENY/ASK/ALLOW/PASSTHROUGH
[4] Safety checks -> if safetyCheck -> immune to all modes
[5] User interaction check -> if requiresUserInteraction -> always ASK
[6] Content-specific ask rules -> bypass-immune ASK
[7] bypassPermissions check -> if bypass -> ALLOW (except above)
[8] Always-allow rules -> if match -> ALLOW
[9] Passthrough -> convert to ASK
[10] Mode transformation:
    +-- dontAsk -> DENY
    +-- auto -> Classifier pipeline
    +-- default/plan -> Interactive prompt
    +-- (acceptEdits: handled by tool at step [3] for file edits;
         non-file-edit ask decisions fall through to interactive prompt)
```

The transformation is deliberately positioned last so that it acts on the **residual** `ask` decisions -- those that no rule, check, or explicit permission resolved. This preserves the deny-first principle: explicit denials are never softened by a permissive mode.

## 4. Rule-Based Permission Hierarchy

Permission rules are loaded from eight sources, listed here in priority order (highest priority first):

| Priority | Source | Persistence | Editability |
|----------|--------|-------------|-------------|
| 1 | Policy settings | Managed deployment | Read-only |
| 2 | Feature flag settings | Feature flags | Read-only |
| 3 | Project settings | `.claude/settings.json` | Read-write |
| 4 | User settings | `~/.claude/settings.json` | Read-write |
| 5 | Local settings | `.claude/settings.local.json` | Read-write |
| 6 | CLI arguments | `--allowed-tools`, `--disallowed-tools` | Session-scoped |
| 7 | Command source | Internal command source | Read-only |
| 8 | Session | In-memory | Ephemeral |

Rules have three behaviors: `allow`, `deny`, and `ask`. Each rule specifies a tool name and optional content pattern (e.g., `Bash(git push:*)` matches any bash command starting with `git push`).

**Evaluation order is deny-first:**
1. Deny rules are checked first. If any deny rule matches, the tool invocation is immediately blocked with a `deny` decision, regardless of mode.
2. Ask rules are checked next. If an ask rule matches, the system proceeds to the tool's permission check method but marks the provisional decision as `ask`.
3. The tool's own permission check runs, producing a domain-specific result.
4. Allow rules are checked last. If the tool returned `passthrough` (no opinion) and an allow rule matches, the invocation is approved.

This ordering ensures that organizational policies always override personal preferences, and that explicit denials are never overridden by allow rules from lower-priority sources.

When a managed-only flag is set in policy settings, all non-policy rule sources are cleared. This gives managed deployments complete control over the permission surface.

Rules support three pattern types for content matching:
- **Exact match**: `Bash(git commit)` matches only `git commit`
- **Legacy prefix**: `Bash(git:*)` matches any command starting with `git`
- **Wildcard**: `Bash(git *)` with glob-style `*` matching

Tools provide custom pattern matching implementations for domain-specific matching. The bash tool, for instance, handles subcommand decomposition (piped commands are split and each segment matched independently).

## 5. Auto Mode

Auto mode is the most complex subsystem within the permission framework. It replaces human judgment with an AI classifier while maintaining safety guarantees through multiple redundant mechanisms.

### 5.1 Entry Conditions

Auto mode activates when all of the following are true:
- The build flag for the classifier is enabled
- The user has selected `auto` mode (or is in `plan` mode with auto semantics active)
- The tool check returned `ask` (no rule made a definitive decision)
- The tool does not require user interaction
- The safety check is not non-classifier-approvable

### 5.2 Three Fast Paths

Before invoking the classifier API, three fast paths are checked:

1. **acceptEdits equivalence test**: The system re-runs the tool's permission check with a synthetic acceptEdits context. If the tool would be allowed in acceptEdits mode, it is allowed in auto mode without a classifier call. This covers file edits within the working directory. The Agent tool and REPL tool are excluded from this fast path because their permission checks return `allow` for acceptEdits mode, which would silently bypass the classifier for delegation and VM escape scenarios.

2. **Safe tool allowlist**: A hardcoded set of tools known to be safe (read-only file operations, search tools, task management, plan mode tools, sleep, etc.) are auto-approved without a classifier call.

3. **Empty classifier input**: If a tool declares no security-relevant input (returns an empty string from its classifier input method), the tool is auto-approved.

### 5.3 The 2-Stage Classifier Pipeline

When no fast path applies, the full classifier pipeline runs. It operates in three possible configurations controlled by dynamic config:

- **`both`** (default): Stage 1 followed by stage 2 on block
- **`fast`**: Stage 1 only (final verdict)
- **`thinking`**: Stage 2 only (skip stage 1)

**Stage 1 (fast)**: A conservative, low-token-budget evaluation. The system sends the transcript and action to the classifier model with a suffix that nudges an immediate block/no-block decision. Max tokens is set to 64 (plus thinking padding for always-on-thinking models). Stop sequences minimize output. If stage 1 allows (no block), the result is returned immediately -- this is the fast path for safe actions. If stage 1 blocks or is unparseable, the system escalates to stage 2.

**Stage 2 (thinking)**: A full chain-of-thought evaluation with 4096 token budget. The suffix instructs the model to follow the classification process carefully, use thinking before responding with a block decision, and verify that user confirmation must be explicit (not suggestive or implicit). Stage 2 can reverse stage 1 blocks, reducing false positives. Thinking content is stripped before parsing block and reason tags to prevent self-influence from chain-of-thought reasoning.

Both stages share the same system prompt and user content, benefiting from prompt caching (1-hour TTL) across calls. Cache control breakpoints are placed on the system prompt and the action block to maximize cache hit rates between stage 1 and stage 2.

### 5.4 Classifier Input Construction

The classifier receives:

1. **System prompt**: A base prompt describing the classification task, combined with a permissions template that contains allow rules, deny rules, and environment descriptions. User-configured auto-mode settings (allow, soft_deny, environment arrays) are substituted into tagged sections of the template. The external template's tagged sections are **replaced** by user values; the internal template's tagged sections are **additive**.

2. **User configuration prefix**: If available, the user's project configuration (CLAUDE.md) is sent as a prefix message with cache control. This gives the classifier context about the user's project intent.

3. **Transcript**: Built from the conversation history. Includes user text messages and assistant tool-use blocks. Each entry is serialized in a compact format. Tools control their representation via their classifier input method.

4. **Action**: The specific tool invocation being evaluated, appended as the final entry in the transcript.

### 5.5 Assistant Text Exclusion

A critical security property of the classifier input: **assistant text content is excluded from the transcript**. Only tool-use blocks from assistant messages are included. This prevents the model from crafting assistant text that influences the classifier's decision -- a form of self-influence attack where the agent writes persuasive text in one turn that the classifier reads in the next.

### 5.6 Failure Modes

The classifier has explicit handling for three failure scenarios:

1. **API error (transient)**: Retried internally (429, 500 errors). Non-transient errors return a block with unavailability marked. A feature gate determines fail-closed (deny) vs. fail-open (fall back to interactive prompting) behavior. The gate defaults to fail-closed and is refreshed periodically.

2. **Transcript too long**: The classifier's context window is exceeded. This is a deterministic condition (same transcript always overflows), so retrying will not help. The system falls back to interactive prompting. In headless mode, this throws an abort error since the condition is permanent and would otherwise waste tokens in a deny-retry-deny loop.

3. **Circuit breaker**: The dynamic config can be set to `disabled` for incident response. When the circuit breaker fires, auto mode is prevented from activating, and users already in auto mode are kicked out to default mode.

## 6. Dangerous Pattern Stripping

When entering auto mode, the system proactively strips overly broad allow rules that would bypass the classifier entirely. The rationale: a user who configured `Bash(python:*)` for convenience in default mode (to avoid being prompted for every Python command) should not have that rule silently bypass the classifier in auto mode, where the classifier is the primary safety gate.

### 6.1 What Gets Stripped

A rule is considered "dangerous" if it would auto-allow commands through code-execution entry points. The system checks for five pattern shapes:

1. **Exact match**: `python` as a rule
2. **Prefix syntax**: `python:*`
3. **Trailing wildcard**: `python*`
4. **Space wildcard**: `python *`
5. **Flag wildcard**: `python -*` (matches `python -c 'code'`)

The dangerous patterns include:

- **Cross-platform interpreters**: python, python3, node, deno, tsx, ruby, perl, php, lua
- **Package runners**: npx, bunx, `npm run`, `yarn run`, `pnpm run`, `bun run`
- **Shells**: bash, sh, zsh, fish
- **Execution primitives**: eval, exec, env, xargs, sudo
- **Remote execution**: ssh
- **PowerShell cmdlets**: iex, invoke-expression, invoke-command, start-process, add-type, new-object, pwsh, powershell, cmd, wsl, and their aliases
- **Agent tool**: Any `Agent` allow rule (prevents delegation attacks)
- **Tool-level allows**: `Bash` or `Bash(*)` with no content restriction

### 6.2 Stripping and Restoration

The stripping is implemented as a **stash-and-restore** pattern:

1. On entering auto mode, the system scans all allow rules, identifies dangerous ones, removes them from the active context, and stores them in a stash on the permission context.

2. Each stripped rule is logged to debug output with its display string and source.

3. On exiting auto mode, the system reads the stash, re-adds the rules to the context, and clears the stash. A second exit is a no-op.

The stripping only removes rules from the in-memory context -- it does not modify settings files on disk. The user's configured rules are preserved and restored when they leave auto mode.

### 6.3 Plan Mode Interaction

When plan mode activates auto semantics (via the "use auto mode during plan" setting), dangerous pattern stripping also applies. A reconciliation function handles the desired auto-mode state against the actual state when settings change mid-plan, re-stripping rules if auto is activated or restoring them if deactivated.

## 7. Denial Limit Tracking

The denial limit system is a circuit breaker that prevents the classifier from being exploited by repeatedly probing with variations of a dangerous command. After too many denials, the system forces fallback to human review.

### 7.1 Thresholds

Two independent thresholds trigger the fallback:

| Threshold | Value | Trigger Condition |
|-----------|-------|-------------------|
| Consecutive | 3 | Three denials in a row without any successful tool use |
| Total | 20 | Twenty total denials in the session |

### 7.2 State Management

Denial state is tracked as a record with two counters: consecutive denials and total denials.

- Recording a denial increments both counters.
- Recording a success resets consecutive denials to zero (total is unchanged).
- The fallback check returns true when either threshold is reached.

Consecutive denials reset on **any** successful tool use, including tool uses that were auto-allowed by rules (before the classifier was even consulted). This means a successful file read between two blocked shell commands resets the consecutive counter.

### 7.3 Isolated State Per Agent

Main agents and subagents maintain **isolated** denial tracking state. Subagents use a local mutable reference that is mutated in-place (since subagent global state updates are a no-op). Main agents use the global application state.

This isolation prevents a subagent's denial streak from triggering the main agent's fallback, and vice versa.

### 7.4 Fallback Behavior

When a limit is exceeded:

- **Interactive mode**: The system returns an `ask` decision with the classifier's reason appended to a warning message ("N consecutive actions were blocked. Please review the transcript before continuing."). The user sees the permission prompt and can approve or deny. When the total limit is hit, both counters are reset to zero to allow the session to continue.

- **Headless mode**: An abort error is thrown, terminating the agent. There is no human to review, so continuing would waste resources.

## 8. Human-in-the-Loop Dialog

When the permission system determines that a human must approve an action (the final decision is `ask` after all transformations), a multi-source approval race begins. This race model ensures that the fastest available decision source wins, while the human always retains override capability.

### 8.1 The 5-Way Race

Five decision sources run in parallel:

1. **Local terminal input**: The user types directly in the CLI. Options include approve, deny, approve-always (with rule suggestion), and deny-always.

2. **IDE bridge**: Approval from VS Code, JetBrains, or other IDE integrations that communicate via a bridge protocol.

3. **Channel relay**: Approval via external channels (Telegram, iMessage, etc.) for remote or mobile oversight.

4. **Permission hooks**: External systems (enterprise integrations, CI/CD pipelines) can inject allow or deny decisions via the hook system.

5. **Async classifier**: A background auto-approval classifier that runs concurrently with the user prompt. If enabled, this can auto-approve the action before the user responds.

### 8.2 Atomic Claim

An atomic claim guard ensures that exactly one source resolves the decision. Once claimed, all other sources' responses are discarded. This prevents race conditions where two sources both try to approve (or one approves while another denies).

### 8.3 200ms Grace Period

A 200ms grace period prevents accidental keypresses from canceling the classifier. When the classifier is checking in the background, user input within the first 200ms is held to ensure the user intentionally wants to override the classifier's pending decision.

### 8.4 Classifier Auto-Approval Display

When the async classifier wins the race and auto-approves an action, a checkmark is briefly displayed (3 seconds when the terminal is focused, 1 second when unfocused) that the user can dismiss. This provides visual feedback that the classifier made a decision on the user's behalf.

### 8.5 Handler Variants

Three handler variants serve different execution contexts:

- **Interactive handler**: Full 5-way race. Used by interactive CLI agents.
- **Coordinator handler**: Sequential hooks, then classifier, then dialog. Used by coordinator workers in multi-agent setups.
- **Swarm worker handler**: Runs the classifier locally, then forwards unresolved decisions to the swarm leader via mailbox IPC.

## 9. Tool-Level Permission Checks

Each tool implements domain-specific safety logic through its permission check method. The permission mode system interacts with these checks in specific ways.

### 9.1 SafetyCheck Decisions

The safety check decision type has a boolean field that determines how the check interacts with auto mode:

- **Not classifier-approvable**: The check is immune to ALL auto-approve paths, including the AI classifier, the acceptEdits fast path, the safe tool allowlist, and bypass mode. Examples: Windows path bypass attempts, cross-machine bridge messages. These always require interactive approval.

- **Classifier-approvable**: The classifier can evaluate this check. Examples: sensitive file paths (`.claude/`, `.git/`, shell configurations) where the conversation context matters for deciding safety. The acceptEdits fast path and safe tool allowlist do not fire for these (because the tool's permission check still returns `ask`), but the classifier sees the full context and can make an informed decision.

### 9.2 Domain-Specific Checks by Tool

| Tool | Permission Check Logic |
|------|----------------------|
| **Bash** | AST-based security parse via tree-sitter, sandbox auto-allow logic, exact/prefix rule matching, classifier for command description matching, pipe/redirect validation |
| **File Edit / File Write** | Deny rules (symlink-aware), path safety checks, session scoping, working directory containment |
| **Agent** | Auto-allows in non-auto modes, defers to classifier in auto mode (passthrough) |
| **MCP** | Always defers to the general permission framework (passthrough) |
| **PowerShell** | Requires explicit user permission in auto mode unless a specific build flag is enabled |

### 9.3 Sandbox Auto-Allow

When sandboxing is enabled and the sandbox auto-allow setting is true, bash commands that will be sandboxed can skip ask rules. The sandbox provides OS-level isolation that substitutes for interactive approval. Commands excluded from sandboxing (via the excluded commands list or explicit sandbox disable) still require approval.

## 10. Permission Configuration

### 10.1 Settings Files

Permission rules are persisted in JSON settings files at three levels:

```
~/.claude/settings.json          (user settings)
<project>/.claude/settings.json  (project settings)
<project>/.claude/settings.local.json (local settings)
```

The settings schema includes:

```json
{
  "permissions": {
    "allow": ["Bash(git:*)", "Read"],
    "deny": ["Bash(rm -rf:*)"],
    "ask": ["Bash(npm publish:*)"],
    "defaultMode": "default",
    "disableBypassPermissionsMode": "disable",
    "disableAutoMode": "disable",
    "additionalDirectories": ["/path/to/other/project"]
  },
  "autoMode": {
    "allow": ["Running test commands"],
    "soft_deny": ["Deleting production data"],
    "environment": ["This is a development environment"]
  }
}
```

Managed settings from deployment infrastructure take the highest priority and can set a managed-only flag to override all other sources.

### 10.2 CLI Arguments

- `--permission-mode <mode>`: Set the initial permission mode
- `--dangerously-skip-permissions`: Alias for `--permission-mode bypassPermissions`
- `--allowed-tools <tools>`: Add allow rules from the CLI (comma or space separated)
- `--disallowed-tools <tools>`: Add deny rules from the CLI
- `--base-tools <preset|list>`: Define the base tool set (tools not in the set are denied)
- `--add-dir <path>`: Add additional working directories
- `--enable-auto-mode`: Opt in to auto mode availability

### 10.3 Session Rules

Rules can be added during a session through the permission prompt UI. When a user approves an action and selects "always allow," the system generates a permission update that can be persisted to either session scope (ephemeral) or a settings file (persistent). Session-scoped rules live only in the in-memory permission context and are lost when the session ends.

### 10.4 Mode Cycling

In the interactive CLI, users cycle through modes with Shift+Tab. The cycle order depends on the user type and available modes:

- **External users**: default -> acceptEdits -> plan -> [bypassPermissions] -> [auto] -> default
- **Anthropic users**: default -> [bypassPermissions] -> [auto] -> default (skip acceptEdits and plan)

Modes in brackets are conditional: bypassPermissions appears only if not disabled, auto appears only if the gate is enabled and the model supports it.

Each mode transition goes through a single transition function, which handles side effects: plan mode stashes the previous mode, auto mode activates the classifier and strips dangerous rules, exiting auto mode restores rules and marks a notification attachment.

## 11. Interaction with Other Systems

### 11.1 Tools

The permission mode is carried on the permission context, which is an immutable snapshot passed to every tool's permission check method. Tools read the mode to make mode-specific decisions:

- The Agent tool returns `allow` in non-auto modes (so subagent spawns don't prompt) but `passthrough` in auto mode (so the classifier evaluates the subagent's prompt).
- The Bash tool checks sandbox auto-allow status, which considers both sandboxing state and rule configuration.
- File tools use the mode to determine whether the acceptEdits fast path applies.

### 11.2 Agents and Subagents

Subagents (spawned via the Agent tool) inherit the parent's permission mode but maintain isolated denial tracking state. In auto mode, subagent tool invocations go through the same classifier pipeline as the main agent. Headless subagents (those that should avoid permission prompts) cannot show permission prompts; they run permission hooks first, then auto-deny if no hook provides a decision.

A flag on the context controls whether automated checks (hooks, classifier) should complete before showing the interactive dialog. This is used by coordinator workers to ensure sequential processing.

### 11.3 Bridge and IDE Integration

The permission mode is communicated to IDE bridges (VS Code, JetBrains) so they can display the current mode in their UI. Mode changes from the bridge (e.g., the user clicks a mode button in VS Code) flow back through the same transition function, ensuring consistent side effects.

The bridge also participates in the 5-way permission race: when the IDE is connected, it can approve or deny permission requests alongside the terminal and classifier.

### 11.4 Commands

Several slash commands interact with the permission system:

- `/permissions` -- Display and manage permission rules
- `/mode` or Shift+Tab -- Cycle through permission modes
- `/config` -- Modify settings including permission configuration
- `/auto-mode` -- Manage auto mode rules (defaults, critique)

### 11.5 SDK and Headless Contexts

When Claude Code runs as an SDK agent (headless), certain modes behave differently:

- Auto mode's denial limit fallback throws an abort error instead of prompting.
- Transcript-too-long errors in auto mode throw an abort error (permanent condition).
- Headless agents route through permission hooks before auto-deny.
- Remote environments restrict the default mode to safe modes only (acceptEdits, plan, default).

## 12. Design Principles

### 12.1 Deny-First Evaluation

Deny rules always win, regardless of mode. A deny rule from any source blocks a tool invocation before the mode transformation even runs. This ensures that organizational policies cannot be overridden by a permissive mode setting.

### 12.2 Safety Checks Are Mode-Immune

Tool-specific safety checks marked as non-classifier-approvable are immune to all modes, including bypass mode and auto mode. These represent hard safety boundaries (like preventing Windows NTLM credential leaks via UNC paths) that no trust posture should override.

### 12.3 Fail-Closed Defaults

The system defaults to fail-closed behavior throughout:
- Unparseable classifier responses are treated as blocks.
- API errors default to deny (controlled by a feature gate).
- Unknown or invalid permission modes fall back to `default`.
- New tools that don't implement permission checks get `passthrough`, which becomes `ask` (requiring approval).

### 12.4 Transparent Stripping

Dangerous pattern stripping is transparent: rules are logged when stripped and restored when auto mode is exited. The user's disk-persisted settings are never modified by the stripping process.

### 12.5 Isolated Agent State

Each agent (main or sub) maintains its own denial tracking state. A subagent's risky behavior cannot trigger the main agent's circuit breaker, and vice versa. This prevents cross-agent interference in multi-agent scenarios.

### 12.6 Composable Layers

The mode system composes with, rather than replaces, other safety layers. The mode transformation is one layer in a defense-in-depth model. A permissive mode (like bypass) still has deny rules, safety checks, and sandbox enforcement active. A restrictive mode (like dontAsk) still respects allow rules for explicitly trusted tools.

### 12.7 Human Override Capability

Even in auto mode, the human retains override capability through denial limits (which force fallback to prompting), the IDE bridge (which can inject decisions), channel relays (for remote oversight), and the ability to exit auto mode at any time via Shift+Tab.

### 12.8 Atomic Transitions

Mode transitions are atomic: the transition function handles all side effects (plan mode stashing, auto mode activation/deactivation, dangerous pattern stripping/restoration) in a single function call. This prevents inconsistent states where the mode has changed but the side effects have not been applied.
