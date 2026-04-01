# Claude Code: Feature Flag System — Design Specification

This document analyzes the two-tier feature flag architecture of Claude Code — Anthropic's
agentic CLI tool — covering compile-time dead-code elimination, runtime server-side gating,
progressive rollout, kill switches, and the interaction between flags and the command,
tool, and service subsystems.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Compile-Time Flags](#2-compile-time-flags)
- [3. Runtime Flags](#3-runtime-flags)
- [4. Notable Compile-Time Flags](#4-notable-compile-time-flags)
- [5. Notable Runtime Gates](#5-notable-runtime-gates)
- [6. Flag Evaluation](#6-flag-evaluation)
- [7. Progressive Rollout](#7-progressive-rollout)
- [8. Interaction with Commands](#8-interaction-with-commands)
- [9. Interaction with Tools](#9-interaction-with-tools)
- [10. Interaction with Services](#10-interaction-with-services)
- [11. Kill Switch Pattern](#11-kill-switch-pattern)
- [12. Design Principles](#12-design-principles)

---

## 1. Overview

Claude Code gates features behind a **two-tier flag system** that separates build-time
decisions from runtime decisions:

| Tier | Mechanism | When Evaluated | Cost When Off |
|---|---|---|---|
| **Compile-time** | Bun's `feature()` macro via `bun:bundle` | Build time (bundler pass) | Zero — dead code eliminated |
| **Runtime** | GrowthBook remote evaluation | Process startup + periodic refresh | Negligible — cached disk read |

The two tiers exist because they solve different problems. Compile-time flags control
**what code ships** in a given build. They prevent ant-only (internal Anthropic) features
from leaking string literals, module imports, or implementation details into the external
(public) binary. Runtime flags control **what code executes** for a given user within a
build that already contains the feature. They enable gradual percentage rollout, targeting
by organization/subscription/platform, A/B experiments, and emergency kill switches —
all without deploying new code.

The architecture means a feature typically passes through three lifecycle stages:

```
  Development                Controlled rollout           General availability
  ─────────────              ──────────────────           ────────────────────
  feature('X') = false       feature('X') = true          feature('X') = true
  (code exists, stripped)    + tengu_x gate (0→100%)      + gate removed or 100%
```

A feature whose compile-time flag is `false` has truly zero runtime cost — the bundler
eliminates the guarded code and all its transitive imports from the output. No string
literals from the feature appear in the binary, no module initialization runs, and no
GrowthBook lookups execute.

---

## 2. Compile-Time Flags

### 2.1 The `feature()` Macro

> **Source:** `import { feature } from 'bun:bundle'`

Bun's bundler provides the `feature()` function as a build-time macro. During bundling,
each `feature('FLAG_NAME')` call is replaced with a boolean literal (`true` or `false`)
based on the build configuration. Bun's dead-code elimination (DCE) then removes
unreachable branches.

The import appears in approximately 40 source files across the codebase:

```typescript
import { feature } from 'bun:bundle'

// At build time, this becomes: if (false) { ... }
// The bundler then strips the entire block.
if (feature('DAEMON') && args[0] === '--daemon-worker') {
  const { runDaemonWorker } = await import('../daemon/workerRegistry.js')
  await runDaemonWorker(args[1])
  return
}
```

### 2.2 Dead-Code Elimination Mechanics

When `feature('X')` resolves to `false`:

1. The boolean literal `false` replaces the call
2. Any `if (false && ...)` or `false ? ... : null` becomes unreachable
3. The bundler eliminates the unreachable branch entirely
4. Dynamic `require()` or `await import()` calls inside the branch are never emitted
5. The required modules and their transitive dependencies are not included in the bundle

This means the external build does not contain:
- Module code for ant-only features
- String literals (GrowthBook flag names, API endpoints, error messages)
- Tool definitions, command handlers, or service initialization for gated features

### 2.3 The Positive Ternary Pattern

A critical implementation detail documented across the codebase is the **positive ternary
pattern**. The bundler's DCE handles positive conditional patterns correctly but does not
eliminate inline string literals from negative guard patterns:

```typescript
// CORRECT — positive ternary: string literal eliminated when feature is false
export function isBridgeEnabled(): boolean {
  return feature('BRIDGE_MODE')
    ? getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)
    : false
}

// INCORRECT — negative guard: string literal 'tengu_ccr_bridge' leaks into output
export function isBridgeEnabled(): boolean {
  if (!feature('BRIDGE_MODE')) return false
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)
}
```

This pattern appears throughout `src/bridge/bridgeEnabled.ts` and is referenced by
inline comments such as `"Positive ternary pattern — see docs/feature-gating.md"`.

### 2.4 Module-Level Feature Guards

Many compile-time flags gate module-level `require()` calls that produce either a module
reference or `null`:

```typescript
const SnipTool = feature('HISTORY_SNIP')
  ? require('./tools/SnipTool/SnipTool.js').SnipTool
  : null
```

The resulting variable is then conditionally included in tool arrays:

```typescript
...(SnipTool ? [SnipTool] : [])
```

This pattern is used extensively in `src/tools.ts`, `src/skills/bundled/index.ts`,
`src/cli/print.ts`, and `src/entrypoints/cli.tsx`.

### 2.5 Separation from Runtime Gates

The `src/query/config.ts` file explicitly documents the distinction:

```typescript
// Intentionally excludes feature() gates — those are tree-shaking boundaries
// and must stay inline at the guarded blocks for dead-code elimination.
export type QueryConfig = {
  gates: {
    // Runtime gates (env/statsig). NOT feature() gates — see above.
    streamingToolExecution: boolean
    // ...
  }
}
```

Compile-time flags **must** appear inline at the guarded blocks — they cannot be
abstracted into config objects or passed as parameters, because the bundler needs to
see the literal `feature('NAME')` call to perform elimination.

---

## 3. Runtime Flags

### 3.1 GrowthBook Integration

> **Source:** `src/services/analytics/growthbook.ts`

Runtime flags are managed by [GrowthBook](https://www.growthbook.io/), an open-source
feature flagging platform. Claude Code uses GrowthBook in **remote evaluation** mode
(`remoteEval: true`), meaning the GrowthBook server evaluates all flag rules server-side
and returns pre-computed values for the current user. The client never sees targeting
rules or percentage allocations.

**Client initialization:**

```typescript
const thisClient = new GrowthBook({
  apiHost: 'https://api.anthropic.com/',
  clientKey: getGrowthBookClientKey(),  // different keys for ant vs external
  attributes: getUserAttributes(),
  remoteEval: true,
  cacheKeyAttributes: ['id', 'organizationUUID'],
})
```

**SDK keys** (`src/constants/keys.ts`) differ by build type:
- Ant builds: `sdk-xRVcrliHIlrg4og4` (production) or `sdk-yZQvlplybuXjYh6L` (dev)
- External builds: `sdk-zAZezfDKGoZuXXKe`

This separation ensures internal experiments never leak to external users even if the
same GrowthBook project is shared.

### 3.2 User Attributes for Targeting

> **Source:** `src/services/analytics/growthbook.ts`, `getUserAttributes()`

The following attributes are sent to GrowthBook for flag targeting:

| Attribute | Description |
|---|---|
| `id` / `deviceID` | Stable device identifier |
| `sessionId` | Per-session identifier |
| `platform` | `win32`, `darwin`, or `linux` |
| `apiBaseUrlHost` | Non-Anthropic proxy hostname (enterprise deployments) |
| `organizationUUID` | OAuth organization (when authenticated) |
| `accountUUID` | OAuth account (when authenticated) |
| `userType` | `ant` or undefined |
| `subscriptionType` | Subscription tier |
| `rateLimitTier` | API rate limit tier |
| `firstTokenTime` | First API token timestamp (cohort tracking) |
| `email` | User email (ant builds, from OAuth) |
| `appVersion` | CLI version string |

These attributes enable targeting by organization (enterprise rollout), subscription
(feature entitlement), platform (OS-specific features), and percentage (gradual rollout
by device ID).

### 3.3 Value Resolution Priority

When a runtime flag is queried, the value comes from the highest-priority source:

```
1. Environment variable overrides    (CLAUDE_INTERNAL_FC_OVERRIDES, ant-only)
2. Config file overrides             (/config Gates tab, ant-only)
3. In-memory remote eval cache       (populated after successful GrowthBook init)
4. Disk cache                        (~/.claude.json → cachedGrowthBookFeatures)
5. Default value                     (passed by the calling code)
```

The environment override exists for eval harnesses that need deterministic flag values.
The config override exists for developers testing specific flag states interactively.
Both are ant-only.

### 3.4 Accessor Functions

GrowthBook values are accessed through a family of functions with explicit semantics
about blocking and staleness:

| Function | Blocks? | Staleness | Use Case |
|---|---|---|---|
| `getFeatureValue_CACHED_MAY_BE_STALE<T>()` | No | May be stale | Startup-critical paths, render loops, sync contexts |
| `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` | No | May be stale | Boolean gates (migration wrapper) |
| `checkGate_CACHED_OR_BLOCKING()` | Conditionally | Fresh if blocking | Entitlement checks where stale `false` is harmful |
| `checkSecurityRestrictionGate()` | If re-init pending | Waits for auth change | Security-critical gates |
| `getDynamicConfig_CACHED_MAY_BE_STALE<T>()` | No | May be stale | JSON config objects |
| `getDynamicConfig_BLOCKS_ON_INIT<T>()` | Yes | Fresh | Config that must be fresh (kill switches) |
| `getFeatureValue_DEPRECATED<T>()` | Yes | Fresh | Legacy — avoid in new code |

The preferred accessor for most code paths is `getFeatureValue_CACHED_MAY_BE_STALE`,
which reads from the in-memory cache first, falls back to disk cache, and never blocks.

`checkGate_CACHED_OR_BLOCKING` implements a hybrid strategy: if the disk cache already
says `true`, it returns immediately (fast path). If the cache says `false` or is missing,
it awaits GrowthBook initialization to fetch the fresh server value (slow path, max ~5s).
This ensures a stale `false` from a previous session does not block access to a feature
the user is now entitled to.

---

## 4. Notable Compile-Time Flags

The following is a catalog of significant compile-time flags observed across the codebase,
grouped by functional area.

### 4.1 Mode and Entrypoint Flags

| Flag | Purpose |
|---|---|
| `BRIDGE_MODE` | Remote Control (bridge) — serve local machine as a bridge environment. Gates the entire `claude remote-control` entrypoint, bridge config, REPL bridge initialization, and all `tengu_ccr_*` / `tengu_bridge_*` runtime lookups. |
| `DAEMON` | Long-running daemon supervisor. Gates `claude daemon` and `--daemon-worker` entrypoints. |
| `KAIROS` | Autonomous assistant mode. Gates proactive tools (SleepTool, SendUserFileTool, PushNotificationTool), daily-log memory, session resumption, and brief mode integration. |
| `KAIROS_CHANNELS` | Channel notification subsystem for KAIROS. |
| `KAIROS_DREAM` | The `/dream` skill for nightly memory distillation. |
| `KAIROS_PUSH_NOTIFICATION` | Push notification tool (can be enabled independently of full KAIROS). |
| `KAIROS_GITHUB_WEBHOOKS` | SubscribePRTool for GitHub webhook integration. |
| `PROACTIVE` | Proactive mode — often combined with KAIROS via `feature('PROACTIVE') \|\| feature('KAIROS')`. |
| `COORDINATOR_MODE` | Multi-agent coordinator mode. Gates coordinator tools and role-based tool filtering. |
| `BG_SESSIONS` | Background sessions — `claude ps`, `claude logs`, `claude attach`, `claude kill`, and `--bg`/`--background` flags. |
| `VOICE_MODE` | Voice input via streaming speech-to-text. Gates VoiceProvider, useVoice hooks, and STT service. |

### 4.2 Context and Compaction Flags

| Flag | Purpose |
|---|---|
| `REACTIVE_COMPACT` | Emergency compaction on API "prompt too long" errors. |
| `CONTEXT_COLLAPSE` | Alternative compaction strategy with context collapse semantics. |
| `HISTORY_SNIP` | Fine-grained snip-based compaction of specific message segments. |
| `CACHED_MICROCOMPACT` | Micro-compaction that operates within the prompt cache window. |
| `BREAK_CACHE_COMMAND` | System prompt cache-breaking injection for debugging. |

### 4.3 Memory and Extraction Flags

| Flag | Purpose |
|---|---|
| `EXTRACT_MEMORIES` | Background memory extraction agent. |
| `TEAMMEM` | Team memory synchronization — shared `team/` subdirectory under auto-memory. |
| `MEMORY_SHAPE_TELEMETRY` | Telemetry for memory file structure analysis. |

### 4.4 Tool and Capability Flags

| Flag | Purpose |
|---|---|
| `AGENT_TRIGGERS` | Cron scheduling tools (CronCreate, CronDelete, CronList) and `/loop` skill. |
| `AGENT_TRIGGERS_REMOTE` | Remote trigger tool and `/schedule-remote-agents` skill. |
| `WORKFLOW_SCRIPTS` | WorkflowTool with bundled workflow initialization. |
| `MONITOR_TOOL` | MonitorTool for persistent observation tasks. |
| `WEB_BROWSER_TOOL` | Web browser automation tool. |
| `OVERFLOW_TEST_TOOL` | Testing tool for context overflow scenarios (ant-only). |
| `TERMINAL_PANEL` | TerminalCaptureTool for terminal screenshot integration. |
| `UDS_INBOX` | Unix domain socket messaging and ListPeersTool. |
| `TOKEN_BUDGET` | Task-level token budget tracking and enforcement. |

### 4.5 Infrastructure and Integration Flags

| Flag | Purpose |
|---|---|
| `TEMPLATES` | Template job system — `claude new`, `claude list`, `claude reply`. |
| `COMMIT_ATTRIBUTION` | Git commit attribution hooks (ant-only). |
| `CHICAGO_MCP` | Computer-use MCP server entrypoint and in-query MCP orchestration. |
| `CONNECTOR_TEXT` | Summarize-connector-text API beta header. |
| `TRANSCRIPT_CLASSIFIER` | AFK mode beta header and transcript classification. |
| `ABLATION_BASELINE` | Harness-science L0 ablation baseline for eval. |
| `DUMP_SYSTEM_PROMPT` | `--dump-system-prompt` flag for prompt sensitivity evals (ant-only). |
| `BYOC_ENVIRONMENT_RUNNER` | Headless BYOC (Bring Your Own Container) runner. |
| `SELF_HOSTED_RUNNER` | Self-hosted runner targeting the SelfHostedRunnerWorkerService API. |
| `CCR_AUTO_CONNECT` | Auto-connect to CCR (Claude Code Remote) on session start. |
| `CCR_MIRROR` | Outbound-only CCR mirror mode for local sessions. |
| `DOWNLOAD_USER_SETTINGS` | Remote settings sync from server. |
| `EXPERIMENTAL_SKILL_SEARCH` | Prefetch for skill search at query start. |
| `REVIEW_ARTIFACT` | `/hunter` skill for artifact review. |
| `BUILDING_CLAUDE_APPS` | `/claude-api` skill for building Claude API applications. |
| `RUN_SKILL_GENERATOR` | `/run-skill-generator` meta-skill. |
| `BUDDY` | Buddy system prompt integration. |
| `FILE_PERSISTENCE` | File persistence tracking for turn-level file access. |
| `STREAMLINED_OUTPUT` | Streamlined output formatting. |
| `BASH_CLASSIFIER` | Bash command classifier for retry logic. |
| `UNATTENDED_RETRY` | Unattended retry configuration. |
| `VERIFICATION_AGENT` | Verification agent integration with TodoWriteTool. |
| `COWORKER_TYPE_TELEMETRY` | Coworker type tracking in analytics metadata. |
| `PROMPT_CACHE_BREAK_DETECTION` | Detection of prompt cache invalidation events. |
| `LODESTONE` | Lodestone interactive helper integration. |
| `KAIROS_BRIEF` | Brief mode styling specific to KAIROS sessions. |

---

## 5. Notable Runtime Gates

Runtime gates follow the naming convention `tengu_*` (the internal codename for Claude
Code's analytics and feature system). The names are intentionally opaque — they use
codename-style identifiers (nature words, random compounds) to avoid leaking feature
semantics into telemetry pipelines and external binaries.

### 5.1 Feature Entitlement Gates

| Gate | Purpose |
|---|---|
| `tengu_ccr_bridge` | Remote Control entitlement — checked via `checkGate_CACHED_OR_BLOCKING` for blocking accuracy. |
| `tengu_ccr_bridge_multi_session` | Multi-session support within Remote Control. |
| `tengu_harbor` | KAIROS mode activation entitlement. |
| `tengu_cobalt_harbor` | Auto-connect to CCR by default (opt-out rather than opt-in). |
| `tengu_ccr_mirror` | CCR mirror mode rollout. |

### 5.2 Memory and Extraction Gates

| Gate | Purpose |
|---|---|
| `tengu_passport_quail` | Activates the background memory extraction agent. |
| `tengu_slate_thimble` | Enables memory extraction in non-interactive (headless) sessions. |
| `tengu_coral_fern` | Adds "searching past context" instructions to the memory prompt. |
| `tengu_moth_copse` | Skips MEMORY.md index inclusion in the system prompt. |
| `tengu_herring_clock` | Team memory cohort tracking telemetry. |
| `tengu_amber_prism` | Memory correction hints shown when user rejects a model action. |
| `tengu_bramble_lintel` | Controls memory extraction frequency multiplier. |

### 5.3 Bridge and Remote Control Gates

| Gate | Purpose |
|---|---|
| `tengu_bridge_repl_v2` | Env-less (v2) REPL bridge path — gates transport implementation, not availability. |
| `tengu_bridge_repl_v2_cse_shim_enabled` | Kill switch for the `cse_*` to `session_*` client-side retag shim. |
| `tengu_bridge_min_version` | Dynamic config (JSON) specifying minimum CLI version for v1 bridge. |
| `tengu_bridge_initial_history_cap` | Caps initial history messages sent over bridge. |
| `tengu_bridge_poll_interval_config` | Polling interval configuration for bridge communication. |

### 5.4 Model and Query Gates

| Gate | Purpose |
|---|---|
| `tengu_streaming_tool_execution2` | Streaming tool execution — parallel tool dispatch during stream. |
| `tengu_otk_slot_v1` | Output token cap escalation. |
| `tengu_willow_mode` | Controls idle-return behavior (off/soft/hard). |
| `tengu_ant_model_override` | Model override for ant builds. |
| `tengu_hive_evidence` | Evidence-gathering mode in system prompt. |
| `tengu_miraculo_the_bard` | Controls background refresh behavior. |
| `tengu_cicada_nap_ms` | Background refresh throttle interval in milliseconds. |

### 5.5 Safety and Security Gates

| Gate | Purpose |
|---|---|
| `tengu_iron_gate_closed` | **Fail-closed behavior** when the auto-mode classifier is unavailable. When `true` (default), classifier unavailability denies the tool call. When `false`, falls back to manual user approval. |
| `tengu_attribution_header` | Controls whether attribution headers are sent with API requests. |

### 5.6 UI and Experience Gates

| Gate | Purpose |
|---|---|
| `tengu_terminal_sidebar` | Terminal sidebar tab status display. |
| `tengu_kairos_brief` | Brief mode tool availability in KAIROS sessions. |
| `tengu_kairos_brief_config` | Brief mode configuration (JSON). |
| `tengu_chomp_inflection` | Prompt suggestion feature activation. |
| `tengu_sedge_lantern` | Away-summary feature gate. |
| `tengu_remote_backend` | Remote TUI backend for headless rendering. |
| `tengu_cobalt_frost` | Nova 3 voice model gate for speech-to-text. |

### 5.7 Analytics and Infrastructure Gates

| Gate | Purpose |
|---|---|
| `tengu_log_datadog_events` | Datadog analytics sink activation. |
| `tengu_frond_boric` | Per-sink analytics killswitch (JSON config: `{datadog?: boolean, firstParty?: boolean}`). |
| `tengu_event_sampling_config` | Event sampling rates (JSON config). |
| `tengu_1p_event_batch_config` | First-party event batching configuration (JSON config). |
| `tengu_strap_foyer` | Settings sync feature gate. |

### 5.8 Version and Update Gates

| Gate | Purpose |
|---|---|
| `tengu_max_version_config` | **Version kill switch** — JSON config with `max_version`, `ant_message`, and `external_message` fields. Forces users above a specified version to downgrade, or blocks specific versions. Uses `getDynamicConfig_BLOCKS_ON_INIT` for freshness. |

### 5.9 Cron and Scheduling Gates

| Gate | Purpose |
|---|---|
| `tengu_kairos_cron` | Enables cron/scheduled task execution. Refreshed on a 5-minute window. |
| `tengu_kairos_cron_durable` | Durable cron persistence gate. |
| `tengu_kairos_cron_config` | Cron jitter and scheduling configuration (JSON). |

---

## 6. Flag Evaluation

### 6.1 Compile-Time Evaluation

Compile-time flags are evaluated **exactly once** — at build time by Bun's bundler. The
build system maintains a mapping of flag names to boolean values for each build variant
(ant vs. external). The `feature()` call is a macro that the bundler replaces with the
appropriate literal.

No runtime cost is incurred. The guarded code either exists in the binary or does not.

### 6.2 Runtime Evaluation: Initialization

GrowthBook initializes lazily on first access. The initialization sequence:

1. `getGrowthBookClient()` creates the client with user attributes and auth headers
2. `client.init({ timeout: 5000 })` fetches flag values from the server (5-second timeout)
3. `processRemoteEvalPayload()` transforms the response and populates `remoteEvalFeatureValues`
4. `syncRemoteEvalToDisk()` writes all values to `~/.claude.json` under `cachedGrowthBookFeatures`
5. `refreshed.emit()` notifies subscribers that values are fresh

If auth is not yet available (before the trust dialog), the client skips HTTP init and
relies on disk-cached values. When auth becomes available later, `initializeGrowthBook()`
detects the change, destroys the old client, and reinitializes with fresh auth headers.

### 6.3 Runtime Evaluation: Caching

Values are cached at three levels:

1. **In-memory map** (`remoteEvalFeatureValues`): Populated by `processRemoteEvalPayload`,
   authoritative once GrowthBook init completes. This is the fastest path — a `Map.get()`.

2. **Disk cache** (`~/.claude.json → cachedGrowthBookFeatures`): Written synchronously
   after every successful payload (init and periodic refresh). Survives process restarts.
   Read via `getGlobalConfig()`, which is itself memory-cached.

3. **Legacy Statsig cache** (`~/.claude.json → cachedStatsigGates`): Fallback for the
   migration period from Statsig to GrowthBook. Checked after GrowthBook cache by
   `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()`.

### 6.4 Runtime Evaluation: Periodic Refresh

Long-running sessions (KAIROS, daemon) would go stale without periodic refresh:

```typescript
const GROWTHBOOK_REFRESH_INTERVAL_MS =
  process.env.USER_TYPE !== 'ant'
    ? 6 * 60 * 60 * 1000  // 6 hours (external)
    : 20 * 60 * 1000       // 20 minutes (ant)
```

The `setupPeriodicGrowthBookRefresh()` function starts an unref'd interval timer that
calls `refreshGrowthBookFeatures()`. This performs a light refresh — it re-fetches from
the server using the existing client (no destroy/recreate) and processes the payload to
update both in-memory and disk caches. The timer is unref'd so it does not prevent
natural process exit.

Subscribers registered via `onGrowthBookRefresh()` are notified after each successful
refresh, allowing systems that bake flag values into long-lived objects (model selection,
skill registration, event logger configuration) to rebuild.

### 6.5 Experiment Exposure Logging

When a flag is backed by a GrowthBook experiment, accessing the flag value triggers
**experiment exposure logging** — a first-party event recording which experiment variant
the user saw. Exposures are deduplicated per session (each feature logged at most once)
via `loggedExposures`. If the flag is accessed before GrowthBook init completes, the
feature is added to `pendingExposures` and logged after init succeeds.

---

## 7. Progressive Rollout

The two-tier system enables a structured rollout pipeline:

### 7.1 Stage 1: Code Lands Behind Compile-Time Flag

A new feature is developed with its code guarded by `feature('NEW_FEATURE')`. In the
ant build, the flag is `true`; in the external build, it is `false`. The feature is
invisible to external users — no code, no strings, no telemetry.

### 7.2 Stage 2: Internal Dogfooding with Runtime Gate

Within the ant build, a GrowthBook gate (e.g., `tengu_new_feature`) is created at 0%.
The feature is rolled out to internal users by percentage, organization, or individual
email. Bugs are caught before any external exposure.

### 7.3 Stage 3: External Build Enablement

Once the feature is stable, the compile-time flag is flipped to `true` in the external
build. The runtime gate controls rollout: 0% initially, ramping to 100% over days or
weeks. The gate's targeting attributes (organization, subscription, platform) enable
controlled external rollout.

### 7.4 Stage 4: Flag Removal

When the feature is generally available and stable, both the compile-time flag and the
runtime gate can be removed. The code becomes unconditional.

### 7.5 Example: Bridge Mode

Bridge mode illustrates the full pipeline. The compile-time flag `BRIDGE_MODE` gates all
bridge code and string literals. Within that gate, multiple runtime flags provide
fine-grained control:

```
feature('BRIDGE_MODE')
  └── tengu_ccr_bridge          (entitlement: org-level)
  └── tengu_bridge_repl_v2      (implementation: v1 vs v2 transport)
  └── tengu_bridge_min_version  (config: minimum CLI version)
  └── tengu_cobalt_harbor       (behavior: auto-connect default)
  └── tengu_ccr_mirror          (behavior: mirror mode opt-in)
```

---

## 8. Interaction with Commands

### 8.1 CLI Entrypoint Routing

> **Source:** `src/entrypoints/cli.tsx`

The CLI entrypoint uses compile-time flags to gate entire subcommand paths. Each
feature's subcommand handler is a "fast path" — a dynamic import that only loads
the feature's module graph when the flag is enabled and the subcommand matches:

```typescript
if (feature('DAEMON') && args[0] === 'daemon') {
  const { daemonMain } = await import('../daemon/main.js')
  await daemonMain(args.slice(1))
  return
}

if (feature('BG_SESSIONS') && (args[0] === 'ps' || args[0] === 'logs' || ...)) {
  const bg = await import('../cli/bg.js')
  // ...dispatch to handler
  return
}

if (feature('TEMPLATES') && (args[0] === 'new' || args[0] === 'list' || ...)) {
  const { templatesMain } = await import('../cli/handlers/templateJobs.js')
  await templatesMain(args)
  process.exit(0)
}
```

When the compile-time flag is `false`, the bundler eliminates the entire block. The
subcommand string (`'daemon'`, `'ps'`, `'new'`) never appears in the binary, and
the import target module is not bundled.

### 8.2 Two-Layer Gating for Subcommands

Some subcommands require both compile-time and runtime gates. Bridge mode is the
canonical example:

```typescript
// Compile-time: code exists in binary
if (feature('BRIDGE_MODE') && (args[0] === 'remote-control' || ...)) {
  // Runtime: auth check first (GrowthBook needs auth context)
  if (!getClaudeAIOAuthTokens()?.accessToken) {
    exitWithError(BRIDGE_LOGIN_ERROR)
  }
  // Runtime: GrowthBook entitlement check (blocking)
  const disabledReason = await getBridgeDisabledReason()
  if (disabledReason) {
    exitWithError(`Error: ${disabledReason}`)
  }
  // Runtime: version floor check
  const versionError = checkBridgeMinVersion()
  if (versionError) {
    exitWithError(versionError)
  }
  await bridgeMain(args.slice(1))
}
```

The compile-time flag ensures the code is present. The runtime gates ensure the
user is entitled, authenticated, and running a sufficiently recent version.

---

## 9. Interaction with Tools

### 9.1 Conditional Tool Registration

> **Source:** `src/tools.ts`

Tools are conditionally included in the tool pool through compile-time flags.
The pattern uses module-level `feature()` guards with dynamic `require()`:

```typescript
const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool
    : null

const cronTools = feature('AGENT_TRIGGERS')
  ? [
      require('./tools/ScheduleCronTool/CronCreateTool.js').CronCreateTool,
      require('./tools/ScheduleCronTool/CronDeleteTool.js').CronDeleteTool,
      require('./tools/ScheduleCronTool/CronListTool.js').CronListTool,
    ]
  : []
```

These are then spread into `getAllBaseTools()`:

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,        // always present
    BashTool,         // always present
    // ...
    ...(SleepTool ? [SleepTool] : []),
    ...cronTools,
    ...(SnipTool ? [SnipTool] : []),
    ...(WorkflowTool ? [WorkflowTool] : []),
  ]
}
```

### 9.2 Tool-Level `isEnabled()` Runtime Gating

Individual tools implement an `isEnabled()` method that performs runtime checks.
This is evaluated at request time by `getTools()`:

```typescript
const isEnabled = allowedTools.map(_ => _.isEnabled())
return allowedTools.filter((_, i) => isEnabled[i])
```

A tool that is present in the binary (compile-time flag `true`) but whose
`isEnabled()` returns `false` (runtime gate off) is excluded from the tool
descriptions sent to the model. The model never sees it and cannot call it.

### 9.3 Coordinator Mode Tool Filtering

Coordinator mode (`COORDINATOR_MODE`) dynamically reshapes the tool pool based on
agent role:

```typescript
if (
  feature('COORDINATOR_MODE') &&
  coordinatorModeModule?.isCoordinatorMode()
) {
  simpleTools.push(AgentTool, TaskStopTool, getSendMessageTool())
}
```

The compile-time flag ensures coordinator module code exists; the runtime
`isCoordinatorMode()` check determines whether the current session is operating
in coordinator mode.

### 9.4 Voice Mode Integration

Voice mode uses compile-time flags to conditionally require the entire voice
subsystem, including React hooks, providers, and the STT service:

```typescript
const VoiceProvider = feature('VOICE_MODE')
  ? require('../context/voice.js').VoiceProvider
  : ({ children }) => children

const useVoiceIntegration = feature('VOICE_MODE')
  ? require('../hooks/useVoiceIntegration.js').useVoiceIntegration
  : () => ({ /* no-op defaults */ })
```

Within the voice module, a runtime gate (`tengu_cobalt_frost`) controls which STT
model is used:

```typescript
const isNova3 = getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_frost', false)
```

---

## 10. Interaction with Services

### 10.1 Memory Extraction Service

> **Source:** `src/memdir/paths.ts`, `src/query/stopHooks.ts`

The background memory extraction agent is gated at two tiers:

```typescript
// Compile-time: module exists in binary
const extractMemoriesModule = feature('EXTRACT_MEMORIES')
  ? await import('./services/extractMemories/extractMemories.js')
  : null

// Runtime: per-user activation (called inside the handler)
function isExtractModeActive(): boolean {
  if (!getFeatureValue_CACHED_MAY_BE_STALE('tengu_passport_quail', false)) {
    return false
  }
  return (
    !getIsNonInteractiveSession() ||
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_slate_thimble', false)
  )
}
```

The comment in `src/memdir/paths.ts` explicitly documents the separation:

> "Callers must also gate on `feature('EXTRACT_MEMORIES')` — that check cannot live
> inside this helper because `feature()` only tree-shakes when used directly in an
> `if` condition."

### 10.2 Team Memory Sync Service

> **Source:** `src/setup.ts`, `src/services/teamMemorySync/`

```typescript
if (feature('TEAMMEM')) {
  void import('./services/teamMemorySync/watcher.js').then(m =>
    m.startTeamMemoryWatcher(),
  )
}
```

The team memory watcher starts only when the compile-time flag is enabled. Within
the watcher, runtime gates (`tengu_herring_clock`) control telemetry behavior.

### 10.3 Context Collapse Service

```typescript
if (feature('CONTEXT_COLLAPSE')) {
  require('./services/contextCollapse/index.js').initContextCollapse()
}
```

Context collapse initialization is gated at build time. The runtime behavior
(when to trigger context collapse vs. traditional compaction) is controlled by
the presence of the initialized service.

### 10.4 Analytics Sinks

Analytics infrastructure itself uses runtime flags:

- `tengu_log_datadog_events`: Enables/disables the Datadog analytics sink
- `tengu_frond_boric`: Per-sink killswitch (JSON config mapping sink names to booleans)
- `tengu_event_sampling_config`: Event sampling rates by event name
- `tengu_1p_event_batch_config`: Batching configuration for the first-party event logger

These are purely runtime — the analytics modules ship in all builds. The kill switches
provide emergency control over analytics data flow.

### 10.5 Settings Sync Service

> **Source:** `src/services/settingsSync/index.ts`

```typescript
if (!getFeatureValue_CACHED_MAY_BE_STALE('tengu_strap_foyer', false) || ...) {
  return  // skip sync
}
```

Settings synchronization from the server is gated by a runtime flag, enabling
controlled rollout of remote configuration.

---

## 11. Kill Switch Pattern

The kill switch is the most operationally critical pattern in the flag system. It
enables Anthropic to remotely disable a feature or enforce constraints without
deploying new code.

### 11.1 Version Kill Switch

> **Source:** `src/utils/autoUpdater.ts`

`tengu_max_version_config` is a JSON dynamic config fetched via
`getDynamicConfig_BLOCKS_ON_INIT` (blocking — must be fresh). Its schema:

```typescript
type MaxVersionConfig = {
  max_version?: string        // SemVer ceiling
  ant_message?: string        // Custom message for ant builds
  external_message?: string   // Custom message for external builds
}
```

If the current CLI version exceeds `max_version`, the system can display a message
or force downgrade. The blocking accessor ensures this check sees the most recent
server value, not a stale disk cache from a previous session.

The GrowthBook periodic refresh (20 min for ants, 6 hours for external) ensures
long-running sessions pick up version-kill changes. The `processRemoteEvalPayload`
processing on refresh was specifically added because "the init-time snapshot" broke
`tengu_max_version_config` for long-running sessions.

### 11.2 Classifier Fail-Closed Gate

> **Source:** `src/utils/permissions/permissions.ts`

`tengu_iron_gate_closed` controls behavior when the auto-mode safety classifier is
unavailable (network error, timeout):

- **`true` (default)**: Fail closed — deny the tool call and return a guidance message.
  The model sees the denial and can retry or ask the user.
- **`false`**: Fail open — fall back to normal permission handling (prompt the user).

This is a security kill switch: in an incident where the classifier API is degraded,
operators can flip this gate to control whether all auto-mode sessions fail closed
(safe but disruptive) or fail open (less safe but functional).

### 11.3 Analytics Sink Kill Switch

> **Source:** `src/services/analytics/sinkKillswitch.ts`

`tengu_frond_boric` is a JSON config that disables individual analytics sinks:

```typescript
export function isSinkKilled(sink: SinkName): boolean {
  const config = getDynamicConfig_CACHED_MAY_BE_STALE<
    Partial<Record<SinkName, boolean>>
  >(SINK_KILLSWITCH_CONFIG_NAME, {})
  return config?.[sink] === true
}
```

Setting `{ "datadog": true }` stops all Datadog dispatch immediately (within the
cache refresh window). Setting `{ "firstParty": true }` stops first-party event
logging. Default `{}` means all sinks are active. Fail-open: missing or malformed
config keeps sinks running.

### 11.4 Cron Scheduling Kill Switch

`tengu_kairos_cron` can be disabled to stop all scheduled task execution across the
fleet. Individual cron configuration (`tengu_kairos_cron_config`) can override jitter
ranges, max concurrent tasks, and other parameters during an incident.

### 11.5 Bridge Version Floor

`tengu_bridge_min_version` specifies the minimum CLI version required for Remote
Control. If a user's version is below the floor, they receive an error message
instructing them to update. This enables forcing upgrades when a critical bridge
bug is discovered.

---

## 12. Design Principles

### 12.1 Zero-Cost Absence

When a feature is off at compile time, its cost is literally zero. No code executes,
no strings appear in the binary, no modules are loaded, no memory is allocated. This
is not a runtime `if (false)` — the code does not exist in the output. This property
is essential for the external build, where internal tooling, debug utilities, and
in-development features must leave no trace.

### 12.2 Explicit Tier Separation

Compile-time and runtime flags serve different purposes and must not be conflated.
Compile-time flags are structural (what code ships). Runtime flags are behavioral
(what code runs for whom). The `QueryConfig` type explicitly documents this:
"Intentionally excludes `feature()` gates — those are tree-shaking boundaries and
must stay inline at the guarded blocks for dead-code elimination."

### 12.3 Fail-Safe Defaults

Runtime flags default to safe values. Boolean gates default to `false` (feature off).
JSON configs default to empty objects or safe structures. The version kill switch
defaults to `{ minVersion: '0.0.0' }` (no version blocked). The analytics kill switch
defaults to `{}` (all sinks active). The iron gate defaults to `true` (fail closed).

If GrowthBook is unreachable, disk cache provides continuity. If disk cache is empty,
defaults apply. If defaults are wrong, the feature is simply off — never unexpectedly on.

### 12.4 Cache-First, Refresh-Eventually

The preferred runtime accessor (`_CACHED_MAY_BE_STALE`) never blocks. It reads from
the in-memory map (microseconds) or disk cache (single JSON parse). Fresh values
arrive asynchronously via init and periodic refresh. This design ensures feature flag
evaluation never adds latency to the critical path — tool selection, permission checks,
and model requests proceed without waiting for network responses.

The exception is `_BLOCKS_ON_INIT` and `_CACHED_OR_BLOCKING` for gates where staleness
is harmful (entitlement checks, version kill switches). These block only on first access
and only when the disk cache is missing or says `false`.

### 12.5 Layered Override for Debugging

The override priority chain (env var > config file > remote eval > disk cache > default)
provides escape hatches at every level:

- **Eval harnesses** use `CLAUDE_INTERNAL_FC_OVERRIDES` for deterministic flag states
- **Developers** use the `/config` Gates tab for interactive testing
- **Operators** use GrowthBook dashboard for fleet-wide changes
- **Cross-process persistence** uses disk cache for continuity

All override mechanisms are ant-only except the GrowthBook dashboard, which controls
both ant and external builds via separate SDK keys.

### 12.6 Naming Opacity

Runtime gate names (`tengu_*`) are intentionally opaque. They do not describe their
function in their name. This prevents feature semantics from leaking through telemetry
pipelines, log aggregators, or binary inspection. The mapping from opaque name to
feature purpose lives in the GrowthBook dashboard and internal documentation, not in
the shipped code.

### 12.7 Composability

Features are routinely gated by multiple flags in combination:

```typescript
feature('PROACTIVE') || feature('KAIROS')   // either mode enables proactive tools
feature('KAIROS') && getKairosActive()       // compile + runtime activation
feature('BRIDGE_MODE') ? gate(...) : false   // compile wraps runtime
```

The two-tier system composes naturally: compile-time flags set the boundary of what is
possible; runtime flags refine what is active within that boundary.

### 12.8 Auth-Aware Initialization

GrowthBook targeting depends on user attributes (organization, subscription, email) that
are only available after authentication. The system handles the auth-not-yet-available
state gracefully:

1. Create client without auth headers — relies on disk cache
2. When auth becomes available, detect the change
3. Destroy old client, recreate with fresh auth headers
4. Re-initialize and refresh all values
5. Track `reinitializingPromise` so security gates can await completion

This ensures security-critical gates (`checkSecurityRestrictionGate`) always evaluate
against the current user's attributes, even if the initial evaluation used stale data.

### 12.9 Observable Flag State

The system provides introspection for operators and developers:

- `getAllGrowthBookFeatures()` returns the complete resolved flag map
- `getGrowthBookConfigOverrides()` shows active local overrides
- Ant builds log all flag evaluations via `logForDebugging`
- Experiment exposures are tracked for A/B test analysis
- The `/config` Gates tab provides a live UI for viewing and overriding flags

### 12.10 Incremental Migration

The system supports incremental migration between flag providers. The
`checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` function falls back to the legacy
Statsig cache when GrowthBook cache is empty, enabling a gradual migration from
Statsig to GrowthBook without a flag-day cutover. The fallback chain
(GrowthBook cache > Statsig cache > default) ensures no flag value is lost during
transition.
