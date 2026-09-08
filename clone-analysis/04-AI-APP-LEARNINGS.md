# Learnings for Building AI Applications

> **Scope**: Concrete, actionable patterns extracted from Claude Code's source that apply to building any AI-powered application — from chatbots to coding agents to autonomous systems.

---

## Table of Contents

1. [The Tool Abstraction: Make AI Actions First-Class](#1-the-tool-abstraction)
2. [Permission Architecture: Trust Is Layered](#2-permission-architecture)
3. [Streaming-First Design](#3-streaming-first-design)
4. [Memory Is Not One System](#4-memory-is-not-one-system)
5. [Context Window Is a Managed Resource](#5-context-window-management)
6. [Feature Flags at Two Speeds](#6-feature-flags-at-two-speeds)
7. [Circuit Breakers for AI Loops](#7-circuit-breakers-for-ai-loops)
8. [Provider Abstraction from Day One](#8-provider-abstraction)
9. [The Hook Pattern: Extensibility Without Coupling](#9-the-hook-pattern)
10. [State Management for Long-Running Agents](#10-state-management)
11. [Security as Architecture, Not Afterthought](#11-security-as-architecture)
12. [Multi-Agent Is a Permission Problem](#12-multi-agent-permissions)
13. [Startup Performance Matters for CLI AI](#13-startup-performance)
14. [Observability Drives Design](#14-observability-drives-design)
15. [The Config Hierarchy Pattern](#15-config-hierarchy)
16. [Lazy Loading for Plugin Ecosystems](#16-lazy-loading)
17. [Graceful Degradation Everywhere](#17-graceful-degradation)
18. [Anti-Patterns Found (What NOT to Do)](#18-anti-patterns)

---

## 1. The Tool Abstraction

**Lesson**: Every action the AI can take should go through a unified tool interface — not raw function calls.

### What Claude Code Does

```typescript
interface Tool {
  name: string
  aliases?: string[]
  description(input, opts): string
  call(input, context): Promise<ToolResult>
  schema: JSONSchema
}
```

Every tool — bash commands, file reads, web searches, MCP calls — shares this interface. The 41 tools in Claude Code all submit to the same permission checks, hook system, telemetry, and abort handling.

### Why It Matters for Your App

If you let the AI call arbitrary functions, you can't:
- Add permissions later (too many call sites)
- Add logging later (have to instrument each function)
- Add hooks/plugins (no interception point)
- Cancel operations (no unified abort mechanism)

**Pattern to adopt**: Define a `Tool` interface early. Even if you have 3 tools, the interface pays for itself when you add the 4th.

### The Context Object

Claude Code passes a `ToolUseContext` with 30+ fields to every tool. This is excessive — but the pattern is right:

```typescript
// Good: explicit context, not globals
tool.call(input, { abortController, permissions, state, ... })

// Bad: tools reaching into global state
tool.call(input) // internally: import { globalState } from './state'
```

**Start with 5-10 context fields**, expand as needed. The key is that tools don't reach into globals.

---

## 2. Permission Architecture

**Lesson**: Permissions should be a separate system with multiple sources, not hardcoded checks inside tools.

### The Pattern

```
Rule sources: { global, project, workspace, enterprise, CLI, session, runtime }
Check order: deny → allow → classify → hook → ask
```

### What to Adopt

**Minimum viable permission system:**

```
1. Define rules as data (not code): { tool: "bash", pattern: "rm *", action: "deny" }
2. Load rules from at least 2 sources (user settings + project settings)
3. Check deny rules first (fail-closed)
4. Default to "ask" when uncertain
```

**The deny-first principle**: Claude Code checks deny rules before allow rules. This means a project can't override a user's security restriction. This ordering is critical — reverse it and any `.claude/settings.json` in a repo can grant itself unlimited access.

### The Dangerous Pattern Database

```typescript
const DANGEROUS_BASH_PATTERNS = [
  'python', 'node', 'eval', 'exec', 'sudo', 'ssh', ...
]
```

**You need this too.** Every AI app that executes code needs a list of patterns that should never be auto-approved. Don't wait until someone's `rm -rf /` destroys a system.

---

## 3. Streaming-First Design

**Lesson**: Design your AI layer to stream from the start. Retrofitting streaming onto a request-response architecture is painful.

### What Claude Code Does

```typescript
async *submitMessage(): AsyncGenerator<SDKMessage> {
  // Yields messages as they arrive
}
```

The entire system is built around async generators. The UI renders incrementally. Tool calls are identified and executed mid-stream. The user sees the model "thinking" in real time.

### Why Request-Response Fails

With a request-response model:
- User waits 30+ seconds seeing nothing (model is working)
- Tool execution can't start until full response is received
- Cancellation is all-or-nothing (can't cancel mid-response)
- Memory usage spikes (buffering full response)

### Pattern to Adopt

```typescript
// Your AI service should yield events, not return results
async function* processQuery(input: string): AsyncGenerator<Event> {
  yield { type: 'thinking_started' }
  for await (const chunk of model.stream(input)) {
    yield { type: 'token', content: chunk }
    if (isToolCall(chunk)) {
      yield { type: 'tool_started', tool: chunk.tool }
      const result = await executeTool(chunk)
      yield { type: 'tool_result', result }
    }
  }
  yield { type: 'complete' }
}
```

---

## 4. Memory Is Not One System

**Lesson**: AI memory has different lifetimes, and each needs its own storage and retrieval mechanism.

### Claude Code's Four Memory Systems

| System | Lifetime | Trigger | Storage |
|--------|----------|---------|---------|
| memdir | Permanent | User + auto | MEMORY.md (structured) |
| SessionMemory | Session | Token threshold | Per-session file |
| autoDream | Cross-session | Time + session gates | Consolidated memory |
| extractMemories | Turn-end | Automatic | Feeds into memdir |

### What to Adopt

**At minimum, separate these concerns:**

1. **Turn context**: What the model needs to know for THIS response (recent messages, current file contents)
2. **Session context**: What should persist across turns but not sessions (conversation topic, open files)
3. **Persistent memory**: What should survive across sessions (user preferences, project knowledge)

**The injection pattern**: Claude Code loads `MEMORY.md` (~200 lines, 25KB cap) into the system prompt every turn. This is a hard cap that prevents memory from consuming the entire context window. Your app needs similar budgets.

**The extraction pattern**: Don't ask users to "save" memories. Extract them automatically from conversations. Claude Code runs `extractMemories` at the end of turns with high information content.

---

## 5. Context Window Management

**Lesson**: The context window is a finite resource that must be actively managed, not passively consumed.

### The Compaction Pattern

When conversation history exceeds the context budget:

```
1. Strip images (save tokens)
2. Group messages by API round
3. Send to model for summarization (lossy compression)
4. Re-inject critical file contents (selective restoration)
5. Re-inject triggered skills (capability preservation)
6. Reset with hybrid state: summary + fresh data
```

### Budget Constants to Set in Your App

```typescript
// How much to restore after compaction
POST_COMPACT_TOKEN_BUDGET = 50_000       // Total restoration budget
MAX_FILES_TO_RESTORE = 5                  // Don't restore everything
MAX_TOKENS_PER_FILE = 5_000              // Truncate large files
MAX_TOKENS_PER_SKILL = 5_000             // Cap skill definitions
SKILLS_TOKEN_BUDGET = 25_000             // Total skills budget
MAX_CONSECUTIVE_FAILURES = 3              // Circuit breaker
```

### The Cache Boundary Pattern

```typescript
const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

Split your system prompt into:
- **Static part** (cacheable): Role definition, tool instructions, formatting rules
- **Dynamic part** (per-turn): Memory context, recent state, user-specific data

The static part can be cached for hours, saving significant API costs. Claude Code uses 1-hour TTL on cached prompt sections.

---

## 6. Feature Flags at Two Speeds

**Lesson**: Use build-time flags for features that MUST NOT exist in certain builds, and runtime flags for gradual rollout.

### Build-Time (Compilation)

```typescript
// Feature is PHYSICALLY ABSENT from the binary
if (feature('INTERNAL_ONLY')) {
  require('./internal-module')
}
```

**Use for**: Internal tools, security-sensitive features, platform-specific code.

### Runtime (Gradual)

```typescript
// Feature can be toggled without rebuild
if (getFeatureValue('new_experimental_feature', false)) {
  enableExperiment()
}
```

**Use for**: A/B testing, gradual rollout, kill switches.

### The GrowthBook Pattern

Claude Code uses GrowthBook with:
- **Cached reads** (`_CACHED_MAY_BE_STALE`) for hot paths
- **Background refresh** (non-blocking updates)
- **Auth-aware re-initialization** (feature state changes when user logs in)
- **Experiment deduplication** (one exposure event per session)

**For your app**: Pick any feature flag service. The important patterns are:
1. Cache aggressively (don't fetch on every check)
2. Refresh in background (don't block on updates)
3. Have defaults for offline mode (feature flags shouldn't break your app)

---

## 7. Circuit Breakers for AI Loops

**Lesson**: Every loop involving AI calls needs a failure limit. AI + retry = infinite cost.

### The Discovery Story

Claude Code's compaction retry had no limit. BigQuery analysis revealed **250,000 wasted API calls per day** from sessions stuck in compaction-retry loops. Fix: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`.

### Where You Need Circuit Breakers

| Loop | Risk | Solution |
|------|------|----------|
| Model retries on 529 (overload) | Making overload worse | `MAX_529_RETRIES = 3` + bail for background tasks |
| Auto-compaction | Infinite API calls | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` |
| Memory extraction | Runaway processing | Sequential execution (one at a time) |
| Tool execution | Agent loops | `maxTurns` limit per conversation |
| Cost accumulation | Budget blowout | `maxBudgetUsd` hard cap |

### Implementation Pattern

```typescript
let consecutiveFailures = 0
const MAX_FAILURES = 3

async function withCircuitBreaker(operation) {
  if (consecutiveFailures >= MAX_FAILURES) {
    throw new CircuitBreakerOpen()
  }
  try {
    const result = await operation()
    consecutiveFailures = 0  // Reset on success
    return result
  } catch (error) {
    consecutiveFailures++
    throw error
  }
}
```

**The background vs foreground distinction**: Claude Code retries 529 errors for foreground tasks but bails immediately for background tasks (cron, autoDream). Background retries amplify overload.

---

## 8. Provider Abstraction

**Lesson**: Abstract the AI provider from day one. You will switch or add providers.

### Claude Code Supports 4 Providers

| Provider | Auth Method | Region Logic |
|----------|-----------|-------------|
| Direct API | API key + OAuth | N/A |
| AWS Bedrock | AWS credentials | Model-specific regions |
| Vertex AI | Google Cloud auth | Per-model env vars |
| Azure Foundry | Resource/credential | Foundry URLs |

### The Abstraction

```typescript
async function getAnthropicClient({
  apiKey?, maxRetries, model?, source?
}): Promise<Anthropic>
```

A single function returns a configured client regardless of provider. The provider is determined by environment variables and settings.

### What to Adopt

Even if you use only one provider today:

```typescript
interface AIProvider {
  stream(messages, config): AsyncGenerator<StreamEvent>
  countTokens(content): number
  supportedModels(): string[]
}
```

When (not if) you add a second provider, this interface saves weeks of refactoring.

---

## 9. The Hook Pattern

**Lesson**: Let users inject behavior at defined points without modifying your code.

### Claude Code's Hook Points

| Hook | When | Can Block? |
|------|------|-----------|
| `PreToolUse` | Before tool executes | Yes |
| `PostToolUse` | After tool executes | No (but can modify) |
| `PostSampling` | After model responds | No (but can modify) |
| `UserPromptSubmit` | When user sends message | Yes |

### Hook Execution Modes

- **File-based**: Shell script with argument substitution (`$ARGUMENTS`, `$0`, `$1`)
- **HTTP-based**: External webhook (for integration with other services)
- **Agent-based**: Run through the model (for complex decisions)

### The SSRF Guard

Hooks that make HTTP calls go through an SSRF (Server-Side Request Forgery) guard. Your hook-based extension system needs this too — otherwise a malicious hook config could make your app attack internal services.

### Pattern to Adopt

```typescript
// Define named extension points
type HookPoint = 'before_tool' | 'after_tool' | 'before_send' | 'after_receive'

// Hooks return structured responses
interface HookResponse {
  ok: boolean
  reason?: string
  modifiedInput?: unknown
}

// 5-second timeout enforcement
const result = await withTimeout(hook.execute(context), 5000)
```

---

## 10. State Management

**Lesson**: Long-running AI agents need explicit state management, not implicit conversation history.

### What Claude Code Tracks

```typescript
// Not just messages — structured state
{
  mutableMessages: Message[]          // Conversation
  readFileState: Map<string, Hash>    // File dedup
  discoveredSkillNames: Set<string>   // Skill tracking
  permissionDenials: ToolDenial[]     // Trust history
  loadedNestedMemoryPaths: Set<string>// Memory dedup
  costs: CostTracker                  // Budget
  invokedSkills: Map<string, Skill>   // Persists compaction
  sessionCronTasks: CronTask[]        // Scheduled work
}
```

### Why Conversation History Isn't Enough

If your "state" is just the message array:
- You can't deduplicate file reads (no content hash tracking)
- You can't enforce budgets (no cost accumulation)
- You can't survive compaction (skills and files lost)
- You can't track permissions (no denial history)

### Pattern to Adopt

Separate your state into:

```typescript
interface AgentState {
  // Conversation (may be compacted)
  messages: Message[]
  
  // Session tracking (persists across compaction)
  toolUsage: Map<string, number>     // How often each tool was used
  fileCache: Map<string, string>     // Files already read
  permissions: PermissionRecord[]    // What was allowed/denied
  
  // Budget
  tokensUsed: number
  costUsd: number
  turnsCompleted: number
}
```

---

## 11. Security as Architecture

**Lesson**: Security in AI apps isn't a layer — it's threaded through every subsystem.

### Claude Code's Security Surface

| Threat | Defense | Location |
|--------|---------|----------|
| Command injection | tree-sitter AST parsing | bashParser.ts |
| Overly broad permissions | Dangerous pattern stripping | dangerousPatterns.ts |
| Prompt injection | Explicit warnings in system prompt | prompts.ts |
| Data exfiltration | SSRF guard on hooks | ssrfGuard.ts |
| Competitive distillation | Fake tool injection + text summarization | claude.ts |
| Internal info leakage | Undercover mode | undercover.ts |
| Session hijacking | Transport-layer attestation | Bun HTTP stack |
| Path traversal | Sandbox adapter + path validation | sandbox-adapter.ts |
| Memory disclosure | 0o600 file permissions | SessionMemory |

### What to Adopt

**Minimum security for an AI app that executes code:**

1. **Parse, don't regex**: Use an AST parser for command validation. Regex misses edge cases that attackers find.
2. **Deny-first permissions**: Check what's forbidden before what's allowed.
3. **Prompt injection warnings**: Tell the model about injection attacks in its system prompt.
4. **Budget limits**: Hard caps on cost, turns, and time. AI loops are expensive.
5. **File permissions**: Memory files should be user-readable only (0o600).
6. **Path validation**: Validate every file path against an allowlist, not a denylist.

---

## 12. Multi-Agent Is a Permission Problem

**Lesson**: The hardest part of multi-agent systems isn't spawning agents — it's deciding what they're allowed to do.

### Claude Code's Approach

| Agent Mode | Permissions |
|-----------|------------|
| Coordinator worker (simple) | Bash + Read + Edit only |
| Coordinator worker (full) | All standard tools + Skills |
| Swarm teammate | Full list with leader approval for dangerous ops |
| Background (KAIROS) | Read-only bash during dreams |

### The Permission Proxy Pattern

Workers can't prompt the user directly. Instead:
1. Worker requests permission
2. Request routes to leader (via bridge or mailbox)
3. Leader shows prompt to user
4. User decision routes back to worker

### What to Adopt

Before building multi-agent, answer:
- Can a sub-agent execute bash commands? Which ones?
- Can a sub-agent write files? To which paths?
- Can a sub-agent make API calls? To which services?
- Who approves dangerous operations — the spawning agent or the user?
- What happens if the user is offline when approval is needed?

---

## 13. Startup Performance

**Lesson**: AI CLI tools must feel instant despite heavy initialization.

### Claude Code's Strategy

```
Parallel during module load:
  ├── Profiler checkpoint
  ├── MDM read (subprocess worker)
  ├── Keychain prefetch (macOS ~65ms I/O)
  └── Feature flag initialization

Lazy after startup:
  ├── Command modules (loaded on first use)
  ├── MCP connections (connected on first tool call)
  └── Plugin initialization (loaded on demand)
```

### What to Adopt

1. **Measure first**: Add a startup profiler before optimizing
2. **Parallelize I/O**: Network, filesystem, and keychain reads can overlap
3. **Lazy-load features**: 80+ commands don't need to load at startup
4. **Break circular deps**: Use lazy `require()` instead of top-level imports
5. **Dead code eliminate**: Features behind build-time flags don't load

---

## 14. Observability Drives Design

**Lesson**: Instrument first, then optimize. Production data reveals problems your tests can't.

### Claude Code's Telemetry

```typescript
// 10+ OpenTelemetry meters
meter, sessionCounter, locCounter, prCounter,
commitCounter, costCounter, tokenCounter, 
codeEditToolDecisionCounter, activeTimeCounter
```

### Design Decisions Driven by Data

| Discovery | Source | Design Change |
|-----------|--------|--------------|
| 250K wasted API calls/day | BigQuery | Added compaction circuit breaker |
| 529 retries amplifying overload | Server metrics | Background tasks bail immediately |
| Stale features after login | GrowthBook logs | Auth-aware re-initialization |
| FPS drops during streaming | Terminal metrics | `OffscreenFreeze` component |
| Duplicate experiment exposure | Analytics | Session-level dedup |

### Pattern to Adopt

Track at minimum:
- **API calls**: Count, latency, tokens, cost per session
- **Tool usage**: Which tools, how often, success/failure
- **Errors**: Classified by type (transient vs permanent)
- **Session lifecycle**: Duration, turns, compaction count, memory size

---

## 15. The Config Hierarchy Pattern

**Lesson**: AI apps serving individuals AND organizations need a multi-source config system.

### The Merge Order

```
Enterprise (remote) → User (global) → Project → Workspace → CLI → Session → Runtime
```

Each source has different:
- **Trust levels**: Enterprise settings are trusted; project settings are not
- **Persistence**: Enterprise is remote-cached; session is ephemeral
- **Override behavior**: Later sources override earlier, except deny rules (cumulative)

### What to Build

```typescript
interface ConfigSource {
  name: string
  trustLevel: 'trusted' | 'untrusted'
  persistence: 'remote' | 'disk' | 'memory' | 'ephemeral'
  load(): Promise<Config>
}

function mergeConfigs(sources: ConfigSource[]): Config {
  // Deny rules: union (cumulative)
  // Allow rules: latest wins
  // Settings: latest wins
  // Untrusted sources: validated before merge
}
```

---

## 16. Lazy Loading

**Lesson**: AI apps have many capabilities. Load them on demand, not at startup.

### What Gets Lazy-Loaded

| Component | When Loaded | Why Lazy |
|-----------|------------|----------|
| 80+ commands | First invocation | Startup speed |
| MCP servers | First tool call | Network latency |
| Skill definitions | First trigger | Memory |
| Plugin modules | First enable | User didn't install all plugins |
| Coordinator module | If COORDINATOR_MODE | Build-time absence |
| Voice module | If claude.ai | Platform-specific |

### The Import Pattern

```typescript
// Static import: loads at module parse time
import { assistant } from './commands/assistant'  // SLOW

// Dynamic import: loads on first use
const command = {
  load: () => import('./commands/assistant.js')   // FAST
}
```

---

## 17. Graceful Degradation

**Lesson**: AI apps should work with partial capability, not fail completely.

### Degradation Examples in Claude Code

| Failure | Degradation |
|---------|------------|
| GrowthBook unreachable | Use disk-cached flags, then defaults |
| Policy limits API down | Features continue without restrictions |
| MCP server disconnected | Tool pool shrinks; rest works |
| Token counting data missing | Fall back to top-level counters |
| Remote settings checksum mismatch | Keep previous settings |
| OAuth token expired | Attempt refresh; fall back to API key |
| Compaction fails 3x | Stop trying (circuit breaker) |
| Background dream fails | Rewind lock for next session retry |

### Pattern

```typescript
// Every external dependency gets a fallback
async function getFeatureFlag(name, defaultValue) {
  try {
    return await fetchRemoteFlag(name)
  } catch {
    try {
      return readDiskCache(name)
    } catch {
      return defaultValue  // Always have a hardcoded default
    }
  }
}
```

---

## 18. Anti-Patterns Found

Not everything in Claude Code should be copied. Some patterns to avoid:

### 1. The God Object

The 1,300-line bootstrap singleton (`state.ts`) is imported everywhere. It makes dependency analysis impossible and testing painful. Use proper dependency injection or a state management library.

### 2. Dual Parsers in Security Paths

The deprecated bash command splitter runs alongside the new tree-sitter parser in 8 security validators. They disagree on `\r`. This is a liability, not defense in depth.

**Lesson**: When replacing a security-critical parser, cut over completely. Don't run both.

### 3. Magic Numbers in Comments

```typescript
// @[MODEL LAUNCH] — update on new model release
```

Comments as update tracking is fragile. Use a configuration file or constant mapping that's validated by tests.

### 4. Overloaded Single Files

`bootstrap/state.ts` (1,300 lines), `prompts.ts` (250+ lines of prompt assembly), `tools.ts` (tool registry + assembly logic). These should be split.

### 5. Cache Staleness Is Invisible

`getFeatureValue_CACHED_MAY_BE_STALE` is honest naming but invisible to callers. A stale cache returning the wrong feature flag value is indistinguishable from the correct value at the call site.

**Lesson**: If you use stale caches, add observability (log when stale value is returned, track drift).

---

## Summary: The Top 10 Lessons

| # | Lesson | Complexity if Ignored |
|---|--------|----------------------|
| 1 | Unified tool interface with context | Can't add permissions/hooks/logging later |
| 2 | Deny-first layered permissions | Security vulnerabilities |
| 3 | Streaming from day one | 30s blank screens; can't cancel |
| 4 | Separate memory lifetimes | Context window consumed by stale data |
| 5 | Active context window management | Conversations die at token limits |
| 6 | Circuit breakers on all AI loops | $$$$ runaway costs |
| 7 | Provider abstraction | Vendor lock-in |
| 8 | Hook system for extensibility | Forks and patches |
| 9 | Explicit state, not just messages | Can't survive compaction |
| 10 | Instrument before optimizing | Invisible problems stay invisible |

---

*Next: [05-ARCHITECTURE.md](05-ARCHITECTURE.md) — Independent architecture analysis from code*
