# Developer Experience & Engineering Culture

> **Scope**: What the source code reveals about how this product was built — the engineering culture, development practices, trade-offs that were made, and signals about how a team ships a complex AI product.

---

## Table of Contents

1. [Code Archaeology: What Comments Reveal](#1-code-archaeology)
2. [Naming Conventions: Honesty in APIs](#2-naming-conventions)
3. [The Internal/External Split](#3-internal-external-split)
4. [Error Handling Philosophy](#4-error-handling-philosophy)
5. [Testing Approach](#5-testing-approach)
6. [Performance Culture](#6-performance-culture)
7. [Security Posture](#7-security-posture)
8. [Technical Debt Signals](#8-technical-debt-signals)
9. [Codename Culture](#9-codename-culture)
10. [Observability-Driven Development](#10-observability-driven-development)
11. [The Enterprise Layer](#11-the-enterprise-layer)
12. [Design Trade-Offs Made](#12-design-trade-offs)

---

## 1. Code Archaeology

The codebase contains markers that reveal its evolution:

### `@[MODEL LAUNCH]` Comments

```typescript
// @[MODEL LAUNCH] — update these on new model release
const CLAUDE_4_5_OR_4_6_MODEL_IDS = {
  opus: 'claude-opus-4-6',
  sonnet: 'claude-sonnet-4-6',
  haiku: 'claude-haiku-4-5-20251001',
}
```

These comments serve as **search markers** for model launch checklists. Instead of a config file that tests enforce, they rely on grep + human discipline. This suggests a team that ships model updates manually, not through automation.

### Data-Driven Constants

```typescript
// Discovered via BigQuery: 250K wasted API calls/day
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// Empirical risk, not universal
...(process.env.USER_TYPE === 'ant'
  ? ['curl', 'wget', 'git', 'kubectl', ...]
  : [])
```

Constants are annotated with **why they exist**, not just what they are. The dangerous pattern list for Anthropic employees is different because real usage data showed different risk profiles.

### The Deprecation Trail

```typescript
splitCommand_DEPRECATED() // Still used in 8 security validators
```

The `_DEPRECATED` suffix is honest — but the function isn't deprecated in practice. It's still in the critical security path alongside its replacement. This suggests a codebase where backward compatibility outweighs cleanup.

---

## 2. Naming Conventions

### Honest Cache APIs

```typescript
getFeatureValue_CACHED_MAY_BE_STALE('tengu_anti_distill_fake_tool_injection', false)
```

The function name **tells you it might lie**. Most codebases hide staleness behind a clean API. Claude Code puts the caveat in the name itself. This is a team that optimizes for the reader's understanding, not the API's aesthetics.

### Codename System

Every feature has an internal codename prefixed with `tengu_`:

```
tengu_thinkback       — Year in Review
tengu_kairos_cron     — Cron scheduling
tengu_turtle_carbon   — Extended thinking
tengu_scratch         — Coordinator scratchpad
tengu_anti_distill_*  — Anti-distillation variants
```

`tengu` appears to be the project codename. Feature flags are namespaced under it, creating a grep-friendly pattern.

### Ant vs External

The term "ant" refers to Anthropic employees throughout the codebase:

```typescript
process.env.USER_TYPE === 'ant'
// ant-only tools: REPLTool, SuggestBackgroundPRTool, MonitorTool
// ant-only retry: persistent retry mode with indefinite backoff
```

Not "internal" or "employee" — specifically "ant." This is a culture that names things for quick typing, not for outsider readability.

---

## 3. The Internal/External Split

The codebase maintains a **single source with conditional compilation**, not separate repositories:

### Build-Time Gating

```typescript
if (feature('KAIROS')) {
  // This entire section removed for external builds
  require('./commands/assistant')
}
```

### What Ants Get

| Feature | External | Internal (Ant) |
|---------|----------|----------------|
| KAIROS (background agent) | Absent | Available |
| Coordinator mode | Absent | Available |
| Anti-distillation | Absent | Active |
| Transcript classifier | Absent | Available |
| Persistent retry | No | Indefinite with 5min cap |
| REPLTool | No | Yes |
| MonitorTool | No | Yes |
| SuggestBackgroundPRTool | No | Yes |
| SendUserFileTool | No | Yes |
| PushNotificationTool | No | Yes |
| Additional dangerous patterns | No | curl, wget, git, kubectl, aws, gcloud |

### The Implication

The source code is a **superset** of any shipped binary. Reading the source shows features that no user has ever seen. The leak is more revealing than the product.

---

## 4. Error Handling Philosophy

### Fail-Open for External Dependencies

```typescript
// Policy limits API down? Continue without restrictions.
// GrowthBook unreachable? Use cached/default values.
// Remote settings checksum mismatch? Keep previous settings.
// OAuth token expired? Attempt refresh, fall back to API key.
```

The pattern is consistent: **never block the user because a backend service is unreachable**. This is pragmatic but creates a security surface — if the policy limits API is down, all policies are disabled.

### Fail-Closed for Security

```typescript
// Deny rules checked FIRST (before allow rules)
// Auto-mode strips overly-broad permissions
// Default action: ASK (not allow)
```

For permissions, the philosophy inverts: **block by default, require explicit approval**. The asymmetry is intentional — availability is valued for features, security is valued for actions.

### Circuit Breakers, Not Infinite Retry

Every retry loop has a maximum:
- Compaction: 3 failures
- API 529: 3 retries (foreground), 0 retries (background)
- OAuth refresh: 1 attempt
- MCP auth: 15-minute cooldown

This is learned behavior. The comments indicate these limits were added **after production incidents**, not designed upfront.

---

## 5. Testing Approach

### What Exists

- `daemon/auth.test.ts` — Authentication tests (Bun/vitest)
- Bash parser validated against **3,449-input golden corpus** from WASM implementation
- JSON snapshot references for tool and subsystem validation

### What Doesn't Exist

- No `src/__tests__/` directory
- No visible integration test suite for the main loop
- No end-to-end tests for the permission system
- No tests for the compaction pipeline

### The Implication

Testing is **validation-focused** (does the parser produce correct output?) rather than behavior-focused (does the system behave correctly end-to-end?). For a security-critical system (bash command parsing, permission decisions), this is a notable gap.

The golden corpus approach for the bash parser is strong — 3,449 inputs is substantial. But the parser's interaction with the permission system (which uses both the new parser AND the deprecated one) appears untested.

---

## 6. Performance Culture

### Startup Profiling

```typescript
profileCheckpoint('main_tsx_entry')
// ... startup work ...
profileCheckpoint('cli_parsed')
// ... more work ...
profileCheckpoint('first_render')
```

Performance checkpoints are **built into the application**, not an afterthought. The startup sequence is explicitly optimized with parallel initialization.

### Terminal FPS Tracking

```tsx
<FpsMetricsProvider>
  {/* React components with memoization */}
</FpsMetricsProvider>
```

The team measures **terminal rendering frame rate**. This is unusual for a CLI — most command-line tools don't think about FPS. But when you're streaming token output while rendering tool results and diffs, jank is visible.

### Lazy Everything

```typescript
// Commands: loaded on first use (not at startup)
command.load = () => import('./commands/vim.js')

// MCP: connected on first tool call (not at startup)
// Modules: lazy require() for circular dependency breaking
// Skills: rendered on first trigger
```

The pattern is "don't load it until you need it." With 80+ commands, 41+ tools, and N MCP servers, eager loading would make startup noticeably slow.

### Write-Through Caching

```typescript
memoizeWithTTL(fn, 300_000)
// Stale? Return old, refresh in background
// Empty? Block and compute
// Error on refresh? Delete cache (force recompute next time)
```

The cache never makes the hot path slower. Stale values are returned instantly while a background refresh runs. This is the right trade-off for feature flags and config values where millisecond staleness is acceptable.

---

## 7. Security Posture

### Defense in Depth

The codebase applies security at multiple layers simultaneously:

```
Layer 1: Build-time removal of internal features
Layer 2: Feature flag gating (runtime)  
Layer 3: Permission rules (7 sources)
Layer 4: Bash AST parsing (not regex)
Layer 5: Dangerous pattern database
Layer 6: Hook-based intervention
Layer 7: Sandbox runtime
Layer 8: Transport-layer attestation
Layer 9: Anti-distillation (fake tools + summarization)
Layer 10: Undercover mode (info scrubbing)
```

### But...

- The deprecated bash parser runs alongside the new one
- No visible tests for the permission decision flow
- The god object (`state.ts`) is imported by security-critical code
- `_DEPRECATED` functions are still in the security path

The security posture is **architecturally strong but implementation-fragile**. The layers are right, but the connections between layers have gaps.

---

## 8. Technical Debt Signals

### The 1,300-Line Singleton

`bootstrap/state.ts` with 100+ fields imported everywhere. This is acknowledged tech debt — the patterns suggest it grew organically as features were added.

### The Dual Parser Problem

```
New: tree-sitter AST (pure TS, byte offsets, 50ms timeout)
Old: splitCommand_DEPRECATED (regex, char offsets, no timeout)
Both: active in security validators, disagree on \r
```

This is a classic migration gap. The new parser is better in every way, but the old one can't be removed because 8 validators depend on its behavior.

### Magic Comments as Process

- `@[MODEL LAUNCH]` — search-based update tracking
- `// TODO` markers (standard but no automation)
- `// Ant-only` — inline audience markers instead of proper gating

### Overloaded Files

| File | Lines | Should Be |
|------|-------|-----------|
| `bootstrap/state.ts` | 1,300+ | 5+ focused modules |
| `constants/prompts.ts` | 250+ | Separate prompt builder |
| `tools.ts` | 150+ | Registry + assembler split |

---

## 9. Codename Culture

### Project Codenames

| Codename | System |
|----------|--------|
| `tengu` | Project-level feature flag prefix |
| `KAIROS` | Background agent / assistant mode |
| `BUDDY` | Companion Tamagotchi system |
| `Capybara` | One of 18 companion species |
| `ant` | Anthropic employee |
| `CCR` | Claude Code Remote |
| `MDM` | Mobile Device Management |

### The Undercover System

The irony: the codebase has a system (`undercover.ts`) specifically designed to scrub codenames from public contexts — but the leak exposed all the codenames this system was designed to hide.

---

## 10. Observability-Driven Development

### What Gets Measured

```typescript
// 10+ OpenTelemetry meters
meter                        // Base meter
sessionCounter               // Session lifecycle
locCounter                   // Lines of code added/removed
prCounter                    // Pull requests created
commitCounter                // Commits made
costCounter                  // API spend
tokenCounter                 // Token consumption
codeEditToolDecisionCounter  // Edit tool choices
activeTimeCounter            // User active time
```

### Events That Drive Design

| Metric | Design Decision It Drove |
|--------|-------------------------|
| API call volume | Compaction circuit breaker (3 max failures) |
| 529 error rate | Background tasks bail immediately |
| Startup timing | Parallel initialization + lazy loading |
| Terminal FPS | OffscreenFreeze, memoization |
| Feature flag latency | Cached-may-be-stale API design |
| Experiment exposure count | Session-level deduplication |

The pattern: **measure → discover → fix**. Not **design → implement → measure**. This is a team that runs the product and learns from its behavior.

---

## 11. The Enterprise Layer

### What Enterprise Customers Get

```
Remote Managed Settings
  ├── Policy enforcement (feature restrictions)
  ├── Custom system prompts (org-specific instructions)
  ├── Default configurations (centralized)
  ├── Checksum validation (tamper detection)
  └── Disk-cached with background refresh (60min)

Policy Limits
  ├── Organization-level feature restrictions
  ├── Team/Enterprise subscription gating
  ├── Fail-open (don't block on API failure)
  └── Memory-only cache (fresh each session)

MDM (Mobile Device Management)
  ├── Reads enterprise config at startup
  ├── Subprocess worker (parallel, non-blocking)
  └── Platform-specific (macOS)
```

### OAuth + API Key Dual Path

```typescript
await checkAndRefreshOAuthTokenIfNeeded()
if (!isClaudeAISubscriber()) {
  await configureApiKeyHeaders(defaultHeaders, getIsNonInteractiveSession())
}
```

The system supports both subscription-based OAuth (claude.ai, Teams) and direct API key access. This dual path adds complexity to every authentication flow.

---

## 12. Design Trade-Offs Made

### Chose: Performance over Purity

- God object that's fast to access vs proper dependency injection that's slower
- Mutable message arrays vs immutable data structures
- Lazy require (breaks circular deps) vs clean module architecture
- Write-through cache (stale values) vs always-fresh reads

### Chose: Security over Simplicity

- 24-file permission system vs simple allow/deny
- Two parallel bash parsers vs one clean parser
- 7 rule sources vs one config file
- SSRF guard on hooks vs trusting hook authors
- Transport-layer attestation vs application-layer auth

### Chose: Availability over Consistency

- Fail-open for policy limits vs blocking on unreachable server
- Cached-may-be-stale flags vs always-fresh evaluation
- Default values for everything vs failing on missing config
- Circuit breakers that stop trying vs exhaustive retry

### Chose: Velocity over Elegance

- Single state file (easy to add fields) vs proper state management
- Comments as process markers vs automated tooling
- Feature flags for everything vs clean feature isolation
- `_DEPRECATED` suffixes vs actual removal

---

## The Team Profile (Inferred)

From the code, this appears to be:

- A **product-oriented team** (feature flags, A/B testing, stickers command alongside AI tools)
- With **strong observability culture** (telemetry drives decisions, not guesses)
- That **ships fast** (tech debt accumulates, feature flags everywhere, lazy cleanup)
- With **security expertise** (defense in depth, multiple parsers, undercover mode)
- Running a **live service** (circuit breakers from production incidents, not design)
- That **values user experience** (FPS tracking, startup optimization, streaming)
- Using **codenames extensively** (tengu, KAIROS, BUDDY — internal vocabulary)
- With **enterprise customers** (MDM, policy limits, remote settings)
- And **competitive awareness** (anti-distillation is not a typical concern)

The codebase reads like a product that grew from a prototype to a production system while maintaining velocity. The architectural choices favor speed-of-addition over structural purity. This is consistent with a team shipping a competitive AI product where being first matters.

---

*Next: [Excalidraw diagrams](./diagrams/) — Visual architecture and flow diagrams*
