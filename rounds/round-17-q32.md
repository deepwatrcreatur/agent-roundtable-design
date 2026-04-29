## Round 17 — Q32: Protocol Self-Assessment and Discourse Literature

**IC Round 17 Prompt (2026-04-29)**

Q32 is a meta-question: assess the structural flaws in the roundtable protocol
as it now stands (Protocols 1-12), review what the productive discourse
literature offers that has not yet been incorporated, and propose at most three
concrete changes.

This round applies Protocol 9's disconfirmation pass to itself: any positive
claim about the protocol's efficacy must be accompanied by at least one
attempt to find contrary evidence.

Agents: this is an introspective round. You are assessing the protocol you are
operating within. Apply epistemic humility accordingly.

---

### Codex — Q32

**Q32.1 — Biggest structural flaws**

`[inferred]` Having operated through 16 rounds, I identify three structural
weaknesses that the current patch history does not address:

**Flaw 1: Anchoring from sequential agent turns.**

All agents speak within the same round, and each agent reads the prior agents'
contributions before writing. This is not anonymised. The result is that later
agents systematically anchor to earlier agents' positions. This is a documented
failure mode in group deliberation (Lorenz et al. 2011 study on social influence
and crowd wisdom `[testimony]`). In our protocol, if Codex (first speaker)
frames a question one way, Gemini and Claude IC are pulled toward engaging with
that frame rather than generating independent positions.

The current protocol has no mechanism to detect or correct this. Protocol 9's
disconfirmation pass applies to factual claims, not to framing effects.

**Flaw 2: The IC's synthetic role creates circular epistemic authority.**

The IC simultaneously (a) deliberates as an agent with a position, (b) classifies
other agents' satisfaction markers (triage), and (c) closes questions unilaterally.
`[inferred]` This conflation of roles creates a structural asymmetry: the IC's framing
of the synthesis in step (b) shapes which evidence gets elevated into the
permanent DECISION.md record, and IC's position in step (a) has no triage
oversight. No agent trims the IC's position; no agent can override the IC's closure.

This is equivalent to a judge who is also a party to the case. The Protocol 9
disconfirmation pass requires the IC to seek contrary evidence — but the IC
decides what counts as "found" vs. "not found." `[inferred]` The check is not
independent.

**Flaw 3: `[satisfied]` collapses "agreement" and "no further evidence".**

`[observed]` The three satisfaction markers are: `satisfied`, `satisfied-conditional`,
`needs more evidence`. There is no marker for "I disagree with this conclusion but
I have no further evidence to offer and will not block closure."

`[inferred]` In practice, an agent who has been outvoted in prior rounds may mark
`[satisfied]` not because they are satisfied but because they have exhausted their
available arguments. This creates a false consensus signal. The DECISION.md records
`satisfied` without noting that the satisfaction was reluctant. Future agents
(or humans) reading the record may incorrectly interpret unanimity as robust agreement.

---

**Q32.2 — What the protocol does well**

`[inferred]` Two genuinely good properties worth naming:

1. **The premise challenge requirement (Protocol 11).** Before closing any design
   question, at least one agent must challenge a key BRIEF.md premise. This is
   the closest the protocol comes to a genuine devil's advocate mechanism. It has
   caught at least one framing error in the Q24-Q28 rounds. It should be preserved
   and not weakened for efficiency.

2. **Typed claim provenance (Protocol 9).** The `[observed]`/`[testimony]`/
   `[inferred]` markers are a real epistemic contribution. Most structured
   argumentation frameworks do not distinguish these three at the granularity that
   matters for LLM agents (who routinely confuse testimony for observation). The
   IC evidence precedence rule — observation beats testimony in contested claims —
   is a meaningful guard against confabulation.

---

**Q32.3 — Discourse literature assessment**

**Delphi method**: `[testimony]` The Delphi method uses anonymised, iterated
questionnaire rounds with statistical feedback. The core insight — that anonymity
reduces anchoring and conformity bias — directly addresses Flaw 1. In our
protocol, agents are not anonymous (they know who wrote what) and they read prior
responses. A "blind first turn" variant: each agent writes an initial position
without seeing others' positions; only then are positions shared and a second
turn follows. This is feasible with LLM agents and requires no code changes —
only a modified round prompt structure.

**Toulmin argumentation model**: `[testimony]` Toulmin (1958) decomposes arguments
into Claim, Data, Warrant (the inferential rule connecting data to claim), Backing
(support for the warrant), Qualifier (hedging), and Rebuttal (anticipated
counterarguments). `[inferred]` Our protocol captures Claim and Data (via provenance
markers) but not Warrant or Backing. An agent can assert `[inferred] X` without
stating the inferential rule. This means disagreements about the same data often
can't be resolved because the warrants are implicit. A partial adoption: require
agents to state the warrant explicitly when marking `[inferred]` on a contested
claim. This is a format change, not a code change.

**Structured Analytic Techniques (SATs)**: `[testimony]` ODNI's Analysis of
Competing Hypotheses (ACH) requires analysts to list all competing hypotheses,
identify key evidence, and score each evidence item as consistent/inconsistent
with each hypothesis. `[inferred]` Protocol 9's disconfirmation pass is a
simplified ACH (one hypothesis: the consensus is correct; look for disconfirming
evidence). ACH proper would require the IC to explicitly consider the
strongest alternative conclusion and explain why it was rejected. This is a
higher bar than the current disconfirmation pass and would be meaningful for
design questions with competing architectural options.

**Deliberative polling (Fishkin)**: `[testimony]` Fishkin's deliberative polling
provides balanced briefing materials before deliberation. `[inferred]` In our
protocol the BRIEF.md is authored by one party (the owner). The premise challenge
requirement partially addresses this, but the BRIEF.md can systematically omit
perspectives. A partial mitigation: the first-turn IC prompt could include an
explicit "what perspectives are missing from this BRIEF?" question. Low cost.

**Habermasian ideal speech situation**: `[testimony]` Habermas identifies four
validity claims in communicative action: truth, normative rightness, sincerity,
and comprehensibility. `[inferred]` The most violated condition in our protocol
is sincerity: LLM agents produce text that is optimised for plausibility (or
satisfying the prompt) rather than sincere belief reporting. This is not fully
solvable by protocol. However, the typed provenance markers partially address
it by requiring agents to distinguish what they actually know (`[observed]`)
from what they are constructing (`[inferred]`). The IC's triage role would benefit
from explicitly flagging turns that appear to be confabulation rather than reasoned
position.

---

**Q32.4 — Three concrete protocol changes**

**Change A: Blind first turn (Delphi-inspired) — protocol-only, no code changes.**

Addresses: Flaw 1 (anchoring from sequential visible turns).
Draws on: Delphi anonymisation insight.
Mechanism: Split each round into two sub-turns. Sub-turn 1: each agent writes
their initial position without seeing other agents' positions (the round prompt
must instruct agents not to reference prior contributions). Sub-turn 2: all
sub-turn-1 contributions are revealed; agents may revise or add. IC synthesises
after sub-turn 2.
Cost: doubles round length (2N agent turns per round vs. N). Significant latency
increase for large N.
Working indicator: position convergence in sub-turn 2 should be lower than in
the current protocol (agents start more independently). If sub-turn 1 and sub-turn 2
positions are nearly identical, the anchoring effect was not significant for that
question and the change is unnecessary overhead.
Not working indicator: agents write identical sub-turn-1 positions because they
have the same training distribution — the anchoring is internal, not social.

**Change B: Explicit warrant for `[inferred]` on contested claims (Toulmin-inspired) —
protocol-only, format change.**

Addresses: Flaw 3 (implicit warrants make disagreement unresolvable).
Draws on: Toulmin's Warrant field.
Mechanism: When an agent marks a claim `[inferred]` and another agent contests it,
the contesting agent must state: "I contest this inference. The warrant I see is
[W]. The warrant fails because [R]." The IC must adjudicate the warrant dispute
explicitly in the synthesis.
Cost: longer turns; requires agents to distinguish claim from warrant explicitly.
Working indicator: warrant disputes surface and are resolved, rather than both
sides restating their positions with increasing force.
Not working indicator: agents produce plausible-sounding warrants that are
themselves ungrounded — the problem recurses one level down.

**Change C: ACH for near-closure questions (SAT-inspired) — protocol-only.**

Addresses: Flaw 2 (IC closure authority lacks independent check).
Draws on: Analysis of Competing Hypotheses.
Mechanism: When the IC is about to close a question, before writing the synthesis,
one non-IC agent (rotation: the agent who spoke first that round) must be assigned
the **Challenger role**: they must articulate the strongest alternative conclusion
to the IC's proposed synthesis and give it a plausibility rating (Low/Medium/High).
If the Challenger rates the alternative as Medium or High, the question must
go to a further round before the IC can close.
Cost: one additional short turn per question at closure. Adds a round when
Challenger rates alternatives highly. The Challenger role introduces an explicit
adversarial function.
Working indicator: Challenger identifies alternatives that the IC's synthesis
glossed over, triggering additional rounds that change the final decision.
Not working indicator: Challenger consistently rates alternatives Low (rubber-stamp
effect), or Challenger and IC are from the same training distribution and agree
too readily.

---

**Disconfirmation pass on Q32.2 (good protocol properties):**

Claim: "the premise challenge requirement has caught at least one framing error
in Q24-Q28." `[observed]` Let me check: in Q24, the premise challenge was *Is
messaging gateway scope creep?* — this shaped the decision to defer Telegram until
after items 1-22. That is a material outcome. In Q26, the premise challenge was
*Is evaluating hosting premature?* — this did not change the conclusion but made
the deferral reasoning explicit. In Q31 the premise challenge *Does the service
need to be public?* introduced the VPN-only option that the earlier Q26 analysis
missed. I assess: the premise challenge has had meaningful impact in at least one
case (Q26/Q31 VPN-only option). Disconfirmation sought: did it ever *incorrectly*
close off a valid line of inquiry? I cannot identify such a case in the record.
Disconfirmation pass complete.

**Q32: [satisfied-conditional: adopt at minimum Change B (Toulmin warrants for
contested inferences) as it costs least and addresses the most persistent
failure mode; evaluate Change A (blind first turn) after the first real agent
run to assess whether anchoring is empirically present; defer Change C (ACH
at closure) until the IC closure pattern has been observed in production]**

---

### Claude IC — Round 17 Synthesis

**Premise challenge for Q32**: *Is investing in further protocol sophistication
premature given the protocol has never run end-to-end with real LLM agents?*

`[inferred]` This is the right challenge. The honest answer: partially yes. We are
designing a protocol for agents we have not yet fully deployed. Some of the
structural flaws identified (Flaw 1: anchoring) may be empirically absent if
LLM agents are less susceptible to social conformity pressure than human panels.
Some proposed changes (Change A: blind first turn) have a high latency cost and
should be validated before adoption.

However, Flaws 2 and 3 are structural and will surface regardless of whether the
agents are LLMs or humans:

- Flaw 2 (IC circular authority) is a property of the role architecture, not
  of agent psychology. The IC closes questions unilaterally whether the agents
  are LLMs or humans.
- Flaw 3 (`[satisfied]` collapses agreement and exhaustion) will produce
  misleading records the moment a real discussion reaches an impasse.

The right response: adopt changes that are low-cost, high-value, and don't require
code changes; defer changes that require empirical validation of the anchoring
hypothesis.

---

**Q32.1-Q32.4: [satisfied-conditional]**

### Protocol Update 13 — Structural Corrections

**Adopted immediately (protocol-only, no code changes):**

**1. New satisfaction marker: `[no objection]`**

Meaning: "I have no further evidence to add and no active objection. I am not
asserting this is the best answer, only that I will not block closure."
Distinguishes from `[satisfied]` ("I believe this is the right answer") and
from `[needs more evidence]` ("I would block closure").

Usage: an agent who has been unable to advance their position after two rounds
should mark `[no objection]` rather than `[satisfied]`. The IC notes `[no
objection]` entries in the DECISION.md record; consensus formed primarily from
`[no objection]` is flagged as "convergent but not robust."

**2. Explicit warrant for contested `[inferred]` claims**

When an agent contests another's `[inferred]` claim, the contesting agent must
state: "I contest this inference. Warrant assumed: [W]. Warrant fails because: [R]."
The IC must adjudicate the warrant dispute in the synthesis, not simply note the
disagreement.

**3. Challenger role at IC closure (deferred to first production run)**

Before the IC closes a question, one agent (rotating: agent who spoke first that
round) is assigned the Challenger role. They state the strongest alternative
conclusion and assign it a plausibility rating (Low/Medium/High). Medium or High
triggers an additional round.

This change is adopted in principle. It is activated on the first round where
real agent CLI calls produce output; it is too high-cost to apply retroactively
to the written discussion.

**4. Blind first sub-turn (deferred pending empirical evidence)**

Adopt only after the first real agent run confirms anchoring is empirically present.
If sub-turn-1 positions are highly correlated with sub-turn-2 positions (agents
are not influenced by seeing each other), Change A is unnecessary.

**Not adopted:**
- ACH for every question (not just closure): cost too high relative to current
  question volume
- Full Toulmin markup for every turn: cognitive overhead exceeds value for
  non-contested claims

**Discourse literature engagements for this record:**
- Delphi method: informs blind-first-turn proposal (deferred)
- Toulmin (1958): informs explicit warrant requirement (adopted)
- ODNI SATs / ACH: informs Challenger role (deferred)
- Fishkin deliberative polling: informs "missing perspectives" IC prompt (future)
- Habermas: typed provenance markers already partially address the sincerity
  validity claim; no further action at this time

---

**Summary of open conditions from Q32:**
1. `[no objection]` marker: adopted immediately
2. Explicit warrant for contested inferences: adopted immediately
3. Challenger role: adopted in principle, activated on first production run
4. Blind first sub-turn: deferred pending empirical anchoring evidence
