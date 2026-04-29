# Decision: Autonomous Roundtable Orchestrator Architecture

**Date:** 2026-04-26
**Status:** Closed
**Evidence:** `ACTIVE_DISCUSSION.md` — six questions, three agents (Codex ×3,
Gemini ×3, IC ×3), four rounds

---

## Summary

Build a roundtable orchestrator **on top of Jido 2.0** in Elixir, packaged as
a Nix flake app. Active per-question discussion lives in GitHub Issues;
durable artifacts (BRIEF, DECISION, transcripts) stay in git. Agents are
invoked as CLI subprocesses; the orchestrator owns all GitHub side effects.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  Nix flake app  (`roundtable`)                       │
│  ┌───────────────────────────────────────────────┐   │
│  │  Jido.Agent  — Orchestrator                   │   │
│  │  cmd/2: round state in → updated state +      │   │
│  │          directives out                        │   │
│  │                                                │   │
│  │  Actions           Signals                     │   │
│  │  ├ RunCliAgent     ├ QuestionCommentPosted     │   │
│  │  ├ GhIssueView     ├ QuestionSatisfied         │   │
│  │  ├ GhIssueComment  ├ QuestionNeedsMoreEvidence │   │
│  │  ├ GhIssueLabel    └ RoundTimedOut             │   │
│  │  └ GhIssueClose                                │   │
│  │                                                │   │
│  │  Directives (side-effect descriptors)          │   │
│  │  ├ PostIssueComment                            │   │
│  │  ├ SetIssueLabels                              │   │
│  │  ├ CloseIssue                                  │   │
│  │  └ ScheduleNextTurn                            │   │
│  └───────────────────────────────────────────────┘   │
│                                                       │
│  Domain modules (not provided by Jido)                │
│  ├ Roundtable.Satisfaction   — label/marker logic     │
│  ├ Roundtable.Scheduler      — round-robin policy     │
│  ├ Roundtable.Prompt         — BRIEF + issue → prompt │
│  └ Roundtable.Gh             — System.cmd gh wrappers │
└─────────────────────────────────────────────────────┘

Shared state
├ GitHub Issues   — active per-question discussion
│  ├ Comments     — signed agent positions
│  ├ Labels       — satisfied / needs-more-evidence / satisfied-conditional
│  └ Open/closed  — question lifecycle state
└ Git repo        — BRIEF.md, DECISION.md, ATTRIBUTION.md, transcripts
   └ ACTIVE_DISCUSSION.md  — index: Q# → issue number, orchestration rules
```

---

## Decisions

### Foundation — Jido 2.0

Use `{:jido, "~> 2.0"}` as the runtime foundation. Do not roll a custom
`GenServer` + `System.cmd/3` inline loop.

- Each CLI agent invocation is a `Jido.Action` (`RunCliAgent`).
- State transitions (`continue → satisfied → closed`) are the orchestrator
  agent's `cmd/2` function — pure, unit-testable without running any LLMs or
  hitting GitHub.
- Side effects (`gh issue comment`, `gh issue edit`, `gh issue close`) are
  `Directive`s executed by the Jido runtime, not inline in action logic.
- `Jido.Signal`s carry events between the orchestrator and any future
  sub-agents or watchers.
- OTP supervision and fault tolerance are inherited from Jido's runtime — no
  custom supervisor tree needed for v1.

**Defer `jido_ai` to v2.** IC triage in v1 uses a raw `claude -p` CLI call
via `RunCliAgent`, same as participant invocations. `jido_ai` becomes relevant
if an Elixir-native selector, summarizer, or fallback judge is needed later.

### Shared State — Hybrid

| Content | Medium | Rationale |
|---|---|---|
| Active per-question turns | GitHub Issues | Atomic writes, no merge conflicts, parallel-agent safe |
| Question satisfaction state | Issue labels | Machine-readable without prose parsing |
| Question lifecycle | Issue open/closed | Natural termination signal |
| BRIEF, DECISION, ATTRIBUTION | Git-tracked files | Durable, reviewable, version-controlled |
| Session index (Q# → issue#) | `ACTIVE_DISCUSSION.md` in git | Stable pointer; not written during live rounds |
| Transcripts/archives | Git-tracked files | Persist after issues close |

Do not write agent positions to `ACTIVE_DISCUSSION.md` during live rounds.
That file is updated by the orchestrator only when opening or closing a
discussion, not turn-by-turn.

### Turn Protocol — Round-Robin + IC Close

- Fixed agent order within a question: `[codex, gemini, claude_ic]`.
  `claude_ic` runs last as the IC; it synthesises, issues the next prompt,
  and decides whether to continue or close.
- Questions can run in parallel across GitHub Issues using
  `Task.async_stream/3` (or Jido's equivalent `await_all`). Questions with
  declared dependencies must be sequenced; independent questions are parallel.
- One round = all agents have commented once on a given question.

### Termination — Labels Primary, IC Triage Fallback

1. After each agent posts, the orchestrator reads
   `gh issue view <n> --json labels,state,comments`.
2. If all active agents have a `satisfied` or `satisfied-conditional` label
   and none have `needs-more-evidence`, the issue is closed.
3. If an agent's response contains no detectable satisfaction marker, the
   orchestrator invokes `claude_ic` with a triage prompt: *"Does the latest
   comment on this question indicate satisfaction? Reply: satisfied /
   satisfied-conditional / needs-more-evidence."* The IC response sets the label.
4. `max_rounds` (default 5) reached without consensus → orchestrator posts a
   summary comment, adds a `needs-human-review` label, leaves the issue open,
   and exits.

### CLI Agent Invocation

The orchestrator, not the agent, handles all GitHub mutations. Agents only
produce prose.

```
# Prompt construction
prompt = Roundtable.Prompt.build(brief_path, issue_json, agent_role)

# Invocation (per-agent headless flags confirmed against installed versions)
claude 2.1.83:    claude -p --output-format json  <<< prompt
codex 0.116.0:    printf prompt | codex exec - --json --output-last-message /tmp/out
gemini 0.35.0:    gemini -p prompt --output-format json

# Posting (orchestrator side effect)
gh issue comment <n> --body-file <rendered_response>
gh issue edit    <n> --add-label satisfied --remove-label needs-more-evidence
gh issue close   <n>
```

**Q1 caveat (Codex):** headless flags and auth preconditions confirmed
locally. One live end-to-end scripted run per agent is still needed to
characterise output truncation/edge cases before hardening the parser.
This is a production-hardening item, not a blocker for the v1 scaffold.

### Implementation Form — Elixir/Mix + Nix Flake

- `mix new roundtable --sup` with `{:jido, "~> 2.0"}` in `mix.exs`.
- Entry point: `mix run -e 'Roundtable.CLI.main(["docs/design/BRIEF.md"])'`
- Nix flake provides a `devShell` with Elixir + Erlang + `gh` + the three
  CLI agents. A `packages.default` app output wraps `mix run` as `roundtable`.
- No Docker, no external database, no message broker.

---

## What Needs Building (Domain Pieces)

Jido provides the runtime. These modules are project-specific:

| Module | Responsibility |
|---|---|
| `Roundtable.Actions.RunCliAgent` | `System.cmd/3` wrapper for claude/codex/gemini |
| `Roundtable.Actions.Gh.*` | `gh issue view/comment/edit/close` wrappers |
| `Roundtable.Prompt` | Assemble BRIEF.md + issue JSON + agent role → prompt string |
| `Roundtable.Satisfaction` | Parse/apply label policy from agent response |
| `Roundtable.Scheduler` | Round-robin agent order, dependency-aware question sequencing |
| `Roundtable.CLI` | Entry point: parse BRIEF, create/load issues, run rounds |

---

## What Is Deferred

- `jido_ai` integration — defer to v2; IC triage uses raw `claude -p` in v1
- Filesystem-only offline fallback mode — defer; GitHub Issues is the primary path
- Parallel question execution — implement after sequential single-question
  proof-of-concept works end-to-end
- Automated issue creation from BRIEF.md — v1 may create issues manually;
  automate in v2
- `OpenCodeHarness` backend — defer to v2; enables GitHub Copilot and Opencode
  Go as first-class participants via `opencode serve` HTTP API
- `GitHubAPI` and `CodeStorage` git backends — defer to v2; `LocalGit` is
  sufficient for v1 finalization writes

## Open Questions (not yet decided)

**PR review as a coordination surface**

The current orchestrator design handles the discussion loop (Issues) and the
artifact write loop (git commits). It does not yet handle the implementation
loop: agent opens PR → review bot comments → agent addresses comments → PR
merges. This is a distinct event stream (PR review comments, CI checks, bot
feedback) that the orchestrator will eventually need to drive. Defer to v2;
the PR review loop requires `Roundtable.Actions.Gh` to be extended with
`pr_create`, `pr_view`, and `pr_comment` wrappers, and the Orchestrator
state machine to add a `:pr_review` state between `:round_in_progress` and
`:satisfied`.

**Q10.3 — Mid-discussion join context**

What compressed context does a new agent receive when joining a discussion that
is already in progress (e.g., round 3 of 5)?

Options:
- Full issue comment history (accurate but token-expensive at round 3+)
- Last N comments only, same as a turn prompt (cheap but loses early positions)
- IC-generated summary comment posted to the issue before the join turn (accurate,
  bounded, but requires an extra IC invocation)
- Current satisfaction state only: open questions + current labels, no prose

No decision made. Discover empirically when the orchestrator first attempts to
add a late-joining agent. The `Roundtable.Prompt` `join: true` path (item 05)
should leave this configurable rather than hardcoding one approach.

---

## Starting Point

```
mix new roundtable --sup
cd roundtable
# Add to mix.exs deps: {:jido, "~> 2.0"}
mix deps.get
# Implement in order:
# 1. Roundtable.Actions.Gh       — gh CLI wrappers (unit-testable with mock)
# 2. Roundtable.Actions.RunCliAgent — CLI agent invocation
# 3. Roundtable.Satisfaction     — label policy
# 4. Roundtable.Prompt           — prompt assembly
# 5. Roundtable.Orchestrator     — Jido.Agent with cmd/2 loop
# 6. Roundtable.CLI              — entry point
# 7. flake.nix                   — devShell + app wrapper
```

Build `Gh` actions first — they are the most testable (mock `System.cmd/3`)
and the most likely source of environment-specific surprises (auth, rate limits,
field names in `gh --json` output).

---

## Protocol Update 6 — Mobile Supervision Architecture (Q18, 2026-04-28)

**Decision:** Companion REST + SSE API is the canonical mobile contract; LiveView dashboard is the primary browser UI.

### Ruled out
- **LiveView Native** — archived February 10, 2026. Do not use.
- **OpenCode fork as primary path** — satisfaction labels and round triggering absent from OpenCode data model; upstream moves at ~1 release/day. Deferred to v2 if supervision scope expands.

### Required additions (v1)

**Push notifications:**
- Orchestrator emits HTTP POST to ntfy.sh on `consensus_reached` and `needs_human_review` events.
- Config: `NTFY_TOPIC` env var. When unset, notifications are silently skipped.
- ntfy.sh is the default backend (self-hostable, free iOS app). Pushover is an acceptable alternative via the same abstraction.

**Companion API (new module: `RoundtableWeb.ApiController`):**
```
GET  /api/state             — map of issue_number => state_map (same shape as get_discussion_state/1)
GET  /api/events            — SSE stream; events: agent_done, round_start, consensus_reached, needs_human_review
POST /api/questions         — body: {text: String} — calls inject_question/3
POST /api/rounds/trigger    — body: {} — calls start_discussion/2 in background Task
```
Authentication: bearer token in `Authorization` header; token set via `ROUNDTABLE_API_TOKEN` env var.

**PWA:**
- LiveView dashboard served with `manifest.json` (name, icons, display: standalone, start_url: /).
- iOS 16.4+ Web Push via Service Worker is the alerting path for home-screen installs.

### Mobile supervision feature classification
| Task | Mechanism | Real-time |
|---|---|---|
| Watch | SSE `/api/events` | Yes |
| Alert | ntfy.sh / Pushover push | Yes (out-of-app) |
| Inject | POST `/api/questions` | No |
| Trigger | POST `/api/rounds/trigger` | No |

Apple Shortcuts can drive Inject and Trigger against the companion API with no native app required.


---

## Protocol Update 7 — Orchestrator Structural Improvements (Q19, 2026-04-28)

**Decision:** Three concrete structural improvements derived from the agent orchestration framework survey (Symphony, LangGraph, AutoGen/AG2, CrewAI, Temporal).

### New work items to implement (items 11, 12, 13)

**Item 11 — `Roundtable.RoundRun` persisted state**
```elixir
%Roundtable.RoundRun{
  issue_number: pos_integer(),
  phase: :awaiting_turns | :triage_missing_markers | :consensus_check
        | :closed | :needs_human_review | :needs_human_input,
  expected_speakers: [atom()],
  completed_speakers: [atom()],
  last_comment_ids: [String.t()],
  satisfaction_map: %{atom() => atom()},
  retry_count: non_neg_integer()
}
```
Persisted to ETS + periodic flush to `state/` git-tracked directory. On restart, reconcile from `gh issue view --json labels,state,comments`.

**Item 12 — Explicit phase state machine in `Roundtable.Orchestrator`**
Replace recursive `do_rounds/7` with named phase transition functions. Each phase function is pure: takes `RoundRun`, returns `{next_run, [effect]}`. Effects (gh calls, CLI invocations) are applied separately. Makes phases testable and replay-safe.

Phase transitions:
```
:awaiting_turns
  → all expected speakers completed → :triage_missing_markers
  → max_rounds exceeded → :needs_human_review

:triage_missing_markers
  → all markers present → :consensus_check
  → IC triage completes → :consensus_check

:consensus_check
  → all [satisfied|satisfied-conditional], no [needs-more-evidence] → :closed
  → any [needs-more-evidence] → :awaiting_turns (next round)
  → max_rounds → :needs_human_review

:needs_human_input  (new — HITL interrupt)
  → operator approves/dismisses → resumes from suspended phase
```

**Item 13 — OTEL span taxonomy**
Define and emit spans for each orchestrator event:
```
roundtable.issue.poll       — gh issue view call
roundtable.agent.turn       — RunCliAgent invocation (includes agent, issue_number)
roundtable.gh.comment       — gh issue comment post
roundtable.satisfaction.parse — marker extraction from agent response
roundtable.ic.triage        — IC classification call
roundtable.consensus.check  — Satisfaction.consensus? evaluation
roundtable.issue.close      — gh issue close
```
Wire via Jido telemetry. Export to OTEL collector in prod; log to structured stdout in dev.

### LiveView dashboard updates (item 10 extension)
- Display current `RoundRun.phase` per question alongside satisfaction badge
- Show `completed_speakers` vs `expected_speakers` progress
- Add approve/dismiss button for `:needs_human_input` phase

### What was ruled out
- Importing LangGraph, Temporal, or AutoGen as runtimes — borrow patterns, not runtimes
- Conductor pre-stage (Symphony-style agent selector) — deferred to v2
- Directed IC routing (re-invoke specific agent) — deferred to v2
- Replacing Jido with another runtime — Jido is the right substrate

### Correction — Symphony is Elixir, not Python (2026-04-28)

The Q19 IC synthesis incorrectly characterised `openai/symphony` as Python-based. **Correction:**

- **Symphony the spec:** language-agnostic (`SPEC.md`).
- **Symphony the reference implementation:** written in **Elixir** (95.5% of repo). OpenAI's April 27, 2026 post explicitly states: *"The reference implementation is written in Elixir"* and cites concurrency and OTP supervision as the reason.
- Gemini's Q19.1 Elixir claim was correct; the IC's rejection of it was wrong and is retracted.
- Consequence: Symphony is a **directly relevant Elixir reference architecture**, not just a tangential comparison. Its workspace isolation, boot reconciliation, and `WORKFLOW.md` policy patterns are worth studying before extending the v2 implementation loop.
- Work items 11, 12, 13 (derived from architectural patterns, not implementation language) are unaffected.

---

## Protocol Update 8 — Coordinator Failover and Degraded-Mode Continuity (Q20 precondition, 2026-04-28)

**Decision:** The system must treat **coordinator/IC unavailability** as a
first-class orchestration failure mode, not an ad hoc human-relay event.

### Failure mode observed

During the Q20 handoff, the primary discussion leader became unable to
continue orchestration due to provider overload. The prompt already existed,
but round continuity depended on another agent informally noticing the stall,
taking over, and writing a continuity note.

That is operationally unsafe. A future autonomous orchestrator must detect
"leader stalled / provider degraded / synthesis not progressing" and either
resume via a standby coordinator or surface a structured human-review state.

### Required robustness additions

**Coordinator lease + heartbeat**
- Each active round has a recorded coordinator identity and lease timestamp.
- Coordinator writes a heartbeat on meaningful progress boundaries:
  prompt posted, agent position received, synthesis started, synthesis posted.
- If the lease expires without heartbeat, another eligible coordinator may take
  over and must record a continuity note.

**Resumable synthesis state**
- `Roundtable.RoundRun` must persist coordination metadata in addition to
  speaker progress:
  - `coordinator`
  - `coordinator_lease_expires_at`
  - `last_progress_at`
  - `suspended_phase`
  - `takeover_count`
- A takeover resumes from persisted state; it must not require rereading the
  full discussion from scratch to determine who already responded and what
  remains open.

**Degraded-mode phase**
- Add `:coordinator_unavailable` as an explicit orchestrator phase.
- Transition into it when coordinator retries/timeouts exceed threshold or
  provider health for the assigned coordinator is degraded.
- Exit paths:
  - standby coordinator takeover → resume prior suspended phase
  - human operator ack → `:needs_human_input`
  - repeated takeover failure → `:needs_human_review`

**Provider diversity for the coordinator role**
- The same provider family should not be a single point of failure for both
  participant reasoning and IC synthesis.
- Coordinator assignment must be swappable by config and by live takeover.

**Auditability**
- Every takeover must emit a structured event and write a durable continuity
  note to the discussion artifact.

### New work item

**Item 14 — Coordinator failover**
- standby coordinator selection policy
- lease/heartbeat persistence in `RoundRun`
- takeover transition rules in the phase state machine
- telemetry spans for takeover and degraded mode
- LiveView display of coordinator identity and degraded-state banner

---

## Protocol Update 9 — Epistemically Grounded Closure Rules (Q20, 2026-04-28)

**Decision:** Four protocol changes derived from philosophy of mind literature
(predictive processing, social epistemology, BonJour's coherentism critique)
to improve reality-testing and reduce collective confabulation.

### Theoretical grounding

The Q19 failure was a coherentist closure: the IC selected the more
internally consistent position without verifying against external reality.
BonJour's Isolation Objection (*The Structure of Empirical Knowledge*, 1985)
provides the precise diagnosis: coherence alone cannot constitute empirical
knowledge. Longino's uptake conditions (1990) explain why the IC's authority
structure failed: valid criticism (Gemini's Elixir claim) did not produce
uptake. Corlett/Fletcher's predictive-processing account of delusion explains
the mechanism: priors (Codex's plausible citation) were weighted above
prediction error (Gemini's disconfirming claim).

### Change 1 — Typed claim provenance (effective immediately)

Every contested factual claim should carry a basis tag:

- `[observed]` — agent directly ran the command, read the file, or fetched and quoted from the URL
- `[testimony]` — agent reports what a source or another agent said
- `[inferred]` — derived from other claims

A URL quote is `[testimony]` unless the question is specifically "what does
this source say?" A quote is not an observation of reality; it is an
observation of a text.

### Change 2 — IC evidence precedence rule (effective immediately)

`[observed]` > `[quoted testimony]` > `[testimony]` > `[inferred]`

When two `[testimony]` claims conflict, the sub-question must remain
`[needs more evidence]` until an `[observed]` claim resolves it. The IC may
not close a contested factual sub-question on majority testimony, narrative
coherence, or citation count alone.

### Change 3 — Disconfirmation pass when fast consensus (effective immediately)

If all agents mark a factual sub-question `[satisfied]` within 2 turns, the
IC must assign one agent a **disconfirmation pass** before closing: find one
`[observed]` piece of evidence that could contradict the consensus, or
explicitly state what was looked for and not found. This guards against
Jump-to-Conclusions closure.

### Change 4 — Brief premise challenge before design closure (next round onwards)

Before closing any design question, at least one agent must answer: *"What
if a key premise in the BRIEF's framing of this question is false? What would
change?"* The IC includes this as a required prompt step when a question is
near closure. Guards against shared delusion from a false BRIEF premise.

### What was ruled out

- DTD (`Draws to Decision`) counter tag — the disconfirmation pass achieves
  the structural goal without requiring agents to self-report an
  unverifiable count
- Replacing the discussion format — all four changes are protocol changes,
  not architectural ones

---

## Q24 — Messaging Gateway (Round 15, 2026-04-29)

**Decision:** Telegram is the recommended first messaging gateway. Email (IMAP/SMTP) as a
secondary option. iMessage, WhatsApp, Signal ruled out for server-side receipt reasons or
approval complexity.

Telegram bot ingests prompts → roundtable service authenticates by Telegram user ID
bound to GitHub identity in Authentik custom attributes. The Telegram channel
complements the LiveView UI (it handles prompt injection and notifications); the
LiveView UI remains the right home for discussion repo management, round history
browsing, and satisfaction-state visualization.

**Brief premise challenge resolved:** Messaging gateway is in-scope once the core
orchestrator is functional. It is explicitly deferred until after items 1-22 ship.

---

## Q25 — Authentication Strategy (Round 15, 2026-04-29)

**Decision:** Authentik as OIDC proxy for GitHub OAuth (option c). The roundtable
Elixir app is an OIDC relying party to Authentik; Authentik federates GitHub as a
social provider. This gives:
- Centralized session management and SSO across homelab services
- MFA available via Authentik without custom code in the app
- GitHub identity available as a claim in the OIDC token

GitHub OAuth scopes required: `read:user`, `user:email`, and `repo` (for checking
collaborator access to private discussion repos). Permission checks against the GitHub
API; no local authorization table required.

**Collaborator flow:** GitHub OAuth → Authentik → OIDC token to app → app queries
`GET /repos/:owner/:repo/collaborators/:username` to gate access per discussion repo.

**Brief premise challenge resolved:** GitHub unavailability during homelab-only access
is acceptable for a personal tool. VPN (Tailscale) as a fallback for LAN-only access
mitigates the risk.

---

## Q26 — Service Hosting (Round 15, 2026-04-29)

**Decision:** Homelab first. Fly.io documented as fallback only if external collaborator
access is needed without VPN.

Gigalixir noted as technically viable (Elixir-native, no sleep on free tier) but not
the primary recommendation given homelab infrastructure already present.

Vercel ruled out: serverless/edge model is incompatible with long-running OTP processes
and LiveView WebSockets.

**Rationale:** See Q31 for updated analysis with full homelab context.

---

## Q27 — Discussion Repo Discovery (Round 15, 2026-04-29)

**Decision:** `roundtable.toml` existence is the canonical identity signal. GitHub
topic `roundtable-discussion` is an optional convention for public-facing discoverability.
"Paste owner/repo slug" is the primary day-one UX for repo registration.

Auto-discovery (scanning user repos for `roundtable.toml`) deferred until a concrete
multi-user need exists. Rate limit concern: O(N) existence checks are acceptable for
a user with <100 repos; pagination + caching mitigates for larger accounts.

---

## Q28 — GitHub Alternatives (Round 15, 2026-04-29)

**Decision:** SourceForge ruled out. GitHub remains the sole supported platform.
`DiscussionRepo.Backend` behaviour provides the abstraction point for future
Forgejo/Gitea support if needed.

SourceForge: declining platform, no modern OAuth API, reputation damage from 2015 adware
incident. Ruled out explicitly — will not resurface.

Forgejo: GitHub-compatible REST API surface; a `Roundtable.Adapters.Forgejo`
implementation would require minimal changes beyond `Roundtable.Adapters.GitHub`.
Not a current priority.

---

## Q29 — Discussion Repo Co-evolution (Round 16, 2026-04-29)

**Decision:** Standalone discussion repos are the default. Embedded (`docs/discussion/`)
is a valid opt-in for solo/public projects. Architecture supports both via the existing
`source` detection in `CLI.start_discussion/2`.

**Retrofit convention adopted:**
- Round 0 file: `round-00-retrofit-snapshot.md` with header
  `# Retrofit: Current State at <commit>`
- BRIEF.md opens with a retrofit notice block
- Service root: `DISCUSSION_REPO.md` (standalone only)

**`roundtable.toml` v1.1 additions:**
```toml
embedded = false
service_repo = ""
service_commit_at_start = ""
retrofit = false
```

---

## Q30 — Collaborator Permission Scoping (Round 16, 2026-04-29)

**Decision:** Service-mediated contributions (LiveView injection, Telegram bot) are
the primary model. The service acts as the GitHub write principal; discussion
contributors authenticate to the service, not directly to the repo. This makes
GitHub write access optional for discussion contributors.

For standalone public discussion repos: any GitHub user can read; the service gates
writes. For private standalone repos: `read` collaborator access is sufficient.

GitHub Discussions ruled out as round medium: content is not in git history and
cannot be forked (violates Q23 forkability requirement).

GitHub Organizations not required for solo+occasional-collaborator use case.

---

## Q31 — Homelab Infrastructure Revisit (Round 16, 2026-04-29)

**Decision:** Q26 recommendation confirmed and strengthened. Homelab is the correct
first-deploy target given:
- `deepwatercreature.com` publicly routable domain
- `router` machine running Caddy handles TLS automatically (Let's Encrypt) and
  WebSocket proxying for Phoenix LiveView without special config
- Unified `unified-nix-configuration` NixOS flake: adding the service is ~20 lines
  of NixOS module + one Caddy virtual host block
- NixOS `nixos-rebuild switch` is already the owner's deployment workflow; `fly deploy`
  adds a Docker build step with no net simplification for this user

**Caddy WebSocket note:** `reverse_proxy localhost:4000` forwards WebSocket upgrade
headers by default. Phoenix requires `PHX_HOST` set to the public hostname to prevent
CSRF errors — this is a one-line NixOS module config.

**Remaining open decisions (conditions for full Q31 closure):**
1. Public (`roundtable.deepwatercreature.com`) vs. VPN-only for external collaborator access
2. `PHX_HOST` value confirmed in NixOS module

**Fly.io:** Retained as documented fallback only if public access without VPN is required.

### Protocol Update 12 — Co-evolution and Deployment Conventions

See ACTIVE_DISCUSSION.md Round 16 for full text. Summary:
1. Standalone-first with embedded opt-in; `embedded` field in `roundtable.toml`
2. Retrofit conventions: `round-00-retrofit-snapshot.md`, retrofit BRIEF notice, `DISCUSSION_REPO.md`
3. Service-mediated contributions as primary path for discussion contributors
4. Homelab (NixOS + Caddy) confirmed as first-deploy target
5. `roundtable.toml` schema v1.1 additions: `embedded`, `service_repo`, `service_commit_at_start`, `retrofit`

---

## Q32 — Protocol Self-Assessment (Round 17, 2026-04-29)

**Three structural flaws identified:**

1. **Anchoring** — agents speak sequentially and see prior responses; later agents
   anchor to earlier positions (Delphi literature)
2. **IC circular authority** — IC deliberates, triages, and closes unilaterally;
   no independent check on IC's framing or closure decision
3. **`[satisfied]` marker conflates genuine agreement with exhausted opposition**;
   creates misleading unanimity in the record

**What is preserved:** premise challenge requirement (Protocol 11); typed claim
provenance `[observed]`/`[testimony]`/`[inferred]` (Protocol 9).

### Protocol Update 13 — Structural Corrections

**Adopted immediately:**

**New marker: `[no objection]`**
Meaning: "No further evidence; not blocking closure; not asserting this is the
best answer." Distinguishes from `[satisfied]` (active agreement) and
`[needs more evidence]` (blocking). DECISION.md flags consensus formed primarily
from `[no objection]` as "convergent but not robust."

**Explicit warrant for contested `[inferred]` claims**
When agent A contests agent B's `[inferred]` claim: A must state the assumed
warrant and why it fails. IC must adjudicate the warrant dispute in the synthesis.

**Adopted in principle, deferred to first production run:**

**Challenger role at IC closure**
One agent (the first speaker that round, rotating) must articulate the strongest
alternative to the IC's proposed synthesis and rate it Low/Medium/High plausibility.
Medium or High triggers an additional round before closure.

**Deferred pending empirical evidence:**

**Blind first sub-turn** (Delphi-inspired): each agent writes initial position
without seeing others'; positions shared; second sub-turn follows. Adopted only
if anchoring is empirically present in first real agent run.

### Discourse Literature Incorporated

- Delphi method → blind first sub-turn proposal (deferred)
- Toulmin (1958) → explicit warrant for contested inferences (adopted)
- ODNI SAT/ACH → Challenger role at closure (deferred)
- Fishkin deliberative polling → "missing perspectives" IC prompt (future work item)
- Habermas ideal speech → typed provenance already addresses sincerity claim

### Q32 Addendum — Organizational Behaviour Literature (Round 17, 2026-04-29)

**Key finding:** LLMs inherit human group dynamics through training on group
outputs (committee reports, meeting summaries, consensus documents) — not just
individual writing. The sycophancy literature provides direct empirical support.
Structural corrections motivated on LLM-specific grounds.

**Adopted (Protocol Update 13 Addendum, all protocol-only):**

**IC mindguard check**
IC synthesis opens by quoting or closely paraphrasing the strongest dissenting
position. If no dissent: "No dissenting position recorded in this round."
Prevents silent absorption of minority views. *Source: Janis (1972) groupthink.*

**Double-loop framing check at question round 1**
IC prompt at first round of each question: *"What framings would lead to
different sub-questions than those in the BRIEF?"* Structural (not
agent-triggered) double-loop check against motivated BRIEF framing.
*Source: Argyris (1990) skilled incompetence + defensive routines.*

**Unique contribution prompt appendix**
Agent prompts end: *"Include at least one consideration not already raised
this round, or explicitly state you cannot identify one."*
Targets information sampling bias; "no unique contribution" is a useful
low-information-entropy signal. *Source: Stasser & Titus (1985).*

**Blind first sub-turn: re-prioritised**
Moved from "deferred" to "implement and evaluate at first real agent run."
Procedural context-conditioning (LLM-specific) is sufficient justification
independent of social anchoring. *Source: Delbecq & Van de Ven (1971) NGT.*

**Unchanged:** Challenger role at closure (Mason & Mitroff 1981 devil's advocacy
confirmed as correct scope — per-turn devil's advocacy would produce dialectical
inquiry, which is less effective for non-adversarial settings).

---

## Q33 — Adding DeepSeek as a Roundtable Agent (Round 18, 2026-04-29)

**Decision:** Add `:deepseek` (DeepSeek-V3) as a **fourth agent** via direct
Elixir HTTP (`Req`), not via `opencode` CLI. Default roster becomes
`[:codex, :gemini, :deepseek, :claude_ic]`.

### Model

- Regular agent turns: `deepseek-chat` (V3) — ~$0.0003/turn, fast, structured
- Challenger role at closure: `deepseek-reasoner` (R1) — activated at first
  production run; configurable via `:deepseek_model` option

### Invocation

Direct Elixir HTTP in `RunCliAgent`. `run/2` detects `:deepseek` and calls
`run_deepseek/2` via `Req`. Response text returned as `%{stdout: text}` —
no `extract_text/2` changes needed. This removes the CLI dependency for
DeepSeek; works in dev, CI, and production with only `DEEPSEEK_API_KEY` set.

### Role in roster

Fourth agent, speaks before IC. `@agent_roles` description: *"You are DeepSeek,
an AI agent developed by DeepSeek AI. Bring independent analytical perspective
from a distinct training distribution. Focus on rigorous reasoning and
considerations potentially underweighted by agents trained on primarily
English-language corpora."*

**Noted alternative** (not adopted; revisit after first run): replace `:codex`
with `:deepseek` for cost efficiency if Codex/DeepSeek position diversity proves low.

### Key management

`DEEPSEEK_API_KEY` env var. Homelab: agenix secret + NixOS module `EnvironmentFile`.
Dev: `.env` / shell export.

### Protocol Update 14

See ACTIVE_DISCUSSION.md Round 18. Changes to implement (item 27):
1. `RunCliAgent`: add `:deepseek` to schema + `run_deepseek/2` HTTP handler
2. `Orchestrator`: add `:deepseek` to `@default_agents` + `@agent_roles`
3. NixOS module (item 26): add `deepseekApiKeyFile` option
