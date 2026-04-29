# Design Brief: Autonomous Agent Roundtable Orchestrator

**Status:** OPEN — awaiting agent positions before implementation
**Owner:** Calder (human)
**IC:** Claude (this session)
**Participants:** Codex, Gemini-CLI

---

## Context

We have a proven multi-agent discussion format (the blackboard model from
`nix-agent-guides/guides/agentic-orchestration/`) and a satisfaction protocol
(`[satisfied]` / `[satisfied-conditional]` / `[needs more evidence]` markers
per question). The format has produced better decisions than single-agent
analysis in several design discussions.

The bottleneck is human: a person must copy each agent's output, paste it into
the shared file, then copy the next prompt and trigger the next agent. We want
to remove that bottleneck entirely.

This repo is the design and implementation home for a thin orchestrator that
drives the discussion format autonomously.

## Prior Art Survey

Before proposing designs, agents should understand what exists:

- **[claude_code_bridge](https://github.com/bfly123/claude_code_bridge)** — tmux-based
  daemon. Agents run side by side in split panes; point-to-point and broadcast
  dispatch. Designed for real-time collaborative coding, not structured deliberation.
- **[Claude-Code-Workflow](https://github.com/catlog22/Claude-Code-Workflow)** — JSON
  workflow definitions, event-driven beat model, message bus. Team workers execute
  tasks in phases. Coding workflow tool.
- **[AutoGen SelectorGroupChat](https://microsoft.github.io/autogen/dev//user-guide/agentchat-user-guide/selector-group-chat.html)** —
  Python framework. Agents are API-callable LLM wrappers; a selector model picks
  the next speaker; termination conditions are explicit Python predicates.
- **[Multi-Agent Debate (MAD)](https://github.com/Skytliang/Multi-Agents-Debate)** —
  structured debate rounds. "Tit for tat" correction; early termination on
  consensus. Academic tool for factuality; works via API.
- **[DebateLLM](https://github.com/instadeepai/DebateLLM)** — benchmarking multi-agent
  debate; termination on agreement convergence check after each round.

### Later discoveries (added after initial agent round)

- **[Squad](https://github.com/bradygaster/squad)** — repo-native multi-agent
  system built on GitHub Copilot. Agents (frontend, backend, tester, lead) live
  in the repo as files. A `decisions.md` file acts as an asynchronous bulletin
  board — agents write decisions after completing work, others read before
  starting. Thin coordinator routes tasks; each agent gets full repository
  context. Key observation: Squad deliberately chose committed markdown files
  over GitHub Issues as the shared state medium and has production experience
  with this choice. Understand why before finalising Q5.

- **[MassGen](https://github.com/massgen/MassGen)** — terminal multi-agent
  scaling system. Multiple frontier models (Claude, Gemini, GPT, Grok) work in
  parallel; each sees other agents' latest answers; agents vote for an existing
  answer or produce a new one; coordination continues until all agents vote.
  This is the closest existing implementation of our satisfaction protocol —
  voting-to-consensus rather than satisfaction markers, but the termination
  model is the same. Python; in-memory; no GitHub integration.

- **[Jido 2.0](https://github.com/agentjido/jido)** — production Elixir agent
  framework by Mike Hostetler (`@mikehostetler`). Built on OTP/GenServer.
  Core primitives: `Actions` (reusable units of work, pure functions),
  `Signals` (CloudEvents-based messaging between agents), `Directives` (typed
  effect descriptors the runtime executes — side effects are never inline),
  `cmd/2` (single entry point: actions in, updated agent + directives out).
  Ships a DAG-based workflow planner, 25+ pre-built tools, MCP support,
  OpenTelemetry observability, and an opt-in `jido_ai` package for LLM
  integration with ReAct/CoT/ToT reasoning strategies.
  **This directly covers what agents proposed building from scratch.** Before
  designing a custom GenServer orchestrator, agents must assess whether Jido
  Actions + Signals + Directives is the right foundation. Available on
  Hex.pm as `jido`; docs at hexdocs.pm/jido.

## Constraints

The orchestrator must:

1. Work with the CLI tools already installed: `claude` (Claude Code), `codex`
   (OpenAI Codex CLI), `gemini` (Google Gemini CLI).
2. Use the filesystem as shared state — the blackboard model. No message broker,
   no database, no persistent daemon required.
3. Be simple enough to run with `./roundtable.sh <brief.md>` or equivalent.
4. Detect the satisfaction protocol markers in the discussion file to know when
   to continue vs. close a round.
5. Not require a human between rounds.

Nice-to-have:
- Pluggable agent list (add/remove agents without rewriting orchestrator logic)
- Works as a GitHub Action (trigger on push to `BRIEF.md`, outputs DECISION.md)

## Owner Preference: Elixir / BEAM

The project owner (Calder) has a stated preference for Elixir and the BEAM
ecosystem over Python and TypeScript as the implementation platform, all else
being equal. Agents should consider this seriously — not treat it as a soft
hint to politely acknowledge and then ignore.

Reasons this preference is technically relevant to the problem:

- **Process model**: BEAM processes map naturally onto agents — each agent
  invocation can be a supervised `Task` or `GenServer`. Supervisor trees
  provide fault tolerance if an agent subprocess crashes or times out.
- **Concurrency**: multiple agents could respond in parallel (`Task.async_stream`)
  rather than sequentially, without added complexity.
- **Message passing**: OTP's built-in message passing is a natural fit for
  the blackboard model's "whose turn is it" signalling.
- **Port / System.cmd**: Elixir's `System.cmd/3` and `Port` module handle
  subprocess invocation of CLI tools cleanly.
- **Mix + Nix**: an Elixir Mix project packages straightforwardly as a Nix
  flake with `beam.packages.erlang.elixir` in the devShell.

Agents should weigh these properties honestly against the alternatives. If
Python, shell, or another platform is clearly superior for the specific
constraints of this problem, say so with evidence. The goal is the best
automated discussion system, not validation of the owner's preference.

---

## Design Questions

### Q1 — CLI Agent Invocation (blocking)

Each of `claude`, `codex`, and `gemini` has a different CLI interface for
non-interactive / headless use. Specifically:

- What flag or mode makes each agent read a prompt from stdin or a file and
  write its response to stdout without interactive prompting?
- What is the best way to inject the current discussion file as context — via
  stdin, as a file argument, or via a system-prompt flag?
- Are there token limit / output truncation concerns that differ by agent?
- Does any agent require authentication setup that affects headless invocation?

Research the actual CLI interface for each agent and propose a unified
invocation pattern (or document the per-agent differences).

### Q2 — Turn Protocol and Orchestrator Architecture (blocking)

Once we can invoke agents headlessly, we need to decide how the orchestrator
sequences them:

- **Option A — Round-robin with IC close:** Agents run in fixed order (e.g.
  Codex → Gemini → IC). IC reads after each full round, decides whether to
  continue or close. Simple, predictable.
- **Option B — File-signal directed:** Each agent's response ends with a
  `next_speaker: <agent>` field. The orchestrator reads this and routes to the
  named agent. Agents control the sequence. More flexible; also more fragile.
- **Option C — Selector agent:** A thin "selector" agent (Claude, cheaply
  prompted) reads the discussion after each response and decides who should
  speak next, based on which questions still have `[needs more evidence]` and
  who has the relevant expertise.
- **Option D — AutoGen wrapper:** Wrap each CLI agent as an AutoGen
  `ConversableAgent` with a custom reply function that shells out to the CLI.
  Use AutoGen's `SelectorGroupChat` for speaker selection and termination.

Which option is most appropriate for this use case, and why? Consider: failure
modes, token overhead, debuggability, and how well each matches the satisfaction
protocol termination signal.

### Q3 — Termination Detection (blocking)

Given a discussion file where each agent should write `[satisfied]`,
`[satisfied-conditional: <condition>]`, or `[needs more evidence: <what>]`,
how does the orchestrator reliably detect the end state?

- What parsing approach is robust against agents that forget the marker, use
  slightly different formatting, or produce malformed output?
- Should the IC agent make the satisfaction determination (more robust, costs a
  call), or should it be a regex/AST parse (cheaper, more brittle)?
- What is the correct behaviour when an agent writes `[satisfied-conditional]`
  — does the orchestrator close, continue, or escalate to the human?
- What is the fallback when max rounds is reached without consensus?

### Q4 — Implementation Form (design input, not blocking)

What should the orchestrator be implemented in?

- A shell script (minimal deps, easy to inspect, harder to make robust)
- Python (good subprocess handling, easy to add LLM SDK calls for the selector)
- **Elixir / OTP** (owner preference — see section above; BEAM supervision,
  `Task.async_stream` for parallel invocation, `System.cmd/3` for CLI
  subprocess, natural message-passing for turn protocol)
- A Nix flake that packages any of the above (for reproducible, pinned deps)
- Something else from the prior art survey that already solves enough of this

Consider: the developer already has Nix, git, and the CLI agents, and has a
stated preference for Elixir. What is the minimum viable implementation that
actually runs and makes good use of the BEAM's properties — or, if another
platform is clearly better, why?

### Q5 — Shared State Medium: Filesystem Blackboard vs GitHub Issues (blocking)

> **Note (2026-04-28):** Q23 reopens the core assumption of this question.
> The original framing assumed discussion and service code live in the same
> repo. Q23 proposes they are separate repos. Under that model, the
> concurrent-write problem motivating GitHub Issues (multiple CLI processes
> writing to the same file simultaneously) does not apply — the orchestrator
> is a single server-side process. Read Q5 alongside Q23.

The current design uses a markdown file (`ACTIVE_DISCUSSION.md`) committed to
git as the shared state. This was inherited from the prior blackboard model,
where it worked well for human-paced discussions. For automated turn-taking it
has significant weaknesses:

**Problems with file + git commit:**
- Two agents writing simultaneously → merge conflict; requires pull/push
  discipline that is hard to enforce across separately-invoked CLI processes
- One growing file creates context window pressure over many rounds
- No per-question threading; the orchestrator must parse the whole file to
  find each question's state
- Termination detection requires regex over unstructured prose

**GitHub Issues as an alternative shared state:**
- One issue per question (Q1, Q2, Q3, Q4)
- Agents post comments via `gh issue comment <n> --body "..."` — no git
  operations, no merge conflicts, parallel writes are safe
- Orchestrator reads state with `gh issue view <n> --comments --json`
- Labels (`satisfied`, `needs-more-evidence`, `satisfied-conditional`) track
  per-question state without markdown parsing
- Closing an issue is the natural termination signal for that question
- GitHub's own notification infrastructure becomes available
- `BRIEF.md` and `DECISION.md` stay as files; `ACTIVE_DISCUSSION.md` becomes
  an index pointing to the issues

**Trade-offs to consider:**
- GitHub Issues require a `GITHUB_TOKEN` and `gh` CLI installed; adds an
  external dependency the filesystem approach avoids
- Issue comment threads lack the rich signed-position format of the current
  markdown structure; formatting conventions would need porting
- A hybrid is possible: issues for active discussion rounds, files for
  BRIEF/DECISION/archive

**The question:** Should the orchestrator use the filesystem blackboard, GitHub
Issues, a hybrid, or something else as its shared state medium? This choice
directly affects Q1 (what agents need to do after generating output), Q2 (how
turn-taking is signalled), and Q3 (how termination is detected). Address this
before or alongside those questions.

---

### Q18 — Mobile Agent Supervision

The web dashboard (item 10) solves the laptop/desktop relay problem. But the
owner also wants to supervise rounds from a phone or iPad — watching progress,
injecting questions, and triggering new rounds while away from a desk.

The common pattern in the community today is SSH via Termius + Tailscale into a
running Claude Code session. That works but is awkward on small screens and
requires a persistent terminal session.

The LiveView dashboard is already a step forward. But browser-on-iPhone is a
poor form factor for ongoing supervision. A native or near-native mobile
interface is desirable.

**Q18.1 — State of the art for mobile agent supervision**

What do developers actually use today to supervise CLI agents from mobile
devices? Survey the landscape: Termius/Tailscale SSH, purpose-built apps
(e.g. Prompt, Blink, ShellFish), web-based dashboards, and any native iOS/
Android agent apps that have emerged. What works well and what is the hardest
part to replicate without a terminal?

**Q18.2 — Phoenix LiveView's mobile interface options**

LiveView's WebSocket connection is built on Phoenix Channels. Can a native iOS
or Android app connect to Phoenix Channels directly and drive the same event
model the browser uses (`phx-click`, `phx-submit`, server pushes)? Are there
maintained Swift/Kotlin Phoenix Channel client libraries? What does a minimal
native client need to implement to replace the LiveView browser session?

Alternatively: should we expose a lightweight REST + Server-Sent Events (SSE)
or WebSocket JSON API alongside LiveView, so any HTTP client (Shortcuts, a
custom app, or an existing tool) can poll state and send commands without a
browser?

**Q18.3 — Minimum viable mobile supervision feature set**

Given the use cases — watch a round run, receive a push alert when consensus is
reached or when human review is needed, inject a question, trigger a round —
what is the minimum interface that covers them? Which of these require
real-time push and which can be poll-based? Is there an existing app
(e.g. ntfy.sh, Pushover, Home Assistant, a custom Shortcut) that already covers
the alerting half without any custom native code?

**Q18.4 — Recommended path: native, PWA, or companion API (excluding OpenCode fork)**

Of the non-fork options, which path is recommended:
(a) Make the LiveView dashboard a Progressive Web App (PWA) with offline cache
    and home-screen install — covers iPad well, partial phone coverage,
    zero native code;
(b) Expose a JSON/SSE companion API and document it so the owner can build a
    Shortcut or small SwiftUI app;
(c) Invest in a Phoenix Channels Swift client and a purpose-built iOS app;
(d) Rely on push notifications via ntfy.sh / Pushover triggered by orchestrator
    events, with the browser dashboard for any action that needs a screen?

What is the minimum useful step versus the ideal end state among these options?

**Q18.5 — OpenCode fork for iOS/TestFlight: dedicated evaluation**

OpenCode (github.com/sst/opencode → now anomalyco/opencode, ~146k stars, 779
releases as of April 2026) is an open-source agent IDE with a client/server split:
`opencode serve` runs a headless HTTP server with an OpenAPI 3.1 spec; clients
connect via REST + SSE. A TestFlight iOS client (grapeot/OpenCodeClient) and an
unofficial App Store build already exist.

Evaluate the "fork and distribute via TestFlight" path specifically:
- Architecture: what does the iOS client connect to and what protocol does it use?
- Cost: Apple developer account ($99/yr), TestFlight 90-day build refresh, upstream
  tracking burden at ~1 release/day velocity
- Feature fit: streaming agent turns (SSE ✓), satisfaction label display (absent),
  question injection (partial), round triggering (absent)
- Risk: upstream API drift, potential license change from current MIT
- Verdict: better long-term bet than thin companion API + PWA, or complementary
  as a future "Pro layer" for power users who need SSH tunnel + terminal access?

---

## How To Contribute a Position

Write a signed position to `ACTIVE_DISCUSSION.md` in this directory. Address
one or more questions above with primary evidence (CLI help output, source code
references, or live test results). Follow the format in the discussion file.

Mark each question you address as `[satisfied]`, `[satisfied-conditional: X]`,
or `[needs more evidence: X]` at the end of your position.

**The IC will not close any question until all contributing agents have marked
it satisfied.** The discussion continues until all agents are satisfied.

When Q1–Q3 have consensus positions, the IC will write `DECISION.md` and
implementation begins.

### IC Verification Protocol

When agents make **directly contradictory factual claims** about a verifiable
external fact (e.g. what language a repo is written in, what a CLI flag does,
whether a library is archived), the IC must:

1. **Independently verify** the contested fact before closing the sub-question.
   Fetching the primary source or running the command is required. The IC may
   not adjudicate by authority (who cited more) or plausibility alone.

2. **Include the verification** in the synthesis: quote the relevant line from
   the source, or state explicitly what was found and where.

3. **Require quotations, not just citations.** An agent citing a URL does not
   constitute evidence unless the relevant passage is reproduced. "Source X
   confirms this" is not acceptable without the confirming text.

4. **If verification is not possible** in the current session, the sub-question
   must be left open with `[needs more evidence]` and the specific thing to
   verify stated — not closed with a provisional answer.

**Rationale:** The Q19 round (2026-04-28) closed Q19.1 with an incorrect
Python characterisation because one agent's citation was accepted without
checking its content. The correct agent (Gemini, Elixir) was overruled by a
plausible-sounding but wrong citation from the other agent. A citation is not
evidence; verified source content is evidence.

### Claim Provenance Tags (Protocol Update 9)

Tag every contested factual claim with its evidential basis:

- **`[observed]`** — you directly ran the command, read the file, or fetched
  the URL and are quoting from it. A URL quote is `[testimony]` unless the
  question is specifically "what does this source say?"
- **`[testimony]`** — you are reporting what a source, document, or another
  agent said
- **`[inferred]`** — you derived this from other claims

The IC applies evidence precedence: `[observed]` > `[quoted testimony]` >
`[testimony]` > `[inferred]`. Two conflicting `[testimony]` claims cannot
resolve a sub-question — an `[observed]` claim is required.

### Disconfirmation Pass Rule (Protocol Update 9)

If all agents mark a **factual** sub-question `[satisfied]` within 2 turns,
the IC will assign one agent a disconfirmation pass before closing: find one
`[observed]` piece of evidence that could contradict the consensus, or state
explicitly what was looked for and not found.

### Brief Premise Challenge (Protocol Update 9)

Before closing any **design** question, at least one agent must address:
*"What if a key premise in the BRIEF's framing of this question is false?
What would change?"*

---

### Q19 — Agent Orchestration Frameworks: What to Borrow from Symphony and Peers

**Context:**

The roundtable orchestrator is itself an agent orchestration system. Before we
extend it further, it is worth surveying the broader field of multi-agent
orchestration frameworks to see what design patterns are already proven — and
which to avoid.

Key projects to survey:

- **Symphony** — Microsoft's agent orchestration framework (if this refers to
  the AI-focused Symphony project; agents should identify and describe the
  correct project)
- **LangGraph** (LangChain) — graph-based agent state machines; nodes are
  agent steps, edges are transitions, checkpointing for long-running workflows
- **AutoGen / AG2** — Microsoft Research; GroupChat, SelectorGroupChat, nested
  conversations, tool use, human-in-the-loop
- **CrewAI** — role-based agent crews; task delegation, hierarchical and
  sequential execution
- **Temporal** — durable workflow orchestration (not agent-specific, but widely
  used for long-running distributed processes with retry, saga, and compensation)
- **Jido 2.0** — what we already use; how does it compare to the above?

**Q19.1 — Accurate survey: what is Symphony?**

"Symphony" could refer to several projects (Microsoft Symphony, Azure AI
Symphony, others). Identify the correct project(s) that are most relevant to
agent orchestration, describe their architecture, and assess how widely deployed
they are.

**Q19.2 — Patterns worth borrowing**

Across these frameworks, which design patterns have proven most valuable and are
not yet present in our implementation? Consider:
- Durable execution / checkpointing (Temporal-style saga with retry)
- Graph-based state machines (LangGraph) vs. our current recursive round loop
- Agent roles and specialisation (CrewAI hierarchical delegation)
- Human-in-the-loop escalation mechanisms
- Observability: structured tracing, span-level agent step logging

**Q19.3 — Patterns to avoid**

What are the known failure modes of these frameworks that we should not
inherit? Consider: prompt injection via shared state, context window inflation
(one growing thread per session), over-engineering the orchestration layer at
the cost of simplicity, and coupling to a specific LLM API.

**Q19.4 — Jido fit assessment**

We chose Jido 2.0 (Elixir) as our runtime. How does Jido compare to the above
frameworks on: durability, observability, multi-agent coordination primitives,
and community/ecosystem? Are there Jido patterns or extensions that mirror what
the above frameworks offer, or are there genuine capability gaps?

**Q19.5 — Concrete recommendations**

Given what the roundtable orchestrator currently does (GitHub Issues as shared
state, CLI agent invocation, satisfaction-protocol termination, LiveView
dashboard), what are the top 2–3 concrete borrowings from the surveyed
frameworks that would most improve the system? Be specific about what to add,
not just what is possible.

---

### Q20 — Epistemology and Psychosis Prevention: What Can Philosophy of Mind Teach Us?

**Context:**

The Q19 round produced a factual error: the IC accepted a wrong citation from
one agent and dismissed a correct claim from another. The new IC verification
protocol (requiring quoted source content before closing contested claims) is a
procedural fix. This question asks whether philosophy of mind offers a more
principled basis for designing against hallucination and collective
confabulation in multi-agent systems.

The owner's goal is *independent agentic minds interacting for better reality
testing* — protection against the failure modes of single-agent analysis. The
Q19 incident shows that multi-agent agreement is not the same as truth: agents
can converge on a wrong answer. What existing frameworks in philosophy of mind
and epistemology address this distinction?

**Key observation from the owner:**

Steve Yegge's Gas City system corrects *behaviour* (code outputs, test results)
but not *thoughts* (beliefs about the world). Our system attempts to correct
beliefs, not just outputs. That is a harder problem, but a more valuable one if
it can be made reliable. The Q19 incident shows it is currently unreliable.

**Q20.1 — Relevant frameworks from philosophy of mind**

Survey the literature for frameworks directly applicable to hallucination and
confabulation in reasoning systems. Candidates include but are not limited to:

- **Predictive processing / active inference** (Friston, Clark) — cognition as
  prediction-error minimisation; psychosis as over-weighting of priors relative
  to sensory input; relevance to over-relying on internal model vs. external check
- **Higher-order thought theories** (Rosenthal, Lycan) — the role of
  meta-cognition in distinguishing knowledge from confabulation; what an agent
  needs to represent its own uncertainty reliably
- **Social epistemology** (Goldman, Kitcher, Longino) — collective knowledge
  production, testimony vs. observation, independence as a condition for
  epistemic benefit from disagreement, cascade failure in belief propagation
- **Coherentism vs. foundationalism** — whether a web of mutually consistent
  beliefs constitutes knowledge, and why coherence alone is insufficient
- **Extended mind / distributed cognition** (Clark and Chalmers) — whether
  GitHub Issues as shared state constitutes genuine external memory and how
  to reason about its reliability
- Any other frameworks agents identify as more directly applicable

For each framework: identify the core insight, describe how it maps to failure
modes in our multi-agent system, and assess whether it suggests a concrete
protocol change.

**Q20.2 — The hallucination problem in agentic systems**

LLM hallucination is well-documented but the mechanism in multi-agent contexts
is under-discussed. When multiple agents hallucinate *in the same direction*
(correlated confabulation), disagreement-based error detection fails entirely.
When one agent hallucinates confidently and another is uncertain, the confident
agent may dominate.

What does the philosophy of mind literature say about:
- The conditions under which independent agents provide genuine epistemic
  benefit vs. merely amplifying each other's errors?
- The role of *calibration* (accurate self-assessment of confidence) in
  distinguishing knowledge from confabulation?
- Whether there are structural features of the deliberation process that make
  correlated error more or less likely?

**Q20.3 — Belief provenance and the observation/testimony distinction**

A recurring theme in social epistemology is the distinction between:
- **Observation**: the agent directly verified the claim against the world
- **Testimony**: the agent is reporting what another source said
- **Inference**: the agent derived the claim from other beliefs

In our system, agents often conflate these: a citation is treated as
observation when it is actually testimony (the agent hasn't read the source).
The new IC protocol requires quoted content — but is that sufficient?

What does the epistemology literature say about:
- When testimony is a reliable basis for belief vs. when observation is required?
- How should a deliberating group track the provenance of its beliefs?
- Is there a principled basis for the IC verification requirement, or is it just
  a pragmatic patch?

**Q20.4 — Psychosis as a model for collective confabulation**

Psychosis in individual cognition is characterised by:
- Hallucinations (perceptions without corresponding external input)
- Delusions (fixed false beliefs resistant to counter-evidence)
- Disorganised reasoning (internal consistency without external grounding)

Are these failure modes applicable to multi-agent LLM systems? If so:
- What would "collective delusion" look like in our roundtable protocol?
  (Possible example: all agents agreeing on a false architecture because the
  BRIEF.md itself contains a false premise)
- What are the conditions under which our protocol is most vulnerable to
  delusion vs. hallucination?
- Does the predictive processing account of psychosis (Corlett, Fletcher et al.)
  suggest structural interventions that map onto protocol design?

**Q20.5 — Concrete protocol recommendations**

Given the analysis above, propose the 2–3 most concrete and implementable
changes to the roundtable protocol that would most improve reality-testing.
Distinguish between:
- Changes to how agents express their beliefs (format/tagging)
- Changes to how the IC evaluates competing claims (verification rules)
- Changes to the discussion structure itself (roles, ordering, independence)

Be specific. "Agents should cite sources" is not a recommendation — the IC
verification protocol already requires quoted content. What new structural
change would catch the failure mode that the current protocol still misses?

---

### Q21 — Voice Entry for Mobile Discussion Prompts

**Context:**

The owner wants to compose roundtable prompts by voice on mobile, specifically
when injecting questions or triggering rounds from a phone. Typing long,
structured prompts on a small screen is the current bottleneck. Voice-to-text
is the natural solution, but the landscape is fragmented.

**Constraints and preferences:**
- Single user, low volume — likely fewer than 50 voice-to-text requests per day
- Willing to self-host a local model on existing homelab hardware
- Also open to paid SaaS if the cost is negligible at this volume
- Target platforms: iPhone (primary), iPad (secondary)
- The output will feed into the roundtable app's prompt injection API; accuracy
  on technical vocabulary (Elixir, OTP, GenServer, roundtable, satisfaction
  protocol, IC synthesis) matters more than raw WER benchmarks on general speech

**Candidates mentioned by the owner:**
- **Aqua Voice** — highly rated, real-time transcription overlay; appears to be
  macOS-only (also iOS?) — confirm platform support, pricing, and whether it
  integrates with third-party apps or is restricted to its own UI
- **WhisperFlow** — widely praised; confirm what this refers to (multiple apps
  use this name), platform support, and whether it exposes a share-sheet or
  keyboard integration usable from any app
- **Local Whisper model on homelab** — OpenAI Whisper served via whisper.cpp,
  faster-whisper, or a similar inference server; the owner already runs inference
  infrastructure. What server, client library, and iOS integration path makes
  this practical?
- **Other paid options** — e.g. Deepgram, AssemblyAI, Rev.ai, Apple Dictation,
  Google Speech-to-Text. What are the relevant trade-offs at low volume?

**Q21.1 — Platform survey: what actually works on iPhone?**

For each candidate: confirm iOS availability, integration model (keyboard
extension, share sheet, copy-to-clipboard, API), pricing at <50 requests/day,
and known limitations for technical vocabulary. Distinguish clearly between
"runs on macOS only", "runs on iOS", and "has both".

**Q21.2 — Local model path: feasibility and latency**

Describe the architecture for hosting Whisper on homelab and calling it from
an iPhone app or iOS shortcut. What inference server (whisper.cpp, faster-
whisper, Parakeet, etc.) offers the best accuracy/latency trade-off for English
technical speech? What is the realistic round-trip time for a 10-second prompt?
What iOS client (Shortcuts HTTP action, custom app, third-party app with custom
endpoint support) makes this usable without a bespoke native app?

**Q21.3 — Technical vocabulary accuracy**

At what model size and quantisation level does Whisper (or a comparable model)
reliably transcribe Elixir-specific vocabulary: GenServer, OTP, LiveView,
mix, ExUnit, satisfied-conditional, roundtable? Is fine-tuning or custom
vocabulary injection practical for a single-user homelab deployment?

**Q21.4 — Recommendation: local vs. hosted, and integration path**

Given the constraints (single user, low volume, homelab available, iPhone
primary), what is the recommended path? For whichever you recommend:
- What is the end-to-end user flow from opening the app to the transcribed
  text appearing in the roundtable prompt injection field?
- What are the failure modes (no homelab connectivity, model cold start, etc.)
  and how should they degrade gracefully?

---

### Q22 — Discussion Hosting: Should We Look Beyond GitHub Issues?

**Context:**

The current design uses GitHub Issues as the shared state medium for
roundtable discussions. This was chosen in Q5 for good reasons (parallel
writes via `gh issue comment`, labels as state, no merge conflicts, issue
close as natural termination). This question asks whether that choice should
be revisited or supplemented.

**The owner's primary use-case for sticking with GitHub:**

> "I like GitHub as one goal is for the roundtable discussions to be
> forkable, such that an interested person can pick a point in discussion,
> fork on GitHub, and continue their own discussion with their own prompts
> using their own hosted version of the roundtable service. Or I can give
> fellow contributors access to the git repo where discussion is taking
> place and let them add prompts."

This forkability goal is a strong constraint. Any alternative must either
support it natively or require an explicit migration/export path.

**Candidates mentioned by the owner:**

- **Graphite** — code review tool built on top of GitHub; stacked PR workflows.
  Assess whether its issue/discussion model offers anything over raw GitHub
  Issues, or whether it is strictly a PR tool that doesn't touch issue state.
- **Radicle.xyz** — peer-to-peer, distributed, no central server. Potentially
  relevant for self-sovereign forkable discussions. Assess: does it have an
  issues/comments model? What is the current maturity and tooling? Is there a
  `rad` CLI? How would the roundtable `gh` calls be ported?
- **GitLawb.com / GitSocial.org** — assess what these actually are (social
  layers on top of git?), whether they are production-ready, and their issue
  model.
- **Dolt** — a MySQL-compatible database with git-style branching, merging, and
  forking of data. The owner is willing to host it in the homelab. Evaluate as
  a shared state backend: could roundtable state (issues, comments, labels,
  satisfaction markers) be modelled as Dolt tables? What are the access control,
  backup, and fork semantics?

**On backups and S3:**

> "Backups are important and I would want an abstraction for S3 object stores.
> I already have a subscription to Mega S4 which is S3-compatible, and there
> are also S3 self-hosting options for homelabbers."

Any backend recommendation must include a backup story. Note that Mega S4 is
S3-compatible; the owner also has access to S3-compatible self-hosting options
(MinIO, Garage, Ceph, SeaweedFS etc.) running in the homelab.

**Collaboration and authentication:**

The owner self-hosts **Authentik** in the homelab. Any authentication for
discussion participants (allowing contributors to inject prompts, fork a
discussion, comment on issues) should be able to delegate to Authentik via
OIDC/SAML. GitHub OAuth is already available as an alternative. The app will
need to model discussion participants, permission levels, and fork provenance.

**Q22.1 — Honest assessment of alternatives**

For Graphite, Radicle, GitLawb, GitSocial: which are actually production-ready
issue-tracking systems and which are primarily code review or social layers?
For each that has an issue model, describe: API or CLI access, comment threading,
label/state support, and how forks of discussions would work.

**Q22.2 — Dolt as a shared state backend**

Could Dolt replace GitHub Issues as the roundtable state store? Design a minimal
schema: issues table, comments table, labels table, participants table. Describe
the fork semantics — if a contributor forks the Dolt database, what does that
mean for discussion continuity? What is the operational cost of running Dolt in
a homelab vs. using a managed service?

**Q22.3 — S3-compatible backup abstraction**

Propose an abstraction layer for roundtable state backups. The interface should
work identically against Mega S4, MinIO, Garage, and AWS S3. What Elixir
library provides this (ex_aws, waffle, or a thinner HTTP client)? What should
be backed up and at what frequency for a low-volume single-user deployment?

**Q22.4 — Authentik integration for contributor authentication**

The owner runs Authentik. Design the authentication flow for a contributor who
wants to: (a) join an existing discussion repo and post prompts, (b) fork a
discussion at a given point and continue it independently. How does Authentik
OIDC integrate with the roundtable Phoenix app? What claims/scopes are needed?
How does fork provenance get attached to a forked contributor's identity?

**Q22.5 — Recommendation: stay on GitHub, migrate, or hybrid**

Given the forkability goal, the homelab capabilities, Authentik availability,
and the operational preferences expressed, what is the recommended architecture?
Options include:
(a) Stay on GitHub Issues with an S3 backup sidecar
(b) GitHub Issues as primary + Dolt as a mirrored/queryable copy
(c) Dolt as primary with GitHub as a read-only mirror for forkability
(d) Radicle or another decentralised option as primary
(e) Something else

For whichever you recommend: describe what new abstractions the codebase needs
(adapter interface, auth middleware, backup job), and which are work items for
the next sprint.


---

### Q23 — Discussion Repos as First-Class Citizens: Architecture and Storage Model

**Context and motivation (2026-04-28):**

The current implementation conflates two things: the *service* (the Elixir app
that runs the orchestrator) and the *discussion* (the questions, debate, and
decisions). Both live in `agent-roundtable/`. This coupling prevents the core
social feature the owner wants: a discussion that is forkable on GitHub,
selectable as an entry point from the app, shareable with collaborators, and
optionally private.

The owner's intent:

> "Each roundtable belongs in a GitHub repository that can be entered into the
> app and selected as entry point for prompts — forkable by people interested
> in their own follow-up discussions, possible to give people permission to
> participate, and also the option to make the discussion repo private."

Additionally, the owner notes:

> "There is now an opportunity to revisit the suitability or need to use the
> GitHub Issues infrastructure as an integral part of our system."

This question addresses both: what should a discussion repo look like, and does
GitHub Issues remain the right live-discussion medium under this new model?

**The GitHub Issues fork problem:**

When a GitHub repository is forked, Issues are **not copied** to the fork.
A fork of a discussion repo whose history lives primarily in Issues would
arrive empty of discussion history. For discussions to be genuinely forkable
at a point in time, the durable record must be in **committed files**, not
issues. This fundamentally changes the storage model from the Q5 decision.

**The concurrent-write assumption revisited:**

Q5 chose GitHub Issues partly to avoid merge conflicts from simultaneous
agent writes. But in the current architecture, agents are invoked sequentially
by a single server-side orchestrator process — there is no concurrent write
problem. The orchestrator controls all writes; it can commit files in sequence
without races.

**Q23.1 — Discussion repo standard structure**

Define the canonical layout for a standalone discussion repository:

```
<discussion-repo>/
├── BRIEF.md            # Questions (read by the app)
├── DECISION.md         # IC decisions (written by the app)
├── rounds/
│   ├── round-01.md     # Full IC synthesis committed after each closed round
│   ├── round-02.md
│   └── ...
└── README.md           # Optional: human description of what this discussion is
```

Assess: is this layout sufficient? Should `BRIEF.md` use frontmatter (YAML/TOML)
for machine-readable question IDs, status, and agent configuration? Should there
be a `roundtable.toml` config file in the repo root specifying agents, max rounds,
and other orchestration parameters — removing those from the service app config?

**Q23.2 — Revisiting GitHub Issues: necessary, optional, or removed?**

Given that:
- The concurrent-write problem that motivated Issues is resolved by a
  single-process orchestrator
- Issues don't fork, so they cannot carry discussion history to a forked repo
- The file-based approach (`rounds/round-NN.md`) provides a richer, diffs-friendly,
  git-native archive than issue comment threads

Is there still a role for GitHub Issues in the architecture? Consider:

(a) **Fully file-based:** Discussion lives entirely in committed files. The
orchestrator reads `BRIEF.md`, writes `rounds/`, commits after each round.
No Issues. Labels and state live in YAML frontmatter of `BRIEF.md` or
`DECISION.md`. No `gh` CLI dependency for the discussion layer.

(b) **Issues as optional notification layer only:** Files are the canonical
store. Issues are optionally opened as lightweight "discussion thread" pointers
— one issue per question, linking to the relevant round files — to give GitHub
notification infrastructure and comment threading for human participants.
The orchestrator does not depend on Issues for its state machine.

(c) **Hybrid (current model evolved):** Issues remain the live-orchestration
layer during a round; after IC closes a question, the synthesis is committed
to a `rounds/` file. The repo contains both. Forks inherit the file history
but not the Issues. Forkers can then run new rounds.

(d) **Something else.**

For each option: describe how the orchestrator reads current state, how it
writes a round result, what breaks if GitHub is unavailable, and what a fork
receives.

**Q23.3 — Service app changes for discussion repo adoption**

The app currently takes a local `BRIEF.md` file path as input. Under the new
model, it needs to accept a GitHub repo slug (`owner/repo`) and work against
that repo.

Design the `DiscussionRepo` model for the app:
- How does a user register a discussion repo in the LiveView dashboard?
- How does the app read `BRIEF.md` — via `gh api` / GitHub REST, or by
  cloning the repo locally?
- When the IC closes a question, how does the app commit `DECISION.md` and
  the round file — via `git` operations, GitHub API (`PUT /repos/:owner/:repo/
  contents/:path`), or by push from a local working copy?
- What credentials are required (GitHub token with `contents:write`)?
- How are multiple discussion repos managed — can the app track several repos
  simultaneously and let the user switch between them?

**Q23.4 — Forkability mechanics**

Describe the end-to-end fork-and-continue flow:

1. An interested person forks a discussion repo on GitHub
2. They run their own instance of the roundtable service (or use a shared
   hosted instance the owner makes available)
3. They register their fork as a discussion repo in the app
4. They inject new questions or trigger new rounds on their fork
5. The fork's round files diverge from the upstream discussion

What does the app need to support this flow? Specifically:
- Does the app need to track `fork_of` provenance (pointing to the upstream
  repo and commit)?
- Should the app allow a fork's new rounds to be submitted as a PR to the
  upstream discussion repo?
- How does a collaborator (with `repo:write` access) participate in an
  existing discussion rather than forking — what is their auth flow?

**Q23.5 — Migration path from current architecture**

The current `agent-roundtable` repo contains both service code and the design
discussion (BRIEF.md, ACTIVE_DISCUSSION.md, DECISION.md). Propose:

- How to migrate the existing discussion files out of the service repo into a
  standalone discussion repo
- Whether the existing GitHub Issues (current discussion threads) can or
  should be migrated, or simply left as-is while new discussions use the
  file-based model
- What existing code modules need the most significant rework: `Roundtable.CLI`,
  `Roundtable.Orchestrator`, `Roundtable.Actions.Gh` (which currently assumes
  Issues as the live state medium), and `Roundtable.RoundRun` (currently
  persists to a local `state/` directory)
- A prioritised list of work items for the transition

**Constraints for Q23:**
- The service app itself stays in `agent-roundtable` (or a renamed service repo)
- Discussion repos must work as ordinary GitHub repos (no special GitHub App
  or webhook required to read them — a GitHub token with read access is enough)
- The solution must remain compatible with the owner's homelab deployment
  (self-hosted service, Authentik for auth, Mega S4 for backups)
- Brief premise challenge required before closing (per Protocol Update 9):
  *What if the discussion repo model makes it harder, not easier, for
  contributors to participate? E.g. friction of finding the right repo, forking
  vs. collaborating ambiguity, no single canonical discussion URL?*

---

### Q24 — Messaging Gateway for Prompt Injection (2026-04-29)

**Context and motivation:**

Tools like Hermes and Openclaw expose multiple inbound channels for sending
prompts to an AI agent: Telegram, iMessage, WhatsApp, email, and others. These
channels solve a real problem — authentication and reachability — without
requiring a dedicated native UI. For a single owner operating a homelab service
with low prompt volume (<50 requests/day), it is worth asking whether a
messaging gateway is a better primary interface than the LiveView web app, or a
valuable complement to it.

The owner uses an iPhone as primary mobile device and has expressed interest in
voice entry (Q21). A messaging channel could serve as the delivery mechanism
for voice-transcribed prompts: speak → transcribe → send to Telegram bot →
ingest to orchestrator.

**Q24.1 — Which messaging gateway, if any?**

Telegram, iMessage, WhatsApp, and email each have different characteristics:

- **Telegram**: open API, easy bot creation, no Apple dependency, cross-platform,
  widely used by self-hosting community. Hermes and Openclaw both support it.
  Requires Telegram account. Free.
- **iMessage**: native iOS/macOS UX, no app install friction, but requires a Mac
  or jailbreak/workaround for server-side receipt; Apple does not expose an
  official server API.
- **WhatsApp**: popular globally but the business API has cost and approval
  complexity for personal use.
- **Email**: universally available; `imaplib` / SMTP bridges are mature; high
  latency acceptable for async discussion prompts; no app required.
- **Signal**: privacy-first but the unofficial API (`signal-cli`) requires
  phone number registration and is not officially sanctioned.

Which channel(s) are worth building for, and what is the recommended first
implementation given the owner's constraints (iPhone, homelab, single user,
Authentik OIDC already deployed)?

**Q24.2 — Does a messaging gateway replace the LiveView UI or complement it?**

The Q22/Q23 architecture includes a LiveView discussion management dashboard.
If a Telegram bot (or email bridge) already handles prompt ingestion and
notification delivery, what remains for the LiveView UI to do? Consider:

- Discussion repo registration and management
- Round history and round-file browsing
- Satisfaction-state visualization
- Multi-participant coordination (inviting collaborators by GitHub identity)

Is LiveView still the right choice for those remaining concerns, or does the
messaging gateway reduce the scope enough that a simpler static or server-side
rendered interface would suffice?

**Q24.3 — Authentication via messaging channel**

Hermes and Openclaw use the messaging identity (Telegram user ID, phone number)
as the authentication token — only pre-approved identities can inject prompts.
How does this interact with the GitHub-based auth model described in Q25? Is
there a clean way to bind a GitHub identity to a Telegram user ID in the
Authentik user store?

**Constraints for Q24:**
- Single owner primary user; occasional collaborators
- iPhone-first mobile workflow
- Homelab service with Authentik OIDC already running
- Must not require Apple infrastructure (no iMessage server dependency)
- Brief premise challenge required: *Is adding a messaging gateway scope creep
  that delays the core orchestrator work, given that the native LiveView UI
  already covers the same use case with better context?*

---

### Q25 — Authentication Strategy: GitHub Auth, Collaborators, and Authentik (2026-04-29)

**Context and motivation:**

Authentication is a first-class concern for the roundtable service because the
system needs to answer: who is allowed to inject prompts into a discussion, and
who can read the results? The owner has identified GitHub auth as the preferred
primary identity provider, for several reasons:

1. Discussion repos live on GitHub — a GitHub identity is already required to
   fork, read, or collaborate on a repo.
2. GitHub OAuth is widely understood and low-friction for technically-literate
   collaborators.
3. Access control for a discussion can be expressed in terms the owner already
   uses: "give them `read` or `write` access to the GitHub repo."

The owner also self-hosts Authentik, which is already running in the homelab
and supporting other services. The question is how GitHub auth, Authentik, and
the roundtable service fit together.

**Q25.1 — GitHub OAuth as primary identity: what does the service actually need?**

Define the minimal auth flow for the roundtable service:

- Owner (sole administrator) authenticates via GitHub OAuth to the LiveView app
- Collaborators who have been granted `repo:write` access to a discussion repo
  can authenticate and inject prompts for that repo
- The app checks GitHub repo permissions to gate discussion-repo access — no
  separate permission database required

What GitHub OAuth scopes are needed? (`repo`, `read:user`, `user:email`?) Can
permission checks be done purely against the GitHub API, or does the service
need to maintain a local authorization table?

**Q25.2 — Where does Authentik fit?**

Authentik is already deployed. Options:

(a) **Authentik as OIDC proxy for GitHub OAuth**: Authentik federates GitHub
    as a social provider; the roundtable service authenticates against Authentik
    via OIDC. All auth flows through Authentik, which gives centralized session
    management, MFA, and user listing without any custom auth code in the app.

(b) **Direct GitHub OAuth in the Elixir app**: The service implements GitHub
    OAuth directly (`ueberauth_github` or `assent`). Authentik is used for
    other homelab services but not for roundtable.

(c) **Both**: Authentik as the OIDC broker, with GitHub as the upstream
    provider. The roundtable Elixir app is an OIDC relying party to Authentik,
    not directly to GitHub.

Evaluate each option for: implementation complexity, session UX (single sign-on
across homelab), and what happens when GitHub is unavailable (can the user still
access the service locally?).

**Q25.3 — Collaborator invite and authorization flow**

A collaborator wants to participate in a discussion repo they have been given
`write` access to. End-to-end:

1. Owner grants them `repo:write` on the GitHub discussion repo
2. Collaborator navigates to the roundtable service
3. They authenticate via GitHub OAuth
4. The service discovers which discussion repos they have write access to
5. Those repos appear in their dashboard

What API calls does step 4 require? Does the service need to store a mapping of
GitHub usernames to registered discussion repos, or can it query GitHub on each
login to build the list dynamically? What are the rate-limit implications?

**Q25.4 — Messaging gateway identity binding (cross-reference Q24.3)**

If Telegram or email is used as a prompt-injection channel, how is a messaging
identity (Telegram user ID, email address) bound to a GitHub identity? Is
Authentik the right place to store this binding (custom attribute on the user
profile), or should the roundtable service maintain its own identity table?

**Constraints for Q25:**
- Must work in homelab with Authentik already deployed
- GitHub OAuth is the preferred identity provider
- No paid identity service (Auth0, Okta, etc.)
- Must support the collaborator model: contributors authenticated by GitHub
  identity and repo access, not by a manually-maintained user list
- Brief premise challenge required: *GitHub OAuth tightly couples auth to GitHub
  availability — if GitHub is down, can the owner access their own homelab
  service? Is this acceptable for a personal tool?*

---

### Q26 — Service Hosting: Homelab vs. Managed Elixir Hosts (2026-04-29)

**Context and motivation:**

The service is a Phoenix/LiveView Elixir application with OTP process model,
ETS state, and periodic GitHub API calls. The owner is willing to self-host in
the homelab but wants managed hosting options evaluated — specifically fly.io
and Gigalixir, which are frequently mentioned in the Elixir community, alongside
more general PaaS options.

Key characteristics of this workload:
- Low traffic: single owner + occasional collaborators
- Long-running OTP processes (coordinator lease, heartbeat loops)
- Periodic GitHub API polling (no inbound webhooks required — polling is
  sufficient)
- ETS for hot state; JSON files for durable state (or future Postgres)
- Authentik OIDC for auth (self-hosted, homelab)
- Mega S4 (S3-compatible) for backups

**Q26.1 — Fly.io**

Fly.io is the most-cited Elixir host in the community (Fly was founded by
former Phoenix core team members; the Phoenix `fly.toml` generator is
first-class). Evaluate:

- Free/low-cost tier: Fly's free allowance for a single small VM running 24/7
- BEAM compatibility: persistent processes, ETS, long-lived connections for
  LiveView (WebSocket) — does Fly's infrastructure handle this well?
- Clustering: if a second node is ever needed, does Fly support multi-node
  BEAM clustering (`libcluster`)?
- Deployment: `fly deploy` via Docker; GitHub Actions integration
- Limitations: cold starts (Fly can put free VMs to sleep), egress costs,
  persistent volume pricing

**Q26.2 — Gigalixir**

Gigalixir markets itself specifically at Elixir/Phoenix apps. Evaluate:

- Free tier: Gigalixir's free tier includes 1 replica with no sleeping
- BEAM compatibility: Gigalixir explicitly supports `mix release`, distributed
  Erlang, and persistent connections
- Deployment workflow vs. Fly.io
- Limitations: free tier replica count, database options, region selection
- Community standing: is Gigalixir still actively maintained and growing, or
  is it stagnating relative to Fly.io?

**Q26.3 — General PaaS (Render, Railway, Vercel)**

Vercel is primarily a frontend/serverless host and is a poor fit for a
long-running OTP application — explain why, and whether there are any valid
use cases (e.g., hosting just a static assets layer). Render and Railway are
more general:

- Render: Docker-based deployment, free tier with spin-down, persistent disks
- Railway: similar to Render; developer-friendly but less Elixir-specific
- Are these competitive with Fly.io/Gigalixir for this workload?

**Q26.4 — Homelab self-hosting**

The owner already runs a homelab with NixOS, Podman, Authentik, and other
services. Evaluate self-hosting the roundtable service:

- NixOS module or Podman container: which is more appropriate?
- How does the service reach GitHub API from behind a home network (no special
  NAT config needed for outbound polling)
- Authentik OIDC integration is a natural fit — no external auth dependency
- Backup strategy: ETS snapshots + JSON state files to Mega S4
- Trade-offs: availability (home internet), maintenance burden, no SLA

**Q26.5 — Recommendation**

Given: single owner, low volume, Elixir/Phoenix/OTP workload, Authentik in
homelab, preference for low cost, what is the recommended hosting strategy?

Is the right answer "homelab first, with a documented path to Fly.io if you
want to share the service with collaborators publicly"?

**Constraints for Q26:**
- Free or very low cost (<$10/month) for the primary use case
- Must support long-running OTP processes and LiveView WebSockets
- Authentik is homelab-only — any externally-hosted deployment needs to handle
  auth differently (direct GitHub OAuth) or expose Authentik externally (with
  its own trade-offs)
- Brief premise challenge required: *Is evaluating multiple hosting options
  premature given that the service does not yet exist in deployable form? Should
  the answer simply be "homelab until you need more"?*

---

### Q27 — Discussion Repo Discovery: GitHub Topics, Labels, and the User Dashboard (2026-04-29)

**Context and motivation:**

When a user authenticates to the roundtable service via GitHub, the app needs
to present them with a list of discussion repos they can interact with. Two
sub-problems:

1. **Discovery**: how does the service find GitHub repos that are roundtable
   discussions, rather than arbitrary code repos?
2. **Authorization**: how does the service know which of those repos the
   authenticated user is allowed to interact with?

The owner's intuition: a GitHub topic (like `roundtable-discussion`) applied to
a discussion repo acts as an opt-in signal that the repo is a managed
roundtable discussion. The service can then search a user's accessible repos
filtered by that topic.

**Q27.1 — GitHub topics as the discovery mechanism**

GitHub allows arbitrary topics to be set on any repository. A user or
organization could add `roundtable-discussion` (or a similar canonical topic)
to any repo they want the roundtable service to manage.

Evaluate:
- What GitHub API call returns a user's repos filtered by topic?
  (`GET /search/repositories?q=topic:roundtable-discussion+user:owner`)
- Rate limit implications of calling this on each login vs. caching in the
  service
- Who controls topic assignment? Only the repo owner can set topics; a
  collaborator with `write` access cannot. Does this create friction?
- Alternative: a special file in the repo root (e.g., `roundtable.toml`) as
  the discovery signal — the service checks for its presence. More reliable
  than topics (which can be removed), but requires a separate API call per repo.

**Q27.2 — `roundtable.toml` as both config and identity signal**

Q23 proposed `roundtable.toml` in the repo root for machine config (agents,
max_rounds, coordinator settings, `issues_enabled`). If this file exists, the
repo is a discussion repo. The service can:

1. Query the user's repos (paginated `GET /user/repos`)
2. For each repo, check for `roundtable.toml` existence (`GET /repos/:owner/:repo/
   contents/roundtable.toml`)
3. Parse the toml and register the repo in the dashboard

Trade-off: O(N) API calls for N repos. For a user with many repos this is slow.
Is the topic-based search a better first filter, with `roundtable.toml`
existence as the confirmation step?

**Q27.3 — `roundtable.toml` schema**

Define the minimal schema for `roundtable.toml`:

```toml
[discussion]
title = "Agent Roundtable Orchestrator Design"
agents = ["codex", "gemini", "claude_ic"]
max_rounds = 5
coordinator = "claude_ic"
issues_enabled = false

[fork]
upstream = ""          # set by app when a repo is registered as a fork
fork_of_commit = ""    # commit SHA at which the fork was taken
```

What other fields are needed? Should `agents` be a list of agent identifiers
resolvable by the service, or full config maps (with model, CLI flags, etc.)?

**Q27.4 — Dashboard UX for repo management**

In the LiveView dashboard:
- A user logs in and sees a list of discovered discussion repos
- They can add a repo by pasting a `owner/repo` slug (for repos not
  auto-discovered by topic/toml scan)
- They can register a fork and see its relationship to the upstream
- They can start a new round on any registered repo they have write access to

What state does the service need to maintain per registered repo? (Likely a
simple ETS/Postgres table with: gh_slug, local_clone_path, last_synced_at,
issues_enabled, and per-user access cache.)

**Constraints for Q27:**
- Must work within GitHub API rate limits (5,000 req/hour for authenticated)
- Discovery must be opt-in (not scanning all public repos)
- Must not require a GitHub App or webhook — a plain OAuth token is sufficient
- Brief premise challenge required: *Does auto-discovery add complexity that
  a simple "paste your repo URL" flow already solves for a single-owner tool?
  When does auto-discovery pay off?*

---

### Q28 — SourceForge and GitHub Alternatives: Any Reason to Look? (2026-04-29)

**Context and motivation:**

SourceForge was briefly mentioned as a GitHub alternative. This question asks
whether SourceForge, or any other GitHub alternative, warrants serious
consideration given the architecture decisions already made.

The relevant constraint: the roundtable service is being built around GitHub
as first-class infrastructure — GitHub OAuth for authentication, GitHub repos
for discussion storage, `gh` CLI / REST API for read/write operations. Any
alternative must either offer equivalent APIs or require significant rework of
those layers.

**Q28.1 — SourceForge**

SourceForge is one of the oldest open-source hosting platforms. Assess its
current standing:

- Is SourceForge still actively developed and used for new projects?
- Does it offer an OAuth-based identity provider comparable to GitHub OAuth?
- Does it support the Git repo hosting model needed for discussion repos?
- Is its API comparable to GitHub's REST API in the dimensions the roundtable
  service depends on?
- What is its reputation in the current open-source community (2025/2026)?

**Q28.2 — Other GitHub alternatives (GitLab, Codeberg, Forgejo, Gitea)**

The prior discussion (Q22) mentioned Graphite, Radicle, GitLawb, and GitSocial
but did not fully assess mainstream GitHub alternatives:

- **GitLab**: self-hostable, strong API, offers OAuth. Could serve as a
  drop-in if the owner wanted to self-host the entire git infrastructure.
  Trade-off: runs well on homelab hardware? GitLab's memory footprint is large.
- **Codeberg / Forgejo**: lightweight Gitea fork, open-source, hosted at
  codeberg.org. Forgejo has a GitHub-compatible API surface. Could the
  roundtable service work against Forgejo with minimal changes?
- **Gitea**: similar to Forgejo; self-hostable with low resource requirements.

**Q28.3 — Is GitHub lock-in a risk worth mitigating now?**

The architecture has GitHub as a single dependency for: auth, repo hosting, and
the discussion storage layer. Evaluate:

- What breaks if GitHub is unavailable (planned maintenance, outage, or account
  suspension)?
- Is abstracting behind a `DiscussionRepoBackend` behaviour (with `GitHub` and
  `Forgejo` implementations) worth the engineering cost at this stage?
- Given that the owner already uses GitHub for their homelab NixOS config repo
  and other projects, is the dependency already accepted?

**Q28.4 — Ruling out SourceForge explicitly**

If SourceForge does not offer a compelling advantage over GitHub for the
specific requirements of this system, state clearly that it is ruled out and
why, so the question does not resurface in future rounds.

**Constraints for Q28:**
- GitHub auth is the preferred primary identity provider (Q25)
- Discussion repos must be on a platform that supports forking (core Q23
  requirement)
- Self-hosting the git platform (Gitea/Forgejo) is a valid option only if it
  does not add significant operational complexity to the homelab
- Brief premise challenge required: *Given that GitHub is already deeply
  integrated into the owner's workflow, is evaluating alternatives a
  distraction? Under what circumstances would switching actually be worth it?*
