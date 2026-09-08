# Unique Elements: What Only Claude Code Does

> **Scope**: Features, patterns, and design choices that are genuinely novel — things you won't find in other AI coding tools or CLI applications.

---

## Table of Contents

1. [Anti-Distillation as a Feature](#1-anti-distillation-as-a-feature)
2. [BUDDY: A Deterministic Tamagotchi](#2-buddy-a-deterministic-tamagotchi)
3. [Undercover Mode: AI That Hides Its Own Identity](#3-undercover-mode)
4. [autoDream: The AI That Thinks While You Sleep](#4-autodream)
5. [Bun's Transport-Layer Attestation](#5-buns-transport-layer-attestation)
6. [Build-Time Feature Flags as Dead Code Elimination](#6-build-time-feature-flags)
7. [The Bash Parser That Counts Bytes, Not Characters](#7-the-bash-parser)
8. [KAIROS: A Background Agent That Lives Beyond Sessions](#8-kairos)
9. [Teleportation: Moving Sessions Between Environments](#9-teleportation)
10. [The 3-Gate Dream System](#10-the-3-gate-dream-system)
11. [Species Encoded as Character Codes](#11-species-encoded-as-character-codes)
12. [Connector Text Summarization with Signatures](#12-connector-text-summarization)
13. [Permission Proxying Through Mailboxes](#13-permission-proxying)
14. [The Stickers Command](#14-the-stickers-command)
15. [A Circuit Breaker Discovered by Data](#15-circuit-breaker-by-data)

---

## 1. Anti-Distillation as a Feature

Most AI products worry about users exploiting the model. Claude Code worries about **competitors training on its outputs**.

### How It Works

Two independent mechanisms:

**Fake Tool Injection:**
```typescript
if (feature('ANTI_DISTILLATION_CC') && 
    process.env.CLAUDE_CODE_ENTRYPOINT === 'cli' &&
    getFeatureValue('tengu_anti_distill_fake_tool_injection', false)) {
  result.anti_distillation = ['fake_tools']
}
```

The server injects fake tool definitions into the conversation. A legitimate model ignores them (it knows which tools are real). A distilled model that learned from captured conversations would try to use these fake tools, producing detectable garbage.

**Connector Text Summarization:**

Text the model produces between tool calls is summarized server-side with a cryptographic signature. The original text is recoverable by the legitimate client but looks like a compressed blob to anyone scraping the API responses for training data.

### Why This Is Unique

No other coding tool treats its own API traffic as a potential training corpus for competitors. This is a defensive strategy at the **protocol level** — not a terms-of-service defense, but a mathematical one.

---

## 2. BUDDY: A Deterministic Tamagotchi

A virtual companion species generated from your user ID.

### The Generation Algorithm

```typescript
// Mulberry32 PRNG seeded from user hash
function mulberry32(a: number): () => number {
  return function() {
    a |= 0; a = a + 0x6D2B79F5 | 0
    var t = Math.imul(a ^ a >>> 15, 1 | a)
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t
    return ((t ^ t >>> 14) >>> 0) / 4294967296
  }
}

// Hash(userId + "SALT-2026") → species + rarity
```

### Why It's Deterministic

Users can't edit their way to a legendary companion. The species and rarity are **derived** from the user ID, not stored. Every time the system reads the companion data, it regenerates from the same seed. Change your user ID, get a different companion. Edit the config file — the system recomputes and overwrites.

### 18 Species, 5 Rarities

- Common (probability floor: 5), Uncommon (15), Rare (30), Epic (40), Legendary (50)
- Each species has 3-frame ASCII art animations
- 500ms animation tick with memoized roll caching

### The Anti-Collision Trick

Species names are stored as **character code arrays**, not strings, to avoid collisions with internal model codenames. If a species were named "Opus" or "Sonnet", it could trigger unintended behavior in prompts or logs.

---

## 3. Undercover Mode

When Claude Code operates on public repositories, it activates **undercover mode** — automatically scrubbing references to:

- Internal model codenames
- Unreleased model versions
- Internal PR and commit metadata
- Build-time feature names

### Default-ON Security

```typescript
// Off ONLY if repo matches INTERNAL_MODEL_REPOS allowlist
// No force-OFF flag exists — stays ON unless whitelisted
```

This is defensive by design. There's no `--disable-undercover` flag. The only way to turn it off is to be working in a known internal repository. The logic is: if there's any chance the work could become public, scrub everything.

### Build-Time Removal

For external builds, undercover mode code is **constant-folded away** by the bundler. In the shipped binary, the scrubbing logic doesn't exist because there's nothing to scrub — the internal codenames were never included.

---

## 4. autoDream

Most AI tools consolidate memory when you tell them to. Claude Code does it **automatically, in the background, between sessions**.

### The Concept

When you close a Claude Code session, a background process may wake up to "dream" — processing your recent interactions and consolidating learnings into long-term memory. This is not triggered by the user; it's triggered by conditions.

### The 3-Gate System

```
Gate 1: Time (has enough time passed since last dream?)
  → Gate 2: Sessions (enough sessions since last dream?)
    → Gate 3: Lock (no other process currently dreaming?)
      → Fork subprocess running /dream command
```

Gates are ordered cheapest-first:
- **Time gate**: File stat on the dream timestamp file (microseconds)
- **Session gate**: Counter comparison (nanoseconds)
- **Lock gate**: Filesystem lock acquisition (milliseconds)

If a dream fails, the system **rewinds the lock's modification time** so the next session retriggers it. The dream doesn't just fail — it sets up conditions for retry.

### Read-Only During Dreams

While dreaming, the bash tool is restricted to read-only commands (ls, grep, cat). The agent can observe but not modify. This prevents a consolidation process from making destructive changes without user oversight.

---

## 5. Bun's Transport-Layer Attestation

The application includes an attribution header with a placeholder:

```typescript
// In the request body
headers['x-attribution'] = '...cch=00000...'
```

This `00000` is **overwritten by Bun's native HTTP stack** at the transport layer with a real attestation token. The application code never sees the real token, can't log it, can't leak it.

### Why This Matters

- Source code analysis reveals nothing about the attestation mechanism
- The token is generated at a layer below the JavaScript runtime
- Even a compromised application can't extract or forge the token
- It's authentication below the application boundary

This is the only known use of Bun's HTTP stack for **security enforcement below the application layer**.

---

## 6. Build-Time Feature Flags

Most feature flag systems are runtime: check a config, toggle behavior. Claude Code uses `bun:bundle` to make feature flags a **compilation artifact**.

```typescript
import { feature } from 'bun:bundle'

if (feature('KAIROS')) {
  // This entire code block is REMOVED FROM THE BINARY for external builds
  require('./commands/assistant')
}
```

The bundler evaluates `feature()` at build time and performs dead code elimination. This means:

- Internal features are **physically absent** from external builds
- No runtime overhead for checking disabled features
- No accidental leakage of internal feature code
- The source code is a superset; no single build contains all paths

Combined with runtime GrowthBook flags, this creates a two-phase feature system where some features can't exist (build-time) and others can be toggled (runtime).

---

## 7. The Bash Parser

A pure TypeScript implementation of a tree-sitter-bash-compatible parser. Not a wrapper around WASM. Not a binding. A from-scratch parser.

### Byte Offsets, Not Character Offsets

```typescript
// UTF-8: ñ = 2 bytes, 1 JS character
// The parser tracks BYTE positions for tree-sitter compatibility
// This matters for security: a misparse could allow command injection
```

The parser maintains a lazy byte table: ASCII characters use a fast path (1 byte = 1 char). The moment a non-ASCII character appears, it builds a full byte→char mapping table.

### Protection Against Adversarial Input

- **50ms wall-clock timeout** — prevents pathological input from blocking the thread
- **50,000 node budget** — prevents memory exhaustion from deeply nested structures
- Validated against a **3,449-input golden corpus** from the WASM tree-sitter implementation

### The Legacy Problem

The deprecated `splitCommand_DEPRECATED()` still runs alongside the new parser in 8 security validators. They disagree on `\r` handling. Both are kept because removing either would change security behavior — a classic "can't fix it without breaking it" situation.

---

## 8. KAIROS

An always-on background agent mode that persists beyond individual sessions:

- **Cron scheduling** with distributed locking (PID liveness probes + owner key)
- **GitHub webhook subscriptions** for automated PR monitoring
- **Session history pagination** from server-side event storage
- **Bridge sessions** for remote execution across devices

This isn't "a chatbot that remembers things." It's an **autonomous agent** that watches your repositories, runs scheduled tasks, and can be interacted with remotely.

### Distributed Lock Implementation

Multiple Claude Code processes might try to run the same cron task. The lock system uses:

1. Filesystem lock file
2. PID liveness probe (is the locking process still alive?)
3. Owner key comparison (is it the same Claude Code instance?)
4. Jitter configuration (GrowthBook-tunable) for load distribution

---

## 9. Teleportation

Moving a Claude Code session from one environment to another:

- CLI → Web (move terminal session to claude.ai)
- Web → CLI (bring web session back to terminal)
- Local → Remote (shift context across machines)

### The Protocol

```typescript
// Session schema covers:
// - Git sources (bundles, not clones)
// - Knowledge base sources
// - Custom prompts
// - GitHub PR context links
// - OAuth tokens + org UUID
```

The system bundles the minimum context needed (git state, memory, conversation) and transmits it. On the receiving end, the session reconstructs and continues.

Retry logic: exponential backoff (2s → 4s → 8s → 16s) with transient error detection (retry 5xx, not 4xx).

---

## 10. The 3-Gate Dream System

Already covered in section 4, but the gate ordering deserves its own highlight: **cheapest check first**.

This is a pattern you rarely see applied to background processing:

1. File stat (microseconds) — is it too soon to dream?
2. Counter comparison (nanoseconds) — enough sessions?
3. Lock acquisition (milliseconds) — can we get exclusive access?

If a million instances check gate 1 and 999,999 fail, you've spent microseconds per failure. Only the rare success proceeds to the more expensive checks.

---

## 11. Species Encoded as Character Codes

In the companion system, species names are stored as arrays of character codes:

```typescript
// Instead of: name: "Capybara"
// Uses: name: [67, 97, 112, 121, 98, 97, 114, 97]
```

This prevents the string "Capybara" (or whatever species) from appearing as a literal in the codebase, where it might:

- Trigger model behavior if it matches an internal codename
- Appear in search results for internal audits
- Be confused with system metadata in prompts

It's steganographic naming — the names exist only at render time.

---

## 12. Connector Text Summarization

When the model produces text between tool calls, the server can:

1. Capture the original text
2. Produce a compressed summary
3. Attach a cryptographic signature to the summary
4. Return the summary in place of the original

The legitimate client can verify and restore the original. A data scrapers sees only the summary — useless for training without the restoration key.

This is the same mechanism used for thinking blocks, repurposed as an anti-distillation measure.

---

## 13. Permission Proxying Through Mailboxes

When a swarm worker needs permission for a dangerous operation:

**Bridge mode**: The worker's permission request routes to the leader's UI. The user sees the prompt in their terminal, approves or denies, and the decision routes back.

**Mailbox mode**: The request goes into an async mailbox. The leader processes it when it reaches that point in its own execution. This is for cases where the leader isn't waiting — it's doing its own work.

This creates a **distributed permission system** where trust decisions for multiple agents funnel through a single human approval point.

---

## 14. The Stickers Command

`/stickers` — an interactive command for ordering physical merchandise (stickers) from within the CLI. 

It exists alongside `ultraplan` (multi-agent planning), `teleport` (session transfer), and `bughunter` (code vulnerability scanning). The same framework that spawns distributed AI agents also handles sticker orders.

This isn't technically unique, but it reveals a design philosophy: **the command system is fully general**. Any interaction — AI, commerce, configuration — is a command with the same lifecycle.

---

## 15. A Circuit Breaker Discovered by Data

The compaction retry circuit breaker (`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`) exists because of a specific discovery:

> BigQuery analysis revealed 250,000 wasted API calls per day from infinite compaction retry loops.

This wasn't designed into the system. It was **discovered through observability** and fixed with a simple constant. The codebase is unusual in that you can find these "archaeology markers" — comments revealing that production data drove design decisions.

Similarly, the 529-error retry limit (MAX_529_RETRIES = 3) for background tasks was added after observing cascade amplification: failing background retries making the overload worse.

---

## Summary: What's Genuinely Novel

| Element | Why It's Unique |
|---------|----------------|
| Anti-distillation (fake tools + summarization) | Protocol-level defense against model training on API traffic |
| Deterministic companion from user hash | Can't be manipulated; recomputes every read |
| Undercover mode (default-ON, no force-off) | AI self-censorship for operational security |
| autoDream (background consolidation) | AI that processes experiences autonomously between sessions |
| Transport-layer attestation | Security below the application runtime |
| Build-time dead code elimination flags | Features that are physically absent, not just disabled |
| Pure TS bash parser with byte offsets | Security-critical parsing without WASM |
| KAIROS (persistent background agent) | Agent that outlives its sessions |
| Session teleportation | Seamless context transfer between environments |
| Permission mailbox proxying | Distributed trust through async channels |
| Species as character code arrays | Steganographic naming to avoid model interference |

---

*Next: [03-SIMPLE-YET-COMPLEX.md](03-SIMPLE-YET-COMPLEX.md) — Things that look simple until you understand why they aren't*
