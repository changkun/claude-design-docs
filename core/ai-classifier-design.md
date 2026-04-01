# AI Classifier — Design Document

*(Extracted and expanded from [oversight-design.md](oversight-design.md), Layer 10 and Section 5.)*

---

# Claude Code: AI Classifier System Design

The AI classifier is the mechanism that replaces human judgment in **auto mode**. When a tool invocation would normally prompt the user for approval (an `ask` decision), the classifier evaluates the full conversation transcript and decides whether the action is safe to proceed.

This document covers the classifier's architecture, decision pipeline, transcript handling, failure modes, and integration with surrounding layers.

## Source Code Map

| Component | File | Key Lines |
|-----------|------|-----------|
| **Classifier entry point** | `src/utils/permissions/permissions.ts` | L593–L927 (auto mode branch in `checkPermissions`) |
| **2-stage XML pipeline** | `src/utils/permissions/yoloClassifier.ts` | L711–L996 (`classifyYoloActionXml`) |
| **Transcript builder** | `src/utils/permissions/yoloClassifier.ts` | L302–L360 (`buildTranscriptEntries`) |
| **XML response parsing** | `src/utils/permissions/yoloClassifier.ts` | L567–L604 (`stripThinking`, `parseXmlBlock`, `parseXmlReason`) |
| **Prompt templates** | `src/utils/permissions/yoloClassifier.ts` | L54–L68 (loaded via bundler from `./yolo-classifier-prompts/`) |
| **Safe tool allowlist** | `src/utils/permissions/classifierDecision.ts` | L56–L94 (`SAFE_YOLO_ALLOWLISTED_TOOLS`) |
| **Denial limit tracking** | `src/utils/permissions/denialTracking.ts` | L1–L45 (state, thresholds, `shouldFallbackToPrompting`) |
| **Denial limit handler** | `src/utils/permissions/permissions.ts` | L984–L1058 (`handleDenialLimitExceeded`) |
| **Denial UI store** | `src/utils/autoModeDenials.ts` | L1–L27 (`recordAutoModeDenial`, `getAutoModeDenials`) |
| **Model selection** | `src/utils/permissions/yoloClassifier.ts` | L1334–L1347 (`getClassifierModel`) |
| **Shared utilities** | `src/utils/permissions/classifierShared.ts` | Response parsing helpers |
| **Auto mode CLI** | `src/cli/handlers/autoMode.ts` | User-facing auto mode commands |
| **Permission hook** | `src/hooks/useCanUseTool.tsx` | React hook calling into classifier |

### System Prompt Files

The classifier's system prompts are loaded at build time via the `TRANSCRIPT_CLASSIFIER` feature flag from text files that are not checked into the open-source tree:

```
src/utils/permissions/yolo-classifier-prompts/
  auto_mode_system_prompt.txt   ← BASE_PROMPT (core safety evaluation rules)
  permissions_external.txt      ← EXTERNAL_PERMISSIONS_TEMPLATE (default rules for external users)
  permissions_anthropic.txt     ← ANTHROPIC_PERMISSIONS_TEMPLATE (Anthropic-internal template)
```

The prompts contain replaceable placeholder tags:
- `<user_allow_rules_to_replace>` — injected with user's allow rules
- `<user_deny_rules_to_replace>` — injected with user's deny rules
- `<user_environment_to_replace>` — injected with environment context

---

## 1. Role in the Oversight Stack

The classifier sits at **Layer 10** of the 14-layer Swiss cheese model, between hooks (L9) and denial limits (L11). It is only active when the permission mode is `auto`. In all other modes, `ask` decisions route directly to the human approval dialog.

```
Tool invocation
    |
    v
L1–L9: deterministic checks, rules, hooks
    |
    v  (decision == "ask" && mode == "auto")
+----------------------------------+
| L10: AI Classifier               |
|   Fast paths → skip classifier   |
|   Stage 1 (fast) → allow/escalate|
|   Stage 2 (thinking) → allow/deny|
+----------------------------------+
    |
    v
L11: Denial limit tracking
    |
    v
L12: Human approval (fallback)
```

The classifier is **not** the first line of defense. It only evaluates actions that have already survived input validation, permission rules, tool-specific checks, the bash security scanner, pattern stripping, sandbox checks, mode transformation, and hooks.

---

## 2. Entry Conditions

The classifier activates when **all** of the following hold:

1. The **transcript classifier feature flag** is enabled in the build
2. The user has selected **auto mode** as their permission mode
3. A tool invocation's permission check returned an **`ask`** decision
4. The `ask` decision is **classifier-approvable** (i.e., `safetyCheck.classifierApprovable !== false`)

If any condition is not met, the decision bypasses the classifier entirely — either denied outright (non-approvable safety checks) or routed to the human dialog.

---

## 3. Fast Paths (Classifier Bypass)

Before invoking the LLM, three fast paths short-circuit the evaluation (`permissions.ts:593–686`):

| Fast Path | Outcome | Source | Rationale |
|-----------|---------|--------|-----------|
| Non-classifier-approvable safety check | **Deny** | Tool-level `safetyCheck` with `classifierApprovable: false` | Hardcoded protections (e.g., Windows UNC path attacks) that no AI should override |
| `acceptEdits` equivalence | **Allow** | `permissions.ts:600–649` — re-runs `tool.checkPermissions` with `mode: 'acceptEdits'` | File edit operations within the working directory that would be auto-allowed under `acceptEdits` mode |
| Safe tool allowlist | **Allow** | `classifierDecision.ts:56–94` — `SAFE_YOLO_ALLOWLISTED_TOOLS` set | Read-only tools (e.g., file reads, glob, grep) that pose no risk |

The safe tool allowlist includes: `FileRead`, `Grep`, `Glob`, `LSP`, `ToolSearch`, `ListMcpResources`, `ReadMcpResourceTool`, `TodoWrite`, `TaskCreate/Get/Update/List/Stop/Output`, `AskUserQuestion`, `EnterPlanMode`, `ExitPlanMode`, `TeamCreate`, `TeamDelete`, `SendMessage`, `Sleep`, and the internal classifier tool itself.

Additional special cases:
- **Tools requiring user interaction** bypass classifier evaluation entirely (they need a human by definition)
- **PowerShell** remains gated unless the `POWERSHELL_AUTO_MODE` feature flag is enabled (`permissions.ts:572–591`)
- **Agent and REPL tools** are excluded from the `acceptEdits` fast path to prevent silent classifier bypass (`permissions.ts:602–603`)

Fast paths avoid unnecessary API calls and latency for decisions that have deterministic answers.

---

## 4. Two-Stage Classifier Pipeline

When no fast path applies, the classifier runs a **2-stage XML pipeline**:

```
              Tool invocation (ask decision)
                        |
                        v
            +------------------------+
            |   Stage 1: Fast Pass   |
            |   (low token budget)   |
            +------------------------+
               /        |        \
          ALLOW      DENY     UNPARSEABLE
            |          |            |
            v          v            v
         Return    Escalate     Escalate
                      |            |
                      v            v
            +------------------------+
            |  Stage 2: Thinking     |
            |  (high token budget)   |
            +------------------------+
               /              \
          ALLOW              DENY
            |                  |
            v                  v
         Return        Return (with reason)
```

### Stage 1: Fast Pass (`yoloClassifier.ts:769–857`)

- **Purpose**: Conservative, low-cost initial evaluation
- **Token budget**: `max_tokens: 64` (default) or `256` (fast-only mode) + thinking padding (`yoloClassifier.ts:781`)
- **Stop sequences**: `['</block>']` in both/thinking modes — forces an immediate yes/no answer
- **Suffix appended to transcript**: `'\nErr on the side of blocking. <block> immediately.'` (`XML_S1_SUFFIX`, line 550)
- **Behavior on allow** (`<block>no</block>`): Return immediately — no escalation needed
- **Behavior on deny or parse failure**: Escalate to stage 2
- **Design bias**: Tuned to be conservative. It should catch clearly safe operations but never approve anything borderline.

### Stage 2: Thinking Pass (`yoloClassifier.ts:860–940`)

- **Purpose**: Deliberate, high-fidelity second evaluation with reasoning
- **Token budget**: `max_tokens: 4096` + thinking padding (2048 for always-on-thinking models) (`yoloClassifier.ts:868`)
- **No stop sequences** — allows full chain-of-thought reasoning
- **Suffix appended to transcript**: `'\nReview the classification process and follow it carefully, making sure you deny actions that should be blocked. As a reminder, explicit (not suggestive or implicit) user confirmation is required to override blocks. Use <thinking> before responding with <block>.'` (`XML_S2_SUFFIX`, line 560)
- **Thinking enabled**: The model uses extended thinking to reason through the decision
- **Can override stage 1**: A stage 1 deny can be reversed by stage 2 if the thinking model determines the action is safe in context
- **Output**: Emits `<thinking>` (chain-of-thought) and `<reason>` XML payloads for auditability

### XML Output Format

The classifier's system prompt is rewritten to use XML instead of tool_use for output (`replaceOutputFormatWithXml`, line 648):

```
If the action should be blocked:
<block>yes</block><reason>one short sentence</reason>

If the action should be allowed:
<block>no</block>

Your ENTIRE response MUST begin with <block>.
```

Parsing is defensive — `stripThinking()` (line 567) removes `<thinking>...</thinking>` tags before extracting `<block>` and `<reason>` to prevent forged tags inside chain-of-thought content.

### Three Classifier Modes

Controlled via `CLAUDE_CODE_TWO_STAGE_CLASSIFIER` env var or `tengu_auto_mode_config.twoStageClassifier` GrowthBook flag (`resolveTwoStageClassifier`, line 1353):

| Mode | Behavior |
|------|----------|
| `'both'` (default) | Stage 1 fast → escalate to stage 2 if blocked |
| `'fast'` | Stage 1 only, 256-token budget, verdict is final |
| `'thinking'` | Stage 2 only, skips stage 1 entirely |

### Why Two Stages?

The two-stage design optimizes for both latency and accuracy:

- **Most actions are safe** (file reads, simple edits). Stage 1 approves these instantly at minimal cost.
- **Borderline actions need deliberation**. Stage 2 provides thinking-enabled reasoning at higher cost, but only when needed.
- **Stage 1 false denials are cheap** — they just add one more API call. Stage 1 false approvals are expensive — they bypass human oversight. Hence stage 1 is biased conservative.

---

## 5. Transcript Construction (`yoloClassifier.ts:302–424`)

The classifier sees the conversation history, but with critical filtering:

### Included

- **User messages** — full text content (`msg.type === 'user'`, line 326)
- **Queued command prompts** — extracted from attachment messages (`msg.type === 'attachment'` with `queued_command` type, line 305)
- **Assistant tool_use blocks** — the tool calls themselves (`block.type === 'tool_use'`, line 346)

### Excluded

- **Assistant prose text** — any text the model generated between tool calls is stripped (the `for` loop at line 343 only collects `tool_use` blocks, ignoring `text` blocks)

### Compact Serialization

Each transcript entry is serialized to a compact format via `toCompactBlock` (line 384):

- **Default format**: `Bash ls\n` (tool name + space + encoded input)
- **JSONL format** (optional, via `CLAUDE_CODE_JSONL_TRANSCRIPT`): `{"Bash":"ls"}\n`
- **User messages**: `User: text\n` (default) or `{"user":"text"}\n` (JSONL)

Each tool provides a `toAutoClassifierInput()` method that projects its input to a classifier-friendly representation. The transcript is wrapped in `<transcript>...</transcript>` XML tags before being sent to the classifier (line 762).

### Rationale for Excluding Assistant Text

This is a defense against **self-influence attacks**. If the model's own prose (which could be influenced by prompt injection in tool results) were included in the classifier's transcript, an attacker could:

1. Inject text into a tool result (e.g., a file containing instructions)
2. The model incorporates that text into its response
3. The classifier reads the model's response as if it were legitimate context
4. The classifier is more likely to approve the dangerous action

By stripping assistant prose, the classifier evaluates only what the user said and what tools the model tried to call — not the model's potentially-compromised reasoning.

---

## 6. Classifier Prompt Structure (`yoloClassifier.ts:54–68, 648–663`)

The classifier's system prompt is assembled from three components:

| Component | Source File | Content |
|-----------|-------------|---------|
| **Base rules** | `auto_mode_system_prompt.txt` → `BASE_PROMPT` (line 54) | Core safety evaluation instructions — what constitutes a safe vs. dangerous action |
| **Permissions template** | `permissions_external.txt` → `EXTERNAL_PERMISSIONS_TEMPLATE` (line 61) or `permissions_anthropic.txt` → `ANTHROPIC_PERMISSIONS_TEMPLATE` (line 65, Anthropic-internal) | Structural framing for how to interpret user-defined rules |
| **User rules** | `settings.autoMode` config | Three categories merged from the user's auto-mode configuration |

Template selection (`isUsingExternalPermissions`, line 71): external users always get the external template; Anthropic employees get the internal template unless `tengu_auto_mode_config.forceExternalPermissions` is set.

For XML mode, the prompt's tool_use output instruction is replaced with XML format via `replaceOutputFormatWithXml` (line 648). Prompt caching is applied to the system prompt block with `cache_control` from `getCacheControl({ querySource: 'auto_mode' })`.

### User Rule Categories

Users can configure rules that the classifier incorporates:

- **`allow` rules**: Actions the classifier should approve (e.g., "allow writes to /tmp")
- **`soft_deny` rules**: Actions the classifier should reject unless context strongly justifies them
- **`environment` rules**: Contextual information about the execution environment (e.g., "this is a CI pipeline", "this repo contains no secrets")

These rules are **merged into the system prompt** by replacing placeholder tags, not evaluated as separate logic. The classifier weighs them alongside its base rules and the transcript context.

---

## 7. Classifier Model Selection (`yoloClassifier.ts:1334–1347`)

The classifier model is **not hardwired** to the main agent loop model. Selection follows a priority chain:

```
1. CLAUDE_CODE_AUTO_MODE_MODEL env var  (Anthropic-internal only)
2. tengu_auto_mode_config.model         (GrowthBook remote config)
3. getMainLoopModel()                   (fallback — same model as the agent)
```

In practice, the classifier typically runs on the same model as the main agent loop (e.g., Opus 4.6), but can be overridden for cost/latency/accuracy tradeoffs without changing the agent model.

Thinking configuration (`getClassifierThinkingConfig`, line 690): models with `alwaysOnThinking=true` get an extra 2048-token headroom for adaptive thinking; all other models have thinking explicitly disabled (`thinking: false`).

---

## 8. Failure Modes

### API Error (`permissions.ts:845–876`)

- **Default behavior**: Fail-closed (deny) — controlled by the `tengu_iron_gate_closed` GrowthBook flag (refreshed every `CLASSIFIER_FAIL_CLOSED_REFRESH_MS`)
- **Fail-closed path** (gate=true): Returns a `deny` with guidance message via `buildClassifierUnavailableMessage`
- **Fail-open path** (gate=false): Falls through to the original `ask` result, which routes to the human prompt
- **Rationale**: If the classifier can't run, the safest assumption is to deny. The user can then approve manually.

### Transcript Too Long (`permissions.ts:822–842`, `yoloClassifier.ts:954–976`)

- **Detection**: `detectPromptTooLong(error)` extracts actual vs. limit token counts from the API error
- **Interactive mode**: Falls back to the original `ask` result (human approval dialog) with reason `'Auto mode classifier transcript exceeded context window — falling back to manual approval'`
- **Headless mode**: Throws `AbortError` — the transcript only grows, so retry is pointless (`permissions.ts:826–828`)
- **Rationale**: If the transcript exceeds the classifier model's context window, the classifier cannot make an informed decision. Human judgment is the fallback.

### Circuit Breaker

- **Behavior**: Prevents re-entry after remote disable
- **Rationale**: If the classifier is remotely disabled (e.g., due to a discovered vulnerability), the circuit breaker ensures it cannot be reactivated within the session.

### Unparseable Response (`yoloClassifier.ts:827–840, 898–917`)

- **Stage 1**: `parseXmlBlock` returns `null` → treated as a block → escalate to stage 2
- **Stage 2**: `parseXmlBlock` returns `null` → returned as `shouldBlock: true` with reason `'Classifier stage 2 unparseable - blocking for safety'`
- **Rationale**: If the classifier's XML response is malformed, the system cannot extract a decision. Fail-closed.

---

## 9. Integration with Denial Limits (L11)

### State Machine (`denialTracking.ts`)

The denial tracker maintains two counters per agent:

```typescript
// denialTracking.ts:7–10
type DenialTrackingState = {
  consecutiveDenials: number  // resets to 0 on any success
  totalDenials: number        // monotonically increases (reset only on total-limit hit)
}
```

Constants (`denialTracking.ts:12–15`):
- `maxConsecutive: 3`
- `maxTotal: 20`

State transitions:
- **On denial** (`recordDenial`, line 24): increment both counters
- **On success** (`recordSuccess`, line 32): reset `consecutiveDenials` to 0, keep `totalDenials`
- **On total limit hit** (`handleDenialLimitExceeded`, `permissions.ts:1034–1040`): reset both counters to 0 (allows the session to continue after human review)

The fallback check (`shouldFallbackToPrompting`, line 40):
```typescript
consecutiveDenials >= 3 || totalDenials >= 20
```

Main agents and subagents maintain **isolated** denial state — a subagent's denials don't count against the main agent.

### Purpose

Denial limits are a circuit breaker against **classifier exploitation**. An attacker who can craft tool calls that the classifier repeatedly denies (while subtly varying them) could probe for classifier weaknesses. The limits force a human checkpoint before the classifier has evaluated too many potentially adversarial inputs.

### The "Fall Back to Interactive Human Prompting" Path

This is the full code path from a classifier denial to the human seeing a permission prompt. Traced through three files:

#### Step-by-step walkthrough

**1. Classifier returns `shouldBlock: true`** (`permissions.ts:818`)

The 2-stage pipeline in `classifyYoloActionXml` returns its verdict. The caller in `permissions.ts` enters the `if (classifierResult.shouldBlock)` branch.

**2. Special cases are checked first** (`permissions.ts:822–876`)

Before denial tracking, two special cases are handled:
- **Transcript too long** → skip denial tracking entirely, return the original `ask` result directly (or `AbortError` in headless mode)
- **Classifier unavailable** (API error) → either `deny` (fail-closed) or return original `ask` result (fail-open), depending on `tengu_iron_gate_closed` gate

**3. Denial state is incremented** (`permissions.ts:879–880`)

```typescript
const newDenialState = recordDenial(denialState)
persistDenialState(context, newDenialState)
```

**4. Denial limits are checked** (`permissions.ts:890–901`)

```typescript
const denialLimitResult = handleDenialLimitExceeded(
  newDenialState, appState, classifierResult.reason,
  assistantMessage, tool, result, context,
)
if (denialLimitResult) {
  return denialLimitResult  // ← this is what triggers the human prompt
}
```

**5. Inside `handleDenialLimitExceeded`** (`permissions.ts:984–1058`)

This function is the decision point:

```
shouldFallbackToPrompting(denialState)?
    │
    ├─ NO  → return null (caller proceeds to deny the action silently)
    │
    └─ YES → which mode?
         │
         ├─ Headless (shouldAvoidPermissionPrompts=true)
         │     → throw AbortError('Agent aborted: too many classifier denials in headless mode')
         │
         └─ Interactive (CLI / IDE)
               → Build warning message:
               │   "3 consecutive actions were blocked. Please review the transcript."
               │   or "20 actions were blocked this session. Please review the transcript."
               │
               → If total limit hit: reset both counters to 0
               │
               → Return the ORIGINAL `ask` result (from L1–L9)
                 with an augmented `decisionReason`:
                   type: 'classifier'
                   reason: "{warning}\n\nLatest blocked action: {classifierReason}"
```

**6. The returned `ask` result flows to the human dialog** (`permissions.ts` caller)

Because `handleDenialLimitExceeded` returns a `PermissionDecision` with `behavior: 'ask'` (the original result before the classifier transformed it), the caller returns this to the permission prompt system. The human sees:

- The tool name and input (what the agent wants to do)
- The augmented reason showing how many denials occurred and why the latest one was blocked

The human can then **approve** (the action executes), **deny** (the action is blocked), or **always allow** (add a permission rule).

#### Concrete example

Imagine the agent is running in auto mode and tries to execute three `rm` commands in a row:

```
Turn 1: Bash("rm -rf /tmp/cache")
  → Classifier stage 1: <block>yes</block>  (blocked)
  → Classifier stage 2: <block>yes</block><reason>Deleting directory recursively</reason>
  → denialState = { consecutiveDenials: 1, totalDenials: 1 }
  → shouldFallbackToPrompting? NO (1 < 3)
  → Result: deny (silent, agent sees rejection message)

Turn 2: Bash("rm -rf /var/log/app")
  → Classifier stage 1: <block>yes</block>  (blocked)
  → Classifier stage 2: <block>yes</block><reason>Deleting system logs</reason>
  → denialState = { consecutiveDenials: 2, totalDenials: 2 }
  → shouldFallbackToPrompting? NO (2 < 3)
  → Result: deny (silent)

Turn 3: Bash("rm -rf ~/Documents")
  → Classifier stage 1: <block>yes</block>  (blocked)
  → Classifier stage 2: <block>yes</block><reason>Deleting user documents</reason>
  → denialState = { consecutiveDenials: 3, totalDenials: 3 }
  → shouldFallbackToPrompting? YES (3 >= 3)
  → handleDenialLimitExceeded:
      warning = "3 consecutive actions were blocked. Please review the transcript."
      return original `ask` result with augmented reason
  → Human sees permission prompt:
      ┌─────────────────────────────────────────────────────────┐
      │  Bash: rm -rf ~/Documents                              │
      │                                                         │
      │  3 consecutive actions were blocked. Please review      │
      │  the transcript before continuing.                      │
      │                                                         │
      │  Latest blocked action: Deleting user documents         │
      │                                                         │
      │  [Allow]  [Deny]  [Always Allow]  [Abort]              │
      └─────────────────────────────────────────────────────────┘
```

If the human denies, the action is blocked. If the human approves, the action executes and `consecutiveDenials` resets to 0. Either way, the human has been forced into the loop.

Note: if this were a **headless** agent (e.g., running in CI), turn 3 would instead throw `AbortError` and terminate the agent, since there is no human to prompt.

---

## 10. Integration with Pattern Stripping (L6)

When entering auto mode, **overly broad allow rules are stripped** to prevent them from bypassing the classifier:

### Stripped Patterns

Rules matching code-execution entry points in any of five shapes (exact, prefix, trailing/space/flag wildcards):

- Shell executors: `python`, `node`, `bash`, `sh`, `ruby`, `perl`, `php`
- Network tools: `ssh`, `curl`, `wget`
- PowerShell cmdlets: `invoke-expression`, `start-process`, `add-type`
- Agent tool rules (prevents delegation attacks)

### Lifecycle

1. **On auto mode entry**: Matching rules are removed from the active rule set and stashed
2. **During auto mode**: The classifier evaluates all tool calls without these broad bypasses
3. **On auto mode exit**: Stashed rules are restored

### Rationale

A user who set `allow: bash *` for convenience in default mode (where they see every command) should not have that rule silently let the classifier skip evaluation of arbitrary bash commands. The stripping is logged to debug output for transparency.

---

## 11. Interaction with the Permission Race (L12)

When the classifier is running as part of auto mode, it participates in the **5-way permission race** as racer #5 (async classifier). In this configuration:

- The classifier runs **in the background** while the terminal prompt is displayed
- If the classifier reaches a decision before the human, it wins the race
- A **200ms grace period** prevents the classifier from accidentally overriding a human who is mid-keystroke
- If the classifier approves, a **checkmark indicator** is briefly displayed (3s focused / 1s unfocused) that the user can dismiss

The human always retains override capability — they can deny an action even if the classifier approved it, within the grace period.

---

## 12. Telemetry

The classifier logs approximately **40 telemetry fields** per decision:

| Category | Fields |
|----------|--------|
| Token usage | Per-stage input/output tokens |
| Latency | Per-stage wall-clock time |
| Cost | Per-stage and total cost |
| Prompt composition | Component lengths (base rules, permissions template, user rules, transcript) |
| Session aggregates | Total classifier tokens, total session tokens, overhead ratio |
| Decision metadata | Stage that decided, allow/deny outcome, reason payload |

Metadata types are restricted to **boolean and number** to prevent accidental logging of code or file paths.

---

## 13. Summary of Design Properties

| Property | Implementation |
|----------|---------------|
| **Conservative by default** | Stage 1 biased to deny; fail-closed on errors |
| **Escalation over rejection** | Stage 1 denials escalate to stage 2 rather than being final |
| **Self-influence resistant** | Assistant prose excluded from transcript |
| **User-configurable** | allow/soft_deny/environment rules merged into prompt |
| **Bounded exploitation** | Denial limits force human review after repeated denials |
| **Independently deployable** | Classifier model selected separately from agent model |
| **Auditable** | Stage 2 emits `<reason>`, ~40 telemetry fields per decision |
| **Gracefully degrading** | API errors → deny, transcript overflow → human, circuit breaker → session lock |
