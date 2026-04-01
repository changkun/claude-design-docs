# Feature Flags — Design Document

## Part A: Design Document

### Feature Flag System -- Design Specification

#### 1. Overview

Claude Code gates features behind a **two-tier flag system** that separates build-time decisions from runtime decisions:

| Tier | Mechanism | When Evaluated | Cost When Off |
|---|---|---|---|
| **Compile-time** | Bundler macro (dead-code elimination) | Build time (bundler pass) | Zero -- dead code eliminated |
| **Runtime** | GrowthBook remote evaluation | Process startup + periodic refresh | Negligible -- cached disk read |

The two tiers solve different problems:
- **Compile-time flags** control **what code ships** in a given build. They prevent internal-only features from leaking string literals, module imports, or implementation details into the external (public) binary.
- **Runtime flags** control **what code executes** for a given user within a build that already contains the feature. They enable gradual percentage rollout, targeting by organization/subscription/platform, A/B experiments, and emergency kill switches -- all without deploying new code.

A feature typically passes through three lifecycle stages:

```
  Development                Controlled rollout           General availability
  ─────────────              ──────────────────           ────────────────────
  compile flag = false       compile flag = true          compile flag = true
  (code exists, stripped)    + runtime gate (0→100%)      + gate removed or 100%
```

A feature whose compile-time flag is `false` has truly zero runtime cost -- the bundler eliminates the guarded code and all its transitive imports from the output. No string literals from the feature appear in the binary, no module initialization runs, and no runtime flag lookups execute.

---

#### 2. Compile-Time Flags

##### 2.1 Dead-Code Elimination Mechanics

The bundler provides a `feature()` function as a build-time macro. During bundling, each `feature('FLAG_NAME')` call is replaced with a boolean literal (`true` or `false`) based on the build configuration. The bundler's dead-code elimination (DCE) then removes unreachable branches.

When `feature('X')` resolves to `false`:

1. The boolean literal `false` replaces the call
2. Any `if (false && ...)` or `false ? ... : null` becomes unreachable
3. The bundler eliminates the unreachable branch entirely
4. Dynamic imports inside the branch are never emitted
5. The required modules and their transitive dependencies are not included in the bundle

This means the external build does not contain:
- Module code for internal-only features
- String literals (flag names, API endpoints, error messages)
- Tool definitions, command handlers, or service initialization for gated features

##### 2.2 The Positive Ternary Pattern

A critical implementation constraint: the bundler's DCE handles positive conditional patterns correctly but does not eliminate inline string literals from negative guard patterns.

- **Correct (positive ternary):** `feature('X') ? someExpr : fallback` -- string literals in `someExpr` are eliminated when the flag is false.
- **Incorrect (negative guard):** `if (!feature('X')) return; someExpr` -- string literals in `someExpr` may leak into the output even when the flag is false.

This means all compile-time gating must use the positive ternary form, not the early-return negation form.

##### 2.3 Module-Level Feature Guards

Compile-time flags commonly gate module-level `require()` calls that produce either a module reference or `null`. The resulting variable is then conditionally spread into arrays:

```
flag → (module | null) → ...(module ? [module] : [])
```

This pattern is used for tool registration, skill registration, and other modular feature sets.

##### 2.4 Separation from Runtime Gates

Compile-time flags **must** appear inline at the guarded blocks -- they cannot be abstracted into config objects or passed as parameters, because the bundler needs to see the literal `feature('NAME')` call to perform elimination. Runtime gates, by contrast, can be read from config objects, passed as parameters, and evaluated dynamically.

---

#### 3. Runtime Flags

##### 3.1 Remote Evaluation Architecture

Runtime flags use GrowthBook in **remote evaluation** mode (`remoteEval: true`). The GrowthBook server evaluates all flag rules server-side and returns pre-computed values for the current user. The client never sees targeting rules or percentage allocations.

SDK keys differ by build type (internal vs. external), ensuring internal experiments never leak to external users even if the same GrowthBook project is shared.

##### 3.2 User Attributes for Targeting

The following attributes are sent to GrowthBook for flag targeting:

| Attribute | Description |
|---|---|
| Device ID | Stable device identifier |
| Session ID | Per-session identifier |
| Platform | OS type (win32, darwin, linux) |
| API Base URL Host | Non-Anthropic proxy hostname (enterprise) |
| Organization UUID | OAuth organization |
| Account UUID | OAuth account |
| User Type | Internal or external |
| Subscription Type | Subscription tier |
| Rate Limit Tier | API rate limit tier |
| First Token Time | First API token timestamp (cohort tracking) |
| Email | User email (internal builds, from OAuth) |
| App Version | CLI version string |

These attributes enable targeting by organization (enterprise rollout), subscription (feature entitlement), platform (OS-specific features), and percentage (gradual rollout by device ID).

##### 3.3 Value Resolution Priority

When a runtime flag is queried, the value comes from the highest-priority source:

```
1. Environment variable overrides    (internal-only, for eval harnesses)
2. Config file overrides             (internal-only, for interactive developer testing)
3. In-memory remote eval cache       (populated after successful GrowthBook init)
4. Disk cache                        (survives process restarts)
5. Default value                     (passed by the calling code)
```

##### 3.4 Accessor Semantics

Runtime flag values are accessed through functions with explicit semantics about blocking and staleness:

| Behavior | Blocks? | Staleness | Use Case |
|---|---|---|---|
| Cached, may be stale | No | May be stale | Startup-critical paths, render loops, sync contexts |
| Cached or blocking | Conditionally | Fresh if blocking | Entitlement checks where stale `false` is harmful |
| Blocks on init | Yes | Fresh | Config that must be fresh (kill switches) |
| Security restriction | If re-init pending | Waits for auth change | Security-critical gates |

The preferred accessor for most code paths is the "cached, may be stale" variant, which reads from the in-memory cache first, falls back to disk cache, and never blocks.

The "cached or blocking" variant implements a hybrid strategy: if the disk cache already says `true`, it returns immediately (fast path). If the cache says `false` or is missing, it awaits initialization to fetch the fresh server value (slow path, max ~5s). This ensures a stale `false` from a previous session does not block access to a feature the user is now entitled to.

---

#### 4. Flag Evaluation Flow

##### 4.1 Compile-Time Evaluation

Compile-time flags are evaluated **exactly once** at build time. The build system maintains a mapping of flag names to boolean values for each build variant (internal vs. external). No runtime cost is incurred.

##### 4.2 Runtime Initialization Sequence

1. Create GrowthBook client with user attributes and auth headers
2. Fetch flag values from server (5-second timeout)
3. Process response and populate in-memory cache
4. Write all values to disk cache for cross-process persistence
5. Notify subscribers that values are fresh

If auth is not yet available, the client skips HTTP init and relies on disk-cached values. When auth becomes available later, the system detects the change, destroys the old client, and reinitializes with fresh auth headers.

##### 4.3 Caching Layers

Values are cached at three levels:

1. **In-memory map**: Fastest path (map lookup). Authoritative once init completes.
2. **Disk cache**: Survives process restarts. Written after every successful payload.
3. **Legacy provider cache**: Fallback during migration from a previous flag provider.

##### 4.4 Periodic Refresh

Long-running sessions refresh periodically:
- External builds: every 6 hours
- Internal builds: every 20 minutes

The refresh timer is unref'd so it does not prevent natural process exit. Subscribers registered for refresh notifications can rebuild long-lived objects (model selection, skill registration, event logger configuration).

##### 4.5 Experiment Exposure Logging

When a flag is backed by an experiment, accessing the flag value triggers exposure logging. Exposures are deduplicated per session. If the flag is accessed before init completes, the feature is added to a pending set and logged after init succeeds.

---

#### 5. Progressive Rollout Pipeline

##### Stage 1: Development
Feature code lands behind a compile-time flag. Internal build: flag is true. External build: flag is false. Feature is invisible to external users -- no code, no strings, no telemetry.

##### Stage 2: Internal Dogfooding
Within the internal build, a runtime gate is created at 0%. Rollout to internal users by percentage, organization, or individual email. Bugs are caught before external exposure.

##### Stage 3: External Enablement
Compile-time flag flipped to true in external build. Runtime gate controls rollout: 0% initially, ramping to 100%. Targeting attributes enable controlled external rollout.

##### Stage 4: Flag Removal
When generally available and stable, both compile-time flag and runtime gate are removed. Code becomes unconditional.

##### Example: Bridge Mode
Bridge mode illustrates the full pipeline. The compile-time flag gates all bridge code and string literals. Within that gate, multiple runtime flags provide fine-grained control:

```
compile-time flag (code presence)
  └── entitlement gate (org-level)
  └── implementation gate (v1 vs v2 transport)
  └── config gate (minimum CLI version)
  └── behavior gate (auto-connect default)
  └── behavior gate (mirror mode opt-in)
```

---

#### 6. Interaction with Commands

##### 6.1 CLI Entrypoint Routing
The CLI entrypoint uses compile-time flags to gate entire subcommand paths. Each feature's subcommand handler is a "fast path" -- a dynamic import that only loads the feature's module graph when the flag is enabled and the subcommand matches. When the compile-time flag is false, the bundler eliminates the entire block. The subcommand string never appears in the binary, and the target module is not bundled.

##### 6.2 Two-Layer Gating
Some subcommands require both compile-time and runtime gates. The compile-time flag ensures the code is present. The runtime gates ensure the user is entitled, authenticated, and running a sufficiently recent version.

---

#### 7. Interaction with Tools

##### 7.1 Conditional Tool Registration
Tools are conditionally included in the tool pool through compile-time flags using the module-level guard pattern. Feature modules are either loaded or set to null, then conditionally spread into the base tool array.

##### 7.2 Runtime Tool Gating
Individual tools implement an `isEnabled()` method for runtime checks. A tool present in the binary but whose `isEnabled()` returns false is excluded from tool descriptions sent to the model. The model never sees it and cannot call it.

##### 7.3 Coordinator Mode
Coordinator mode dynamically reshapes the tool pool based on agent role. The compile-time flag ensures coordinator module code exists; a runtime check determines whether the current session is operating in coordinator mode.

##### 7.4 Voice Mode
Voice mode uses compile-time flags to conditionally load the entire voice subsystem. Within the voice module, runtime gates control specific behaviors (e.g., which STT model is used).

---

#### 8. Interaction with Services

##### 8.1 Pattern: Compile-Time Module Loading + Runtime Activation
Services follow a consistent two-tier pattern:
1. **Compile-time:** Module is loaded or set to null based on the feature flag
2. **Runtime:** A per-user activation check (GrowthBook gate) determines whether the service runs

This pattern appears in: memory extraction, team memory sync, context collapse, and others.

##### 8.2 Analytics Infrastructure
Analytics modules ship in all builds. Runtime kill switches provide emergency control over analytics data flow (enabling/disabling individual analytics sinks, controlling sampling rates, configuring batching).

##### 8.3 Settings Sync
Settings synchronization from the server is gated by a runtime flag, enabling controlled rollout of remote configuration.

---

#### 9. Kill Switch Patterns

Kill switches are the most operationally critical pattern. They enable remote disabling of features or enforcement of constraints without deploying new code.

##### 9.1 Version Kill Switch
A JSON dynamic config specifies a maximum allowed CLI version. If the current version exceeds the ceiling, the system can display a message or force downgrade. Uses a blocking accessor to ensure freshness. Periodic refresh ensures long-running sessions pick up changes.

##### 9.2 Classifier Fail-Closed Gate
Controls behavior when the auto-mode safety classifier is unavailable:
- **true (default):** Fail closed -- deny the tool call
- **false:** Fail open -- fall back to user approval

This is a security kill switch for controlling auto-mode behavior during classifier API degradation.

##### 9.3 Analytics Sink Kill Switch
A JSON config disables individual analytics sinks. Fail-open: missing or malformed config keeps sinks running.

##### 9.4 Cron Scheduling Kill Switch
Can disable all scheduled task execution across the fleet. Individual configuration can override scheduling parameters during an incident.

##### 9.5 Bridge Version Floor
Specifies minimum CLI version for Remote Control. Forces upgrades when critical bridge bugs are discovered.

---

#### 10. Design Principles

1. **Zero-Cost Absence:** When a feature is off at compile time, its cost is literally zero. No code, no strings, no modules, no memory.

2. **Explicit Tier Separation:** Compile-time flags are structural (what ships). Runtime flags are behavioral (what runs for whom). They must not be conflated.

3. **Fail-Safe Defaults:** Boolean gates default to false. JSON configs default to empty/safe structures. If the flag system is unreachable, features are off, never unexpectedly on.

4. **Cache-First, Refresh-Eventually:** The preferred accessor never blocks. Fresh values arrive asynchronously. Exception: blocking accessors exist for gates where staleness is harmful.

5. **Layered Override for Debugging:** env var > config file > remote eval > disk cache > default. Escape hatches at every level.

6. **Naming Opacity:** Runtime gate names are intentionally opaque to prevent feature semantics from leaking through telemetry, logs, or binary inspection.

7. **Composability:** Compile-time and runtime flags compose naturally: compile flags set the boundary of what is possible; runtime flags refine what is active within that boundary.

8. **Auth-Aware Initialization:** The system handles the auth-not-yet-available state gracefully, re-initializing when auth becomes available and ensuring security-critical gates evaluate against current user attributes.

9. **Observable Flag State:** Introspection is provided for operators and developers: complete flag maps, active overrides, debug logging, experiment exposure tracking, and a live UI.

10. **Incremental Migration:** The system supports provider migration via fallback chains (new provider > legacy provider > default), avoiding flag-day cutovers.
