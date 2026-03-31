# Claude Code: Prompt Cache Architecture — Design Specification

This document analyzes the prompt caching architecture of Claude Code — Anthropic's
agentic CLI tool — focusing on how it maximizes prompt cache hit rates to reduce API
cost and latency across the lifecycle of a session.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Static/Dynamic Boundary](#2-staticdynamic-boundary)
- [3. Static Region](#3-static-region)
- [4. Dynamic Region](#4-dynamic-region)
- [5. System Context Memoization](#5-system-context-memoization)
- [6. Fork Agent Cache Sharing](#6-fork-agent-cache-sharing)
- [7. One-Shot Agent Optimizations](#7-one-shot-agent-optimizations)
- [8. Prompt Cache Break Detection](#8-prompt-cache-break-detection)
- [9. Startup Prefetch](#9-startup-prefetch)
- [10. Compaction and Cache](#10-compaction-and-cache)
- [11. Cost Implications](#11-cost-implications)
- [12. Design Principles](#12-design-principles)

---

## 1. Overview

### What Prompt Caching Is

Anthropic's API supports prompt caching: when the byte-identical prefix of an API
request matches a previously seen request, the server can reuse the already-processed
key-value attention state instead of recomputing it from scratch. This is controlled by
`cache_control` markers on system prompt blocks and message content blocks. Each marker
tells the server "everything up to and including this block is a cacheable prefix." On a
cache hit, those tokens are billed at the **cache read** rate instead of the full
**input** rate, and the server skips the forward pass for those tokens entirely.

### Why It Matters

A typical Claude Code session sends 15,000--40,000 tokens of system prompt, tool
definitions, and early conversation history on every API call. Without caching, every
turn re-processes this entire prefix at full input cost. With caching, the server reuses
previously computed state, delivering two benefits:

1. **Cost reduction.** Cache read tokens are billed at 10% of the full input price.
   For Sonnet-class models, the input rate is $3/MTok while cache reads cost $0.30/MTok
   — a 10x reduction on cached tokens. For Opus 4.6, the reduction is $15/MTok down to
   $1.50/MTok. Given that system prompt and tool definitions typically dominate input
   token counts on early turns, this translates to significant savings across a session.

2. **Latency reduction.** The server does not need to recompute attention for cached
   tokens. For a 30,000-token system prompt, this removes a meaningful amount of
   first-token latency — particularly on larger models where the forward pass per token
   is more expensive.

Claude Code's architecture is designed so that the vast majority of the prompt prefix
is byte-identical across consecutive API calls within a session, and a substantial
portion (the "static region") is byte-identical even across different users and sessions.

---

## 2. Static/Dynamic Boundary

> **Source:** `src/constants/prompts.ts:106-115`, `src/utils/api.ts:296-435`

The cornerstone of Claude Code's prompt caching strategy is the **dynamic boundary
marker** — a sentinel string that explicitly partitions the system prompt into two
regions:

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

This marker is inserted into the system prompt array between the static and dynamic
sections. When the system prompt is serialized into API `TextBlockParam` blocks by
`splitSysPromptPrefix()`, everything before the boundary receives
`cache_control: { type: 'ephemeral', scope: 'global' }`, while everything after
receives no cache scope. The boundary itself is stripped from the final output — it is a
compile-time directive, not content sent to the model.

### Three Splitting Modes

`splitSysPromptPrefix()` implements three strategies depending on the provider and
whether MCP tools are present:

**Mode 1: Global cache with boundary (first-party API, no active MCP tools)**
Returns up to four blocks:
- Attribution header block (no cache scope)
- CLI system prompt prefix block (no cache scope)
- Static content before boundary (cache scope: `'global'`)
- Dynamic content after boundary (no cache scope)

The `'global'` scope means the cached prefix is shared across all users and
organizations on the Anthropic API — any Claude Code session running the same static
system prompt bytes can hit the cache.

**Mode 2: Tool-based cache (first-party API, MCP tools present)**
MCP tool definitions are user-specific, so the tool array already varies per-user.
The system falls back to `'org'`-scoped caching on the system prompt instead:
- Attribution header block (no cache scope)
- CLI system prompt prefix block (cache scope: `'org'`)
- All content concatenated (cache scope: `'org'`)

The `cache_control` marker is placed on the last tool schema instead, so tool
definitions form the cached prefix.

**Mode 3: Default (third-party providers, or boundary missing)**
Falls back to org-scoped caching without global sharing:
- Attribution header block (no cache scope)
- CLI system prompt prefix block (cache scope: `'org'`)
- All content concatenated (cache scope: `'org'`)

### Boundary Placement Contract

The boundary's position is critical. The comment at line 572 of `prompts.ts` reads:

```typescript
// === BOUNDARY MARKER - DO NOT MOVE OR REMOVE ===
...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),
```

Moving anything from the static region into the dynamic region reduces the cache prefix
length. Moving anything dynamic into the static region fragments the global cache — the
2^N variant problem, where N session-varying conditionals each double the number of
distinct cache prefixes. The PR comments (#24490, #24171) document specific incidents
where session-variant conditionals placed before the boundary caused cache fragmentation.

---

## 3. Static Region

> **Source:** `src/constants/prompts.ts:560-571`

The static region contains everything that is identical for all Claude Code users running
the same build. These sections appear before the dynamic boundary and are assembled by
`getSystemPrompt()`:

1. **Intro section** (`getSimpleIntroSection`): Identity framing ("You are an interactive
   agent..."), cyber risk instructions, URL generation policy.

2. **System section** (`getSimpleSystemSection`): Tool permission modes, system reminder
   tags, hook awareness, automatic compression notice.

3. **Task guidance** (`getSimpleDoingTasksSection`): Coding style rules (minimal
   changes, no speculative abstractions, security awareness), code commenting policy,
   user help information.

4. **Action safety** (`getActionsSection`): Reversibility heuristics, blast radius
   assessment, confirmation requirements for destructive operations.

5. **Tool usage** (`getUsingYourToolsSection`): Dedicated tool preferences over Bash,
   parallel tool call guidance, task management instructions.

6. **Tone and style** (`getSimpleToneAndStyleSection`): Emoji policy, code reference
   formatting, pull request link formatting.

7. **Output efficiency** (`getOutputEfficiencySection`): Conciseness guidelines, when
   to communicate vs. when to act silently.

These sections are deterministic at build time for a given `USER_TYPE` (ant vs.
external) and tool set. They do not depend on the user's project, environment, model
choice, or session state. This makes them ideal for global-scope caching: two unrelated
Claude Code users will produce byte-identical static regions and share the same server-
side cached prefix.

### Session-Variant Content Excluded from Static Region

The `getSessionSpecificGuidanceSection()` function (line 352) explicitly collects
conditionals that would otherwise fragment the global cache. Its comment explains the
design rationale:

```typescript
/**
 * Session-variant guidance that would fragment the cacheScope:'global'
 * prefix if placed before SYSTEM_PROMPT_DYNAMIC_BOUNDARY. Each conditional
 * here is a runtime bit that would otherwise multiply the Blake2b prefix
 * hash variants (2^N).
 */
```

This function is resolved in the dynamic region, not the static region. It includes
guidance that varies based on whether the `AskUserQuestion` tool is enabled, whether
the session is interactive, whether fork subagents are enabled, and whether specific
feature flags are active.

---

## 4. Dynamic Region

> **Source:** `src/constants/prompts.ts:491-555`, `src/constants/systemPromptSections.ts`

The dynamic region appears after the boundary and contains per-session content. Each
section is wrapped in a `systemPromptSection()` or `DANGEROUS_uncachedSystemPromptSection()`
registration:

1. **Session-specific guidance** (`session_guidance`): Agent tool instructions, fork
   subagent guidance, skill tool documentation — varies with session type and feature
   flags.

2. **Memory prompt** (`memory`): Auto-memory index from `~/.claude/memory/`, behavioral
   instructions for memory usage.

3. **Environment info** (`env_info_simple`): Working directory, git repo status, platform,
   shell, model name, knowledge cutoff date — varies per project.

4. **Language preference** (`language`): User-configured language setting.

5. **Output style** (`output_style`): Custom output style configuration.

6. **MCP instructions** (`mcp_instructions`): Instructions from connected MCP servers.
   Marked as `DANGEROUS_uncachedSystemPromptSection` because MCP servers can connect
   and disconnect between turns, so the value must be recomputed. When the MCP
   instructions delta feature is enabled, this section is nullified and instructions
   are delivered via persisted attachments instead — specifically to avoid busting the
   prompt cache on late MCP connections.

7. **Scratchpad** (`scratchpad`): Temporary directory instructions when enabled.

8. **Function result clearing** (`frc`): Cached microcompact instructions.

9. **Tool result summarization** (`summarize_tool_results`): Instructions to write down
   important information before tool results are cleared.

### Section Memoization

The `systemPromptSection()` function in `src/constants/systemPromptSections.ts` creates
memoized sections: each section's `compute()` function runs once, and the result is
cached in the system prompt section cache (stored in `src/bootstrap/state.ts`) until
`/clear` or `/compact` resets it. This prevents dynamic sections from producing different
bytes across turns — a change in the dynamic region does not break the cache, because
the memoized value is returned on subsequent calls.

The `DANGEROUS_uncachedSystemPromptSection()` variant explicitly opts out of memoization,
recomputing on every turn. Its signature requires a `_reason` parameter documenting why
cache-breaking is acceptable:

```typescript
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => isMcpInstructionsDeltaEnabled() ? null : getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns',
)
```

---

## 5. System Context Memoization

> **Source:** `src/context.ts`

Two memoized async functions collect context that is prepended to the conversation.
Both use lodash `memoize()`, meaning they compute once per session and return the
cached result on all subsequent calls:

### `getSystemContext()` (line 116)

Collects environment-level context appended to the system prompt:
- Git status snapshot (branch, default branch, short status, recent 5 commits, user
  name). All git commands run in parallel via `Promise.all()`.
- Optional cache breaker injection (ant-only debugging mechanism).

Memoization ensures the git status snapshot taken at session start is reused for the
entire session. This is a deliberate design trade-off: the status may go stale as the
user makes commits, but the alternative — re-reading git status every turn — would
change the system prompt bytes and break the cache on every turn.

### `getUserContext()` (line 155)

Collects user-level context prepended to the message array as a `<system-reminder>`:
- CLAUDE.md hierarchy content (project-level, user-level, and enterprise-level
  instructions discovered by walking the directory tree).
- Current date string.

Memoization is critical here because CLAUDE.md content often includes hundreds or
thousands of characters of project-specific instructions. If this were recomputed and
any file changed mid-session, the first message in the conversation would change bytes,
invalidating the entire conversation prefix cache.

### Cache Invalidation

Both memoize caches can be explicitly cleared:
```typescript
getUserContext.cache.clear?.()
getSystemContext.cache.clear?.()
```

This happens when the system prompt injection changes (for cache-breaking debugging),
and when `/clear` or `/compact` resets the session state. The `clearSystemPromptSections()`
function in `systemPromptSections.ts` also resets all section caches and beta header
latches.

---

## 6. Fork Agent Cache Sharing

> **Source:** `src/tools/AgentTool/forkSubagent.ts`, `src/utils/forkedAgent.ts`

Fork subagents — background workers that inherit the parent's full conversation context
— are designed specifically to share the parent's prompt cache. The architecture achieves
this through three mechanisms:

### Byte-Identical System Prompts

The fork agent definition at line 54 of `forkSubagent.ts` explains the design:

```typescript
/**
 * The getSystemPrompt here is unused: the fork path passes
 * override.systemPrompt with the parent's already-rendered system prompt
 * bytes, threaded via toolUseContext.renderedSystemPrompt. Reconstructing
 * by re-calling getSystemPrompt() can diverge (GrowthBook cold→warm) and
 * bust the prompt cache; threading the rendered bytes is byte-exact.
 */
```

Rather than re-computing the system prompt (which could produce different bytes due to
feature flag evaluation order or GrowthBook state changes), the parent's already-
serialized system prompt bytes are passed directly to the child. This guarantees byte-
identity.

### Identical Tool Pools

Fork children receive the parent's exact tool pool (`tools: ['*']` with
`useExactTools`). Even the `AgentTool` itself is kept in the child's tool pool — not
because the child should use it (recursive forking is blocked at call time by
`isInForkChild()`), but because removing it would change the tool schema bytes and
break cache sharing.

### Byte-Identical Message Prefixes

`buildForkedMessages()` constructs child messages so that all fork children produce
byte-identical API request prefixes. The strategy:

1. The full parent assistant message (all tool_use blocks, thinking, text) is preserved
   identically across all children.
2. A single user message is constructed with tool_result blocks for every tool_use, each
   containing an identical placeholder string: `'Fork started — processing in background'`.
3. Only the final text block in the user message differs per child — it contains the
   per-child directive.

Result: `[...history, assistant(all_tool_uses), user(placeholder_results..., directive)]`

Since the API cache key is a prefix match, the identical bytes up through the placeholder
results are cached once and shared across all fork children. Only the final directive
(typically a few hundred tokens) is unique per child.

### CacheSafeParams

The `CacheSafeParams` type in `src/utils/forkedAgent.ts` (line 57) formalizes the set
of parameters that must match the parent for cache sharing:

```typescript
export type CacheSafeParams = {
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext
  forkContextMessages: Message[]
}
```

A module-level `lastCacheSafeParams` slot is written after each turn so that post-turn
forked agents (prompt suggestions, post-turn summaries, `/btw` commands) can share the
main loop's prompt cache without each caller having to thread parameters through.

---

## 7. One-Shot Agent Optimizations

> **Source:** `src/utils/sideQuery.ts`, `src/utils/forkedAgent.ts`

Not all API calls benefit from the full system prompt and context assembly. Claude Code
uses several lightweight query paths that trade completeness for efficiency:

### Side Queries

`sideQuery()` in `src/utils/sideQuery.ts` is used for internal classifiers, permission
explainers, model validation, and session search. These calls:
- Accept a minimal system prompt (often just a short string, not the full
  `getSystemPrompt()` output).
- Skip CLAUDE.md and git status context entirely.
- Use their own `querySource` identifier for tracking.
- May set `skipSystemPromptPrefix: true` to omit the CLI system prompt prefix (keeping
  only the OAuth attribution header).

These queries do not attempt to share the main session's prompt cache — they are
independent, short-lived calls where a small total token count is more important than
cache reuse.

### Forked Agent Context Trimming

The `runForkedAgent()` function provides `SubagentContextOverrides` that allow callers
to customize what context the forked agent receives. For read-only or single-purpose
agents (memory extraction, prompt suggestion), callers can omit non-essential trailers
or context to reduce token counts.

### Tool Schema Caching

Tool schemas are cached session-stably in `src/utils/toolSchemaCache.ts`. The
`toolToAPISchema()` function computes the base schema (name, description, input_schema,
strict mode, fine-grained streaming) once per tool per session and caches it. This
prevents mid-session GrowthBook flips from changing tool description bytes, which would
break the prompt cache:

```typescript
// Session-stable base schema: name, description, input_schema, strict,
// eager_input_streaming. These are computed once per session and cached to
// prevent mid-session GrowthBook flips from churning the serialized tool array bytes.
const cache = getToolSchemaCache()
let base = cache.get(cacheKey)
```

Per-request overlays (`defer_loading`, `cache_control`) are applied on top of the cached
base without mutating it, so the core bytes remain stable.

---

## 8. Prompt Cache Break Detection

> **Source:** `src/services/api/promptCacheBreakDetection.ts`

Claude Code implements a two-phase detection system that identifies when the prompt cache
breaks and diagnoses why.

### Phase 1: Pre-Call State Recording

`recordPromptState()` is called before each API request. It computes hashes of:
- System prompt blocks (with `cache_control` stripped for content comparison)
- System prompt blocks (with `cache_control` intact for scope/TTL comparison)
- Tool schemas
- Per-tool schema hashes (to identify which specific tool changed)
- Model string, fast mode flag, beta headers, effort value, extra body params
- Global cache strategy, auto-mode state, overage state, cached microcompact state

When any hash differs from the previous call, the system records `PendingChanges`
describing exactly what changed: which tools were added/removed, which tool schemas
changed, character delta in the system prompt, model changes, beta header changes, etc.

### Phase 2: Post-Call Response Analysis

`checkResponseForCacheBreak()` is called after the API response arrives. It compares
`cache_read_input_tokens` from the current response against the previous response. A
cache break is detected when:

1. Cache read tokens dropped more than 5% from the previous call, AND
2. The absolute drop exceeds `MIN_CACHE_MISS_TOKENS = 2000` tokens.

When a break is detected, the system constructs an explanation from the pending changes:
- `"system prompt changed (+342 chars)"` — system prompt bytes differed
- `"tools changed (+2/-0 tools)"` — tool set changed
- `"model changed (claude-sonnet-4-6 → claude-opus-4-6)"` — model switched
- `"fast mode toggled"` — fast/normal mode changed
- `"betas changed (+prompt_caching_scope_beta)"` — beta headers changed
- `"possible 5min TTL expiry (prompt unchanged)"` — no client-side change, but >5
  minutes since last call
- `"likely server-side (prompt unchanged, <5min gap)"` — no client-side explanation

### False Positive Suppression

The detection system suppresses known false positives:
- **Cache deletions**: When cached microcompact sends `cache_edits` deletions, the
  subsequent drop in cache read tokens is expected. `notifyCacheDeletion()` sets a flag
  that causes the next check to skip alerting.
- **Compaction**: `notifyCompaction()` resets the baseline so the natural drop in cache
  read tokens after compaction is not flagged.
- **Haiku model**: Excluded entirely — different caching behavior.
- **First call**: No previous baseline to compare against.

### Diff Generation

When a cache break is detected and the previous diffable content is available, the system
generates a unified diff between the previous and current prompt state (system prompt +
tool schemas) and writes it to a temp file. The file path is included in the debug log
so developers can inspect exactly what changed.

### Analytics

Every detected cache break fires a `tengu_prompt_cache_break` analytics event with
detailed metadata: all change flags, tool add/remove counts, tool names (MCP tools
sanitized to `'mcp'` to avoid leaking filesystem paths), beta header changes, timing
information, and the Anthropic request ID for server-side correlation.

---

## 9. Startup Prefetch

> **Source:** `src/main.tsx:1-20`, `src/utils/secureStorage/keychainPrefetch.ts`,
> `src/utils/settings/mdm/rawRead.ts`

Claude Code performs parallel prefetching during module evaluation to reduce startup
latency. This is not prompt caching per se, but it is a critical optimization that
ensures the data needed to assemble the prompt (API keys, settings) is available without
blocking:

### Execution Order

At the top of `main.tsx`, before any other imports:

```typescript
profileCheckpoint('main_tsx_entry');
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();
import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();
```

### MDM Settings Prefetch (`startMdmRawRead`)

Fires `plutil` (macOS) or `reg query` (Windows) subprocesses to read MDM/managed
settings. These subprocesses run in parallel with the remaining ~135ms of module imports.

### Keychain Prefetch (`startKeychainPrefetch`)

On macOS, fires two `security find-generic-password` subprocesses in parallel:
1. OAuth credentials (`"Claude Code-credentials"`, ~32ms)
2. Legacy API key (`"Claude Code"`, ~33ms)

Without prefetching, these would be read sequentially via `execSync`, costing ~65ms.
With prefetching, both subprocesses run in parallel with module evaluation, and
`ensureKeychainPrefetchCompleted()` is awaited alongside `ensureMdmSettingsLoaded()` in
the Commander `preAction` hook — by which time the subprocesses have long since finished.

### Impact on Cache

These prefetches do not directly affect the prompt cache, but they reduce the time
between CLI invocation and the first API call. Since Anthropic's server-side prompt cache
has a TTL (5 minutes for ephemeral, 1 hour for eligible users), reducing startup latency
means less time for the cache to expire between consecutive commands in interactive usage
patterns.

---

## 10. Compaction and Cache

> **Source:** `src/services/compact/compact.ts`

Compaction — the process of summarizing old conversation messages to free context window
space — interacts with prompt caching in important ways.

### Compaction Direction and Cache Preservation

Partial compaction supports two directions:

**`'from'` direction (default):** Summarizes messages after a pivot point, keeps earlier
messages. This preserves the prompt cache because the kept messages form a prefix of the
original conversation — the bytes the API has already cached remain at the front,
untouched.

**`'up_to'` direction:** Summarizes messages before a pivot point, keeps later messages.
This invalidates the prompt cache because the summary replaces the cached prefix. The
summary text will not match the previously cached bytes, so the server must recompute.

The comment in `compact.ts` at line 852 makes this explicit:

```typescript
// 'up_to' prefix hits cache directly; 'from' sends all (tail wouldn't cache).
```

### Auto-Compact and Cache Baseline Reset

When auto-compact runs, the `notifyCompaction()` function is called on the cache break
detector, resetting the `prevCacheReadTokens` baseline to `null`. This prevents the
natural drop in cache read tokens (from fewer messages in the post-compact conversation)
from being flagged as a cache break.

### Compact Boundary Messages

After compaction, a `compact_boundary` system message is inserted. All subsequent API
calls use `getMessagesAfterCompactBoundary()` to find only messages after the last
boundary, effectively starting a new "cache epoch." The messages before the boundary are
no longer sent to the API, so the new cache prefix consists of:
`system prompt + context + summary + post-boundary messages`.

### Cached Microcompact (Cache Editing)

An advanced variant called "cached microcompact" uses the API's `cache_edits` feature to
delete individual tool results from the cached prefix without invalidating the entire
cache. When a tool result is cleared:

1. The system sends `cache_edits` deletion instructions in the API request.
2. The server removes the specified content from its cached state.
3. Cache read tokens legitimately drop — `notifyCacheDeletion()` suppresses the false
   positive.

This is more cache-friendly than full compaction because it preserves the vast majority
of the cached prefix, only excising specific stale tool results.

---

## 11. Cost Implications

> **Source:** `src/utils/modelCost.ts`

The financial impact of prompt caching is substantial. Claude Code's cost calculation
distinguishes four token types:

| Token Type | Sonnet 4.6 ($/MTok) | Opus 4.6 ($/MTok) |
|---|---|---|
| Input | $3.00 | $15.00 |
| Cache write | $3.75 | $18.75 |
| **Cache read** | **$0.30** | **$1.50** |
| Output | $15.00 | $75.00 |

Cache reads cost 10% of the full input price. On the first API call of a session, the
system prompt and tool definitions are written to the cache (at 1.25x the input price).
On every subsequent call, those tokens are read from the cache at 0.1x the input price.

For a typical session with a ~25,000-token system prompt and tool definitions:
- First call: 25,000 tokens at cache write rate = $0.09 (Sonnet) / $0.47 (Opus)
- Each subsequent call: 25,000 tokens at cache read rate = $0.008 (Sonnet) / $0.038 (Opus)
- Without caching: 25,000 tokens at input rate = $0.075 (Sonnet) / $0.375 (Opus) per call

After just two calls, the cache write cost is amortized and every subsequent call saves
~90% on the cached prefix. Over a 20-turn session, this saves roughly $1.35 (Sonnet) or
$6.38 (Opus) on system prompt tokens alone — and the savings compound because conversation
history tokens are also cached (via `cache_control` markers on the last message block
of each turn).

### Global Cache Scope Savings

The `'global'` cache scope extends sharing beyond a single session. When two users run
the same Claude Code build with the same static system prompt, they share the same
server-side cache. The cache write cost is paid once by whichever user's request arrives
first; all subsequent users get cache reads. At scale, this means the static system
prompt is effectively always in cache for popular builds.

### 1-Hour Cache TTL

Eligible users (first-party, non-Haiku) receive a 1-hour cache TTL instead of the
default 5-minute TTL. This is controlled by `should1hCacheTTL()` and gated by user
eligibility latched at session start for stability — mid-session eligibility flips would
change the `cache_control` TTL field, invalidating the cache. The 1-hour TTL means
users who pause for 10-30 minutes between interactions still get cache hits on return.

---

## 12. Design Principles

The prompt caching architecture reflects several core design principles:

**Byte stability above all.** The system goes to extraordinary lengths to keep prompt
bytes identical across API calls. System context is memoized once per session. Tool
schemas are cached to prevent GrowthBook flip drift. Fork agents inherit serialized
system prompt bytes rather than recomputing them. Section values are memoized across
turns. Every mechanism that touches the bytes sent to the API is designed for stability
first and freshness second.

**Explicit boundary over implicit convention.** The `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`
marker is a machine-readable contract, not a comment. Code that splits the system prompt
looks for this exact string. Code that adds new sections must decide whether to place
them before or after it. The boundary is documented with warnings against moving or
removing it, and the `splitSysPromptPrefix()` function logs analytics events when the
boundary is missing.

**Session-variant content is quarantined.** Any conditional that depends on runtime state
(interactive vs. non-interactive, fork subagent enabled, feature flag evaluation) is
placed in the dynamic region. The `getSessionSpecificGuidanceSection()` function exists
specifically to collect these conditionals and keep them out of the static prefix. The
2^N variant problem is called out explicitly in code comments.

**Cache breaks are detected, diagnosed, and logged.** Rather than hoping caching works,
the system actively monitors `cache_read_input_tokens` across consecutive calls, detects
drops, identifies the root cause (system prompt change, tool change, model change, TTL
expiry, or server-side eviction), and fires structured analytics events. This turns
cache efficiency from a black box into an observable metric.

**Dangerous operations require justification.** The `DANGEROUS_uncachedSystemPromptSection()`
API requires a string reason explaining why cache-breaking recomputation is necessary.
This makes the cost of volatile sections visible at the call site and discourages
casual addition of uncached sections.

**Latches for session stability.** Several values that could change mid-session are
"latched" at session start: cache TTL eligibility, beta header presence, overage state.
The code comments (e.g., "Latch eligibility in bootstrap state for session stability —
prevents mid-session overage flips from changing the cache_control TTL") explain that
each latch prevents a specific cache-breaking scenario.

**Compaction direction respects cache topology.** The `'from'` direction is the default
for partial compaction because it preserves the conversation prefix. The `'up_to'`
direction, which invalidates the cache, is available but not the default. Full
auto-compact resets the cache baseline rather than fighting to preserve a stale prefix.

**Fork children share, not duplicate.** The entire fork subagent design — threading
rendered system prompt bytes, keeping unused tools in the pool, using identical
placeholder tool results — optimizes for one thing: byte-identical prefixes so that
multiple children share a single cache entry. The small per-child directive at the end
is the only unique content.

**Prefetch hides latency, not complexity.** Startup prefetching (MDM settings, keychain
reads) does not change what data is collected — it changes when the I/O happens. By
overlapping subprocess spawning with module evaluation, the system avoids adding latency
to the critical path between CLI invocation and the first API call, which indirectly
protects cache TTL windows.
