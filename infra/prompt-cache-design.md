# Prompt Cache -- Design Document

This document analyzes the prompt caching architecture of Claude Code -- Anthropic's agentic CLI tool -- focusing on how it maximizes prompt cache hit rates to reduce API cost and latency across the lifecycle of a session.

---

### 1. Overview

#### What Prompt Caching Is

Anthropic's API supports prompt caching: when the byte-identical prefix of an API request matches a previously seen request, the server can reuse the already-processed key-value attention state instead of recomputing it from scratch. This is controlled by `cache_control` markers on system prompt blocks and message content blocks. Each marker tells the server "everything up to and including this block is a cacheable prefix." On a cache hit, those tokens are billed at the **cache read** rate instead of the full **input** rate, and the server skips the forward pass for those tokens entirely.

#### Why It Matters

A typical Claude Code session sends 15,000--40,000 tokens of system prompt, tool definitions, and early conversation history on every API call. Without caching, every turn re-processes this entire prefix at full input cost. With caching, the server reuses previously computed state, delivering two benefits:

1. **Cost reduction.** Cache read tokens are billed at 10% of the full input price. For Sonnet-class models, the input rate is $3/MTok while cache reads cost $0.30/MTok -- a 10x reduction on cached tokens. Given that system prompt and tool definitions typically dominate input token counts on early turns, this translates to significant savings across a session.

2. **Latency reduction.** The server does not need to recompute attention for cached tokens. For a 30,000-token system prompt, this removes a meaningful amount of first-token latency -- particularly on larger models where the forward pass per token is more expensive.

Claude Code's architecture is designed so that the vast majority of the prompt prefix is byte-identical across consecutive API calls within a session, and a substantial portion (the "static region") is byte-identical even across different users and sessions.

---

### 2. Static/Dynamic Boundary

The cornerstone of Claude Code's prompt caching strategy is the **dynamic boundary marker** -- a sentinel string that explicitly partitions the system prompt into two regions.

This marker is inserted into the system prompt array between the static and dynamic sections. When the system prompt is serialized into API block params, everything before the boundary receives `cache_control: { type: 'ephemeral', scope: 'global' }`, while everything after receives no cache scope. The boundary itself is stripped from the final output -- it is a compile-time directive, not content sent to the model.

#### Three Splitting Modes

The splitting function implements three strategies depending on the provider and whether MCP tools are present:

**Mode 1: Global cache with boundary (first-party API, no active MCP tools)**
Returns up to four blocks:
- Attribution header block (no cache scope)
- CLI system prompt prefix block (no cache scope)
- Static content before boundary (cache scope: `'global'`)
- Dynamic content after boundary (no cache scope)

The `'global'` scope means the cached prefix is shared across all users and organizations on the Anthropic API -- any Claude Code session running the same static system prompt bytes can hit the cache.

**Mode 2: Tool-based cache (first-party API, MCP tools present)**
MCP tool definitions are user-specific, so the tool array already varies per-user. The system falls back to `'org'`-scoped caching on the system prompt instead:
- Attribution header block (no cache scope)
- CLI system prompt prefix block (cache scope: `'org'`)
- All content concatenated (cache scope: `'org'`)

The `cache_control` marker is placed on the last tool schema instead, so tool definitions form the cached prefix.

**Mode 3: Default (third-party providers, or boundary missing)**
Falls back to org-scoped caching without global sharing:
- Attribution header block (no cache scope)
- CLI system prompt prefix block (cache scope: `'org'`)
- All content concatenated (cache scope: `'org'`)

#### Boundary Placement Contract

The boundary's position is critical. Moving anything from the static region into the dynamic region reduces the cache prefix length. Moving anything dynamic into the static region fragments the global cache -- the 2^N variant problem, where N session-varying conditionals each double the number of distinct cache prefixes. Historical PRs document specific incidents where session-variant conditionals placed before the boundary caused cache fragmentation.

---

### 3. Static Region

The static region contains everything that is identical for all Claude Code users running the same build. These sections appear before the dynamic boundary:

1. **Intro section**: Identity framing ("You are an interactive agent..."), cyber risk instructions, URL generation policy.
2. **System section**: Tool permission modes, system reminder tags, hook awareness, automatic compression notice.
3. **Task guidance**: Coding style rules (minimal changes, no speculative abstractions, security awareness), code commenting policy, user help information.
4. **Action safety**: Reversibility heuristics, blast radius assessment, confirmation requirements for destructive operations.
5. **Tool usage**: Dedicated tool preferences over Bash, parallel tool call guidance, task management instructions.
6. **Tone and style**: Emoji policy, code reference formatting, pull request link formatting.
7. **Output efficiency**: Conciseness guidelines, when to communicate vs. when to act silently.

These sections are deterministic at build time for a given user type and tool set. They do not depend on the user's project, environment, model choice, or session state. This makes them ideal for global-scope caching: two unrelated Claude Code users will produce byte-identical static regions and share the same server-side cached prefix.

#### Session-Variant Content Excluded from Static Region

A dedicated function explicitly collects conditionals that would otherwise fragment the global cache. Its design rationale: each conditional is a runtime bit that would otherwise multiply the prefix hash variants (2^N). This function is resolved in the dynamic region, not the static region. It includes guidance that varies based on whether certain tools are enabled, whether the session is interactive, whether fork subagents are enabled, and whether specific feature flags are active.

---

### 4. Dynamic Region

The dynamic region appears after the boundary and contains per-session content. Each section is registered as either a memoized section or an explicitly uncached section:

1. **Session-specific guidance**: Agent tool instructions, fork subagent guidance, skill tool documentation -- varies with session type and feature flags.
2. **Memory prompt**: Auto-memory index, behavioral instructions for memory usage.
3. **Ant model override**: Model override instructions (ant-only).
4. **Environment info**: Working directory, git repo status, platform, shell, model name, knowledge cutoff date -- varies per project.
5. **Language preference**: User-configured language setting.
6. **Output style**: Custom output style configuration.
7. **MCP instructions**: Instructions from connected MCP servers. Marked as explicitly uncached because MCP servers can connect and disconnect between turns, so the value must be recomputed. When the MCP instructions delta feature is enabled, this section is nullified and instructions are delivered via persisted attachments instead -- specifically to avoid busting the prompt cache on late MCP connections.
8. **Scratchpad**: Temporary directory instructions when enabled.
9. **Function result clearing**: Cached microcompact instructions.
10. **Tool result summarization**: Instructions to write down important information before tool results are cleared.
11. **Numeric length anchors**: Length calibration anchors (ant-only).
12. **Token budget**: Token budget guidance (feature-gated).
13. **Brief**: Brief mode instructions (feature-gated).

#### Section Memoization

Memoized sections run their compute function once, and the result is cached until `/clear` or `/compact` resets it. This prevents dynamic sections from producing different bytes across turns -- a change in the underlying data does not break the cache because the memoized value is returned on subsequent calls.

The explicitly uncached section variant opts out of memoization, recomputing on every turn. Its API requires a reason parameter documenting why cache-breaking recomputation is necessary, making the cost of volatile sections visible at the call site.

---

### 5. System Context Memoization

Two memoized async functions collect context that is prepended to the conversation. Both compute once per session and return the cached result on all subsequent calls:

#### System Context

Collects environment-level context appended to the system prompt:
- Git status snapshot (branch, default branch, short status, recent 5 commits, user name). All git commands run in parallel.
- Optional cache breaker injection (debugging mechanism).

Memoization ensures the git status snapshot taken at session start is reused for the entire session. This is a deliberate design trade-off: the status may go stale as the user makes commits, but the alternative -- re-reading git status every turn -- would change the system prompt bytes and break the cache on every turn.

#### User Context

Collects user-level context prepended to the message array as a system reminder:
- CLAUDE.md hierarchy content (project-level, user-level, and enterprise-level instructions discovered by walking the directory tree).
- Current date string.

Memoization is critical here because CLAUDE.md content often includes hundreds or thousands of characters of project-specific instructions. If this were recomputed and any file changed mid-session, the first message in the conversation would change bytes, invalidating the entire conversation prefix cache.

#### Cache Invalidation

Both memoize caches can be explicitly cleared. This happens when the system prompt injection changes (for cache-breaking debugging), and when `/clear` or `/compact` resets the session state. The section cache reset function also resets all section caches and beta header latches.

---

### 6. Fork Agent Cache Sharing

Fork subagents -- background workers that inherit the parent's full conversation context -- are designed specifically to share the parent's prompt cache. The architecture achieves this through three mechanisms:

#### Byte-Identical System Prompts

Rather than re-computing the system prompt (which could produce different bytes due to feature flag evaluation order or state changes), the parent's already-serialized system prompt bytes are passed directly to the child. This guarantees byte-identity.

#### Identical Tool Pools

Fork children receive the parent's exact tool pool. Even the agent tool itself is kept in the child's tool pool -- not because the child should use it (recursive forking is blocked at call time), but because removing it would change the tool schema bytes and break cache sharing.

#### Byte-Identical Message Prefixes

Child messages are constructed so that all fork children produce byte-identical API request prefixes:

1. The full parent assistant message (all tool_use blocks, thinking, text) is preserved identically across all children.
2. A single user message is constructed with tool_result blocks for every tool_use, each containing an identical placeholder string.
3. Only the final text block in the user message differs per child -- it contains the per-child directive.

Since the API cache key is a prefix match, the identical bytes up through the placeholder results are cached once and shared across all fork children. Only the final directive (typically a few hundred tokens) is unique per child.

#### Cache-Safe Parameters

A dedicated type formalizes the set of parameters that must match the parent for cache sharing: system prompt, user context, system context, tool use context, and fork context messages. A module-level slot is written after each turn so that post-turn forked agents (prompt suggestions, summaries, background commands) can share the main loop's prompt cache without each caller having to thread parameters through.

---

### 7. One-Shot Agent Optimizations

Not all API calls benefit from the full system prompt and context assembly. Claude Code uses several lightweight query paths that trade completeness for efficiency:

#### Side Queries

Internal classifiers, permission explainers, model validation, and session search use a side query mechanism. These calls:
- Accept a minimal system prompt (often just a short string, not the full system prompt).
- Skip CLAUDE.md and git status context entirely.
- Use their own query source identifier for tracking.
- May skip the CLI system prompt prefix (keeping only the OAuth attribution header).

These queries do not attempt to share the main session's prompt cache -- they are independent, short-lived calls where a small total token count is more important than cache reuse.

#### Forked Agent Context Trimming

The forked agent function provides context overrides that allow callers to customize what context the forked agent receives. For read-only or single-purpose agents, callers can omit non-essential trailers or context to reduce token counts.

#### Tool Schema Caching

Tool schemas are cached session-stably. The base schema (name, description, input_schema, strict mode, fine-grained streaming) is computed once per tool per session and cached. This prevents mid-session feature flag flips from changing tool description bytes, which would break the prompt cache. Per-request overlays (defer loading, cache control markers) are applied on top of the cached base without mutating it, so the core bytes remain stable.

---

### 8. Prompt Cache Break Detection

Claude Code implements a two-phase detection system that identifies when the prompt cache breaks and diagnoses why.

#### Phase 1: Pre-Call State Recording

Before each API request, hashes are computed of:
- System prompt blocks (with and without cache_control -- to catch both content and scope/TTL changes)
- Tool schemas, with per-tool hashes
- Model string, fast mode flag, beta headers, effort value, extra body params
- Global cache strategy, auto-mode state, overage state, cached microcompact state

When any hash differs from the previous call, the system records pending changes describing exactly what changed.

#### Phase 2: Post-Call Response Analysis

After the API response arrives, cache read tokens from the current response are compared against the previous response. A cache break is detected when:
1. Cache read tokens dropped more than 5% from the previous call, AND
2. The absolute drop exceeds a minimum threshold of 2,000 tokens.

When a break is detected, the system constructs an explanation from the pending changes (e.g., system prompt changed, tools changed, model changed, fast mode toggled, betas changed, possible TTL expiry, or likely server-side eviction).

#### False Positive Suppression

The detection system suppresses known false positives:
- **Cache deletions**: When cached microcompact sends deletion instructions, the subsequent drop is expected.
- **Compaction**: Resets the baseline so the natural drop after compaction is not flagged.
- **Haiku model**: Excluded entirely -- different caching behavior.
- **First call**: No previous baseline to compare against.

#### Diff Generation

When a cache break is detected and previous diffable content is available, a unified diff between the previous and current prompt state is generated and written to a temp file for developer inspection.

#### Analytics

Every detected cache break fires a structured analytics event with detailed metadata: all change flags, tool add/remove counts, tool names (MCP tools sanitized to avoid leaking filesystem paths), beta header changes, timing information, and server request ID for correlation.

---

### 9. Startup Prefetch

Claude Code performs parallel prefetching during module evaluation to reduce startup latency. This is not prompt caching per se, but it ensures the data needed to assemble the prompt (API keys, settings) is available without blocking:

- **MDM Settings Prefetch**: Fires platform-specific subprocesses (plutil on macOS, reg query on Windows) to read managed settings. These run in parallel with module imports.
- **Keychain Prefetch**: On macOS, fires two keychain read subprocesses in parallel (OAuth credentials and legacy API key). Without prefetching, these would be read sequentially, costing approximately 65ms. With prefetching, both run in parallel with module evaluation.

These prefetches do not directly affect the prompt cache, but they reduce the time between CLI invocation and the first API call. Since the server-side prompt cache has a TTL (5 minutes default, 1 hour for eligible users), reducing startup latency means less time for the cache to expire between consecutive commands.

---

### 10. Compaction and Cache

Compaction -- the process of summarizing old conversation messages to free context window space -- interacts with prompt caching in important ways.

#### Compaction Direction and Cache Preservation

Partial compaction supports two directions:

**`'from'` direction (default):** Summarizes messages after a pivot point, keeps earlier messages. This preserves the prompt cache because the kept messages form a prefix of the original conversation -- the bytes the API has already cached remain at the front, untouched.

**`'up_to'` direction:** Summarizes messages before a pivot point, keeps later messages. This invalidates the prompt cache because the summary replaces the cached prefix.

> **Note:** A related but distinct cache optimization applies to the summarization API call itself. The comment at `compact.ts:852` -- `// 'up_to' prefix hits cache directly; 'from' sends all (tail wouldn't cache).` -- describes which messages are sent to the internal summarization call, not the main conversation cache preservation described above. In the summarization call, `'up_to'` sends only the prefix messages (which hit the cache from the main conversation), while `'from'` must send all messages (because the tail portion alone would not have a cached prefix).

#### Auto-Compact and Cache Baseline Reset

When auto-compact runs, the cache break detector's baseline is reset to null. This prevents the natural drop in cache read tokens (from fewer messages in the post-compact conversation) from being flagged as a cache break.

#### Compact Boundary Messages

After compaction, a boundary marker message is inserted. All subsequent API calls find only messages after the last boundary, effectively starting a new "cache epoch." The new cache prefix consists of: system prompt + context + summary + post-boundary messages.

#### Cached Microcompact (Cache Editing)

An advanced variant uses the API's cache editing feature to delete individual tool results from the cached prefix without invalidating the entire cache. When a tool result is cleared:
1. Deletion instructions are sent in the API request.
2. The server removes the specified content from its cached state.
3. Cache read tokens legitimately drop -- the false positive suppression mechanism handles this.

This is more cache-friendly than full compaction because it preserves the vast majority of the cached prefix, only excising specific stale tool results.

---

### 11. Cost Implications

The financial impact of prompt caching is substantial. Claude Code's cost calculation distinguishes four token types with different rates per model tier.

Cache reads cost 10% of the full input price. On the first API call of a session, the system prompt and tool definitions are written to the cache (at 1.25x the input price). On every subsequent call, those tokens are read from the cache at 0.1x the input price.

After just two calls, the cache write cost is amortized and every subsequent call saves approximately 90% on the cached prefix. Over a multi-turn session, this translates to significant savings on system prompt tokens alone -- and the savings compound because conversation history tokens are also cached (via cache control markers on the last message block of each turn).

#### Global Cache Scope Savings

The `'global'` cache scope extends sharing beyond a single session. When two users run the same Claude Code build with the same static system prompt, they share the same server-side cache. The cache write cost is paid once by whichever user's request arrives first; all subsequent users get cache reads. At scale, this means the static system prompt is effectively always in cache for popular builds.

#### Extended Cache TTL

Eligible users (first-party, non-Haiku) receive a 1-hour cache TTL instead of the default 5-minute TTL. This is gated by user eligibility latched at session start for stability -- mid-session eligibility flips would change the cache control TTL field, invalidating the cache. The 1-hour TTL means users who pause for 10-30 minutes between interactions still get cache hits on return.

---

### 12. Design Principles

**Byte stability above all.** The system goes to extraordinary lengths to keep prompt bytes identical across API calls. System context is memoized once per session. Tool schemas are cached to prevent feature flag flip drift. Fork agents inherit serialized system prompt bytes rather than recomputing them. Section values are memoized across turns.

**Explicit boundary over implicit convention.** The dynamic boundary marker is a machine-readable contract, not a comment. Code that splits the system prompt looks for this exact string. Code that adds new sections must decide whether to place them before or after it.

**Session-variant content is quarantined.** Any conditional that depends on runtime state is placed in the dynamic region. A dedicated function exists specifically to collect these conditionals and keep them out of the static prefix. The 2^N variant problem is called out explicitly.

**Cache breaks are detected, diagnosed, and logged.** Rather than hoping caching works, the system actively monitors cache read tokens across consecutive calls, detects drops, identifies the root cause, and fires structured analytics events. This turns cache efficiency from a black box into an observable metric.

**Dangerous operations require justification.** The uncached section API requires a string reason explaining why cache-breaking recomputation is necessary.

**Latches for session stability.** Several values that could change mid-session are "latched" at session start: cache TTL eligibility, beta header presence, overage state. Each latch prevents a specific cache-breaking scenario.

**Compaction direction respects cache topology.** The `'from'` direction is the default for partial compaction because it preserves the conversation prefix.

**Fork children share, not duplicate.** The entire fork subagent design optimizes for byte-identical prefixes so that multiple children share a single cache entry.

**Prefetch hides latency, not complexity.** Startup prefetching does not change what data is collected -- it changes when the I/O happens.
