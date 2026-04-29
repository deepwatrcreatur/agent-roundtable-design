## Round 11 — Q19: Agent Orchestration Frameworks Survey

**IC prompt (Round 11, 2026-04-28):**

Q19 asks agents to survey the multi-agent orchestration framework landscape — Symphony, LangGraph, AutoGen/AG2, CrewAI, Temporal, and Jido 2.0 — and identify what our orchestrator should borrow and what it should avoid.

Sub-questions:
- Q19.1 — What is Symphony? (identify the right project, architecture, deployment breadth)
- Q19.2 — Patterns worth borrowing (checkpointing, graph state machines, agent roles, HITL, observability)
- Q19.3 — Patterns to avoid (prompt injection via shared state, context inflation, over-engineering, API coupling)
- Q19.4 — Jido 2.0 fit assessment (durability, observability, coordination, community)
- Q19.5 — Top 2–3 concrete borrowings specific to our current system

Agents: research with current sources. Star counts, last-commit dates, and architecture references expected. Mark each sub-question.


---

## Round 11 — Q19 Agent Positions (summary)

**Codex (Q19.1–Q19.5):** Symphony = openai/symphony (April 2026, 17.1k★, Python-based CLI+workspace coding orchestrator, not Elixir). Borrow: checkpointable round state (Symphony/Temporal), explicit state machine (LangGraph idea not runtime), HITL interrupt with approve/edit/reject, OTEL-shaped spans now, limited IC role hierarchy. Avoid: shared-thread context inflation, rigid state machine nodes (OpenAI's own blog says doesn't work well), replay-unsound side effects, LLM API coupling, prompt injection via shared state. Jido assessment: right runtime substrate, but durability must be built explicitly — Jido has no Temporal-style workflow journal. Top 3: (1) Roundtable.RoundRun persisted state per issue, (2) explicit discussion state machine phases, (3) OTEL spans before complexity grows.

**Gemini (Q19.1–Q19.5):** Also identified openai/symphony but claimed it's Elixir/OTP based with named role taxonomy (Conductor/Composer/Score/Performer/Checker) — this is unverified and inconsistent with Codex's direct GitHub/blog citation; not accepted. Good pattern survey: context distillation step to prevent inflation, strict isolation of instruction vs. blackboard content in prompts. Jido assessment: claimed "native persistence layer" — not accepted (overstated vs. hexdocs evidence). Top 3: (1) promote Orchestrator to Jido.Agent, (2) Conductor selector pre-stage, (3) directed routing allowing IC to re-invoke specific agent.

---

## IC Synthesis — Q19 (Round 11, 2026-04-28)

**IC: Claude**

### Resolved fact conflicts

**Symphony architecture:** `openai/symphony` is a Python-based, always-on coding-work orchestrator (tracker-driven dispatch, per-ticket workspaces, CLI agent runners). OpenAI blog April 27 2026; 17.1k stars. Gemini's Elixir/OTP claim and the `symphony-framework-arch` URL are not corroborated and are not accepted. This does not diminish Gemini's Q19 pattern analysis, which remains substantively useful.

**Jido durability:** Jido provides OTP supervision, telemetry, and parent/child process semantics. It does not provide a durable workflow journal comparable to Temporal. Persistence must be built explicitly in Roundtable using GitHub Issues + git artifacts as the external state. Codex's assessment is accepted.

### Q19.1 — What is Symphony: closed

`openai/symphony` is the relevant project: a polling-based, always-on service that monitors an issue tracker, creates isolated workspaces per issue, runs a coding agent per workspace, and controls workflow policy via in-repo `WORKFLOW.md`. Its components are Workflow Loader, Issue Tracker Client, Orchestrator, Workspace Manager, Agent Runner, and Logging. 500% PR landing increase claimed internally at OpenAI.

Relevance to us: Symphony's architecture validates our GitHub Issues + per-question runner model. It is not a deliberation framework; it is a coding-work queue. Relevant to our v2 implementation loop; partially relevant to v1 discussion loop for the reconciliation and workspace patterns.

**Q19.1: [satisfied]**

### Q19.2 — Patterns worth borrowing: closed

Three patterns have strong consensus across both agents and framework evidence:

**a) Explicit discussion state machine (highest priority)**
Replace the recursive round loop with named phases:
`:awaiting_turns → :triage_missing_markers → :consensus_check → :closed | :needs_human_review`
This borrows the LangGraph insight — model lifecycle as explicit transitions — without importing LangGraph. Makes the LiveView dashboard legible and makes restart recovery tractable.

**b) Checkpointable round state per issue (high priority)**
`Roundtable.RoundRun` struct persisted per GitHub Issue: current phase, expected speakers, completed speakers, last seen comment IDs, parsed satisfaction states, retry counter. On boot, reconcile from `gh issue view --json`. Symphony and Temporal both validate this pattern. Enables crash recovery without data loss.

**c) OTEL-shaped spans now (medium priority, high leverage before complexity grows)**
Emit structured events for: issue poll, prompt build, CLI invocation, gh comment post, marker parse, IC triage, consensus decision, escalation. Jido gives the telemetry substrate; we need the roundtable-specific schema. AutoGen/AG2 now emit this shape. Do not wait.

Additionally: HITL as a first-class interrupt state (not just a terminal label) is the fourth borrowing, warranted once the LiveView action surface has approve/edit/reject buttons.

**Q19.2: [satisfied]**

### Q19.3 — Patterns to avoid: closed

All five of Codex's patterns hold; Gemini's additions are compatible:
- **Context inflation:** prompt bounded to issue body + last N comments + current labels + optional IC summary. Never full history replay.
- **Rigid state machine micromanagement:** orchestrator owns lifecycle, not prose generation decisions.
- **Replay-unsound side effects:** `gh issue comment/edit/close` must have idempotency checks before retry.
- **LLM API coupling:** RunCliAgent boundary stays. Do not import model SDKs into the orchestration core.
- **Prompt injection via shared state:** immutable orchestrator prefix, clearly delimited quoted issue context, no agent-written text treated as executable policy.

**Q19.3: [satisfied]**

### Q19.4 — Jido fit: closed

Jido is the right substrate. Genuine strengths: structured telemetry, OTP supervision, parent/child agent semantics, `SpawnAgent` for future worker roles. Genuine gap: no durable workflow journal. We must build `Roundtable.RoundRun` persistence ourselves, using GitHub Issues as the authoritative external state.

Do not replace Jido. Do not add Temporal or LangGraph as a runtime. Borrow their *architectural patterns*, implement them in Elixir on top of Jido.

**Q19.4: [satisfied-conditional: Jido is right if we explicitly build RoundRun persistence and the explicit state machine; without those, the runtime alone does not provide durability]**

### Q19.5 — Concrete recommendations: closed

Ordered by priority:

1. **`Roundtable.RoundRun` — persisted state per issue** (before any other extension)
   Fields: `issue_number`, `phase`, `expected_speakers`, `completed_speakers`, `last_comment_ids`, `satisfaction_map`, `retry_count`. Persisted to a local ETS table + periodic flush to a `state/` git-tracked directory. On restart: reconcile from `gh issue view`. This is the foundation everything else builds on.

2. **Explicit phase state machine in `Roundtable.Orchestrator`**
   Replace recursive `do_rounds/7` with named phase transitions. Each phase function is pure (takes `RoundRun`, returns next `RoundRun` + list of effects). Effects are applied separately, making them testable and replay-safe.

3. **OTEL span schema for roundtable events**
   Define the span taxonomy now: `roundtable.issue.poll`, `roundtable.agent.turn`, `roundtable.gh.comment`, `roundtable.satisfaction.parse`, `roundtable.ic.triage`, `roundtable.consensus.check`, `roundtable.issue.close`. Wire into Jido telemetry. Export to OTEL collector (or log to stdout in dev). This enables the LiveView dashboard to show structured progress rather than raw event strings.

The Conductor pre-stage (Gemini Q19.5 #2) and directed IC routing (Gemini Q19.5 #3) are valid future extensions once the state machine and persistence are in place. They are not v1.

**Q19.5: [satisfied]**

### Protocol Update 7 — Orchestrator structural improvements

- `Roundtable.RoundRun` struct to be implemented (items 11 or 12 to be created as work items)
- `Roundtable.Orchestrator` to be refactored to explicit phase state machine
- OTEL span taxonomy defined; wired via Jido telemetry
- LiveView dashboard to expose phase + speaker progress (not just flash events)
- HITL interrupt state (`needs_human_input`) added to phase machine with LiveView approve/dismiss action

---

## Correction — Q19.1 Symphony Architecture (2026-04-28, post-synthesis)

**The IC synthesis above incorrectly characterised Symphony as Python-based. This is a factual correction based on post-round primary source verification.**

Codex re-checked against the OpenAI blog post published **April 27, 2026** (the day before the Q19 round) and the current `openai/symphony` GitHub repo:

- **Symphony the concept:** a language-agnostic orchestration spec (`SPEC.md`).
- **Symphony the reference implementation:** written in **Elixir**. The OpenAI post states explicitly: *"The reference implementation is written in Elixir"* and *"For the reference implementation, we chose Elixir..."* The repo language breakdown is currently **Elixir 95.5%, Python 3.0%**.
- **Why Elixir:** OpenAI cited the same properties that inform Calder's preference — concurrency primitives and OTP supervision.

**Consequence for the Q19 synthesis:**
- Gemini's Q19.1 claim (Elixir-based) was correct on implementation language. The IC's rejection of that claim was wrong and is retracted.
- The Conductor/Composer/Score/Performer/Checker role taxonomy Gemini attributed to Symphony remains unverified — that detail is not yet confirmed against primary sources.
- The Q19.1 substance (Symphony validates our GitHub Issues + per-question runner model) is unchanged.
- Symphony is now a **directly relevant Elixir reference architecture**, not a tangential Python comparison. Its workspace isolation, reconciliation on boot, and WORKFLOW.md policy patterns are worth studying before extending the orchestrator further.

**Codex verdict quote:** *"My earlier summary in the roundtable notes treated Symphony as Python-based; that is now contradicted by the current OpenAI post and repo."*

Protocol Update 7 work items (11, 12, 13) are unaffected — they were derived from patterns, not from Symphony's language choice.

---

