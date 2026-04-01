# KAIROS — Code Reference

### File Map

| Component | File(s) | Key Exports / Symbols |
|-----------|---------|----------------------|
| State flag | `src/bootstrap/state.ts:72,1085-1090` | `kairosActive` boolean, `getKairosActive()`, `setKairosActive()` |
| State flag (dev channels) | `src/bootstrap/state.ts:33-39` | Per-entry dev channel tracking |
| Assistant module | `src/assistant/index.js` (compiled, not in repo) | `isAssistantMode()`, `markAssistantForced()`, `initializeAssistantTeam()`, `getAssistantSystemPromptAddendum()`, `getAssistantActivationPath()` |
| Entitlement gate | `src/assistant/gate.js` (compiled, not in repo) | `isKairosEnabled()` -- checks GrowthBook `tengu_kairos` |
| Session history | `src/assistant/sessionHistory.ts` | `createHistoryAuthCtx()`, `fetchLatestEvents()`, `fetchOlderEvents()` |
| Brief tool | `src/tools/BriefTool/BriefTool.ts` | `isBriefEntitled()`, `isBriefEnabled()`, tool implementation |
| Brief prompt | `src/tools/BriefTool/prompt.ts` | `BRIEF_TOOL_NAME = 'SendUserMessage'`, `LEGACY_BRIEF_TOOL_NAME = 'Brief'`, `DESCRIPTION` |
| Brief UI | `src/tools/BriefTool/UI.tsx:26-68` | Three rendering modes (transcript, brief-only, default) |
| Brief command | `src/commands/brief.ts:47-130` | `/brief` slash command toggle |
| Messages filter | `src/components/Messages.tsx:93-206` | `filterForBriefTool()` (lines 93-158), `dropTextInBriefTurns()` (lines 169-206) |
| Daily-log memory | `src/memdir/memdir.ts:319-370` | `buildAssistantDailyLogPrompt()` |
| Memory dispatch | `src/memdir/memdir.ts:419-507` | `loadMemoryPrompt()` |
| Memory dispatch (KAIROS branch) | `src/memdir/memdir.ts:427-437` | KAIROS daily-log branch |
| Memory paths | `src/memdir/paths.ts:21-55` | `isAutoMemoryEnabled()` |
| Memory paths (auto mem) | `src/memdir/paths.ts:223-235` | `getAutoMemPath()` |
| Memory paths (daily log) | `src/memdir/paths.ts:246-251` | `getAutoMemDailyLogPath()` |
| Memory types | `src/memdir/memoryTypes.ts:183-195` | Five exclusion rules |
| Bridge integration | `src/hooks/useReplBridge.tsx:155-170` | Perpetual session flag |
| Bridge pointer | `src/bridge/bridgePointer.ts:42-48` | Pointer Zod schema |
| Bridge pointer (worktree) | `src/bridge/bridgePointer.ts:129-184` | `readBridgePointerAcrossWorktrees()` |
| Bridge core | `src/bridge/replBridge.ts:211-212,294-315` | Perpetual mode logic |
| Bridge init | `src/bridge/replBridge.ts:311-477` | Perpetual init flow (read, request, reconnect, create) |
| Bridge reconnect | `src/bridge/replBridge.ts:587-836` | `doReconnect()` two-strategy recovery |
| Bridge pointer refresh | `src/bridge/replBridge.ts:1510-1526` | Hourly timer |
| Bridge teardown (perpetual) | `src/bridge/replBridge.ts:1595-1613` | Perpetual teardown |
| Bridge teardown (standard) | `src/bridge/replBridge.ts:1617-1668` | Standard teardown |
| Channel notifications | `src/services/mcp/channelNotification.ts:1-17` | Channel concept |
| Channel schema | `src/services/mcp/channelNotification.ts:37-47` | Notification Zod schema |
| Channel permission outbound | `src/services/mcp/channelNotification.ts:85-95` | Permission request format |
| Channel permission inbound | `src/services/mcp/channelNotification.ts:64-72` | Permission response schema |
| Channel XML wrap | `src/services/mcp/channelNotification.ts:103-116` | `wrapChannelMessage()`, `SAFE_META_KEY` regex, `escapeXmlAttr()` |
| Channel seven-gate | `src/services/mcp/channelNotification.ts:191-316` | `gateChannelServer()` |
| Channel permissions | `src/services/mcp/channelPermissions.ts` | Permission relay via channels |
| Permission handler | `src/hooks/toolPermission/handlers/interactiveHandler.ts:57-535` | `handleInteractivePermission()` four-way race |
| Permission handler (channels racer) | `src/hooks/toolPermission/handlers/interactiveHandler.ts:300-407` | Channel permission racer |
| Permission dispatch | `src/hooks/useCanUseTool.tsx:93-169` | Handler dispatch order |
| AutoDream | `src/services/autoDream/autoDream.ts:58-100` | Gate sequence, KAIROS exclusion |
| Dream skill | `src/skills/bundled/index.ts:35-40` | `registerDreamSkill()` registration |
| Dream skill impl | `src/skills/bundled/dream.js` (compiled) | Dream skill body |
| Consolidation prompt | `src/services/autoDream/consolidationPrompt.ts:10-65` | Four-phase workflow |
| Dream tool permissions | `src/services/extractMemories/extractMemories.ts:171-222` | `createAutoMemCanUseTool()` |
| Loop skill | `src/skills/bundled/loop.ts` | `/loop` skill |
| Cron create tool | `src/tools/ScheduleCronTool/CronCreateTool.ts:27-42` | Tool Zod schema |
| Cron prompt | `src/tools/ScheduleCronTool/prompt.ts` | Tool prompt text |
| Trust dialog | `src/utils/config.ts:697-743` | `checkHasTrustDialogAccepted()` |
| Trust dialog UI | `src/components/TrustDialog/TrustDialog.tsx:207-264` | Trust dialog rendering |
| Settings schema | `src/utils/settings/types.ts:872-887` | `assistant` / `assistantName` fields (conditional on feature flag) |
| Analytics metadata | `src/services/analytics/metadata.ts:493,736,967` | `kairosActive`, `is_assistant_mode` tracking |
| Datadog tag | `src/services/analytics/datadog.ts:72` | `kairosActive` in tag set |
| Fast mode | `src/utils/fastMode.ts:96-110` | KAIROS exemption from SDK fast-mode block |
| Fast mode (model check) | `src/utils/fastMode.ts:167-176` | Supported model list (currently Opus 4.6 only) |
| Proactive module | `src/utils/systemPrompt.ts:14-20` | Conditional `require('../proactive/index.js')` |
| Proactive prompt | `src/constants/prompts.ts:860-914` | Autonomous work section |
| Proactive composition | `src/utils/systemPrompt.ts:103-113` | Append vs. replace behavior |
| Sleep tool | `src/tools/SleepTool/SleepTool.js` (compiled) | `SleepTool`, `isEnabled() -> isProactiveActive()` |
| Sleep tool gate | `src/tools.ts:25-28` | Conditional require |
| AppState (brief) | `src/state/AppStateStore.ts:96` | `isBriefOnly: boolean` |
| AppState (kairos) | `src/state/AppStateStore.ts:116` | `kairosEnabled: boolean` |
| User prompt UI | `src/components/UserPromptMessage.tsx:51-61` | `useBriefLayout` computation |
| Spinner | `src/components/Spinner.tsx:62-81` | `SpinnerWithVerb` / `BriefSpinner` branching |
| Agent tool async | `src/tools/AgentTool/AgentTool.tsx:566` | `assistantForceAsync` |
| Teammate snapshot | `src/utils/swarm/backends/teammateModeSnapshot.ts:55-67` | `captureTeammateModeSnapshot()` |
| Spawn dispatch | `src/tools/shared/spawnMultiAgent.ts:1040-1078` | `handleSpawn()` backend selection |
| In-process spawn | `src/tools/shared/spawnMultiAgent.ts:840-1034` | `handleSpawnInProcess()` |
| Session viewer CLI | `src/main.tsx:685-700` | Argument parsing for `claude assistant` |
| Session viewer init | `src/main.tsx:3259-3354` | Viewer initialization |
| History hook | `src/hooks/useAssistantHistory.ts:72-239` | `useAssistantHistory` lazy-loading |
| WebSocket | `src/remote/SessionsWebSocket.ts:74-163` | `SessionsWebSocket` |
| Message posting | `src/utils/teleport/api.ts:349-417` | `sendEventToRemoteSession()` |
| Main orchestrator references | `src/main.tsx` | 30+ KAIROS references throughout |

### Key Code Snippets and Line References

**Module Loading (main.tsx:80-81)**:
```typescript
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') as typeof import('./assistant/index.js')
  : null;
const kairosGate = feature('KAIROS')
  ? require('./assistant/gate.js') as typeof import('./assistant/gate.js')
  : null;
```

**Activation Gate (main.tsx:1048-1089)**:
```typescript
// Trust gate check at ~1043-1069
if (!checkHasTrustDialogAccepted()) {
  console.warn('Assistant mode disabled: directory is not trusted...');
}
// Teammate exclusion at ~1059-1066
!(options as { agentId?: unknown }).agentId && kairosGate
// Activation at ~1082-1089
if (kairosEnabled) {
  assistantTeamContext = await assistantModule.initializeAssistantTeam();
}
```

**Permission mode comment (main.tsx:1034-1039)**:
```typescript
// Assistant mode: when .claude/settings.json has assistant: true AND
// the tengu_kairos GrowthBook gate is on, force brief on. Permission
// mode is left to the user.
```

**Brief entitlement chain (BriefTool.ts:88-134)**:
```typescript
// isBriefEntitled(): kairosActive || env || GrowthBook
// isBriefEnabled(): (kairosActive || userMsgOptIn) && isBriefEntitled()
```

**Brief tool schema (BriefTool.ts:20-63)**:
```typescript
// Input
z.strictObject({
  message: z.string(),
  attachments: z.array(z.string()).optional(),
  status: z.enum(['normal', 'proactive']),
})
// Output
z.object({
  message: z.string(),
  attachments: z.array(z.object({
    path: z.string(), size: z.number(), isImage: z.boolean(),
    file_uuid: z.string().optional(),
  })).optional(),
  sentAt: z.string().optional(),
})
```

**Brief tool execution (BriefTool.ts:186-203)**:
```typescript
async call({ message, attachments, status }, context) {
  const sentAt = new Date().toISOString()
  logEvent('tengu_brief_send', { proactive: status === 'proactive', attachment_count: ... })
  // resolve attachments if present, return { data: { message, attachments, sentAt } }
}
```

**Brief prompt names (prompt.ts:1-4)**:
```typescript
export const BRIEF_TOOL_NAME = 'SendUserMessage'
export const LEGACY_BRIEF_TOOL_NAME = 'Brief'
export const DESCRIPTION = 'Send a message to the user'
```

**Brief system prompt integration (main.tsx:2201)**:
```typescript
const briefVisibility = isBriefEnabled()
  ? 'Call SendUserMessage at checkpoints to mark where things stand.'
  : 'The user will see any text you output.';
```

**Brief opt-in paths (main.tsx:1728-1742, 2184-2192, 4622-4652)**: Six paths documented in Section 4.3.

**Memory prompt KAIROS branch (memdir.ts:427-437)**:
```typescript
if (feature('KAIROS') && autoEnabled && getKairosActive()) {
  return buildAssistantDailyLogPrompt(skipIndex)
}
```

**Daily log path (paths.ts:246-251)**:
```typescript
export function getAutoMemDailyLogPath(date = new Date()): string {
  const yyyy = date.getFullYear().toString()
  const mm = (date.getMonth() + 1).toString().padStart(2, '0')
  const dd = date.getDate().toString().padStart(2, '0')
  return join(getAutoMemPath(), 'logs', yyyy, mm, `${yyyy}-${mm}-${dd}.md`)
}
```

**Auto mem path (paths.ts:223-235)**:
```typescript
export const getAutoMemPath = memoize((): string => {
  const override = getAutoMemPathOverride() ?? getAutoMemPathSetting()
  if (override) return override
  const projectsDir = join(getMemoryBaseDir(), 'projects')
  return (join(projectsDir, sanitizePath(getAutoMemBase()), 'memory') + sep).normalize('NFC')
}, () => getProjectRoot())
```

**Perpetual bridge flag (useReplBridge.tsx:155-169)**:
```typescript
let perpetual = false;
if (feature('KAIROS')) {
  const { isAssistantMode } = await import('../assistant/index.js');
  perpetual = isAssistantMode();
}
```

**Bridge pointer schema (bridgePointer.ts:42-48)**:
```typescript
z.object({
  sessionId: z.string(),
  environmentId: z.string(),
  source: z.enum(['standalone', 'repl']),
})
```
TTL: `BRIDGE_POINTER_TTL_MS = 4 * 60 * 60 * 1000` (4 hours).

**Pointer refresh timer (replBridge.ts:1510-1526)**:
```typescript
const pointerRefreshTimer = perpetual
  ? setInterval(() => {
      if (reconnectPromise) return
      void writeBridgePointer(dir, { sessionId, environmentId, source: 'repl' })
    }, 60 * 60_000)
  : null
```

**Channel notification schema (channelNotification.ts:37-47)**:
```typescript
z.object({
  method: z.literal('notifications/claude/channel'),
  params: z.object({
    content: z.string(),
    meta: z.record(z.string(), z.string()).optional(),
  }),
})
```

**Channel XML wrapping (channelNotification.ts:103-116)**:
```typescript
const SAFE_META_KEY = /^[a-zA-Z_][a-zA-Z0-9_]*$/
export function wrapChannelMessage(serverName, content, meta) {
  const attrs = Object.entries(meta ?? {})
    .filter(([k]) => SAFE_META_KEY.test(k))
    .map(([k, v]) => ` ${k}="${escapeXmlAttr(v)}"`)
    .join('')
  return `<channel source="${escapeXmlAttr(serverName)}"${attrs}>\n${content}\n</channel>`
}
```

**Channel permission relay (channelNotification.ts:64-95)**:
```typescript
// Inbound: { method: 'notifications/claude/channel/permission',
//            params: { request_id, behavior: 'allow' | 'deny' } }
// Outbound: { request_id, tool_name, description, input_preview }
// Input preview truncated to 200 chars. Request IDs: 5-letter, no 'l', profanity-filtered.
```

**Cron create tool schema (CronCreateTool.ts:27-42)**:
```typescript
z.strictObject({
  cron: z.string(),   // 5-field cron: "M H DoM Mon DoW"
  prompt: z.string(),
  recurring: z.boolean().optional(),  // default true
  durable: z.boolean().optional(),    // default false
})
```

**Cron persistence files**:
- Durable: `.claude/scheduled_tasks.json`
- Lock: `.claude/scheduled_tasks.lock` -- `{ sessionId, pid, acquiredAt }`

**AutoDream gate (autoDream.ts:95-100)**:
```typescript
function isGateOpen(): boolean {
  if (getKairosActive()) return false
  if (getIsRemoteMode()) return false
  if (!isAutoMemoryEnabled()) return false
  return isAutoDreamEnabled()
}
```

**Dream skill registration (skills/bundled/index.ts:35-40)**:
```typescript
if (feature('KAIROS') || feature('KAIROS_DREAM')) {
  const { registerDreamSkill } = require('./dream.js')
  registerDreamSkill()
}
```

**Consolidation lock file**: `.claude/auto_mem/.consolidate-lock` (PID as body, 1-hour stale threshold).

**Trust dialog (config.ts:697-743)**: Checks session trust, global config, parent directories. Latches `true`.

**Settings schema (settings/types.ts:872-887)**:
```typescript
...(feature('KAIROS') ? {
  assistant: z.boolean().optional(),
  assistantName: z.string().optional(),
} : {}),
```

**Team context precedence (main.tsx:3031-3035)**:
```typescript
teamContext: feature('KAIROS')
  ? assistantTeamContext ?? computeInitialTeamContext?.()
  : computeInitialTeamContext?.()
```

**Agent async forcing (AgentTool.tsx:566)**:
```typescript
const assistantForceAsync = feature('KAIROS') ? appState.kairosEnabled : false;
```

**Fast mode exemption (fastMode.ts:96-110)**:
```typescript
if (getIsNonInteractiveSession() && preferThirdPartyAuthentication() && !getKairosActive()) {
  // Block fast mode in SDK unless explicitly opted in
}
```

**Fast mode model check (fastMode.ts:167-176)**: Currently only Opus 4.6 in supported set.

**Session history API (sessionHistory.ts)**:
- `fetchLatestEvents()`: `GET /v1/sessions/{id}/events?limit=100&anchor_to_latest=true`
- `fetchOlderEvents()`: `GET /v1/sessions/{id}/events?limit=100&before_id={cursor}`
- Auth: OAuth Bearer + `x-organization-uuid` + `anthropic-beta: ccr-byoc-2025-07-29`

**WebSocket URL (SessionsWebSocket.ts:74-163)**: `wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe`

**Message POST (teleport/api.ts:349-417)**: POST to `/v1/sessions/{sessionId}/events`, 30s timeout, echo filtering via `BoundedUUIDSet`.

**Viewer initialization (main.tsx:3259-3354)**:
- `kairosActive = true`, `userMsgOptIn = true`, `isRemoteMode = true`
- `RemoteSessionConfig` with `viewerOnly: true`
- `initialTools: []`, `isBriefOnly: true`

**CLI channel flags (main.tsx:1642, 3844)**: `--channels` and `--dangerously-load-development-channels`.

**Analytics events**:
- `tengu_trust_dialog_shown` / `tengu_trust_dialog_accept` -- logged with risk factors
- `tengu_brief_mode_toggled`, `tengu_brief_mode_enabled` (main.tsx:4623-4652)
- `tengu_brief_send` -- from Brief tool
- `tengu_agent_memory_loaded` -- from memory system

**Analytics metadata**:
```typescript
{ kairosActive: true }            // metadata.ts:736
{ is_assistant_mode: true }       // metadata.ts:967
{ assistantActivationPath: ... }  // main.tsx:2518
```

**Proactive module loading (systemPrompt.ts:14-20)**:
```typescript
const proactiveModule =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('../proactive/index.js')
    : null
```

**Sleep tool gating (tools.ts:25-28)**:
```typescript
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

**GrowthBook accessor functions**:
- `getFeatureValue_CACHED_MAY_BE_STALE<T>(feature, default)` -- preferred for startup-critical/sync contexts
- `getFeatureValue_CACHED_WITH_REFRESH<T>(feature, default, refreshMs)` -- **deprecated**, delegates to `_CACHED_MAY_BE_STALE`

**Memory prompt assembly dispatch (memdir.ts:419-507)**:
1. KAIROS + auto + active -> `buildAssistantDailyLogPrompt(skipIndex)`
2. TEAMMEM enabled -> `buildCombinedMemoryPrompt()`
3. Auto only -> `buildMemoryLines()`
4. Disabled -> log `tengu_memdir_disabled`, return null

**Past-context search (memdir.ts:375-407)**: Gated by `tengu_coral_fern`. Instructs grep of `.claude/auto_mem/` and JSONL transcripts.

**Date rollover note (memdir.ts:329-334)**: Pattern-based path, date derived from `date_change` attachment rather than stale user-context.

**Permission four-way race (interactiveHandler.ts:57-535)**:
| Racer | Lines | Trigger |
|-------|-------|---------|
| Bridge (CCR/claude.ai) | 234-298 | `bridgeCallbacks.sendRequest()` |
| Channels (KAIROS) | 300-407 | `channelCallbacks.onResponse()` |
| Permission hooks | 410-431 | `ctx.runHooks()` |
| Async bash classifier | 433-530 | `executeAsyncClassifierCheck()` |

**Handler dispatch (useCanUseTool.tsx:93-169)**: Three modes -- coordinator (sequential, no channels/bridge), swarm worker (forward to leader), interactive (full four-way race).

**Teammate mode snapshot (teammateModeSnapshot.ts:55-67)**: `captureTeammateModeSnapshot()`, type `'auto' | 'tmux' | 'in-process'`, immutable for session.

**Team context structure**:
```typescript
{
  teamName: string,
  teamFilePath: string,
  leadAgentId: string,
  selfAgentId: string | undefined,
  selfAgentName: string | undefined,
  isLeader: boolean,
  teammates: { [agentId: string]: TeammateInfo },
}
```

**Spawn backend selection (spawnMultiAgent.ts:1040-1078)**: `handleSpawn()` dispatches to `handleSpawnInProcess()`, `handleSpawnSplitPane()`, `handleSpawnSeparateWindow()`, or `markInProcessFallback()`.

**In-process spawn (spawnMultiAgent.ts:840-1034)**: 8-step lifecycle with `AbortController`, `TeammateContext`, `AsyncLocalStorage`, fire-and-forget execution loop.

**Tmux CLI args propagation**: `--agent-id`, `--agent-name`, `--team-name`, `--agent-color`, `--parent-session-id`.

**Streaming text visibility in brief mode**:
```typescript
{streamingText && !isBriefOnly && <Box>...</Box>}
{isStreamingThinkingVisible && streamingThinking && !isBriefOnly && <Box>...</Box>}
```

**Brief layout detection (UserPromptMessage.tsx:51-61)**:
```
(getKairosActive() || getUserMsgOptIn() && entitlement) && isBriefOnly && !isTranscriptMode && !viewingAgentTaskId
```
