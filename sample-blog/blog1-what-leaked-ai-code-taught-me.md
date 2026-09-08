# Claude Code's Source Code Leaked. Here's What It Taught Me About How AI Really Works.

Anthropic pushed a new version of Claude Code, their AI coding assistant, to npm. The package included a 60-megabyte source map file that was never supposed to ship. Source maps are debug files that connect minified production code back to readable source. Inside that map was a URL pointing to a zip archive on Anthropic's cloud storage — publicly accessible, no authentication required. The root cause was a packaging error during a manual deploy step. There's also an open Bun runtime bug that generates source maps in production builds even when they shouldn't be, though whether that played a role or it was purely human error isn't confirmed. The result: roughly 1,900 TypeScript files in the open.

> **Image placement:** [Blog opener illustration]

I spent the past week reading through those files — the actual code. What surprised me wasn't what the AI can do. It's what the engineers built to control what it can't.

Here's what I found.

## Your AI Decides What to Forget

Most people assume AI assistants either remember everything or forget everything between conversations. The reality is more interesting and more deliberate than either.

Claude Code has four separate memory systems. Not one database with different tables. Four independent subsystems, each scoped to a different lifetime — from a single API turn to permanent storage on disk.

**Turn memory** dies the second the model finishes its response. These are transient context variables like "we're debugging the login page" or "the user prefers short answers today." Useful right now, irrelevant by the next message.

**Session memory** survives within a single conversation but is wiped when the chat window closes. Think of it as the working context — which files are open, what you've discussed so far.

**Context memory** lives in instruction files (like `CLAUDE.md`) that get loaded into the model's context window at the start of every session. These define your project's conventions, preferred linting rules, or tech stack.

**Permanent memory** persists indefinitely across sessions. Your preferred coding style, your project structure, which libraries you use. These go into a memory file on disk with a hard cap: 200 lines, 25 kilobytes. That's it. Everything the AI permanently knows about you has to fit on roughly one printed page.

That constraint is the interesting part.

When memory is limited, the system has to make choices. Which facts are worth keeping? Which ones have gone stale? When two memories contradict each other, which one wins?

This is where it stops being a technical detail and starts being a product decision. The AI isn't just storing information. It's curating it. And the engineers decided that forgetting is just as important as remembering, because an AI stuffed with outdated facts makes worse decisions than one that knows less but knows it accurately.

There's even an auto-extraction system that runs at the end of each conversation, pulling out new learnings and feeding them into permanent memory. Like a student reviewing their notes after class.

> **Image placement:** [Memory systems illustration]

## Your AI Has autoDream (And It Can't Move While It Does)

Between sessions, when you're not using the tool, something happens in the background. The system spawns a child process called `autoDream` — a background memory consolidation routine that merges, deduplicates, and prunes your permanent memory.

Here's how it works. Before autoDream can start, three conditions have to pass, checked in order from cheapest to most expensive:

**Time gate.** The system checks a persisted timestamp. If the last autoDream ran less than 24 hours ago, it short-circuits here. No wasted compute.

**Session-count gate.** It counts how many conversations have occurred since the last consolidation. Fewer than five? It stops. Not enough new data to justify the overhead.

**Filesystem lock.** The system attempts to acquire a lock file. If another instance of Claude Code is already running autoDream (maybe you have two terminals open), it backs off. Only one process at a time.

When all three gates pass, the system forks a child process and starts reviewing your recent conversation history. It extracts new learnings, diffs them against existing memory entries, resolves contradictions, converts relative dates into absolute ones, and writes the updated memory file back to disk.

But here's the part that stuck with me: during autoDream, every tool is restricted to read-only permissions. The process can read your files, search your codebase, inspect context. But it cannot modify a single file, run a shell command, or push code.

Why? Because a background process that rewrites memory while you're away could silently change the AI's behavior between sessions. The engineers could have let autoDream write freely — it would have been simpler. Instead, they enforced a hard permission boundary that protects you from an AI that edits its own understanding without your knowledge or consent.

If autoDream crashes midway, the system resets its timestamp so the gating conditions will trigger again on your next session. The failure self-heals.

That single design choice — restricting a background AI process to read-only — tells you more about how these engineers think about safety than any corporate blog post.

> **Image placement:** [autoDream illustration]

## The System That Assumes AI Will Do the Wrong Thing

Every software system you use operates on a basic assumption: actions are permitted unless explicitly blocked. You can open any app, click any button, access any feature. The system trusts you by default.

Claude Code does the opposite. Every action the AI wants to take starts at no.

When the AI decides it needs to invoke a tool — run a Bash command, write a file, fetch a URL — that tool call passes through a multi-layer permission pipeline before anything executes:

**Layer 1 — Deny rules.** A hardcoded blocklist of operations that should never happen. Deleting system files, running destructive commands like `rm -rf /`, accessing sensitive directories. If the tool call matches a deny rule, it's rejected immediately. No appeals.

**Layer 2 — Allow rules.** Patterns that have been explicitly pre-approved. Reading files inside the project directory, running safe read-only commands. If the tool call matches an allow rule, it gets through.

**Layer 3 — Risk classifier.** For tool calls that aren't clearly denied or allowed, the system runs a classifier that evaluates the request in context. "This command creates a file inside the working directory" gets a different risk score than "this command installs a system-level package."

**Layer 4 — Hooks.** Custom middleware that organizations or users can inject. A company could add a pre-tool-use hook that blocks any command touching production databases. These run as external scripts, so they can encode any logic.

**Layer 5 — User approval.** If the tool call passed through all four layers without a clear verdict, the system renders a permission dialog in the terminal. You see the exact command, a risk label, and three options: approve once, approve and auto-allow similar commands, or reject.

The layer ordering is deliberate. Deny rules evaluate before allow rules. This means a project-level configuration file cannot override a user's security restrictions. Even if a repo ships a settings file saying "allow everything," user-scoped deny rules take precedence.

The engineers designed this for a world where the AI will, eventually, try to do something it shouldn't. Not out of malice. Out of misunderstanding a request, following an instruction too literally, or encountering an edge case nobody anticipated. The entire architecture is built on the assumption that the AI will get it wrong, and the job of the system is to catch it before that mistake reaches your machine.

I think that's the single most important lesson from this entire codebase. Not how to make AI smarter. How to make AI safely wrong.

> **Image placement:** [Permission layers illustration]

## Your AI Has a Hidden Companion (And a Defense Mechanism)

Buried in the source is a system called BUDDY. When you run `/buddy`, the system hashes your user ID through a seeded pseudorandom number generator (Mulberry32) and deterministically generates a companion — a small ASCII creature that sits beside your input box in the terminal. There are 18 possible species (duck, dragon, axolotl, capybara, ghost, and more), five rarity tiers weighted from common (60%) down to legendary (1%), and a 1-in-100 chance of a "shiny" variant.

The companion isn't the AI. The system prompt explicitly tells Claude: "You're not {name} — it's a separate watcher." It's a pet that belongs to you. It has idle animations, reacts when you type `/buddy pet`, and occasionally comments in a speech bubble. The AI is instructed to stay out of the way when you talk to your companion.

Nothing is stored in a database. The same user ID always produces the same species, rarity, and stats. You can't edit your way to a rare one.

Then there's something darker: anti-distillation. The system injects decoy tool definitions into its own API traffic. A legitimate instance of Claude Code knows to ignore them. But a competitor trying to train their model by capturing Claude Code's outputs would ingest poisoned data — their model would try to invoke tools that don't exist.

These aren't easter eggs. They're signals that the team treats this as something with identity, something worth protecting.

## What Changes When You Know This

I thought I had a reasonable picture of how AI coding assistants worked: a language model wrapped in a CLI, with some context management and tool integrations on top. Sophisticated, but architecturally straightforward.

This codebase proved me wrong.

What's actually running on your machine is a system with four-tier memory that decides what to remember and what to forget. A background consolidation process (autoDream) that reviews its own experiences between sessions — under strict read-only permissions. A five-layer permission pipeline built on the assumption that the AI will make mistakes, and the system's job is to catch them before they reach your filesystem. A deterministic companion system. An anti-distillation defense against competitive cloning.

The gap between a demo that generates text and a production system like this is the same gap between a concept car and a vehicle that passes crash tests. Most of the engineering isn't in the model. It's in the systems that protect you when the model does something unexpected.

This matters because we're all making decisions about AI tools: which ones to trust, how much autonomy to give them, what to let them do on our machines. Those decisions should be informed by what's actually in the code, not what's in the marketing.

Now you've seen the code.
