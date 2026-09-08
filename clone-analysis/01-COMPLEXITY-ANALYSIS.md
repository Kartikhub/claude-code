# Complexity Analysis: Claude Code Source

> **Scope**: Independent analysis of 1,902 TypeScript files comprising the Claude Code CLI — a terminal-based AI coding agent.  
> **Focus**: Where the real complexity lives, why it exists, and what makes this codebase genuinely hard to reason about.

---

## Table of Contents

1. [Complexity at a Glance](#1-complexity-at-a-glance)
2. [The Feature Flag Labyrinth](#2-the-feature-flag-labyrinth)
3. [Permission System: 24 Files of Trust Decisions](#3-permission-system-24-files-of-trust-decisions)
4. [The Query Engine: A Generator Inside a Loop Inside a State Machine](#4-the-query-engine)
5. [Multi-Agent Orchestration: Coordinator, Swarm, and Workers](#5-multi-agent-orchestration)
6. [Memory: Four Overlapping Systems](#6-memory-four-overlapping-systems)
7. [Message Compaction: Lossy Compression of Conversations](#7-message-compaction)
8. [The Streaming API Layer: Retries, Circuit Breakers, and Provider Abstraction](#8-the-streaming-api-layer)
9. [Startup Performance: Parallel Initialization Under Time Pressure](#9-startup-performance)
10. [State Management: A 1,300-Line Singleton](#10-state-management)
11. [Cross-Cutting Complexity Multipliers](#11-cross-cutting-complexity-multipliers)
12. [Complexity Metrics Summary](#12-complexity-metrics-summary)

---

## 1. Complexity at a Glance

| Dimension | Metric |
|-----------|--------|
| Total files | 1,902 TypeScript files |
| Tools | 41 tool directories, each with schema, permission logic, and execution |
| Commands | 80+ slash commands with platform-specific availability |
| Feature flags | 3 build-time (`bun:bundle`) + 20+ runtime (GrowthBook) |
| Permission sources | 7+ rule sources (global, project, workspace, CLI, command, session, settings) |
| API providers | 4 (Direct, AWS Bedrock, Vertex AI, Azure Foundry) |
| Transport protocols | 6 MCP transports (Stdio, SSE, WebSocket, HTTP, In-Process, SDK Control) |
| Memory systems | 4 distinct (memdir, SessionMemory, autoDream, extractMemories) |
| Agent modes | 4 (normal, coordinator, swarm worker, proactive/KAIROS) |
| Bootstrap state fields | 100+ fields in a 1,300-line singleton |

This isn't a "big" codebase in the way a monolith is big. It's complex because **every subsystem interacts with every other subsystem through shared state and feature flags**. A change to model selection affects permissions, which affects tool availability, which affects the coordinator prompt, which affects the streaming API layer.

---

## 2. The Feature Flag Labyrinth

Claude Code uses a **two-layer feature flag system** that creates combinatorial complexity:

### Layer 1: Build-Time Dead Code Elimination

```typescript
// From bun:bundle — evaluated at bundle time
if (feature('KAIROS')) {
  require('./commands/assistant')
}
if (feature('COORDINATOR_MODE')) {
  require('./coordinator/coordinatorMode')
}
if (feature('ANTI_DISTILLATION_CC')) {
  // fake tool injection logic
}
```

These are resolved by Bun's bundler. For external builds, `feature('KAIROS')` is `false`, and the entire assistant module is **physically absent** from the bundle. This means:

- You cannot test KAIROS features in external builds
- The code paths exist only in the source, never in shipped artifacts
- Reading the source gives you a *superset* of what any single build contains

### Layer 2: Runtime Feature Flags (GrowthBook)

```typescript
getFeatureValue_CACHED_MAY_BE_STALE('tengu_anti_distill_fake_tool_injection', false)
getFeatureValue_CACHED_MAY_BE_STALE('tengu_thinkback', false)
getFeatureValue_CACHED_MAY_BE_STALE('tengu_kairos_cron', false)
```

These are fetched from GrowthBook's remote config, cached with potential staleness, and checked at runtime. The `_CACHED_MAY_BE_STALE` suffix is an honest API: the value you get might be from the last session.

### Why This Is Complex

A single feature like "cron scheduling" requires:

1. `feature('KAIROS')` — build-time gate (is the module even in the bundle?)
2. `feature('AGENT_TRIGGERS')` — another build-time gate for the tool
3. `tengu_kairos_cron` — runtime flag (is cron enabled for this user?)
4. Policy limits check — does the organization allow it?
5. Permission rule check — does the settings file permit it?

**Five independent conditions** to determine if one feature is available. Debugging "why doesn't cron work?" means checking all five layers.

### The Override Hierarchy

GrowthBook has its own override chain:

```
Environment variables → Local config file → Remote evaluation → Disk cache → Hardcoded defaults
```

Combined with the build-time layer, the effective state of any feature flag is determined by up to **7 sources**.

---

## 3. Permission System: 24 Files of Trust Decisions

The permission system is the most complex subsystem measured by interconnection density. It occupies 24 files across `utils/permissions/` and touches every tool invocation.

### The Decision Flow

Every tool call passes through this chain:

```
Tool invocation
  → Check deny rules (7 sources)
  → Check allow rules (7 sources)
  → Check auto-mode classifier (if enabled + bash)
  → Execute pre-tool-use hooks
  → Fall back to interactive prompt
```

### Rule Sources

Rules can come from:

| Source | Persistence | Trust Level |
|--------|------------|-------------|
| `globalSettings` | `~/.claude/settings.json` | User-controlled |
| `projectSettings` | `.claude/settings.json` | Repo-controlled (untrusted) |
| `workspaceSettings` | `.claude/settings.local.json` | Local override |
| `enterpriseSettings` | Remote managed | Org policy |
| `cliArg` | Command-line flags | Session-only |
| `command` | `/permissions` command | Session-only |
| `session` | Runtime additions | Ephemeral |

### Dangerous Pattern Stripping

When entering auto-mode (where the classifier can approve bash commands without user interaction), the system **strips** overly-broad allow rules:

```typescript
export const DANGEROUS_BASH_PATTERNS = [
  'python', 'python3', 'node', 'deno', 'tsx', 'ruby', 'perl',
  'npx', 'bunx', 'bash', 'sh', 'ssh', 'eval', 'exec', 'sudo', ...
]
```

If your settings say "allow all bash commands matching `python*`", that rule is **removed** when auto-mode activates. The system trusts the classifier to make individual decisions instead.

### Bash Classification: Two Parallel Paths

The bash classifier has a deprecated path and a new path **running simultaneously**:

- `splitCommand_DEPRECATED()` — regex-based command splitting, still used in 8 security validators
- `bashParser.ts` — Pure TypeScript tree-sitter-compatible AST parser with UTF-8 byte offsets

The deprecated version disagrees with the new parser on `\r` (carriage return) handling. Both are active because removing the old one would change security behavior.

### The Tree-Sitter Parser Itself

A 50ms-timeout, 50K-node-budget AST parser written in pure TypeScript (no WASM). It produces byte offsets, not character offsets — a subtle but critical distinction when a `ñ` takes 2 bytes but 1 JS character. Getting this wrong means a malicious command could be parsed correctly by the validator but executed differently by the shell.

---

## 4. The Query Engine

`QueryEngine.ts` is the conversation loop. It's an async generator that yields SDK messages:

```typescript
async *submitMessage(): AsyncGenerator<SDKMessage> {
  // 1. Build system prompt from fetchSystemPromptParts()
  // 2. Wrap canUseTool for permission tracking
  // 3. Get model config and app state
  // 4. Parse prompt into cache-boundary blocks
  // 5. Process user input (extract tool calls, format)
  // 6. Stream API response via queryModelWithStreaming()
  // 7. Execute tools via StreamingToolExecutor
  // 8. Yield messages to caller
}
```

### What Makes It Complex

**State that survives across turns:**

- `mutableMessages[]` — the full conversation (mutated in place)
- `permissionDenials[]` — tools the user rejected
- `readFileState` — cache of files already read
- `discoveredSkillNames` — skills triggered this turn
- `loadedNestedMemoryPaths` — which CLAUDE.md files have been injected

**The config surface area:**

```typescript
QueryEngineConfig = {
  cwd, tools, commands, mcpClients, agents,
  canUseTool, getAppState, setAppState,
  initialMessages?, readFileCache,
  customSystemPrompt?, appendSystemPrompt?,
  userSpecifiedModel?, fallbackModel?,
  thinkingConfig?, maxTurns?, maxBudgetUsd?, taskBudget?,
  jsonSchema?, verbose?, replayUserMessages?,
  snipReplay?
}
```

20+ configuration knobs, each changing execution behavior. Some of these affect the API request (model, thinking), some affect local behavior (max turns, budget), some affect prompt construction (custom/append system prompt).

**Tool execution is interleaved with streaming:** As the model streams its response, tool calls are identified and executed concurrently. The results are fed back into the next API call. This means error handling must account for partial tool execution — what happens when 2 of 3 tool calls succeed before the stream is interrupted?

---

## 5. Multi-Agent Orchestration

Three distinct multi-agent systems coexist:

### Coordinator Mode

A single coordinator agent orchestrates workers via `AgentTool` (spawn) and `SendMessageTool` (continue):

- Workers can't see the coordinator's conversation — every prompt must be self-contained
- Workers get a subset of tools (configurable: simple vs full mode)
- A scratchpad provides durable cross-worker key-value storage

The coordinator's system prompt is itself ~200 lines of detailed instructions on when to parallelize, when to delegate, and when to synthesize.

### Swarm Mode

Peer-to-peer teaming with `AsyncLocalStorage` context isolation:

- Permission proxying: worker asks leader's UI for approval, or uses async mailbox
- Bash classifier auto-approval for safe commands (no user interaction)
- Each teammate gets its own permission context via in-process runner
- Reconnection handling for dropped teammates

### KAIROS (Proactive/Assistant)

The background agent mode with:

- Cron scheduling with distributed locking (PID liveness probes)
- Bridge sessions for remote execution
- Session history pagination from server API
- GitHub webhook subscriptions for PR monitoring

**All three can theoretically be active simultaneously.** A KAIROS session could spawn a coordinator, which spawns swarm workers, which execute tools that trigger hooks. The interaction matrix is enormous.

---

## 6. Memory: Four Overlapping Systems

### 1. memdir (Memory Directory)

8 files implementing a structured memory index:

- Four types: user, feedback, project, reference
- `MEMORY.md` index loaded every turn (~200 lines, 25KB cap)
- Frontmatter-based entries with search
- Explicit exclusions for code patterns (those belong in source control)

### 2. SessionMemory

Auto-extraction triggered by threshold:

- Dual thresholds: (tokens + tool calls) OR (tokens + no tools in last turn)
- Runs as forked subagent (doesn't block main conversation)
- Files written with `0o600` permissions (user-only)
- Sequential: one extraction at a time

### 3. autoDream (Background Consolidation)

A 3-gate system:

```
Time gate (has enough time passed since last dream?)
  → Session gate (enough sessions since last dream?)
    → Lock gate (no other process dreaming?)
      → Fork subprocess running /dream command
```

- Cheapest checks first (time is a file stat, sessions is a counter, lock is filesystem)
- On failure, rewinds lock mtime so the next session retries
- Bash tool restricted to read-only during dreams (ls, grep, cat)

### 4. extractMemories (Turn-End Extraction)

Processes the last assistant turn for memory-worthy content. Feeds into memdir.

**The complexity**: All four systems write to overlapping storage. A memory could be created by extractMemories, consolidated by autoDream, served from memdir, and session-scoped by SessionMemory. Deduplication and precedence aren't trivially obvious from the code.

---

## 7. Message Compaction

When the conversation gets too long, the system compresses it:

### The Flow

```
Pre-compact hooks
  → Strip images
  → Group messages by API round
  → Send to model for summarization
  → Parse summary
  → Restore top 5 files (5K tokens each)
  → Restore triggered skills (5K each, 25K total budget)
  → Append session metadata
  → Post-compact hooks
```

### Budget Constants

```typescript
POST_COMPACT_MAX_FILES_TO_RESTORE = 5
POST_COMPACT_TOKEN_BUDGET = 50_000
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

### Why It's Complex

Compaction is **lossy**. The model summarizes, losing detail. But the system tries to be smart about what to preserve — recent file contents and triggered skills get re-injected after compaction. This means the post-compaction state has a **different shape** than the pre-compaction state: summarized conversation + fresh file snapshots + skill definitions. The model must work with this hybrid context.

The failure mode (3 consecutive failures) triggers a circuit breaker — discovered after BigQuery analysis showed 250,000 wasted API calls per day from infinite compaction retry loops.

---

## 8. The Streaming API Layer

### Provider Abstraction

Four API providers behind a single interface:

| Provider | Auth | Region Logic |
|----------|------|-------------|
| Direct (Anthropic) | API key or OAuth | N/A |
| AWS Bedrock | AWS credentials | Model-specific region overrides |
| Vertex AI | Google Cloud auth | `VERTEX_REGION_CLAUDE_3_5_HAIKU`, etc. |
| Azure Foundry | Resource name or `DefaultAzureCredential` | Foundry-specific |

Each provider has different header requirements, credential flows, and error responses. The client layer normalizes these into a single streaming interface.

### Retry Logic with Circuit Breaker

```
429 (rate limit) → retry with backoff
529 (overload) → retry up to 3 times (foreground only)
529 (background) → bail immediately (prevent cascade amplification)
5xx → retry (transient)
4xx → fail (permanent)
```

The foreground vs background distinction matters: a cron job or background task that retries on overload makes the overload worse. Only the main thread and SDK calls get retry patience.

### Anti-Distillation

Two-pronged defense:

1. **Fake tool injection**: The API request includes `anti_distillation: ['fake_tools']` — the server injects fake tool definitions that produce distinguishable outputs if a distilled model tries to use them
2. **Connector text summarization**: Assistant text between tool calls is summarized server-side with a signature, reducing training signal while preserving functionality

### Attestation

The attribution header contains a `cch=00000` placeholder that Bun's native HTTP stack **overwrites at the transport layer** with a real attestation token. This means:

- The application layer never sees the real token
- The token can't be extracted from source code
- Bun's HTTP implementation is doing security-relevant post-processing

---

## 9. Startup Performance

The main entry point (`main.tsx`) is optimized for perceived startup speed:

```typescript
// These run in PARALLEL during module loading:
profileCheckpoint('main_tsx_entry')      // Profiler starts
readMDMRaw()                             // MDM subprocess (enterprise)
prefetchKeychain()                       // macOS keychain I/O (~65ms)
```

Circular dependencies are broken with lazy `require()`:

```typescript
const getTeammateModule = () => require('./teammate')
const getCoordinatorMode = () => require('./coordinator/coordinatorMode')
```

Feature flags gate entire module trees — if `feature('KAIROS')` is false, the entire assistant module, its dependencies, and their transitive imports are never loaded.

---

## 10. State Management

The bootstrap singleton (`bootstrap/state.ts`) is 1,300+ lines managing:

### Session Context
- `cwd`, `projectRoot` — filesystem position
- `sessionId`, `parentSessionId` — session lineage (plan → implementation)
- `costs`, `duration` — billing/telemetry

### Telemetry (10+ OpenTelemetry meters)
```typescript
meter, sessionCounter, locCounter, prCounter,
commitCounter, costCounter, tokenCounter,
codeEditToolDecisionCounter, activeTimeCounter
```

### Runtime State
- `agentColorMap` — persistent color assignment per agent
- `cachedClaudeMdContent` — breaks import cycle in classifier
- `planSlugCache` — maps session IDs to human slugs
- `teleportedSessionInfo` — cross-session reliability logging
- `invokedSkills` — skill cache surviving compaction
- `sessionCronTasks` — non-durable scheduled tasks

This singleton is **imported everywhere**. It's the god object of the application — convenient but making dependency analysis nearly impossible.

---

## 11. Cross-Cutting Complexity Multipliers

### Hook System (17 files)

Every tool invocation can trigger:
- Pre-tool-use hooks (can block execution)
- Post-tool-use hooks (can modify results)
- Post-sampling hooks (can modify model output)
- User prompt submit hooks (can modify user input)

Each hook can be:
- File-based (shell script, Python)
- HTTP-based (external webhook)
- Agent-based (run through model)

Hooks interact with permissions, feature flags, and the streaming pipeline.

### MCP Integration (23 files, 6 transports)

Each MCP server adds dynamic tools to the tool pool. These tools go through the same permission system, appear in the coordinator prompt, and participate in compaction. A single MCP server can change the behavior of the entire system.

### Content Replacement

File edit operations interact with:
- File history snapshots (100 max, circular buffer)
- Diff calculation (npm `diff` package)
- Token counting (for budget tracking)
- Permission rules (write access to paths)
- Sandbox adapter (path pattern validation)

---

## 12. Complexity Metrics Summary

| Subsystem | Files | Interaction Points | Key Complexity Driver |
|-----------|-------|-------------------|----------------------|
| Permissions | 24 | Every tool call | 7 rule sources × 5 check layers |
| Memory | 20+ | Every turn | 4 overlapping systems, dedup unclear |
| Feature flags | Everywhere | Every conditional path | 7 override sources, build+runtime layers |
| API streaming | 10+ | Every model call | 4 providers × retry logic × anti-distillation |
| Multi-agent | 30+ | Agent spawning | 3 modes × permission proxying × context isolation |
| Compaction | 11 | Context overflow | Lossy compression with selective restoration |
| Hook system | 17 | Tool + sampling boundaries | 4 hook types × 3 execution modes |
| State | 1 file (1,300 lines) | Everywhere | God object, imported by everything |
| MCP | 23 | Tool pool + permissions | 6 transports, dynamic tool injection |
| Bash analysis | 12 | Security boundary | Dual parsers, byte vs char offsets |

**The fundamental complexity is not in any single subsystem. It's in the interaction space between them. A model response triggers tool execution, which checks permissions (7 sources), runs hooks (4 types), may spawn agents (3 modes), consumes memory (4 systems), and tracks state (1,300-line singleton) — all through a streaming pipeline with circuit breakers and provider-specific error handling.**

---

*Next: [02-UNIQUE-ELEMENTS.md](02-UNIQUE-ELEMENTS.md) — What makes this codebase unlike anything else*
