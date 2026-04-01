# Permission Modes — Code Reference

*(All code-level details: file paths, function/class/variable names, types, line numbers, implementation-specific references)*

All paths are relative to `src/`.

### Permission Mode Types and Constants

| Component | Location |
|-----------|----------|
| `ExternalPermissionMode` type | `types/permissions.ts:24` |
| `InternalPermissionMode` type | `types/permissions.ts:28` |
| `PermissionMode` type | `types/permissions.ts:29` |
| `EXTERNAL_PERMISSION_MODES` array | `types/permissions.ts:16-22` |
| `INTERNAL_PERMISSION_MODES` array | `types/permissions.ts:33-36` |
| `PERMISSION_MODES` array | `types/permissions.ts:38` |
| Mode config (title, symbol, color) | `utils/permissions/PermissionMode.ts:42-91` |
| `permissionModeFromString()` | `utils/permissions/PermissionMode.ts:117` |
| `permissionModeTitle()` | `utils/permissions/PermissionMode.ts:123` |
| `isExternalPermissionMode()` | `utils/permissions/PermissionMode.ts:97` |
| `toExternalPermissionMode()` | `utils/permissions/PermissionMode.ts:111` |

### Permission Decision Types

| Component | Location |
|-----------|----------|
| `PermissionAllowDecision` | `types/permissions.ts:174` |
| `PermissionAskDecision` | `types/permissions.ts:199` |
| `PermissionDenyDecision` | `types/permissions.ts:231` |
| `PermissionDecision` union | `types/permissions.ts:241` |
| `PermissionResult` (with passthrough) | `types/permissions.ts:251` |
| `PermissionDecisionReason` tagged union | `types/permissions.ts:271-324` |
| `PendingClassifierCheck` | `types/permissions.ts:190` |
| `ToolPermissionContext` | `types/permissions.ts:427-441` |

### Permission Rule Types

| Component | Location |
|-----------|----------|
| `PermissionBehavior` | `types/permissions.ts:44` |
| `PermissionRuleSource` (8 sources) | `types/permissions.ts:54-62` |
| `PermissionRuleValue` | `types/permissions.ts:67-70` |
| `PermissionRule` | `types/permissions.ts:75-79` |
| `PermissionUpdate` | `types/permissions.ts:98-131` |
| `PermissionUpdateDestination` | `types/permissions.ts:88-93` |

### Core Permission Pipeline

| Component | Location |
|-----------|----------|
| `hasPermissionsToUseTool()` (outer function) | `utils/permissions/permissions.ts:473` |
| `hasPermissionsToUseToolInner()` (inner function) | `utils/permissions/permissions.ts:1158` |
| `dontAsk` -> deny transformation | `utils/permissions/permissions.ts:503-517` |
| Auto mode entry point | `utils/permissions/permissions.ts:519-927` |
| Safety check immunity (auto mode) | `utils/permissions/permissions.ts:532-548` |
| `acceptEdits` fast path (within auto) | `utils/permissions/permissions.ts:600-656` |
| Safe tool allowlist check | `utils/permissions/permissions.ts:658-686` |
| Classifier invocation | `utils/permissions/permissions.ts:688-702` |
| Classifier result handling | `utils/permissions/permissions.ts:718-926` |
| Headless agent auto-deny | `utils/permissions/permissions.ts:929-952` |
| Bypass mode check | `utils/permissions/permissions.ts:1262-1281` |
| Always-allow rule check | `utils/permissions/permissions.ts:1283-1297` |
| Passthrough to ask conversion | `utils/permissions/permissions.ts:1299-1310` |
| `checkRuleBasedPermissions()` | `utils/permissions/permissions.ts:1071` |

**Pipeline architecture note**: The outer function (`hasPermissionsToUseTool` at :473) calls the inner function (`hasPermissionsToUseToolInner` at :1158) and then applies mode-specific transformations. The inner function handles steps [1]-[9] of the pipeline (deny rules, ask rules, tool checks, bypass, always-allow, passthrough). The outer function handles step [10] (dontAsk at :503, auto mode at :519, and the fall-through to interactive prompting).

### Rule Loading and Management

| Component | Location |
|-----------|----------|
| `loadAllPermissionRulesFromDisk()` | `utils/permissions/permissionsLoader.ts:120` |
| `settingsJsonToRules()` | `utils/permissions/permissionsLoader.ts:91` |
| `getPermissionRulesForSource()` | `utils/permissions/permissionsLoader.ts:140` |
| `addPermissionRulesToSettings()` | `utils/permissions/permissionsLoader.ts:229` |
| `deletePermissionRuleFromSettings()` | `utils/permissions/permissionsLoader.ts:163` |
| `shouldAllowManagedPermissionRulesOnly()` | `utils/permissions/permissionsLoader.ts:31` |
| `applyPermissionUpdate()` | `utils/permissions/PermissionUpdate.ts:55` |
| `persistPermissionUpdates()` | `utils/permissions/PermissionUpdate.ts:222` |

### Mode Setup and Transitions

| Component | Location |
|-----------|----------|
| `initialPermissionModeFromCLI()` | `utils/permissions/permissionSetup.ts:689` |
| `initializeToolPermissionContext()` | `utils/permissions/permissionSetup.ts:872` |
| `transitionPermissionMode()` | `utils/permissions/permissionSetup.ts:597-646` |
| `prepareContextForPlanMode()` | `utils/permissions/permissionSetup.ts:1462` |
| `transitionPlanAutoMode()` | `utils/permissions/permissionSetup.ts:1502` |
| `getNextPermissionMode()` | `utils/permissions/getNextPermissionMode.ts:34` |
| `cyclePermissionMode()` | `utils/permissions/getNextPermissionMode.ts:88` |

### Auto Mode State

| Component | Location |
|-----------|----------|
| `autoModeActive` flag | `utils/permissions/autoModeState.ts:4` |
| `setAutoModeActive()` | `utils/permissions/autoModeState.ts:11` |
| `isAutoModeActive()` | `utils/permissions/autoModeState.ts:15` |
| `autoModeCircuitBroken` flag | `utils/permissions/autoModeState.ts:9` |
| `setAutoModeCircuitBroken()` | `utils/permissions/autoModeState.ts:27` |
| `isAutoModeCircuitBroken()` | `utils/permissions/autoModeState.ts:31` |
| `autoModeFlagCli` flag | `utils/permissions/autoModeState.ts:5` |
| `isAutoModeGateEnabled()` | `utils/permissions/permissionSetup.ts:1283` |
| `verifyAutoModeGateAccess()` | `utils/permissions/permissionSetup.ts:1078` |
| `getAutoModeEnabledState()` | `utils/permissions/permissionSetup.ts:1328` |
| `AutoModeEnabledState` type | `utils/permissions/permissionSetup.ts:1311` |

### Dangerous Pattern Stripping

| Component | Location |
|-----------|----------|
| `CROSS_PLATFORM_CODE_EXEC` patterns | `utils/permissions/dangerousPatterns.ts:18-42` |
| `DANGEROUS_BASH_PATTERNS` patterns | `utils/permissions/dangerousPatterns.ts:44-80` |
| `isDangerousBashPermission()` | `utils/permissions/permissionSetup.ts:94-147` |
| `isDangerousPowerShellPermission()` | `utils/permissions/permissionSetup.ts:157-233` |
| `isDangerousTaskPermission()` | `utils/permissions/permissionSetup.ts:240-245` |
| `isDangerousClassifierPermission()` | `utils/permissions/permissionSetup.ts:272-285` |
| `findDangerousClassifierPermissions()` | `utils/permissions/permissionSetup.ts:295-342` |
| `stripDangerousPermissionsForAutoMode()` | `utils/permissions/permissionSetup.ts:510-553` |
| `restoreDangerousPermissions()` | `utils/permissions/permissionSetup.ts:561-579` |
| `isOverlyBroadBashAllowRule()` | `utils/permissions/permissionSetup.ts:351` |
| `isOverlyBroadPowerShellAllowRule()` | `utils/permissions/permissionSetup.ts:365` |

### AI Classifier

| Component | Location |
|-----------|----------|
| `classifyYoloAction()` entry | `utils/permissions/yoloClassifier.ts:1012` |
| `classifyYoloActionXml()` 2-stage | `utils/permissions/yoloClassifier.ts:711-996` |
| `buildYoloSystemPrompt()` | `utils/permissions/yoloClassifier.ts:484-540` |
| `buildTranscriptEntries()` | `utils/permissions/yoloClassifier.ts:302-360` |
| `buildClaudeMdMessage()` | `utils/permissions/yoloClassifier.ts:460-477` |
| `formatActionForClassifier()` | `utils/permissions/yoloClassifier.ts:1487` |
| Stage 1 suffix | `utils/permissions/yoloClassifier.ts:550` |
| Stage 2 suffix | `utils/permissions/yoloClassifier.ts:560-561` |
| `replaceOutputFormatWithXml()` | `utils/permissions/yoloClassifier.ts:648` |
| `parseXmlBlock()` | `utils/permissions/yoloClassifier.ts:578` |
| `parseXmlReason()` | `utils/permissions/yoloClassifier.ts:590` |
| `parseXmlThinking()` | `utils/permissions/yoloClassifier.ts:601` |
| `stripThinking()` | `utils/permissions/yoloClassifier.ts:567` |
| `getClassifierModel()` | `utils/permissions/yoloClassifier.ts:1334` |
| `getClassifierThinkingConfig()` | `utils/permissions/yoloClassifier.ts:683` |
| Safe tool allowlist | `utils/permissions/classifierDecision.ts:56-94` |
| `isAutoModeAllowlistedTool()` | `utils/permissions/classifierDecision.ts:96` |
| `YoloClassifierResult` type | `types/permissions.ts:346-397` |
| `AutoModeRules` type | `utils/permissions/yoloClassifier.ts:85-89` |

**Internal naming note**: The classifier is referred to as "yolo" in the codebase (e.g., `yoloClassifier.ts`, `classifyYoloAction`). This is a legacy codename for auto mode. The GrowthBook configs use the prefix `tengu_` (e.g., `tengu_auto_mode_config`, `tengu_disable_bypass_permissions_mode`).

### Denial Tracking

| Component | Location |
|-----------|----------|
| `DenialTrackingState` type | `utils/permissions/denialTracking.ts:7-10` |
| `DENIAL_LIMITS` constants | `utils/permissions/denialTracking.ts:12-15` |
| `createDenialTrackingState()` | `utils/permissions/denialTracking.ts:17` |
| `recordDenial()` | `utils/permissions/denialTracking.ts:24` |
| `recordSuccess()` | `utils/permissions/denialTracking.ts:32` |
| `shouldFallbackToPrompting()` | `utils/permissions/denialTracking.ts:40` |
| `persistDenialState()` | `utils/permissions/permissions.ts:963-978` |
| `handleDenialLimitExceeded()` | `utils/permissions/permissions.ts:984-1058` |

**TypeScript type definition**:
```typescript
type DenialTrackingState = {
  consecutiveDenials: number
  totalDenials: number
}
```

### Bypass Mode Kill Switch

| Component | Location |
|-----------|----------|
| `shouldDisableBypassPermissions()` | `utils/permissions/permissionSetup.ts:1265` |
| `isBypassPermissionsModeDisabled()` | `utils/permissions/permissionSetup.ts:1371` |
| `createDisabledBypassPermissionsContext()` | `utils/permissions/permissionSetup.ts:1389` |
| `checkAndDisableBypassPermissions()` | `utils/permissions/permissionSetup.ts:1411` |
| `checkAndDisableBypassPermissionsIfNeeded()` | `utils/permissions/bypassPermissionsKillswitch.ts:19` |
| `checkAndDisableAutoModeIfNeeded()` | `utils/permissions/bypassPermissionsKillswitch.ts:74` |
| `useKickOffCheckAndDisableBypassPermissionsIfNeeded()` | `utils/permissions/bypassPermissionsKillswitch.ts:57` |
| `useKickOffCheckAndDisableAutoModeIfNeeded()` | `utils/permissions/bypassPermissionsKillswitch.ts:127` |

**Feature gate names**:
- Bypass mode kill switch: `tengu_disable_bypass_permissions_mode`
- Auto mode config: `tengu_auto_mode_config`
- Fail-closed gate: `iron_gate_closed`
- Two-stage classifier config: `twoStageClassifier`

**Build flags**:
- `TRANSCRIPT_CLASSIFIER`: gates auto mode availability
- `POWERSHELL_AUTO_MODE`: gates PowerShell auto-allow in auto mode

**Settings fields**:
- `permissions.disableBypassPermissionsMode: 'disable'`
- `permissions.disableAutoMode: 'disable'`
- `permissions.defaultMode`
- `allowManagedPermissionRulesOnly: true`
- `useAutoModeDuringPlan`

**Context fields**:
- `ToolPermissionContext` at `types/permissions.ts:427-441`
- `prePlanMode`: stashed mode before plan transition
- `isBypassPermissionsModeAvailable`: remembers bypass as origin mode
- `strippedDangerousRules`: stash of rules removed for auto mode
- `awaitAutomatedChecksBeforeDialog`: controls sequential processing in coordinator workers
- `localDenialTracking`: mutable reference for subagent denial state on `ToolUseContext`

**Subagent state mutation**: Subagent denial tracking uses `Object.assign` for in-place mutation because subagent `setAppState` is a no-op (`utils/permissions/permissions.ts:963-978`).

### Human-in-the-Loop Dialog

| Component | Location |
|-----------|----------|
| Interactive handler | `hooks/toolPermission/handlers/interactiveHandler.ts:57` |
| Coordinator handler | `hooks/toolPermission/handlers/coordinatorHandler.ts:26` |
| Swarm worker handler | `hooks/toolPermission/handlers/swarmWorkerHandler.ts:40` |
| `createResolveOnce()` (atomic claim) | `hooks/toolPermission/PermissionContext.ts:75-94` |
| 200ms grace period | `hooks/toolPermission/handlers/interactiveHandler.ts:108-122` |
| Checkmark timer (3s focused, 1s unfocused) | `hooks/toolPermission/handlers/interactiveHandler.ts:495-520` |

### Permission UI Components

| Component | Location |
|-----------|----------|
| `BashPermissionRequest` | `components/permissions/BashPermissionRequest/BashPermissionRequest.tsx` |
| `FileEditPermissionRequest` | `components/permissions/FileEditPermissionRequest/FileEditPermissionRequest.tsx` |
| `PermissionDialog` | `components/permissions/PermissionDialog.tsx` |
| `PermissionPrompt` | `components/permissions/PermissionPrompt.tsx` |
| `PermissionExplanation` (risk levels) | `components/permissions/PermissionExplanation.tsx:41-59` |
| `SandboxPermissionRequest` | `components/permissions/SandboxPermissionRequest.tsx` |
| `PermissionRuleList` | `components/permissions/rules/PermissionRuleList.tsx` |
| `RecentDenialsTab` | `components/permissions/rules/RecentDenialsTab.tsx` |

### Validation Checklist

When validating the design against the codebase, verify:

1. `hasPermissionsToUseTool()` at `:473` calls `hasPermissionsToUseToolInner()` at `:1158` and applies dontAsk (`:503-517`) and auto mode (`:519-927`) transformations on the result.
2. The inner function handles bypass mode (`:1262-1281`), always-allow rules (`:1283-1297`), and passthrough conversion (`:1299-1310`).
3. The `DenialTrackingState` type at `denialTracking.ts:7-10` has exactly two fields: `consecutiveDenials` and `totalDenials`.
4. `DENIAL_LIMITS` at `denialTracking.ts:12-15` defines `consecutive: 3` and `total: 20`.
5. `CROSS_PLATFORM_CODE_EXEC` at `dangerousPatterns.ts:18-42` includes the interpreters, package runners, shells, and execution primitives listed in Section 6.1.
6. `classifyYoloActionXml()` at `yoloClassifier.ts:711-996` implements the 2-stage pipeline with stage 1 (64 max tokens, stop at `</block>`) and stage 2 (4096 max tokens, chain-of-thought).
7. `createResolveOnce()` at `PermissionContext.ts:75-94` implements the atomic claim pattern.
8. The five external modes in `EXTERNAL_PERMISSION_MODES` at `types/permissions.ts:16-22` match: `default`, `plan`, `acceptEdits`, `dontAsk`, `bypassPermissions`.
9. `auto` is in `INTERNAL_PERMISSION_MODES` at `types/permissions.ts:33-36`.
10. The safe tool allowlist at `classifierDecision.ts:56-94` contains the categories listed in Section 5.2 (read-only file ops, search tools, task management, plan mode tools, sleep).
