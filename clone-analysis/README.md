# Claude Code Source Analysis

> Independent analysis of the leaked Claude Code CLI source code — 1,902 TypeScript files comprising a terminal-native AI coding agent.

---

## Documents

| # | Document | Focus |
|---|----------|-------|
| 01 | [Complexity Analysis](01-COMPLEXITY-ANALYSIS.md) | Where the real complexity lives and why — feature flags, permissions, multi-agent, memory |
| 02 | [Unique Elements](02-UNIQUE-ELEMENTS.md) | What only this codebase does — anti-distillation, autoDream, BUDDY, undercover mode |
| 03 | [Simple Yet Complex](03-SIMPLE-YET-COMPLEX.md) | Deceptive simplicity — tool calls, system prompts, file reads that are 15-step pipelines |
| 04 | [AI App Learnings](04-AI-APP-LEARNINGS.md) | Actionable patterns for building AI applications — the top 18 lessons extracted |
| 05 | [Architecture](05-ARCHITECTURE.md) | Complete 9-layer architecture — from CLI shell to infrastructure services |
| 06 | [Developer Experience](06-DEVELOPER-EXPERIENCE.md) | Engineering culture, trade-offs, and what the code reveals about the team |

## Diagrams

| # | Diagram | What It Shows |
|---|---------|---------------|
| 01 | [Architecture Layers](diagrams/01-architecture-layers.excalidraw) | The 9-layer stack with key components per layer |
| 02 | [Tool Call Pipeline](diagrams/02-tool-call-pipeline.excalidraw) | The 15-step journey of a single tool invocation |
| 03 | [Permission Decision Flow](diagrams/03-permission-flow.excalidraw) | How trust decisions cascade through 7 sources and 5 check layers |
| 04 | [Memory Systems](diagrams/04-memory-systems.excalidraw) | The 4 overlapping memory systems and their interactions |
| 05 | [Multi-Agent Modes](diagrams/05-multi-agent-modes.excalidraw) | Coordinator, Swarm, and KAIROS architecture comparison |

## Codebase Stats

| Metric | Value |
|--------|-------|
| Total files | 1,902 TypeScript |
| Runtime | Bun |
| UI framework | React + Ink (terminal) |
| CLI framework | Commander.js |
| Tools | 41 directories |
| Commands | 80+ directories |
| Feature flags | 3 build-time + 20 runtime |
| API providers | 4 (Direct, Bedrock, Vertex, Foundry) |
| MCP transports | 6 |
| Permission sources | 7+ |
| Memory systems | 4 |
| Agent modes | 4 |

## Reading Order

**For understanding complexity**: 01 → 03 → 05  
**For building your own AI app**: 04 → 01 → 03  
**For architecture review**: 05 → 01 → 06  
**For unique insights**: 02 → 06 → 04  
