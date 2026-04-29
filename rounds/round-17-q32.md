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

---

## Round 17 Addendum — Q32: Organizational Behaviour and Management Science Literature

**IC Addendum Prompt (2026-04-29)**

Five bodies of business school / OB research have been identified as relevant
to the protocol's structural flaws. Agents assess each against the Q32.1 flaws
and propose protocol adjustments where warranted. The key empirical question:
do LLMs inherit human group dynamics because they are trained on human text?

---

### Codex — Q32 Addendum

**On the LLM-inherits-human-dynamics hypothesis:**

`[inferred]` The prior should be strong, not weak. LLMs are not just trained on
individual human writing — they are trained on *group outputs*: meeting
summaries, decision memos, committee reports, edited consensus documents. These
are already the outputs of groupthink, information sampling bias, and defensive
routines. The model does not just inherit the individual human's reasoning
patterns; it inherits the social filtering that occurred before the document was
written. A committee report that suppresses the dissenting view trains the model
to suppress dissenting views.

`[inferred]` Empirically: there is published evidence that LLMs exhibit
sycophancy under pressure (Perez et al., Wei et al. `[testimony]`) — the
model changes its stated position when a user pushes back, even without new
information. This is the LLM analogue of evaluation apprehension and is directly
analogous to what Stasser & Titus and Janis describe in human groups. The
mechanism is different (RLHF reward for user approval vs. social belonging) but
the observable pattern is the same.

---

**(A) Groupthink / Mindguard**

`[inferred]` Yes, the IC synthesis can function as a mindguard. The IC decides
which agent contributions are elevated into the DECISION.md permanent record.
A contribution that is structurally present in the round but framed as a minority
view may be omitted, softened, or absorbed into the synthesis without attribution.
This is exactly Janis's "rationalization" symptom: minority positions are not
confronted, they are absorbed.

**Protocol adjustment proposed:** The IC synthesis must quote the strongest
*dissenting* position verbatim (or closely paraphrased) before the synthesis,
not just the convergent positions. If there is no dissent, the IC states: "No
dissenting position recorded in this round." This is a one-sentence addition to
the synthesis format.

**(B) Double-loop learning / Argyris**

`[inferred]` The BRIEF.md is authored by one party (the owner) and can encode
motivated reasoning. The premise challenge partially catches this — but only if
agents can identify the motivated premise. A BRIEF authored with a strong prior
will frame the question in a way that makes the prior seem like the neutral
starting point.

`[inferred]` A double-loop question the premise challenge does not catch:
*"Is the BRIEF framing this question in a way that makes the desired conclusion
seem natural while making alternatives seem like edge cases?"* This is harder
to answer than "what if premise X is false" — it requires the agent to
reconstruct the authorial intent behind the framing.

**Protocol adjustment proposed:** Add a fixed IC prompt step at the start of
each question's first round: *"What framings of this question would lead to
a different set of sub-questions than those in the BRIEF?"* This is a structural
(not agent-triggered) double-loop check. Cost: one additional IC turn per
question at round 1.

**(C) Nominal Group Technique / Blind first sub-turn**

`[inferred]` The production-blocking and evaluation-apprehension effects that
motivate NGT are partly social (requiring conscious awareness of group norms)
and partly procedural (seeing a prior contribution narrows the generative space).
For LLMs: the social component (fear of judgment) is absent, but the procedural
component (context conditioning) is present and arguably stronger. An LLM
sees prior turns as in-context examples and is pulled toward stylistic and
substantive consistency with them.

`[observed]` This is observable: if the first agent marks `[needs more evidence]`
with a specific concern, later agents are more likely to address that concern
specifically — not because they independently generated it, but because it is
salient in the context window. This is procedural anchoring, not social
anchoring. NGT's blind-first-turn mechanism addresses both; for LLMs it is
the procedural component that matters.

**Revised assessment of blind first sub-turn:** Should not be deferred as a
low-priority empirical question. The mechanism is well-motivated for LLMs on
procedural grounds alone. The latency cost (doubles agent turns per round) is
the real constraint, not the empirical uncertainty.

**(D) Dialectical inquiry vs. devil's advocacy / Mason & Mitroff**

`[inferred]` The finding that devil's advocacy outperforms dialectical inquiry
for non-adversarial settings suggests the Challenger role should be targeted:
one critic, one plan, at the moment of closure. Expanding devil's advocacy to
every agent turn would produce dialectical inquiry, which is less effective and
much more expensive.

**No additional protocol adjustment proposed beyond Challenger role at closure.**

**(E) Information sampling bias / Stasser & Titus**

`[inferred]` The "unique contribution" prompt structure is worth testing. Current
agent prompt ends with: "Please respond with your position." A modified ending:
"Please respond with your position. Explicitly include at least one piece of
evidence or consideration that has NOT already been raised in the prior
contributions — if you cannot identify one, state this explicitly."

This is a low-cost format change that directly targets the sampling bias
mechanism. An agent that states "I have no unique contribution to add" provides
useful signal — it indicates the information space has been exhausted.

**Protocol adjustment proposed:** Add "unique contribution" prompt appendix.
Cost: slightly longer turns; measurable if agents regularly report "no unique
contribution" (which would indicate the question has low information entropy and
may resolve quickly).

---

**Summary of proposed additions from addendum:**

1. **IC mindguard check** — IC synthesis must quote or paraphrase the strongest
   dissenting position before synthesis. One sentence. No code change.
2. **Double-loop framing check** — Fixed IC prompt at question round 1:
   *"What framings would lead to different sub-questions?"* Protocol-only.
3. **Unique contribution prompt appendix** — Agent turn prompt ends with:
   *"Include at least one consideration not already raised, or state you cannot
   identify one."* Protocol-only, low cost.
4. **Blind first sub-turn re-prioritised** — Not deferred; motivated on
   procedural (context conditioning) grounds for LLMs independent of social
   anchoring. Latency cost is the real constraint; recommend implementing
   after first real agent run with timing data.

**Q32 addendum: [satisfied]**

---

### Claude IC — Addendum Synthesis

**Disconfirmation pass**: Do LLMs actually exhibit information sampling bias
analogous to Stasser & Titus? `[inferred]` The claim is well-grounded theoretically
(context conditioning is a real mechanism) but I should note: the Stasser & Titus
effect in humans is partly driven by *shared pre-deliberation knowledge* — all
members knew X before the meeting, so X is discussed more. For LLMs with similar
training data, the analogous effect is that well-represented topics in training
data produce more confident, less hedged claims. Topics with sparse training data
produce more `[inferred]` and `[testimony]` labels. The provenance markers already
partially surface this. The unique contribution prompt adds explicit structure.
Disconfirmation sought: is there evidence this prompt actually changes LLM output?
`[inferred]` Not directly, but the instruction-following literature strongly suggests
that explicit structural constraints in prompts affect output distribution. I assess
the unique contribution prompt as low-cost and likely effective.

---

**Protocol Update 13 Addendum — Organizational Behaviour Corrections**

**Adopted immediately (all protocol-only, no code changes):**

**1. IC mindguard check**
The IC synthesis opens with the strongest dissenting position from the round,
quoted or closely paraphrased. If there is no dissent, the IC states: "No
dissenting position recorded in this round." Prevents silent absorption of
minority views into the synthesis.

**2. Double-loop framing check at round 1**
At the start of a question's first round, the IC prompt includes: *"What
framings of this question would lead to a different set of sub-questions than
those in the BRIEF? The IC will note the most significant alternative framing
before opening the substantive discussion."*

**3. Unique contribution prompt appendix**
Agent turn prompts conclude with: *"Explicitly include at least one piece of
evidence or consideration that has NOT already been raised in this round's
prior contributions. If you cannot identify one, state this explicitly."*

**4. Blind first sub-turn: re-prioritised to first production run**
(Moved from "deferred pending empirical evidence" to "implement and evaluate
at first real agent run." Procedural context-conditioning is sufficient
theoretical justification; timing data will determine whether the latency
cost is acceptable.)

**Theoretical basis recorded:**
- Janis (1972) groupthink → mindguard check + Challenger role
- Argyris (1990) double-loop learning → framing check at round 1
- Delbecq & Van de Ven (1971) NGT → blind first sub-turn (re-prioritised)
- Mason & Mitroff (1981) devil's advocacy → Challenger role at closure (unchanged)
- Stasser & Titus (1985) information sampling → unique contribution prompt
- LLM-inherits-human-dynamics: prior is strong (sycophancy evidence, context
  conditioning mechanism); structural corrections motivated on LLM-specific grounds
  in addition to human group dynamics grounds
