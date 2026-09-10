# AGENTS.md — LangGraph Agent Building Workflow + Skill Library

**What this file is.** The kernel of an agent-building pipeline plus the operating manual for its skill library. It defines HOW LangGraph agents get built in this workflow: the process, the order, the gates, the git protocol — and WHICH of the 10 packaged skills to load at each step, and how to load them. It is project-agnostic — it never contains anything specific to one agent project. Everything about WHAT a particular agent is lives in that project's `context/` directory inside its own repo.

**Precedence.** If this file and a `context/` file disagree: this file wins on process, the context file wins on project specifics. If this file and a skill disagree: this file wins on process and sequencing, the skill wins on technique inside its own domain.

---

## 1. The Skill Library

Ten skills, distilled from the Multi-Agent Systems Masterclass (599 pages) into load-on-demand packages under `skills/`. Each package: one `SKILL.md` (workflow + checklist + gotchas) plus `references/` files loaded only when the skill's own table says to load them. One skill per package directory. Never edit a skill's technique to fit a project — if a skill and the project conflict, record the conflict in `progress-tracker.md` and follow the project.

| # | Skill | Stage | What it gives you | Load it when the user says... |
|---|-------|-------|-------------------|-------------------------------|
| 1 | `agent-architecture-advisor` | DESIGN | Whether to build an agent at all, pattern selection (7-question framework), single vs multi, topology, framework choice, cost/latency budget, architecture decision record | "design an agent", "which pattern/topology/framework", "is an agent even the right tool" |
| 2 | `langgraph-builder` | BUILD | Correct single-graph code: state + reducers, edges, Send fan-out, Command routing, checkpointers, streaming — with the engine's sharp edges | "build a LangGraph graph", "add a reducer", "wire conditional edges", "set up a checkpointer", "fan out to N workers" |
| 3 | `agent-tool-designer` | BUILD | Tool interfaces (descriptions, parameter schemas, error contracts, idempotency keys) and agent-facing prompts (instruction hierarchy, completion contracts) | "design the tools", "write the system prompt", "the agent picks the wrong tool" |
| 4 | `multi-agent-builder` | BUILD | Supervisor / handoff / hierarchical / network topologies, the supervisor prompt, inter-agent messaging (A2A/MCP), multi-agent failure detection | "build a supervisor", "add agents that hand off", "multi-agent system" |
| 5 | `hitl-builder` | BUILD | `interrupt()` approval gates, resume sequences, human state edits, audit trails, async approvals | "add human approval", "pause for review", "interrupt and resume" |
| 6 | `agent-memory-builder` | BUILD | Memory taxonomy, Store API, extraction prompts, write/consolidate/forget policies, memory evaluation | "give the agent memory", "remember across sessions", "memory pollution" |
| 7 | `agent-reliability-hardener` | HARDEN | Failure taxonomy, retries + idempotency, fallback ladders, circuit breakers, durable execution, SLOs, fault injection | "make it reliable", "add retries/fallbacks", "survive crashes", "define SLOs" |
| 8 | `agent-eval-builder` | HARDEN | Eval layers, golden datasets, trajectory checks, LLM-as-judge + calibration, CI regression gates, red-team suites | "build evals", "create a golden dataset", "add CI gates", "is the judge any good" |
| 9 | `agent-guardrails-builder` | HARDEN | Threat model, prompt-injection defense, tool security + sandboxing, PII handling, output moderation, red-team cadence | "add guardrails", "secure the tools", "handle PII", "threat model" |
| 10 | `agent-debugger` | OPERATE | Observability wiring (traces, metrics, alerts), 10-minute triage, symptom router, 16 incident runbooks, latency audits | "it's broken / looping / expensive", "instrument tracing", "read this trace" |

**Sibling exclusions are built into every skill's frontmatter** — they prevent loading two skills for one job. When two skills are genuinely both needed (common pairings below), load them sequentially, one as primary.

Common pairings: architecture-advisor → langgraph-builder (design then build) · langgraph-builder → hitl-builder (graph then interrupts) · reliability-hardener → eval-builder (harden then verify) · eval-builder → debugger (eval regression → root-cause).

---

## 2. How to Load a Skill

1. Read `skills/<name>/SKILL.md` only. It is ≤ 400 lines and self-contained.
2. Follow its "When to Load Which Reference File" table. Load a reference **only at the step that names it** — because references are one hop deep and each starts with a "Load this when" note, this keeps context small and current.
3. Execute the skill's Execution Checklist as the working checklist for that stage.
4. Copy code from the skill's `references/templates.md`, replace `{{PLACEHOLDER}}` values, and adapt — templates are starting points, not finished code.
5. One skill is primary at a time. A second skill may be consulted for a bounded question (load only the specific reference you need), never run end-to-end in parallel.

Never load all ten skills. Never paste a whole skill into a prompt. The description frontmatter of each skill is what makes it discoverable; the SKILL.md body is what makes it work.

---

## 3. The Pipeline

Every session executes this loop exactly once, for exactly one feature:

```
INPUT    User provides a GitHub repo URL + a task (feature, fix, or bootstrap)
ACQUIRE  Clone the repo — or fetch only the required paths            (§5)
ORIENT   Read all context files in order — before touching anything   (§6)
PLAN     Pick the ONE next feature; select the PRIMARY SKILL for it   (§7)
BUILD    Follow the Build Ladder, loading skills per phase            (§8)
VERIFY   Pass the current phase's eval gate — record the result       (agent-eval-builder)
RECORD   Update the three living files                                (§11)
SHIP     Commit and push (§10) — work not pushed is work not done
```

Never batch two features into one session. Never end a session with a half-built feature — if a feature cannot be finished, stop at the last clean rung of the ladder and record the exact state in `progress-tracker.md` so the next session resumes cleanly.

---

## 4. Stage → Skill Map

```
DESIGN   agent-architecture-advisor        before any code exists
BUILD    langgraph-builder                 the graph itself
         agent-tool-designer               the tools and prompts it calls
         multi-agent-builder               when roles split into 2+ agents
         hitl-builder                      when humans approve or edit
         agent-memory-builder              when state must survive runs
HARDEN   agent-reliability-hardener        before launch, after features work
         agent-eval-builder                the verification system
         agent-guardrails-builder          the safety layer
OPERATE  agent-debugger                    when live and misbehaving
```

A project may skip HARDEN only if `project-overview.md` explicitly declares it a prototype — and then it ships with the string `prototype: no harden stage` in `progress-tracker.md`, because an unhardened agent in production is an incident on a timer.

---

## 5. Acquire

- Full clone: `git clone <repo_url>` — the default.
- Large repo or task needs only some paths: sparse checkout of the required files only.
- No write access to the repo: fork first, clone the fork, work on a branch, open a pull request against the upstream repo.
- Authentication: `gh` CLI or a PAT from the environment. Never commit credentials, tokens, or `.env` files.
- The skill library lives in this repo (`skills/` + this file). If the agent project repo is separate, the library is cloned alongside it; the paths in this file are relative to the library root.
- If the repo has no `context/` directory — do not start building. Run the Intake procedure (§13) first, which starts with `agent-architecture-advisor`.

---

## 6. Orient — Mandatory Read Order

Read all ten context files, in this order, at the start of every session. Never implement from memory of a previous session.

| # | File | What it tells you |
|---|------|-------------------|
| 1 | `context/project-overview.md` | What this agent is for. Scope in / scope out. Success criteria. |
| 2 | `context/architecture.md` | Stack, folder structure, system boundaries, external services, invariants. |
| 3 | `context/graph-design.md` | The contract. Topology, state schema + reducers, node-by-node specs. |
| 4 | `context/tool-registry.md` | Every tool: signature, behavior, consumers, side effects, eval status. |
| 5 | `context/prompt-registry.md` | Every prompt: node, model, temperature, max tokens, output schema, version. |
| 6 | `context/eval-plan.md` | Eval layers, datasets, graders, gates, thresholds. |
| 7 | `context/code-standards.md` | How code must look in this repo. Naming, patterns, closed lists. |
| 8 | `context/library-docs.md` | Project-specific library patterns. Authority order: MCP docs → installed skills → this file → training knowledge. |
| 9 | `context/build-plan.md` | The phased feature list. The current phase's features in detail. |
| 10 | `context/progress-tracker.md` | What is done, what is next, decisions made so far. |

Why this order: identity → structure → contract → registries → quality bar → process → current state. The first ten minutes of every session are reading. Skipping Orient is the root cause of every drift bug.

**Authority order for agent-building technique:** MCP docs → **installed skills (`skills/`)** → `library-docs.md` → training knowledge. Training data alone is never sufficient, because the skills encode sharp edges (edge double-fire, reducer data loss, resume semantics) that training data does not reliably carry.

---

## 7. Plan — Pick the Feature, Then the Primary Skill

1. Read `build-plan.md` + `progress-tracker.md`, pick the ONE next feature.
2. Name its PRIMARY SKILL from the Stage → Skill Map (§4). If no skill maps, the feature is either out of scope or the plan is wrong — say so before building.
3. If the feature is architectural (new capability class, new agent, new integration), run `agent-architecture-advisor` first and append its decision record to `progress-tracker.md` — because the cost of a wrong architecture is measured in re-built phases, and the advisor's when-NOT-to-build gate is cheaper than any rebuild.

---

## 8. The Build Ladder

The core build principle: **design top-down, build bottom-up.**

`graph-design.md` is written first and is the contract — full topology, state schema, and node specs exist on paper before any real code. Then construction starts at the leaves (tools) and climbs to the root (main graph), evaluating at every layer.

```
Phase 0 — SKELETON     Full graph with STUB nodes + STUB tools (hardcoded returns)
                       PRIMARY SKILL: langgraph-builder (topology + state, stub bodies)
                       Verify topology and state flow in LangGraph Studio / local run
                       GATE: the skeleton runs end-to-end on a fake input
Phase 1 — TOOLS        Real tools, built one at a time per tool-registry.md
                       PRIMARY SKILL: agent-tool-designer (signatures, error contracts,
                                      idempotency keys; langgraph-builder for wiring)
                       GATE: each tool passes its tool-level eval
Phase 2 — SUBGRAPHS    Real node logic, real tools swapped in, one subgraph at a time
                       PRIMARY SKILL: langgraph-builder; multi-agent-builder if a
                                      subgraph is a distinct agent with its own role;
                                      agent-memory-builder when state must persist
                                      across runs (Store, extraction, forgetting)
                       GATE: each subgraph runs green in isolation
Phase 3 — MAIN GRAPH   Wire subgraphs, conditional edges, interrupts, checkpointing
                       PRIMARY SKILL: langgraph-builder; hitl-builder for every
                                      approval/edit point; multi-agent-builder for
                                      supervisor/handoff wiring
                       GATE: end-to-end run succeeds on golden inputs
Phase 4 — EVALS        Full eval suite wired, tracing on, regression gates active
                       PRIMARY SKILL: agent-eval-builder; agent-debugger sets up
                                      the observability the evals read from
                       GATE: all eval layers pass at configured thresholds
```

Rules of the ladder:

- No phase starts until the previous phase's gate passes. Gate results are recorded in `progress-tracker.md`.
- Phase 0 exists so the whole shape is visible on day one — the agent equivalent of "mock UI first". Stub tools return realistic fake data matching the schemas in the registries.
- Within a phase, build one unit at a time: one tool, one node, one subgraph. Finish it, eval it, record it, then move to the next.
- If a tool turns out to need a different signature than `tool-registry.md` declares — stop, update the registry and `graph-design.md` first, then build. The registries are never allowed to lag behind the code.

---

## 9. The Harden Stage (post-ladder, pre-launch)

The ladder proves the agent works. The harden stage proves it survives. Run all three in this order — each assumes the previous one's output:

1. **`agent-reliability-hardener`** — failure taxonomy pass, idempotency, retries, fallback ladders, circuit breakers, durable execution, SLOs. Output: reliability decisions recorded in `progress-tracker.md`; degraded-mode rungs documented in `architecture.md`.
2. **`agent-guardrails-builder`** — threat model, injection defense, tool security, PII, moderation, audit logging. Output: the one-page guardrail spec, stored at `context/guardrail-spec.md` and treated like a registry (update it whenever tools change).
3. **`agent-eval-builder`** — golden datasets, trajectory checks, judge + calibration, CI gates, red-team suite. Output: `eval-plan.md` becomes real datasets and gates.

A feature is launch-ready when the ladder gate AND the three harden outputs exist and pass. Skipping harden for speed is borrowing from the incident budget with interest.

---

## 10. Git Protocol

- `context/` lives inside the agent's repo, at the repo root, versioned with the code. This skill library is versioned in its own repo.
- **Default mode: direct to `main`.** Solo workflow — after a feature passes its gate, commit to `main` and push.
- **Branch mode** (only if the project's `architecture.md` sets `branch_mode: true`): one feature branch per feature, named `<phase>-<feature-slug>` (example: `phase1-search-tool`), push the branch, open a pull request.
- Commit **only after** a feature is complete and its gate passed. Never commit half-done work.
- The three living files are updated in the **same commit** as the feature they describe. A feature commit with stale registries is a broken commit.
- Commit message format:

```
[Phase N.F] feature-slug: one-line description of what was built

Gate: <which eval gate passed> | Registries updated: <yes/no + which> | Skills used: <names>
```

- Never force-push. Never rewrite published history. Never commit `.env`, keys, or secrets.
- Push immediately after committing. Unpushed work at the end of a session is unfinished work.

---

## 11. The Three Living Files

These three files change constantly during the build. They are the drift-control system — the reason the project stays coherent across sessions and across agents.

| File | Update after | What goes in |
|------|--------------|--------------|
| `tool-registry.md` | Any tool added, changed, or re-signed | Signature, args schema, return shape, side effects, consuming nodes, error behavior, eval status |
| `prompt-registry.md` | Any prompt or model config changed | Full prompt text location, model, temperature, max tokens, structured output schema, version, change reason |
| `progress-tracker.md` | Every completed feature (no exceptions) | Checklist state, gate results, decisions made, bugs found, caveats learned, **which skills were used and any technique the project overrode** |

If a living file was not touched, the feature is not done. There is no such thing as a code-only commit in this workflow.

---

## 12. Rules That Never Change

1. **Scope is sacred.** Only build what the current feature requires. Never go beyond scope, even if it seems helpful.
2. **One feature at a time,** fully finished, per session.
3. **Design top-down, build bottom-up.** Never build a tool that is not declared in `graph-design.md`. Never wire a node that is not in the topology.
4. **No gate, no pass.** Every phase gate runs and passes before the next phase starts.
5. **Never mutate state.** Nodes return state updates; reducers merge them. This is non-negotiable.
6. **Every tool and node handles failure.** Wrap external calls in error handling, return structured errors, log them — one failing tool must never crash the whole graph run. `agent-reliability-hardener` defines the standard; Phase 1 applies it to every tool as it lands.
7. **Registries never lag.** Code and its registry entry change in the same commit.
8. **Never hardcode model names, temperatures, or prompts in node code.** They live in `prompt-registry.md` and are loaded from there.
9. **Never invent tools, prompts, or events on the fly.** Add them to the registry first, then build.
10. **Skills over memory.** Load the skill for the stage before writing technique from training data, because the skills encode book-verified sharp edges that memory approximates. Training data alone is never sufficient (see authority order, §6).
11. **Secrets only in `.env` / environment.** Never hardcoded, never committed.
12. **Clean over clever.** Simple, readable, boring code wins. A junior developer must be able to follow the graph.
13. **Do not edit skills mid-project.** A skill gap gets recorded in `progress-tracker.md` and fixed as a library commit after the feature ships — because silently diverging skill copies stop being skills and become folklore.
14. **Human gates before autonomous actions.** Anything irreversible, expensive, or uncertain gets an interrupt point designed in from the start (`hitl-builder`), not bolted on after the first bad refund.

---

## 13. Intake — Bootstrapping a New Project

Run this when the repo has no `context/` directory (new project) or when a structural change makes the context files invalid (re-intake). Do not skip it and do not guess.

### Step 1 — The Interview

Ask the user these questions, in order. Batch them; do not drip one at a time. If an answer is unclear, ask a follow-up before moving on.

**About the agent:**

1. In one sentence — what does this agent do? What exactly goes in, and what exactly comes out?
2. What triggers a run — chat message, API call, schedule, or manual run in Studio?
3. Walk me through one perfect run, step by step, from input to final output.
4. Which steps need external data or external actions? Name the system each step touches. (Each answer is a candidate tool.)
5. Which steps need a human decision before the agent continues? (Human-in-the-loop interrupts.)
6. What must the agent remember within one run vs. across runs? (State schema vs. checkpointer vs. Store.)
7. Is one graph enough, or are there distinct specialist roles? (Subgraph / supervisor candidates.)
8. Which model(s)? Any constraints on cost, latency, or self-hosting?
9. Where must the model return structured output, and what shape must it have?

**About the project:**

10. Python or TypeScript?
11. Name three concrete end-to-end cases that prove the agent works. (These seed `eval-plan.md`.)
12. Does it ship a UI or chat interface, or is it Studio / API only?
13. Existing repo URL — and anything already inside it we must respect?

### Step 2 — Architecture Pass (`agent-architecture-advisor`)

Feed the interview answers to `agent-architecture-advisor` and produce, before any context file is written: the feasibility verdict (agent vs workflow vs single call), the pattern stack, single vs multi-agent + topology, the framework choice, and the cost/latency budget. Write the advisor's one-page decision record as `context/adr-001-architecture.md`. If the advisor says "don't build an agent," stop and tell the user — the cheapest failed project is the one that never started.

### Step 3 — Generate the Context Set

From the answers + the decision record, generate all ten context files (§6 order). Where an answer implies a default the user did not state, choose the simplest option that satisfies the stated success cases and mark it as a decision in `progress-tracker.md`.

Generation order matters: `project-overview.md` → `architecture.md` → `graph-design.md` (the advisor's topology and state schema feed this directly) → `tool-registry.md` (declared, not yet built) → `prompt-registry.md` → `eval-plan.md` → `code-standards.md` → `library-docs.md` → `build-plan.md` (phased per the ladder) → `progress-tracker.md` (all unchecked).

### Step 4 — Review and First Commit

Present the generated files to the user for review. After approval, commit the entire `context/` set as the first commit: `[Phase 0.0] context bootstrap: full context set generated from intake`.

---

## 14. Operating a Live Agent (`agent-debugger`)

Once deployed, incidents and performance questions route to `agent-debugger` — never improvised:

- **Something is broken now** → the debugger's incident workflow: contain first, 10-minute triage, symptom router → one of 16 runbooks → fix → verify against the firing metric → incident retro. The retro produces one new alert and one permanent artifact.
- **It's slow or expensive** → the debugger's latency-audit workflow: measure before optimizing, one change at a time, logged with before/after on a pinned dataset.
- **Eval scores regressed** → debugger for root cause, eval-builder's loop for the fix-then-re-verify discipline.
- **Every incident ends** with `progress-tracker.md` updated: what broke, which runbook applied, what prevention landed.

Recurring incidents are not runbook problems — they are harden-stage debt. Two incidents with the same root cause promote a permanent fix through `agent-reliability-hardener` or `agent-guardrails-builder`.

---

## 15. Skill Maintenance

The library is code. It changes only through deliberate commits:

- **Add a skill** only for a recurring job the ten do not cover — every skill is a maintenance liability, because its gotchas go stale as libraries evolve.
- **Update a skill** when a production incident revealed a missing gotcha (add it as Symptom → Cause → Response), when a reference file's guidance proved wrong, or when the underlying library changed behavior. Record "last verified" dates on gotchas.
- **A skill edit commit** touches one skill directory only, and its message states what triggered the change: `skills(hitl-builder): gotcha — double resume on same thread_id, found in prod incident 42`.
- **Never fork a skill inside a project repo.** If a project needs different behavior, the project context files override (§ Precedence) — the skill stays generic, because it serves every project.

---

## 16. Session Checklists

**Session start:**

- [ ] Pull latest from the repo and from the skill library
- [ ] Read all ten context files, in order (§6)
- [ ] Read `progress-tracker.md` → confirm the ONE feature for this session
- [ ] Confirm the feature is in scope of `project-overview.md` — if not, stop and ask
- [ ] Name the PRIMARY SKILL for the feature (§7) — load its SKILL.md now

**Session end:**

- [ ] Feature at Definition of Done (§17) — or exact resume state recorded
- [ ] Three living files updated
- [ ] Committed and pushed
- [ ] Any skill gaps or conflicts recorded in `progress-tracker.md` for library maintenance (§15)

---

## 17. Definition of Done — Feature

A feature is done when ALL of these are true:

- [ ] Code implemented per `code-standards.md`, using the primary skill's templates as the starting point
- [ ] The phase's eval gate passed — result recorded in `progress-tracker.md`
- [ ] `tool-registry.md` / `prompt-registry.md` updated for anything touched
- [ ] `progress-tracker.md` updated: checklist, decisions, caveats, skills used
- [ ] Lint / typecheck / build green
- [ ] Committed with the standard message format and pushed

---

## 18. Failure Handling

- A failed correction gets exactly one retry with an adjusted approach. If the same problem persists after one corrective attempt — stop, re-read `graph-design.md` and the primary skill's relevant reference, and report back with what you found instead of thrashing.
- A failing eval gate is information, not an obstacle. Record what failed and why in `progress-tracker.md` before fixing anything.
- If the code and the context files have drifted apart (code reality vs. declared reality), fix the context files first, commit that, then continue.
- If a required decision is missing from the context files, do not assume — ask the user.
- If a skill's guidance and observed framework behavior conflict, trust the observed behavior, record the conflict in `progress-tracker.md`, and file the skill update (§15). Stale gotchas are worse than none.
