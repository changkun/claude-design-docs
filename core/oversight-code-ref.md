# Oversight — Code Reference

### Permission Decision Types

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

### L1: System Prompt

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

### L2: Input Validation

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

### L3: Permission Rules

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

### L4: Tool-Specific Checks

| Tool | `checkPermissions` | Key Logic |
|------|--------------------|-----------|
| BashTool | `tools/BashTool/BashTool.tsx:539` -> `tools/BashTool/bashPermissions.ts:1663` | AST parse, sandbox auto-allow, exact/prefix rules, Haiku classifier, pipe validation |
| FileEditTool | `tools/FileEditTool/FileEditTool.ts:125` -> `utils/permissions/filesystem.ts:1205` | Deny rules (symlink-aware), path safety, `.claude/` session scoping |
| FileWriteTool | `tools/FileWriteTool/FileWriteTool.ts:135` -> `utils/permissions/filesystem.ts:1205` | Same shared `checkWritePermissionForTool` |
| AgentTool | `tools/AgentTool/AgentTool.tsx:1281` | Allow in non-auto modes, passthrough in auto |
| MCPTool | `tools/MCPTool/MCPTool.ts:56` | Always passthrough |
| Path safety checks | `utils/permissions/filesystem.ts:620-665` | Windows patterns, Claude config, dangerous files |

### L5: Bash Security Scanner

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

### L6: Dangerous Pattern Stripping

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

### L7: Sandbox

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

### L8: Mode Transformation

| Transformation | Location |
|---------------|----------|
| `hasPermissionsToUseTool()` outer | `utils/permissions/permissions.ts:473` |
| `hasPermissionsToUseToolInner()` | `utils/permissions/permissions.ts:1158` |
| `dontAsk` -> `deny` | `utils/permissions/permissions.ts:503-517` |
| Auto mode entry | `utils/permissions/permissions.ts:519-927` |
| `acceptEdits` fast path | `utils/permissions/permissions.ts:600-656` |
| Safety check immunity | `utils/permissions/permissions.ts:532-548` |
| Headless agent auto-deny | `utils/permissions/permissions.ts:929-952` |

### L9: Hooks

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

### L10: AI Classifier

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

### L11: Denial Limits

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

### L12: Human-in-the-Loop Dialog

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

### L13: Decision Audit

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
| Metadata marker type | `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` (referenced in Section 7) |

### L14: Query Pipeline Safety

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

### Implementation Notes

- The `TRANSCRIPT_CLASSIFIER` feature flag gates auto mode availability at the build level.
- The `iron_gate_closed` feature gate controls fail-closed behavior on classifier API errors (separate from the transcript classifier feature flag).
- The classifier internally uses the name "Yolo" (as seen in `classifyYoloAction`, `YoloClassifierResult`, `buildYoloSystemPrompt`).
- Two deprecated entry points exist in the bash security scanner (`bashCommandIsSafe_DEPRECATED`, `bashCommandIsSafeAsync_DEPRECATED`), suggesting an ongoing migration to a newer API.
- The `preparePermissionMatcher()` function on the Tool interface enables tools to define custom rule-matching logic for L3.
- Stage 1 of the classifier uses a stop sequence of `</block>` to minimize token usage.
- The `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` marker type enforces at the type level that telemetry metadata contains only `boolean | number` values.
