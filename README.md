# LangGraph Agent-Building Skill Library

Ten production-grade agent skills distilled from the **Multi-Agent Systems Masterclass** (599 pages, 22 chapters) into load-on-demand skill packages, wired into an agent-building workflow defined by [`AGENTS.md`](AGENTS.md).

## What's here

```
AGENTS.md            operating manual: the build pipeline + how/when to load each skill
FORMAT_SPEC.md       the mandatory format every skill package follows
skills/
├── agent-architecture-advisor/    DESIGN   — whether/what to build, patterns, topology, framework
├── langgraph-builder/             BUILD    — single-graph code: state, edges, Send, Command, checkpointers
├── agent-tool-designer/           BUILD    — tool interfaces + agent prompts, error contracts
├── multi-agent-builder/           BUILD    — supervisor/handoff/hierarchical systems, A2A/MCP
├── hitl-builder/                  BUILD    — interrupt(), approvals, resume, audit trails
├── agent-memory-builder/          BUILD    — memory taxonomy, Store API, extraction, forgetting
├── agent-reliability-hardener/    HARDEN   — retries, fallback ladders, circuit breakers, SLOs
├── agent-eval-builder/            HARDEN   — golden datasets, LLM-as-judge, CI gates
├── agent-guardrails-builder/      HARDEN   — threat models, injection defense, PII, moderation
└── agent-debugger/                OPERATE  — observability, triage, 16 incident runbooks, latency audits
```

Each package: one `SKILL.md` (≤ 400 lines: workflow + execution checklist + gotchas) and `references/` files loaded only when needed — including runnable Python templates (96 templates, all syntax-checked).

## Using the library

1. Read `AGENTS.md` §1–2 for the pipeline and the skill-loading protocol.
2. Load the skill matching the current stage: `skills/<name>/SKILL.md`.
3. Follow its "When to Load Which Reference File" table — references load on demand, one hop deep.

The lifecycle: **DESIGN** (architecture-advisor) → **BUILD** (langgraph-builder, tool-designer, multi-agent-builder, hitl-builder, memory-builder) → **HARDEN** (reliability-hardener, guardrails-builder, eval-builder) → **OPERATE** (debugger).

## Source

All technique is derived from the Multi-Agent Systems Masterclass blog series (LangGraph / LangChain): design patterns, LangGraph internals, state management, HITL, reliability, performance, observability, evals, guardrails, memory, inter-agent communication, deployment, 70 pro tips, and 16 debugging runbooks.
