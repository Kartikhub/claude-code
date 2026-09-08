# Architecture: Claude Code from Source

> **Scope**: A complete architecture analysis derived purely from reading the source code of the leaked Claude Code CLI. No external documentation or prior analysis influenced this document.

---

## Table of Contents

1. [High-Level Architecture](#1-high-level-architecture)
2. [Layer 1: CLI Shell & Terminal UI](#2-layer-1-cli-shell--terminal-ui)
3. [Layer 2: Conversation Engine](#3-layer-2-conversation-engine)
4. [Layer 3: Tool System](#4-layer-3-tool-system)
5. [Layer 4: Permission & Security](#5-layer-4-permission--security)
6. [Layer 5: API & Provider Abstraction](#6-layer-5-api--provider-abstraction)
7. [Layer 6: Memory & Context Management](#7-layer-6-memory--context-management)
8. [Layer 7: Orchestration (Multi-Agent)](#8-layer-7-orchestration)
9. [Layer 8: Extension Ecosystem](#9-layer-8-extension-ecosystem)
10. [Layer 9: Services & Infrastructure](#10-layer-9-services--infrastructure)
11. [Cross-Cutting Concerns](#11-cross-cutting-concerns)
12. [Data Flow: A Complete Request](#12-data-flow)
13. [Build & Runtime Architecture](#13-build--runtime-architecture)
14. [Component Dependency Map](#14-component-dependency-map)

---

## 1. High-Level Architecture

Claude Code is a **terminal-native AI coding agent** built as a TypeScript application running on the Bun runtime. It uses React + Ink for terminal UI rendering, Commander.js for CLI parsing, and communicates with Claude models through a multi-provider API layer.

### The 9 Layers

```
┌─────────────────────────────────────────────────────┐
│  Layer 1: CLI Shell & Terminal UI                    │
│  (main.tsx, App.tsx, components/, React+Ink)         │
├─────────────────────────────────────────────────────┤
│  Layer 2: Conversation Engine                        │
│  (QueryEngine.ts, messages, streaming)               │
├─────────────────────────────────────────────────────┤
│  Layer 3: Tool System                                │
│  (tools/, Tool.ts, tools.ts, StreamingToolExecutor)  │
├─────────────────────────────────────────────────────┤
│  Layer 4: Permission & Security                      │
│  (permissions/, bash/, sandbox/, hooks/ssrfGuard)    │
├─────────────────────────────────────────────────────┤
│  Layer 5: API & Provider Abstraction                 │
│  (services/api/, client.ts, withRetry.ts)            │
├─────────────────────────────────────────────────────┤
│  Layer 6: Memory & Context Management                │
│  (memdir/, SessionMemory/, autoDream/, compact/)     │
├─────────────────────────────────────────────────────┤
│  Layer 7: Orchestration                              │
│  (coordinator/, swarm/, bridge/, assistant/)         │
├─────────────────────────────────────────────────────┤
│  Layer 8: Extension Ecosystem                        │
│  (commands/, plugins/, skills/, mcp/, hooks/)        │
├─────────────────────────────────────────────────────┤
│  Layer 9: Services & Infrastructure                  │
│  (analytics/, policyLimits/, remoteManagedSettings/) │
└─────────────────────────────────────────────────────┘
```

### Key Directories (from `src/`)

| Directory | Files | Purpose |
|-----------|-------|---------|
| `tools/` | 41 directories | Tool implementations |
| `commands/` | 80+ directories | Slash commands |
| `services/` | 15+ subdirectories | Backend services |
| `utils/` | 20+ subdirectories | Shared utilities |
| `components/` | 60+ files | React terminal components |
| `hooks/` | 80+ files | React hooks (not to be confused with user hooks) |
| `coordinator/` | Multi-file | Coordinator mode system |
| `bridge/` | 25+ files | Remote session bridge |
| `memdir/` | 8 files | Memory directory |
| `buddy/` | 5 files | Companion system |
| `bootstrap/` | 2 files | Startup state |
| `constants/` | 18 files | Configuration constants |
| `assistant/` | 1 file | KAIROS assistant |

---

## 2. Layer 1: CLI Shell & Terminal UI

### Entry Point (`main.tsx`)

The application initializes in a performance-critical startup sequence:

```
┌── Profiler checkpoint (start timing)
├── MDM raw read (subprocess worker, parallel)
├── Keychain prefetch (macOS ~65ms, parallel)
├── Feature flag initialization
├── Dead code elimination gates
│   ├── feature('KAIROS') → require('./commands/assistant')
│   ├── feature('COORDINATOR_MODE') → require('./coordinator')
│   └── feature('TRANSCRIPT_CLASSIFIER') → require('./classifier')
├── Settings/policy loading (async with promise gate)
└── CLI argument parsing (Commander.js)
```

### React + Ink Rendering

The terminal UI is a React application using Ink (React renderer for terminals):

```
<FpsMetricsProvider>       ← Terminal frame rate tracking
  <StatsProvider>          ← Session metrics (LOC, cost)
    <AppStateProvider>     ← Central reactive state
      <Messages/>          ← Polymorphic message renderer
      <InputArea/>         ← User input with vim/normal modes
      <StatusBar/>         ← Model, cost, tool state
    </AppStateProvider>
  </StatsProvider>
</FpsMetricsProvider>
```

**Key components (60+ total):**

- `Message.tsx` — Polymorphic: user, assistant, system, tool-result, thinking, advisor
- `StructuredDiff.tsx` — Unified diff with syntax highlighting and line ranges
- `OffscreenFreeze` — Prevents re-renders of scrolled-off content
- `CollapsedReadSearchContent` — Compresses search results

### Component Architecture

Components are **heavily memoized** with the React compiler (visible in source maps). The `FpsMetricsProvider` exists specifically because terminal rendering has visible jank at high update rates (streaming tokens + tool results simultaneously).

---

## 3. Layer 2: Conversation Engine

### QueryEngine.ts

The core conversation loop — an async generator yielding SDK messages:

```typescript
class QueryEngine {
  private mutableMessages: Message[]      // The conversation
  private readFileState: FileStateCache   // File content dedup
  private discoveredSkillNames: Set       // Skills this turn
  private permissionDenials: ToolDenial[] // Trust history
  private loadedNestedMemoryPaths: Set    // Memory injection tracking
  
  async *submitMessage(): AsyncGenerator<SDKMessage> {
    // 1. Assemble system prompt (10+ dynamic sections)
    // 2. Wrap canUseTool for permission tracking
    // 3. Resolve model config (14 variants × 4 providers)
    // 4. Split prompt at cache boundary
    // 5. Process user input (extract tool calls)
    // 6. Stream API response
    // 7. Execute tools via StreamingToolExecutor
    // 8. Yield messages incrementally
  }
}
```

### Configuration Surface

```typescript
QueryEngineConfig = {
  // Required
  cwd, tools, commands, mcpClients, agents,
  canUseTool, getAppState, setAppState,
  
  // Optional (20+ knobs)
  initialMessages?, readFileCache,
  customSystemPrompt?, appendSystemPrompt?,
  userSpecifiedModel?, fallbackModel?,
  thinkingConfig?, maxTurns?, maxBudgetUsd?, taskBudget?,
  jsonSchema?, verbose?, replayUserMessages?, snipReplay?
}
```

### System Prompt Assembly

```
┌── getSimpleIntroSection()          Role identity + cyber-risk
├── getSimpleSystemSection()         Tool rules, compression, injection warnings
├── getSimpleDoingTasksSection()     Code style, testing, faithfulness
├── getLanguageSection()             Localization
├── getOutputStyleSection()          Custom formatting
├── getMcpInstructionsSection()      MCP server descriptions (truncated 2048 chars)
├── getAntModelOverrideSection()     Ant-only overrides
├── Hooks section                    Hook feedback handling
├── ═══════════════════════════      CACHE BOUNDARY (1h TTL above)
├── Memory context                   CLAUDE.md injection (25KB cap)
├── Coordinator context              If coordinator mode
├── Proactive context                If KAIROS mode
└── System reminders                 Injection markers
```

The `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` marker separates the cacheable static section from the per-turn dynamic section. API prompt caching is applied to everything above the boundary.

---

## 4. Layer 3: Tool System

### Tool Interface

```typescript
interface Tool {
  name: string
  aliases?: string[]
  description(input, opts): string
  userFacingName(input): string
  call(input, context: ToolUseContext): Promise<ToolResult>
  schema: JSONSchema
}
```

### Tool Registry (`tools.ts`)

The tool pool is assembled dynamically:

```
Built-in tools (always available)
  │ Core: AgentTool, BashTool, FileEditTool, FileReadTool, FileWriteTool
  │ Search: GlobTool, GrepTool, WebSearchTool
  │ Code: NotebookEditTool, LSPTool, REPLTool (ant-only)
  │ Tasks: TaskCreateTool, TaskUpdateTool, TaskStopTool, TaskOutputTool
  │ System: ConfigTool, PlanMode, WorktreeMode
  │
  ├── Feature-gated tools
  │    SleepTool (KAIROS), CronTools (AGENT_TRIGGERS)
  │    MonitorTool (KAIROS), WebBrowserTool (feature-gated)
  │    WorkflowTool (WORKFLOW_SCRIPTS)
  │
  ├── MCP tools (dynamic, per-server)
  │    ListMcpResourcesTool, ReadMcpResourceTool
  │    + tools exposed by connected MCP servers
  │
  └── Filter by permissions (deny rules remove tools from pool)
```

### ToolUseContext (30+ fields)

Every tool receives a rich context object:

```typescript
ToolUseContext = {
  // Configuration
  options: { commands, tools, mcpClients, agentDefinitions, thinkingConfig, ... }
  
  // Control
  abortController: AbortController
  
  // State
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f): void
  messages: Message[]
  
  // Integration
  handleElicitation?: (serverName, params) => Promise<ElicitResult>
  setToolJSX?: SetToolJSXFn
  addNotification?: (Notification) => void
  appendSystemMessage?: (SystemMessage) => void
  requestPrompt?: (sourceName) => PromptCallback
  
  // Limits
  fileReadingLimits?: { maxTokens?, maxSizeBytes? }
  globLimits?: { maxResults? }
  queryTracking?: QueryChainTracking
}
```

### 41 Tool Directories

| Category | Tools |
|----------|-------|
| File I/O | FileReadTool, FileWriteTool, FileEditTool, GlobTool, GrepTool |
| Execution | BashTool, PowerShellTool, REPLTool |
| Agent | AgentTool, SendMessageTool, SkillTool |
| Task | TaskCreateTool, TaskUpdateTool, TaskStopTool, TaskOutputTool, TaskGetTool, TaskListTool |
| Web | WebFetchTool, WebSearchTool, WebBrowserTool |
| MCP | MCPTool, ListMcpResourcesTool, ReadMcpResourceTool, McpAuthTool |
| Mode | EnterPlanModeTool, ExitPlanModeTool, EnterWorktreeTool, ExitWorktreeTool |
| System | ConfigTool, BriefTool, SleepTool, ToolSearchTool |
| Scheduling | ScheduleCronTool |
| Team | TeamCreateTool, TeamDeleteTool |
| Notebook | NotebookEditTool |
| Remote | RemoteTriggerTool |
| Misc | AskUserQuestionTool, SyntheticOutputTool, TodoWriteTool |

---

## 5. Layer 4: Permission & Security

### Architecture (24 files in `utils/permissions/`)

```
Permission Request
  │
  ├─ 1. Deny rules check (7 sources, cumulative)
  │     globalSettings, projectSettings, workspaceSettings
  │     enterpriseSettings, cliArg, command, session
  │     → DENIED (if any match)
  │
  ├─ 2. Allow rules check (7 sources, latest wins)
  │     → ALLOWED (if rule matches)
  │
  ├─ 3. Auto-mode classifier (if enabled)
  │     │ Works on bash commands only
  │     │ Uses tree-sitter AST parser
  │     │ Checks DANGEROUS_BASH_PATTERNS database
  │     │ Strips overly-broad allow rules
  │     └─→ ALLOWED or continue
  │
  ├─ 4. Pre-tool-use hooks
  │     │ File-based, HTTP, or agent hooks
  │     │ 5-second timeout
  │     │ Structured response: { ok, reason? }
  │     └─→ BLOCKED or continue
  │
  └─ 5. Interactive prompt (if human available)
        │ Direct UI in CLI
        │ Bridge proxy in coordinator
        │ Mailbox in swarm
        └─→ ALLOWED or DENIED
```

### Bash Security Pipeline

```
Bash command string
  │
  ├── tree-sitter AST parser (pure TypeScript)
  │     50ms timeout, 50K node budget
  │     UTF-8 byte offsets (not char offsets)
  │     Validated against 3,449-input golden corpus
  │
  ├── splitCommand_DEPRECATED (legacy regex)
  │     Still active in 8 security validators
  │     Disagrees on \r handling
  │
  ├── Dangerous patterns check
  │     python, node, eval, exec, sudo, ssh, ...
  │     Platform-specific (ant gets: curl, wget, git, kubectl)
  │
  ├── Path validation
  │     Sandbox adapter pattern resolution
  │     // = absolute, / = relative to settings dir
  │
  └── Classifier decision
        deny | ask | allow
```

### Sandbox System

The sandbox adapter (`sandbox-adapter.ts`) wraps `@anthropic-ai/sandbox-runtime`:

- Path pattern resolution (absolute vs relative)
- FS and network permission integration
- Maps settings-based restrictions to runtime constraints
- **Not a separate permission system** — adapts the existing rule system

---

## 6. Layer 5: API & Provider Abstraction

### Client Architecture

```
getAnthropicClient()
  │
  ├── Direct API (ANTHROPIC_API_KEY or OAuth)
  │     Headers: x-app, User-Agent, Session-Id
  │     Optional: container-id, client-app, additional-protection
  │     Attestation: cch=00000 overwritten by Bun HTTP
  │
  ├── AWS Bedrock (AWS credentials)
  │     Region resolution per model
  │     ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION for Haiku
  │     
  ├── Vertex AI (Google Cloud)
  │     Per-model region env vars (VERTEX_REGION_CLAUDE_*)
  │     Global fallback: CLOUD_ML_REGION
  │
  └── Azure Foundry
        Resource name or full URL
        DefaultAzureCredential fallback chain
```

### Streaming with Retry

```
queryModelWithStreaming()
  │
  ├── Build request (messages, tools, model, thinking config)
  │
  ├── Cache control (1h TTL on cacheable sections)
  │
  ├── Anti-distillation (if enabled)
  │     fake_tools injection + connector text summarization
  │
  ├── withRetry() wrapper
  │     │ 429 (rate limit) → exponential backoff
  │     │ 529 (overload) → MAX_529_RETRIES = 3 (foreground only)
  │     │ 529 (background) → bail immediately
  │     │ 5xx → retry (transient)
  │     │ 4xx → fail (permanent)
  │     └── Persistent retry mode (ant-only): indefinite with 5min cap
  │
  └── Async generator → yields stream events
```

### Model Configuration

14 model variants across 4 providers:

```
Opus:   opus40, opus41, opus45, opus46
Sonnet: sonnet35, sonnet37, sonnet40, sonnet45, sonnet46
Haiku:  haiku35, haiku45
```

Selection cascade: User flag → Env var → Settings → Subscription tier default

Thinking modes: `adaptive`, `enabled` (with token budget), `disabled`

---

## 7. Layer 6: Memory & Context Management

### Four Memory Systems

```
┌─ memdir/ (Permanent structured memory) ──────────────────┐
│  MEMORY.md index: 200 lines max, 25KB cap                │
│  4 types: user, feedback, project, reference              │
│  Loaded into system prompt every turn                     │
│  findRelevantMemories → search with frontmatter           │
└──────────────────────────────────────────────────────────┘

┌─ SessionMemory/ (Per-session extraction) ─────────────────┐
│  Trigger: token threshold + tool call activity            │
│  Runs as forked subagent (non-blocking)                   │
│  Sequential: one extraction at a time                     │
│  Output: 0o600 permission files                           │
└──────────────────────────────────────────────────────────┘

┌─ autoDream/ (Cross-session consolidation) ────────────────┐
│  3-gate system: time → sessions → lock                    │
│  Forks subprocess running /dream command                  │
│  Bash restricted to read-only during dreams               │
│  Lock rewind on failure (retry on next session)           │
└──────────────────────────────────────────────────────────┘

┌─ extractMemories/ (Turn-end extraction) ──────────────────┐
│  Processes last assistant turn for memory content          │
│  Feeds into memdir                                        │
└──────────────────────────────────────────────────────────┘
```

### Compaction (Context Window Management)

```
Conversation exceeds context budget
  │
  ├── Pre-compact hooks execute
  ├── Strip images from messages
  ├── Group by API round
  ├── Send to model for summarization (lossy)
  ├── Parse summary
  ├── Restore top 5 files (5K tokens each, 50K total)
  ├── Restore triggered skills (5K each, 25K total)
  ├── Append session metadata (timestamp, model, costs)
  ├── Post-compact hooks execute
  │
  └── Result: summary + fresh file snapshots + skill definitions
       (hybrid state, different shape than pre-compaction)
```

**Circuit breaker**: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` — stops retrying after 3 failures (discovered via BigQuery: 250K wasted calls/day).

---

## 8. Layer 7: Orchestration

### Three Agent Modes

```
┌─ Coordinator Mode ────────────────────────────────────────┐
│  Single coordinator orchestrates workers                  │
│  Tools: AgentTool (spawn), SendMessageTool (continue)     │
│  Worker types:                                            │
│    Simple: Bash + Read + Edit + MCP                       │
│    Full: All standard tools + Skills                      │
│  Scratchpad: Durable cross-worker KV store                │
│  Workers can't see coordinator conversation               │
│  ~200 line system prompt for coordinator behavior         │
└──────────────────────────────────────────────────────────┘

┌─ Swarm Mode ──────────────────────────────────────────────┐
│  Peer-to-peer teaming                                     │
│  AsyncLocalStorage for context isolation                  │
│  Permission proxying:                                     │
│    Bridge mode: routes to leader's UI                     │
│    Mailbox mode: async request/response                   │
│  In-process runner (not subprocess)                       │
│  Bash classifier auto-approval for safe commands          │
└──────────────────────────────────────────────────────────┘

┌─ KAIROS (Proactive/Assistant) ────────────────────────────┐
│  Background agent that outlives sessions                  │
│  Cron scheduling with distributed locking                 │
│    Lock: PID liveness probe + owner key                   │
│    Jitter: GrowthBook-tunable for load distribution       │
│  Bridge sessions for remote execution                     │
│  Session history pagination (server API)                  │
│  GitHub webhook subscriptions                             │
│  Feature-gated: build-time KAIROS + runtime flags         │
└──────────────────────────────────────────────────────────┘
```

### Bridge Architecture (25+ files)

```
Remote Session Bridge
  │
  ├── Session pool (32 max, multi-session spawn gating)
  │
  ├── Transport: JWT + WebSocket
  │     Proactive refresh (5min before expiry) for v2
  │     Auth recovery: 401/403 → re-queue, not fatal
  │
  ├── State tracking (8 separate maps)
  │     sessions, timers, worktrees, compatibility
  │     
  ├── Smart backoff
  │     Connection errors: separate budget
  │     General errors: separate budget
  │     
  └── Kill signal classification
        User kill vs timeout kill vs server kill
```

---

## 9. Layer 8: Extension Ecosystem

### Commands (80+ directories)

```typescript
Command = {
  name: string
  type: 'local' | 'local-jsx' | 'prompt' | 'plugin-stub'
  load: () => Promise<CommandModule>
  isEnabled?: () => boolean
  isHidden?: boolean
  availability?: ('cli' | 'claude-ai' | 'agent-sdk')[]
}
```

Notable commands:
- `ultraplan` — Remote multi-agent planning (30min timeout)
- `teleport` — Session environment transfer
- `compact` — Context compaction
- `vim` — Toggle input mode
- `voice` — Voice mode (claude.ai only)
- `chrome` — Claude in Chrome settings
- `stickers` — Physical merchandise ordering

### Plugins

Three scopes: user (`~/.claude/plugins/`), project (`.claude/plugins/`), local (session-only).

Plugin lifecycle: install → enable → load → execute → disable → uninstall. Managed plugins (bundled) can't be manually disabled.

### Skills

Bundled + user-defined capability definitions. Tracked per-invocation (survives compaction). Discovery via `SkillTool`.

### MCP (23 files, 6 transports)

```
MCP Integration
  │
  ├── Transports: Stdio, SSE, WebSocket, HTTP, In-Process, SDK Control
  │
  ├── Protocol: Tools, Resources, Prompts
  │
  ├── Auth: ClaudeAuthProvider
  │     XAA IDP login
  │     Step-up detection
  │     15min TTL for "needs-auth" state
  │     Server approval UI dialog
  │
  ├── Resilience
  │     Session expiration: 404 + JSON-RPC -32001
  │     Description truncation: 2048 char cap
  │     Custom error types: McpAuthError, McpToolCallError
  │
  └── Lazy loading
        Skill rendering, computer-use wrapper, Chrome tools
```

### Hooks (17 files in `utils/hooks/`)

```
Hook Types:
  PreToolUse      → Can block tool execution
  PostToolUse     → Can modify tool results
  PostSampling    → Can modify model output
  UserPromptSubmit → Can block user input

Hook Execution Modes:
  File       → Shell script with $ARGUMENTS, $0, $1
  HTTP       → External webhook (SSRF-guarded)
  Agent      → Execute through model

Response Schema: { ok: boolean, reason?: string }
Timeout: 5 seconds
```

---

## 10. Layer 9: Services & Infrastructure

### Analytics

```
Event System
  │
  ├── Deferred sink attachment (queue until sink connects)
  ├── Type-safe metadata (marker types)
  ├── Proto field stripping (_PROTO_* removed before Datadog)
  │
  ├── GrowthBook (feature flags + experiments)
  │     Cached reads (_CACHED_MAY_BE_STALE)
  │     Override: env vars → local config → remote → disk cache → defaults
  │     Auth-aware re-initialization
  │     Experiment dedup (once per session)
  │
  └── OpenTelemetry meters (10+)
        sessions, LOC, PRs, commits, cost, tokens, 
        codeEditDecisions, activeTime
```

### Policy Limits

- Organization-level feature restrictions (Team/Enterprise)
- Fail-open: network errors don't block
- ETag-based cache with 60-minute polling
- Memory-only (no disk persistence)

### Remote Managed Settings

- Enterprise configuration (policy, defaults, custom prompts)
- Checksum validation (UUID + checksum)
- Disk-cached (survives restart)
- Background refresh (60-minute poll)
- Security check before applying

### Cron System

```
Cron Scheduler
  │
  ├── 1-second check interval
  ├── Standard 5-field cron expressions
  ├── Lock-based multi-session coordination
  │     PID liveness probe + owner key
  │     Jitter for load distribution
  │
  ├── Task types:
  │     File-backed (durable, persisted to disk)
  │     Session-only (dies with process)
  │     Permanent (exempt from auto-expiry)
  │
  └── KAIROS gate: tengu_kairos_cron toggles mid-session
```

---

## 11. Cross-Cutting Concerns

### State Singleton (`bootstrap/state.ts`, 1,300+ lines)

The central state object. Every subsystem imports it.

Key state groups:
- **Session identity**: sessionId, parentSessionId, cwd, projectRoot
- **Costs & telemetry**: costs, duration, meter, counters
- **Model**: modelUsage, mainLoopModelOverride
- **Cache**: cachedClaudeMdContent, planSlugCache, readFileState  
- **Runtime**: agentColorMap, invokedSkills, sessionCronTasks
- **Debug**: lastAPIRequest, lastAPIRequestMessages

### AbortController Hierarchy

```
Session AbortController (root)
  └── Child (per tool call, WeakRef to parent)
       └── Grandchild (per subprocess, WeakRef)
```

- Max 50 listeners per controller
- Child abort doesn't propagate to parent
- WeakRef prevents GC leaks
- Fast-path for pre-aborted parents

### File History

```
FileHistoryState
  │
  ├── snapshots[] (100 max, circular buffer)
  │     Per snapshot: files, timestamp, messageId
  │
  ├── trackedFiles: Set<string>
  │
  ├── snapshotSequence: number (activity signal)
  │
  └── DiffStats via npm `diff` package
```

### Token Tracking

```
Token calculation:
  input_tokens + cache_creation_input_tokens + 
  cache_read_input_tokens + output_tokens

Budget formula (excludes cache tokens):
  Matches server-side billing computation

Feeds into:
  maxBudgetUsd, taskBudget, compaction triggers, telemetry
```

---

## 12. Data Flow: A Complete Request

Here's what happens when a user types a message:

```
1. User types in terminal
     ↓
2. React UI captures input (Ink)
     ↓
3. UserPromptSubmit hooks execute (may block)
     ↓
4. QueryEngine.submitMessage() begins
     ↓
5. System prompt assembled (10+ sections, cache boundary)
     ↓
6. Model selected (14 variants × 4 providers)
     ↓
7. queryModelWithStreaming() called
     │
     ├── Anti-distillation injected (if enabled)
     ├── Cache control headers set
     ├── withRetry wraps the call
     └── Async generator begins yielding
           ↓
8. Stream events processed
     │
     ├── Text chunks → rendered in terminal
     ├── Thinking blocks → displayed with timing
     └── Tool calls → identified mid-stream
           ↓
9. StreamingToolExecutor handles tool calls
     │
     ├── Tool looked up in pool (built-in + MCP)
     ├── Permission check (7 sources × 5 layers)
     ├── Pre-tool-use hooks execute
     ├── AbortController created (child of session)
     ├── Tool.call() executes
     │     ├── File reads → readFileState cache
     │     ├── Bash → AST parser → sandbox → execute
     │     ├── MCP → transport → server → response
     │     └── Agent → spawn new QueryEngine
     ├── Post-tool-use hooks execute
     ├── File history snapshot (if write)
     └── Token/cost tracking updated
           ↓
10. Tool results fed back for next model turn
      ↓
11. Repeat 7-10 until model stops emitting tool calls
      ↓
12. Final assistant message yielded
      ↓
13. Post-sampling hooks execute
      ↓
14. extractMemories runs (if threshold met)
      ↓
15. SessionMemory extraction (if triggered)
      ↓
16. Telemetry emitted
      ↓
17. UI renders final state
```

---

## 13. Build & Runtime Architecture

### Bun Runtime

- **Bundler**: `bun:bundle` for dead code elimination via `feature()` gates
- **HTTP stack**: Transport-layer attestation (cch token)
- **Module system**: Standard ESM with lazy `require()` for circular dependency breaking
- **Performance**: Startup profiling, parallel initialization

### Build Variants

| Variant | Feature Flags | Target |
|---------|--------------|--------|
| External (CLI) | KAIROS=false, COORDINATOR=false, ANTI_DISTILL=false | Public release |
| Internal (Ant) | KAIROS=true, COORDINATOR=true, ANTI_DISTILL=true | Anthropic employees |
| Agent SDK | Subset of tools, no UI | Programmatic access |

### Runtime Feature Matrix

```
Build-time (absent from binary):
  feature('KAIROS')
  feature('COORDINATOR_MODE')
  feature('ANTI_DISTILLATION_CC')
  feature('TRANSCRIPT_CLASSIFIER')
  feature('ULTRATHINK')

Runtime (GrowthBook, togglable):
  tengu_anti_distill_fake_tool_injection
  tengu_thinkback
  tengu_kairos_cron
  tengu_turtle_carbon (thinking)
  tengu_scratch (scratchpad)
  + 15 more feature values
```

---

## 14. Component Dependency Map

### High-Traffic Dependencies

```
bootstrap/state.ts ←── (imported by nearly everything)
   │
   ├── QueryEngine.ts ←── hooks/useQueryEngine.tsx
   │     │
   │     ├── services/api/claude.ts (streaming)
   │     ├── tools/ (execution)
   │     ├── utils/permissions/ (security)
   │     ├── constants/prompts.ts (system prompt)
   │     └── services/compact/ (context management)
   │
   ├── coordinator/coordinatorMode.ts
   │     │
   │     ├── tools/AgentTool (worker spawning)
   │     ├── tools/SendMessageTool (worker continuation)
   │     └── bootstrap/state.ts (session tracking)
   │
   ├── services/mcp/client.ts
   │     │
   │     ├── 6 transport implementations
   │     ├── services/analytics/growthbook.ts
   │     └── utils/hooks/ssrfGuard.ts
   │
   └── bridge/bridgeMain.ts
         │
         ├── services/api/client.ts (auth)
         ├── bootstrap/state.ts (session state)
         └── WebSocket transport
```

### Circular Dependency Breakers

```
main.tsx:
  const getTeammateModule = () => require('./teammate')     // Lazy
  const getCoordinatorMode = () => require('./coordinator')  // Lazy

bootstrap/state.ts:
  cachedClaudeMdContent: breaks cycle with classifier

hooks/useCanUseTool.tsx:
  Imports permissions lazily to break UI ↔ security cycle
```

### The Critical Path

For a simple file-read tool call:

```
User input → QueryEngine → API stream → Tool identification → 
Permission check → FileReadTool.call() → Result → API stream (next turn)
```

Minimum files touched: `main.tsx`, `QueryEngine.ts`, `claude.ts`, `withRetry.ts`, `permissions.ts`, `tools.ts`, `FileReadTool/`, `bootstrap/state.ts`, `prompts.ts`

That's **9 files minimum** for the simplest possible tool call. Complex operations (MCP + coordinator + hooks) can touch 30+.

---

*Next: [06-DEVELOPER-EXPERIENCE.md](06-DEVELOPER-EXPERIENCE.md) — Patterns in developer experience and engineering culture*
