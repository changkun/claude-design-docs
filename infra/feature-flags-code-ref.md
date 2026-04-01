# Feature Flags — Code Reference

## Compile-Time Flag Machinery

- **Import:** `import { feature } from 'bun:bundle'` -- appears in approximately 40 source files.
- **Bundler:** Bun's bundler replaces `feature('FLAG_NAME')` with a boolean literal at build time.

## Positive Ternary Pattern

**File:** `src/bridge/bridgeEnabled.ts`

```typescript
// CORRECT — positive ternary
export function isBridgeEnabled(): boolean {
  return feature('BRIDGE_MODE')
    ? getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)
    : false
}

// INCORRECT — negative guard (string 'tengu_ccr_bridge' leaks)
export function isBridgeEnabled(): boolean {
  if (!feature('BRIDGE_MODE')) return false
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)
}
```

Inline comments reference: `"Positive ternary pattern — see docs/feature-gating.md"`

## Module-Level Feature Guard Pattern

**File:** `src/tools.ts`

```typescript
const SnipTool = feature('HISTORY_SNIP')
  ? require('./tools/SnipTool/SnipTool.js').SnipTool
  : null
```

Also used in: `src/skills/bundled/index.ts`, `src/cli/print.ts`, `src/entrypoints/cli.tsx`

## QueryConfig Type

**File:** `src/query/config.ts`

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

## GrowthBook Client Initialization

**File:** `src/services/analytics/growthbook.ts`

```typescript
const thisClient = new GrowthBook({
  apiHost: 'https://api.anthropic.com/',
  clientKey: getGrowthBookClientKey(),
  attributes: getUserAttributes(),
  remoteEval: true,
  cacheKeyAttributes: ['id', 'organizationUUID'],
})
```

User attribute function: `getUserAttributes()` in the same file.

## SDK Keys

**File:** `src/constants/keys.ts`

| Build | Environment | Key |
|---|---|---|
| Ant (production) | prod | `sdk-xRVcrliHIlrg4og4` |
| Ant (dev) | dev | `sdk-yZQvlplybuXjYh6L` |
| External | prod | `sdk-zAZezfDKGoZuXXKe` |

## Environment Variable Override

- **Variable:** `CLAUDE_INTERNAL_FC_OVERRIDES` (ant-only)
- **Config Override:** `/config` Gates tab (ant-only)

## Disk Cache Location

- **Path:** `~/.claude.json`
- **Key (GrowthBook):** `cachedGrowthBookFeatures`
- **Key (Statsig legacy):** `cachedStatsigGates`
- **Read via:** `getGlobalConfig()`

## Accessor Functions

**File:** `src/services/analytics/growthbook.ts`

| Function | Signature |
|---|---|
| `getFeatureValue_CACHED_MAY_BE_STALE<T>()` | Non-blocking, may be stale |
| `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` | Non-blocking, Statsig migration wrapper |
| `checkGate_CACHED_OR_BLOCKING()` | Conditionally blocking |
| `checkSecurityRestrictionGate()` | Blocks if re-init pending |
| `getDynamicConfig_CACHED_MAY_BE_STALE<T>()` | Non-blocking JSON config |
| `getDynamicConfig_BLOCKS_ON_INIT<T>()` | Blocking JSON config |
| `getFeatureValue_DEPRECATED<T>()` | Legacy, blocking -- avoid |

## Periodic Refresh Configuration

**File:** `src/services/analytics/growthbook.ts`

```typescript
const GROWTHBOOK_REFRESH_INTERVAL_MS =
  process.env.USER_TYPE !== 'ant'
    ? 6 * 60 * 60 * 1000  // 6 hours (external)
    : 20 * 60 * 1000       // 20 minutes (ant)
```

Functions: `setupPeriodicGrowthBookRefresh()`, `refreshGrowthBookFeatures()`, `onGrowthBookRefresh()` subscriber registration.

Internal variables: `remoteEvalFeatureValues` (in-memory map), `loggedExposures` (dedup set), `pendingExposures` (pre-init set).

GrowthBook init timeout: `client.init({ timeout: 5000 })` (5 seconds).

## Compile-Time Flag Catalog (by name)

**Mode/Entrypoint:** `BRIDGE_MODE`, `DAEMON`, `KAIROS`, `KAIROS_CHANNELS`, `KAIROS_DREAM`, `KAIROS_PUSH_NOTIFICATION`, `KAIROS_GITHUB_WEBHOOKS`, `PROACTIVE`, `COORDINATOR_MODE`, `BG_SESSIONS`, `VOICE_MODE`

**Context/Compaction:** `REACTIVE_COMPACT`, `CONTEXT_COLLAPSE`, `HISTORY_SNIP`, `CACHED_MICROCOMPACT`, `BREAK_CACHE_COMMAND`

**Memory/Extraction:** `EXTRACT_MEMORIES`, `TEAMMEM`, `MEMORY_SHAPE_TELEMETRY`

**Tool/Capability:** `AGENT_TRIGGERS`, `AGENT_TRIGGERS_REMOTE`, `WORKFLOW_SCRIPTS`, `MONITOR_TOOL`, `WEB_BROWSER_TOOL`, `OVERFLOW_TEST_TOOL`, `TERMINAL_PANEL`, `UDS_INBOX`, `TOKEN_BUDGET`

**Infrastructure/Integration:** `TEMPLATES`, `COMMIT_ATTRIBUTION`, `CHICAGO_MCP`, `CONNECTOR_TEXT`, `TRANSCRIPT_CLASSIFIER`, `ABLATION_BASELINE`, `DUMP_SYSTEM_PROMPT`, `BYOC_ENVIRONMENT_RUNNER`, `SELF_HOSTED_RUNNER`, `CCR_AUTO_CONNECT`, `CCR_MIRROR`, `DOWNLOAD_USER_SETTINGS`, `EXPERIMENTAL_SKILL_SEARCH`, `REVIEW_ARTIFACT`, `BUILDING_CLAUDE_APPS`, `RUN_SKILL_GENERATOR`, `BUDDY`, `FILE_PERSISTENCE`, `STREAMLINED_OUTPUT`, `BASH_CLASSIFIER`, `UNATTENDED_RETRY`, `VERIFICATION_AGENT`, `COWORKER_TYPE_TELEMETRY`, `PROMPT_CACHE_BREAK_DETECTION`, `LODESTONE`, `KAIROS_BRIEF`

## Runtime Gate Catalog (by name)

**Entitlement:** `tengu_ccr_bridge`, `tengu_ccr_bridge_multi_session`, `tengu_harbor`, `tengu_cobalt_harbor`, `tengu_ccr_mirror`

**Memory:** `tengu_passport_quail`, `tengu_slate_thimble`, `tengu_coral_fern`, `tengu_moth_copse`, `tengu_herring_clock`, `tengu_amber_prism`, `tengu_bramble_lintel`

**Bridge:** `tengu_bridge_repl_v2`, `tengu_bridge_repl_v2_cse_shim_enabled`, `tengu_bridge_min_version`, `tengu_bridge_initial_history_cap`, `tengu_bridge_poll_interval_config`

**Model/Query:** `tengu_streaming_tool_execution2`, `tengu_otk_slot_v1`, `tengu_willow_mode`, `tengu_ant_model_override`, `tengu_hive_evidence`, `tengu_miraculo_the_bard`, `tengu_cicada_nap_ms`

**Safety:** `tengu_iron_gate_closed`, `tengu_attribution_header`

**UI:** `tengu_terminal_sidebar`, `tengu_kairos_brief`, `tengu_kairos_brief_config`, `tengu_chomp_inflection`, `tengu_sedge_lantern`, `tengu_remote_backend`, `tengu_cobalt_frost`

**Analytics:** `tengu_log_datadog_events`, `tengu_frond_boric`, `tengu_event_sampling_config`, `tengu_1p_event_batch_config`, `tengu_strap_foyer`

**Version:** `tengu_max_version_config`

**Cron:** `tengu_kairos_cron`, `tengu_kairos_cron_durable`, `tengu_kairos_cron_config`

## CLI Entrypoint Routing

**File:** `src/entrypoints/cli.tsx`

```typescript
if (feature('DAEMON') && args[0] === 'daemon') {
  const { daemonMain } = await import('../daemon/main.js')
  await daemonMain(args.slice(1))
  return
}

if (feature('BG_SESSIONS') && (args[0] === 'ps' || args[0] === 'logs' || ...)) {
  const bg = await import('../cli/bg.js')
  // ...
  return
}

if (feature('TEMPLATES') && (args[0] === 'new' || args[0] === 'list' || ...)) {
  const { templatesMain } = await import('../cli/handlers/templateJobs.js')
  await templatesMain(args)
  process.exit(0)
}
```

Bridge two-layer gating in same file:

```typescript
if (feature('BRIDGE_MODE') && (args[0] === 'remote-control' || ...)) {
  // Auth check, GrowthBook entitlement check (getBridgeDisabledReason()),
  // version floor check (checkBridgeMinVersion())
  await bridgeMain(args.slice(1))
}
```

## Tool Registration

**File:** `src/tools.ts`

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

export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    BashTool,
    // ...
    ...(SleepTool ? [SleepTool] : []),
    ...cronTools,
    ...(SnipTool ? [SnipTool] : []),
    ...(WorkflowTool ? [WorkflowTool] : []),
  ]
}
```

Runtime tool filtering:

```typescript
const isEnabled = allowedTools.map(_ => _.isEnabled())
return allowedTools.filter((_, i) => isEnabled[i])
```

## Coordinator Mode Tool Filtering

```typescript
if (
  feature('COORDINATOR_MODE') &&
  coordinatorModeModule?.isCoordinatorMode()
) {
  simpleTools.push(AgentTool, TaskStopTool, getSendMessageTool())
}
```

## Voice Mode

```typescript
const VoiceProvider = feature('VOICE_MODE')
  ? require('../context/voice.js').VoiceProvider
  : ({ children }) => children

const useVoiceIntegration = feature('VOICE_MODE')
  ? require('../hooks/useVoiceIntegration.js').useVoiceIntegration
  : () => ({ /* no-op defaults */ })

// Runtime within voice module:
const isNova3 = getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_frost', false)
```

## Memory Extraction Service

**Files:** `src/memdir/paths.ts`, `src/query/stopHooks.ts`

```typescript
const extractMemoriesModule = feature('EXTRACT_MEMORIES')
  ? await import('./services/extractMemories/extractMemories.js')
  : null

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

Comment in `src/memdir/paths.ts`: "Callers must also gate on `feature('EXTRACT_MEMORIES')` -- that check cannot live inside this helper because `feature()` only tree-shakes when used directly in an `if` condition."

## Team Memory Sync

**Files:** `src/setup.ts`, `src/services/teamMemorySync/`

```typescript
if (feature('TEAMMEM')) {
  void import('./services/teamMemorySync/watcher.js').then(m =>
    m.startTeamMemoryWatcher(),
  )
}
```

## Context Collapse

```typescript
if (feature('CONTEXT_COLLAPSE')) {
  require('./services/contextCollapse/index.js').initContextCollapse()
}
```

## Settings Sync

**File:** `src/services/settingsSync/index.ts`

```typescript
if (!getFeatureValue_CACHED_MAY_BE_STALE('tengu_strap_foyer', false) || ...) {
  return  // skip sync
}
```

## Version Kill Switch

**File:** `src/utils/autoUpdater.ts`

```typescript
type MaxVersionConfig = {
  max_version?: string
  ant_message?: string
  external_message?: string
}
```

Uses `getDynamicConfig_BLOCKS_ON_INIT` for freshness.

## Classifier Fail-Closed Gate

**File:** `src/utils/permissions/permissions.ts`

Gate: `tengu_iron_gate_closed` -- default `true` (fail closed).

## Analytics Sink Kill Switch

**File:** `src/services/analytics/sinkKillswitch.ts`

```typescript
export function isSinkKilled(sink: SinkName): boolean {
  const config = getDynamicConfig_CACHED_MAY_BE_STALE<
    Partial<Record<SinkName, boolean>>
  >(SINK_KILLSWITCH_CONFIG_NAME, {})
  return config?.[sink] === true
}
```

## Introspection Functions

- `getAllGrowthBookFeatures()` -- returns the complete resolved flag map
- `getGrowthBookConfigOverrides()` -- shows active local overrides
- `logForDebugging` -- ant builds log all flag evaluations
- `/config` Gates tab -- live UI for viewing and overriding flags

## GrowthBook Initialization Internal Variables

- `remoteEvalFeatureValues` -- in-memory `Map` of resolved values
- `processRemoteEvalPayload()` -- transforms server response
- `syncRemoteEvalToDisk()` -- writes to `~/.claude.json`
- `refreshed.emit()` -- notifies subscribers
- `initializeGrowthBook()` -- detects auth changes, destroys/recreates client
- `reinitializingPromise` -- tracked so security gates can await completion
- `loggedExposures` -- per-session dedup set for experiment exposures
- `pendingExposures` -- pre-init exposure queue
