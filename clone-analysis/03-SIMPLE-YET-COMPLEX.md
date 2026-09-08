# Simple Yet Complex: Deceptive Simplicity in Claude Code

> **Scope**: Subsystems that appear straightforward but hide significant depth — the iceberg pattern where 10% is visible and 90% is engineering under the surface.

---

## Table of Contents

1. [Tool Calls: "Just Run the Function"](#1-tool-calls)
2. [System Prompt: "Just a String"](#2-system-prompt)
3. [File Reading: "Just Read a File"](#3-file-reading)
4. [Model Selection: "Just Pick a Model"](#4-model-selection)
5. [Caching: "Just Memoize It"](#5-caching)
6. [The Abort Controller: "Just Cancel It"](#6-abort-controller)
7. [Config Loading: "Just Read settings.json"](#7-config-loading)
8. [Command Registration: "Just Add a Command"](#8-command-registration)
9. [Token Counting: "Just Count Tokens"](#9-token-counting)
10. [Session ID: "Just a UUID"](#10-session-id)
11. [The Spinner: "Just Show Progress"](#11-the-spinner)
12. [Slash Commands: "Just Parse User Input"](#12-slash-commands)

---

## 1. Tool Calls

### What It Looks Like

The model wants to read a file. It emits a tool call. The tool runs. Results go back.

### What Actually Happens

```
Model emits tool_use block in stream
  → StreamingToolExecutor identifies the tool call mid-stream
  → Tool lookup in the assembled pool (built-in + MCP + filtered by permissions)
  → Permission check (7 rule sources × 5 layers)
    → If bash: run through tree-sitter parser + deprecated regex parser (both)
    → If bash: check dangerous patterns database
    → If auto-mode: run classifier (or always-allow for safe categories)
  → Pre-tool-use hooks execute (can block, modify, or log)
    → Hooks can be: shell scripts, HTTP endpoints, or agent-based
  → AbortController created (child of session controller, WeakRef parent link)
  → Tool execution with ToolUseContext (30+ fields)
    → File reads track state for dedup across turns
    → Bash commands go through sandbox adapter
    → MCP tools route through 1 of 6 transport protocols
  → Result captured
    → File history snapshot if write operation
    → Diff stats computed (insertions/deletions)
    → Token count updated
  → Post-tool-use hooks execute (can modify result)
  → Result streamed back to model
  → Telemetry logged (tool name, duration, result type)
  → Skill invocation tracked (survives compaction)
  → Permission denial recorded if applicable
```

A single tool call touches: permissions, hooks, abort controllers, file history, token counting, telemetry, the sandbox, and potentially 6 different transport protocols. The API surface says "call a function." The reality is a 15-step pipeline.

---

## 2. System Prompt

### What It Looks Like

A string that tells the model what it is.

### What Actually Happens

The system prompt is **assembled from 10+ dynamic sections** with a priority system:

```
Priority 1: Override (full replacement, used by loop mode)
Priority 2: Coordinator mode (if COORDINATOR_MODE + env flag)
Priority 3: Agent definition (proactive: append; normal: replace)
Priority 4: Custom (--system-prompt flag)
Priority 5: Default (standard Claude Code prompt)
Priority 6: Append (always added at end)
```

Within the default prompt, these sections are concatenated:

| Section | Content |
|---------|---------|
| `getSimpleIntroSection()` | Role identity + cyber-risk instruction |
| `getSimpleSystemSection()` | Tool execution rules, compression, prompt injection warnings |
| `getSimpleDoingTasksSection()` | Code style, testing, security, faithfulness |
| `getLanguageSection()` | Localization |
| `getOutputStyleSection()` | Custom output formatting |
| `getMcpInstructionsSection()` | MCP server capabilities |
| `getAntModelOverrideSection()` | Ant-only configuration |
| Hooks section | Hook execution feedback handling |
| Memory context | CLAUDE.md files (loaded, deduped, cached) |
| System reminders | Compression markers, injection warnings |

The prompt has a **cache boundary marker** (`__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`) that separates static (cacheable) content from dynamic (user-specific) content. Everything above the boundary can be cached for 1 hour across requests. Everything below changes per turn.

The model name (`Claude Opus 4.6`) is embedded. The available tools list is embedded. The MCP server descriptions are embedded (truncated at 2,048 chars per server). The coordinator instructions are embedded (if applicable). Memory files are embedded.

"Just a string" turns out to be 10+ functions, 6 priority levels, a cache boundary, and dynamic injection from multiple sources.

---

## 3. File Reading

### What It Looks Like

```typescript
tool: "FileReadTool"
input: { path: "/src/main.ts" }
```

### What Actually Happens

**Before the read:**
- Path validated against sandbox adapter (is this path allowed?)
- Permission rules checked (7 sources: can we read from this directory?)
- `readFileState` consulted — has this file been read this session? If so, is the cached version still valid?
- File size checked against `fileReadingLimits.maxSizeBytes`

**During the read:**
- Content loaded with encoding detection
- Token count estimated against `fileReadingLimits.maxTokens`
- If file exceeds token limit: truncation with marker
- Binary file detection (returns metadata, not content)
- Image detection → base64 encoding for multimodal

**After the read:**
- `readFileState` updated with content hash and timestamp
- File added to tracked files set (for file history snapshots)
- If the file is `CLAUDE.md` (nested memory): extracted, deduplicated against `loadedNestedMemoryPaths`, and injected into system prompt for next turn
- Token usage incremented
- Telemetry emitted

**The dedup layer:**
If the model asks to read the same file twice in one session, the second read can return the cached version with a "file unchanged" marker — saving tokens and API cost. But only if the file hasn't been modified by a tool call between reads. This requires **tracking every write operation** in the session.

---

## 4. Model Selection

### What It Looks Like

```
claude-code --model opus
```

### What Actually Happens

The model selection cascade:

```
User-specified model (--model flag)
  → Environment variable (ANTHROPIC_MODEL)
  → Settings file (model field)
  → Subscription tier default:
    → Max/Team Premium → Opus 4.6 (with 1M suffix option)
    → Pro/Team Standard → Sonnet 4.6
    → PAYG/Enterprise → Sonnet 4.6
    → Ant → Configurable, default Opus 1M
```

But the model ID isn't just a string — it's resolved per provider:

| Provider | Resolution |
|----------|-----------|
| Direct API | `claude-opus-4-6` |
| AWS Bedrock | Full ARN with region-specific endpoints |
| Vertex AI | `VERTEX_REGION_CLAUDE_3_5_HAIKU` etc. per model |
| Azure Foundry | Resource name + base URL construction |

Some providers support different model subsets. Haiku has a **per-provider regional override system** with 5+ environment variables each mapping to a specific region.

The model config includes:
- Token limits (input, output)
- Thinking config (adaptive, enabled with budget, disabled)
- Cache control settings
- Provider-specific headers
- Beta features enabled for this model

All of this is resolved before the first API call. "Just pick a model" is a 14-variant × 4-provider × N-region resolution matrix.

---

## 5. Caching

### What It Looks Like

```typescript
const result = memoizeWithTTL(expensiveFunction, 300_000)
```

### What Actually Happens

The `memoizeWithTTL` implementation uses a write-through pattern:

```
Cache state: FRESH
  → Return cached value immediately

Cache state: STALE (past TTL)
  → Return old value immediately (non-blocking)
  → Trigger background refresh
  → On success: update cache
  → On failure: delete cache entry

Cache state: EMPTY
  → Block and compute
  → Store result with timestamp
```

**The subtlety**: A stale value returns instantly while the refresh happens in the background. The user never waits for a cache miss on a previously-cached value. But this means **multiple consumers might see different values** during the refresh window.

**The refresh protection**: A `refreshing: boolean` flag prevents parallel refreshes. If two callers hit a stale cache simultaneously, only one triggers a refresh. The other gets the stale value.

**The error recovery**: If the background refresh fails, the cache entry is **deleted** (not kept stale). The next caller will block and recompute. This prevents serving permanently stale data.

This pattern appears in GrowthBook feature flags, MCP auth state (15-minute TTL), policy limits, and model configs.

---

## 6. Abort Controller

### What It Looks Like

```typescript
const controller = new AbortController()
// ... later
controller.abort()
```

### What Actually Happens

Claude Code's abort system uses `WeakRef` for memory-safe parent-child relationships:

```typescript
function createChildAbortController(parent: AbortController): AbortController {
  // Child aborts when parent aborts (forward propagation)
  // Parent does NOT abort when child aborts (no backward propagation)
  // WeakRef: if parent is GC'd, child is unlinked (no memory leak)
  // Fast path: if parent already aborted, abort child immediately
}
```

**Why WeakRef?** Without it, every spawned agent/tool creates an abort controller that holds a strong reference to its parent. If the parent's session ends but a child is still running (e.g., a cron task), the parent can't be garbage collected — classic memory leak.

**Max listeners**: Set to 50 per controller to prevent Node.js `MaxListenersExceededWarning`. A single coordinator session can spawn dozens of workers, each attaching abort listeners.

**The real complexity**: Abort signals propagate through:
- API calls (cancels the streaming request)
- Tool executions (kills running processes)
- Hook scripts (terminates child processes)
- MCP protocol calls (sends cancel notification)
- WebSocket connections (sends close frame)

A single `abort()` call cascades through 5 different I/O boundaries.

---

## 7. Config Loading

### What It Looks Like

```json
// .claude/settings.json
{ "allowedTools": ["bash:*"] }
```

### What Actually Happens

Settings come from **7+ sources** with a merge hierarchy:

```
Enterprise remote settings (MDM + remote managed)
  ↓ merge
Global settings (~/.claude/settings.json)
  ↓ merge
Project settings (.claude/settings.json)
  ↓ merge  
Workspace local settings (.claude/settings.local.json)
  ↓ merge
CLI arguments
  ↓ merge
Session-only settings (/config command)
  ↓ merge
Runtime additions (hooks, commands)
```

Each layer can:
- Add rules (allow, deny, ask)
- Override rules from lower layers
- Be validated against security checks
- Be cache-invalidated by background refresh (ETag-based, 60-minute poll)

Enterprise settings do a **checksum validation** (UUID + checksum) before applying. The `securityCheck.tsx` component validates settings structure before merge.

Remote managed settings are **disk-cached** (survives process restart). Policy limits are **memory-only** (fresh each session). This asymmetry exists because:
- Settings change rarely and need consistency across restarts
- Policy limits change via admin panels and should be fresh

**The fail-open pattern**: If remote settings can't be fetched (network error), the system continues without restrictions. This is intentional — don't block work because a policy server is down.

---

## 8. Command Registration

### What It Looks Like

```typescript
const command: Command = {
  name: 'vim',
  type: 'local',
  load: () => import('./vim.js'),
}
```

### What Actually Happens

Each command specifies:

```typescript
{
  name: string,
  type: 'local' | 'local-jsx' | 'prompt' | 'plugin-stub',
  load: () => Promise<CommandModule>,
  isEnabled?: () => boolean,
  isHidden?: boolean,
  availability?: Array<'cli' | 'claude-ai' | 'agent-sdk'>,
  userFacingName?: () => string,
}
```

**The availability matrix**: A command can be:
- Available only in CLI (terminal), or only on claude.ai (web), or only in Agent SDK
- Conditionally enabled (`isEnabled` checks feature flags, subscription, platform)
- Hidden from help but still executable
- A stub (placeholder for planned/deprecated features)

**Lazy loading**: Commands use dynamic `import()` — the module code isn't loaded until the command runs. With 80+ commands, this prevents 80 modules from loading at startup.

**Feature-gated loading**: Some commands only exist if a build-time flag is true:

```typescript
const assistantCommand = feature('KAIROS') 
  ? require('./commands/assistant').default 
  : null
```

The command literally doesn't exist in the registry for external builds.

**Platform-specific**: The voice command is claude.ai only. vim is interactive-only (no headless). stickers requires user interaction. chrome requires a specific browser extension. Each command has its own world of constraints.

---

## 9. Token Counting

### What It Looks Like

```typescript
totalTokens = input_tokens + output_tokens
```

### What Actually Happens

```typescript
// The ACTUAL calculation
tokens = input_tokens 
  + cache_creation_input_tokens 
  + cache_read_input_tokens 
  + output_tokens
```

**Cache tokens matter**: Prompt caching means some input tokens are "free" (cache reads) while others are expensive (cache creation). The budget tracking excludes cache tokens to match server-side billing computation.

**Synthetic message filtering**: The system injects synthetic tool results (e.g., post-compaction file restoration). These must be filtered from token counts to prevent artificial inflation.

**Graceful degradation**: If the per-turn usage data is missing (older API responses), fall back to top-level counters. But top-level counters don't separate cache types — so the granularity degrades silently.

**Budget enforcement**: Token counting feeds into:
- `maxBudgetUsd` — hard cost cap
- `taskBudget` — per-task spending limit
- Compaction triggers — "context is getting too long"
- Telemetry counters — OpenTelemetry metrics

Getting token counting wrong means billing is wrong, compaction triggers at the wrong time, and tasks exceed their budgets.

---

## 10. Session ID

### What It Looks Like

```typescript
const sessionId = crypto.randomUUID()
```

### What Actually Happens

The session ID participates in:

| System | How It Uses the Session ID |
|--------|---------------------------|
| API headers | `X-Claude-Code-Session-Id` — correlates requests |
| Bootstrap state | `sessionId` + `parentSessionId` — session lineage |
| Telemetry | Every event tagged with session |
| Coordinator | Worker sessions linked to coordinator session |
| Teleportation | Session identity preserved across environments |
| Bridge | Remote sessions keyed by session ID (32-session pool) |
| Plan slug cache | Session ID → human-readable word slug mapping |
| GrowthBook | Experiment exposure deduplication per session |
| Cron | Lock owner identification (session + PID) |
| File history | Snapshot indexed by session |

A UUID that **is** the identity of an execution context, used for correlation across 10 different systems. Generate it wrong (non-unique) and you get cross-session contamination in telemetry, cron lock stealing, and experiment data corruption.

---

## 11. The Spinner

### What It Looks Like

A spinning animation in the terminal while thinking.

### What Actually Happens

The terminal UI is built with **React + Ink** — a React renderer for the terminal. The spinner is a React component with:

- `FpsMetricsProvider` tracking terminal frame rate
- `OffscreenFreeze` preventing re-renders of hidden content
- `ExpandShellOutputContext` for shared expansion state
- Memoized rendering with React compiler optimizations

The "Messages" display handles:
- User messages, assistant messages, system messages
- Thinking blocks (interleaved, with timing pills)
- Tool use blocks (with progress tracking)
- Redacted thinking (hidden but accounted for)
- Advisor blocks, connector text
- Transcript mode filtering

**FPS tracking**: The terminal is essentially a GUI running at terminal frame rates. Performance matters because the model is streaming tokens while the UI is rendering tool results and the user might be scrolling.

A 200ms spike in render time means visible jank in the terminal. The `FpsMetricsProvider` exists because someone measured this.

---

## 12. Slash Commands

### What It Looks Like

```
/compact
```

User types a word, thing happens.

### What Actually Happens

```
User input received
  → Parse for slash prefix
  → Look up in command registry (80+ commands)
  → Check availability (CLI? claude.ai? SDK?)
  → Check isEnabled (feature flags, subscription, platform)
  → Check isHidden (exclude from autocomplete but allow execution)
  → Lazy-load the command module
  → Execute command:
    → 'local': Runs function, returns result
    → 'local-jsx': Returns React component for rendering
    → 'prompt': Injects text into conversation
    → 'plugin-stub': Placeholder, returns disabled message
  → Some commands modify app state (vim toggles mode)
  → Some commands spawn agents (ultraplan launches remote session)
  → Some commands modify conversation (compact replaces messages)
  → Some commands are interactive (stickers needs user input)
  → Some commands affect settings (config persists changes)
  → Result rendered in terminal UI
```

The command system is a full application framework. `/compact` triggers a multi-step message compression pipeline. `/ultraplan` launches a remote agent session with 30-minute timeout and polling. `/teleport` transfers the session to another environment. `/vim` changes the input mode of the entire terminal.

These are not "chat slash commands." They're application modes disguised as simple text triggers.

---

## The Pattern

Every "simple" feature in Claude Code follows the same pattern:

1. **Visible API**: Clean, minimal interface
2. **Decision layer**: Multiple conditions gate execution
3. **Integration layer**: 3-5 other subsystems are involved
4. **Edge case handling**: Failures, timeouts, degraded modes
5. **Observability**: Telemetry, logging, metrics

The simplicity is a design achievement, not an accident. Making 15 steps feel like 1 step is harder than building the 15 steps.

---

*Next: [04-AI-APP-LEARNINGS.md](04-AI-APP-LEARNINGS.md) — What this teaches us about building AI applications*
