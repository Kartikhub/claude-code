# Reverse-Engineering Claude Code: Memory, Agents, and Permissions in a Production AI System

## TL;DR

If you only have two minutes:

- Every tool call passes through a 5-layer permission cascade that starts at "no." Deny rules beat allow rules. Always.
- The system has four separate memory systems, each with a different lifetime. Overlap is intentional.
- A background process called "autoDream" consolidates memory between sessions using a 3-gate system ordered cheapest-first. During dreams, the AI is locked to read-only.
- Three distinct multi-agent patterns coexist in one codebase: hub-spoke (Coordinator), peer-mesh (Swarm), and persistent background (KAIROS). Each solves a different problem.
- The engineers hid a deterministic Tamagotchi and an anti-distillation defense in the same system. Production code has personality.

---

On March 31, 2026, Anthropic pushed Claude Code v2.1.88 to npm. The package included a 60-megabyte source map file that was never supposed to ship. Source maps are debug files that bridge minified production code back to readable source. Inside that map file: a URL pointing to a zip archive on Anthropic's Cloudflare R2 storage bucket, publicly accessible, no authentication required. The root cause was a packaging error during a manual deploy step. There's also an open Bun runtime bug (filed 20 days earlier) that generates source maps in production builds even when docs say they shouldn't be, though whether that contributed or it was purely human error isn't confirmed. Either way: around 1,900 TypeScript files exposed. Bun runtime. React and Ink for the terminal UI. 40 tool implementations. 80+ slash commands. Four memory systems. Three agent orchestration modes. A 1,500-line state management singleton.

This was the second time it happened. On launch day in February 2025, the same type of source map leak occurred. Thirteen months later, same class of bug.

I spent a week reading through the architecture. What follows are the patterns I found most worth stealing. Not a comprehensive breakdown of every file (the [full analysis](../clone-analysis/README.md) covers that). These are the design decisions that apply to anyone building AI systems that actually run in production.

For the leak details, Sabrina Ramonov's [analysis](https://www.sabrina.dev/p/claude-code-source-leak-analysis) and Varshith Hegde's [dev.to post](https://dev.to/varshithvhegde/the-great-claude-code-leak-of-2026-accident-incompetence-or-the-best-pr-stunt-in-ai-history-3igm) have excellent coverage.

---

## The Permission Layer: Deny First, Ask Later

Every tool call starts at no.

I expected permissions to be a simple allow/deny list. A config file that says "bash: allowed, file-delete: blocked" and that's it. Instead I found a 5-stage pipeline where seven different sources of rules compete for priority.

Here's the cascade, in order:

**1. Deny rules.** Hardcoded and user-defined patterns that block unconditionally. If a deny rule matches, execution stops. No appeals process, no override mechanism.

**2. Allow rules.** Patterns explicitly pre-approved for automatic execution. File reads in the project directory, safe shell commands, known-good operations.

**3. Risk classifier.** Actions that aren't clearly denied or allowed get scored by context. "Create a file in src/" is different from "install a global npm package."

**4. Pre-tool hooks.** External scripts that run before tool execution. Organizations can inject custom logic here: block access to production databases, require approval for cloud operations, log everything to an audit trail.

**5. Ask the user.** If nothing above produced a verdict, the system stops and asks. "Claude wants to run `git push origin main`. Allow?"

The seven rule sources feeding this pipeline come from global settings, project settings, workspace config, enterprise policies, CLI flags, session overrides, and runtime classifications. All merged, with deny rules always winning priority over allow rules.

> [**Diagram: Permission Flow Cascade**](diagrams/blog2-01-permission-flow.excalidraw)

That ordering matters more than the individual layers. Because deny beats allow, a malicious `.claude/settings.json` in a cloned repository cannot grant itself permissions that your personal config blocks. The architecture makes privilege escalation structurally impossible, not just policy-discouraged.

All 40 tools share a single interface: schema definition, permission check, execute. Adding a new tool automatically inherits the full permission cascade. You can't accidentally ship a tool that bypasses security.

**Here's what this means if you're building AI tools:** Design the permission model before you design the capabilities. If you add permissions after shipping 20 tools, you'll have 20 inconsistent approaches to security. If you start with a unified tool interface and a deny-first cascade, every tool you add is safe by default.

---

## Four Memory Systems That Overlap on Purpose

The AI doesn't have one memory. It has four, each with a different expiry date.

I assumed there'd be one database, maybe with tags for "short-term" and "long-term." Instead I found four fully independent systems with deliberately different lifecycles. The overlap isn't a bug. It's the entire architecture.

**memdir (permanent).** A structured memory directory capped at 200 lines and 25 kilobytes. Loaded into the system prompt on every single turn. Contains user preferences, project knowledge, and accumulated learnings. That cap forces a curation loop: when memory gets full, something has to be dropped.

> **Your permanent memory fits on a single printed page.** That constraint isn't a limitation. It's a design decision that prevents memory from consuming the context window.

**SessionMemory (per-session).** Extracted mid-session when token usage crosses a threshold. Runs as a forked subprocess so it doesn't block the main conversation. Session files are created with 0o600 permissions (readable only by the owner). Dies when the session ends.

**extractMemories (turn-end).** Runs automatically after assistant turns with high information density. Processes what the AI just said, pulls out anything worth remembering, and feeds it into memdir. This is the bridge between ephemeral conversation and permanent knowledge.

**Compaction (context management).** When conversation history exceeds the context budget, the system runs lossy compression: strip images, group messages, summarize, then selectively restore the top 5 most-referenced files (5,000 tokens each) and any triggered skills (25,000 token budget). A circuit breaker trips after 3 consecutive compaction failures to prevent infinite summarization loops.

> [**Diagram: Memory Systems with Lifetime Timeline**](diagrams/blog2-02-memory-systems.excalidraw)

Why four systems? Because different knowledge has different half-lives. A preference ("I use tabs over spaces") should survive forever. A debugging context ("we're fixing the auth bypass in middleware.ts") should die when the session ends. A pattern you used three times this week should be consolidated into permanent memory. A conversation from an hour ago should be compressible.

If you put all of that in one system, you get one of two failure modes: permanent facts get flushed during compaction, or temporary context never gets cleaned up and slowly poisons the AI's understanding.

**Pattern to steal:** Don't build one memory system with categories. Build several memory systems with explicit lifetimes, and let them feed each other through automated extraction.

---

## autoDream: When Your AI Sleeps On It

Between sessions, the AI reviews what happened and updates its permanent memory. The team calls this "dreaming."

The trigger logic is a 3-gate system, and the gate ordering is the part worth paying attention to. They're sequenced cheapest-first:

**Gate 1: Time.** A single `stat()` call on a lock file to check its modification timestamp. Cost: microseconds. If it's been less than 24 hours since the last dream, stop here. You've spent almost nothing.

**Gate 2: Sessions.** Count how many conversation files have a modification time newer than the last dream. Cost: milliseconds (parallel stat calls across session files). Default threshold: 5 sessions. If the user has only had 2 conversations since the last dream, stop. Not enough new material.

**Gate 3: Lock.** Acquire a filesystem advisory lock. The lock file body contains the PID of the current holder. If another process already holds it (verified by checking if that PID is still alive), back off. A stale lock from a dead process (older than 1 hour) gets reclaimed.

> [**Diagram: autoDream 3-Gate Flow**](diagrams/blog2-03-autodream-gates.excalidraw)

When all three gates pass, the system forks a subprocess and runs a 4-phase consolidation:

1. **Orient.** Read the existing memory index and skim topic files.
2. **Gather.** Search recent session transcripts for new information.
3. **Consolidate.** Merge new facts, resolve contradictions, convert relative dates to absolute ones (so "yesterday" doesn't become meaningless in a week).
4. **Prune.** Keep the memory index under 150 lines. Remove stale pointers.

During the entire dream, bash is restricted to read-only commands. `ls`, `grep`, `cat` work. Anything that writes is blocked. A consolidation process that can modify files without user oversight is a process that can silently change system behavior.

The gate ordering surprised me. They didn't just check "should we dream?" They ordered the checks so the cheapest one runs first. If it fails, you've wasted microseconds, not milliseconds. This is a pattern worth applying to any background process with multiple preconditions: order your checks by cost, fail cheaply.

If a dream crashes mid-execution, the system rolls back the lock file's modification timestamp. This resets Gate 1, which means the next session will re-trigger the dream. Failures are self-healing, not terminal.

---

## Three Ways to Orchestrate Multiple Agents

Not all multi-agent systems work the same way. This codebase has three distinct patterns, and the tradeoffs between them are the real lesson.

**Coordinator (hub-spoke).** One parent agent decomposes a task and spawns sub-agents via AgentTool. Each child gets an isolated conversation fork. The parent controls tool access, collects results, and synthesizes a response. Maximum isolation, minimum context sharing. The child can't see the parent's full conversation history.

**Swarm (peer-to-peer mesh).** Agents communicate via SendMessageTool within a shared tmux session managed by TeamCreateTool. The full conversation context travels between teammates. Agents share a common permission set. No central coordinator. Think of it as a group chat where each participant can see the full thread. Maximum context sharing, but every agent inherits the same permission surface.

**KAIROS (persistent background).** A background agent that survives across sessions. Runs with a strict least-privilege sandbox (read-only tools by default). Can schedule recurring work via cron expressions (50-job cap, 7-day default auto-expiry for recurring tasks). Communicates through filesystem signals, not direct message passing. Maximum autonomy, strictest constraints.

| | Coordinator | Swarm | KAIROS |
|---|---|---|---|
| **Topology** | Hub to spoke | Peer to peer | Main to background |
| **Lifetime** | Parent turn only | Until task completes | Cross-session |
| **Permissions** | Inherited, restricted | Shared policy set | Strict least-privilege |
| **Communication** | Return values | SendMessage in shared tmux | Filesystem signals |
| **Best for** | Parallel independent tasks | Multi-persona workflows | Long-running autonomous work |

> [**Diagram: Multi-Agent Modes**](diagrams/blog2-04-multi-agent-modes.excalidraw)

Which pattern fits your use case? If you need parallel independent tasks where isolation matters: Coordinator. If you need agents that hand context to each other like a conversation: Swarm. If you need a process that outlives any single session: KAIROS.

The real tradeoff is a triangle: isolation, context sharing, and lifetime. You can optimize for two. Coordinator picks isolation and short lifetime (cheap, safe, but sub-agents are context-starved). Swarm picks context sharing and medium lifetime (rich, but permission boundaries blur). KAIROS picks lifetime and isolation (persistent, but can barely touch anything).

There is no single right multi-agent architecture. There are three, and the choice depends on what you're willing to sacrifice.

---

## One More Thing

The codebase has personality. Literally.

**BUDDY** is a Tamagotchi-style virtual pet hidden in the codebase. Remember those pocket digital pets from the 90s you had to feed and take care of? Same idea, except this one lives in your terminal. Your user ID gets hashed and fed into a Mulberry32 pseudo-random number generator (a 1990s algorithm that fits in 7 lines). The output determines one of 18 species across 5 rarity tiers. Each species has 3-frame ASCII art animations running at 500ms intervals. The generation is deterministic: same user ID, same companion, every time. You can't edit or cheat your way to a legendary. The system recomputes and overwrites.

Species names are stored as character code arrays, not strings, because a species called "Opus" or "Sonnet" would collide with internal model codenames in prompt processing.

**Anti-distillation** is equally deliberate. The system injects fake tool definitions into API traffic. A legitimate Claude Code instance knows which tools are real and ignores the fakes. A model being trained on captured Claude Code outputs would learn to call tools that don't exist, producing detectable garbage. This is a defense against competitive model training at the protocol level, not a terms-of-service defense.

The code has personality. The defenses have teeth. Production software doesn't have to be sterile.

---

## If You're Building an AI Tool

Not a conclusion. A checklist.

1. **Design your permission model before your capabilities.** A unified tool interface with deny-first security is easy to add on day one and nearly impossible to retrofit.

2. **Build multiple memory systems with explicit lifetimes.** One memory system with categories will fail. Several systems with different expiry dates, feeding each other through automated extraction, will not.

3. **Order your background gates cheapest-first.** Any process with preconditions should check the cheapest condition first. Fail in microseconds, not milliseconds.

4. **Pick your multi-agent pattern based on what you'll sacrifice.** Isolation, context sharing, or persistence. You get two.

5. **Assume your AI will do the wrong thing. Build for that.** Default-deny. Read-only background processes. Circuit breakers on loops. Self-healing failures. The system that assumes AI will be wrong is the system that stays safe when it eventually is.

6. **Make your failures self-healing, not terminal.** A crashed dream resets its own trigger. A tripped circuit breaker eventually retries. Design recovery into the failure path, not as a separate monitoring system.
