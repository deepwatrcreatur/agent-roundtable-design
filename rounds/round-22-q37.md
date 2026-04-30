## Round 22 — Q37: Vaglio Evaluation Framework (2026-04-30)

**IC prompt:**

Q37 asks how to evaluate whether vaglio (the multi-agent protocol) actually
produces better outcomes than a single capable model given the same question.
The core claim has never been tested. Sub-questions cover metrics (Q37.1),
task design (Q37.2), infrastructure (Q37.3), statistical validity (Q37.4),
and falsifiability (Q37.5).

This is the first round under the vaglio name. Agents should propose concrete,
implementable eval approaches — not theoretical frameworks. The owner wants
proposals that can become work tasks.

---

### Codex — Q37

**Premise challenge:** *Is formal evaluation necessary for a personal tool?*

`[inferred]` The premise challenge has real force here. The owner has used vaglio
for 21 rounds and found it valuable — that is evidence. However, the project
aspires to be more than a personal tool (Protocol Update 14 added a fourth agent;
Q26/Q31 discuss public hosting). If vaglio is ever shared, "I think it works" is
insufficient. Even for personal use, knowing *where* multi-agent adds value vs.
where it is wasted cost helps the owner decide when to use vaglio vs. a single
model for a given question.

Formal eval is not necessary for continued use. It is necessary for informed use.

**Q37.1 — Metrics**

`[inferred]` Four metrics are feasible with reasonable automation:

1. **Consideration coverage** (automatable): Count unique substantive
   considerations in the output. Define a "consideration" as a distinct claim
   with a recommendation or assessment. Use an LLM judge (a fifth model not in
   the roster, or Claude in a separate context) to extract and deduplicate
   considerations from both vaglio and single-model outputs. Metric: count
   of unique considerations.

2. **Self-consistency** (automatable): Does the output contradict itself?
   Extract claims and check for logical contradictions within the same output.
   Vaglio's IC synthesis should reduce internal contradictions relative to a
   single model that may contradict itself across a long response.

3. **Blind preference** (human judge): Present the owner with two outputs
   (vaglio vs. single-model, unlabelled) and ask: "which analysis is more
   useful for making a decision?" Binary forced choice. This is the gold
   metric — it measures what the owner actually cares about.

4. **Cost ratio** (automatable): Total tokens consumed (input + output across
   all agents) for vaglio vs. single-model. Express as a multiplier: vaglio
   costs Nx a single-model run. The question is whether the quality improvement
   (if any) justifies the cost.

Dropping: calibration (requires many runs with known outcomes), decision quality
(requires downstream measurement), error detection (requires ground truth).
These are important but too expensive to measure in a first eval pass.

**Q37.1: [satisfied]**

**Q37.2 — Task design**

`[inferred]` Three task types, in order of value:

1. **Replay prior questions** (highest value, lowest design cost): Take 5
   questions from the existing Q1–Q36 set — chosen for diversity (one
   architecture question, one process question, one tooling question, one
   hosting question, one naming/creative question). Run each through a single
   model (Claude, with the same BRIEF.md context) and compare output to the
   vaglio discussion record. This reuses existing work.

2. **Synthetic design questions** (medium value, medium cost): Create 3–5 new
   design questions that are plausible vaglio use cases — e.g., "design a
   rate limiting strategy for the orchestrator API," "evaluate three
   approaches to conversation persistence." Run both modes blind.

3. **Code review** (specific, measurable): Give both modes a PR diff and ask
   for a review. Count issues found, categorise by severity. This is the
   most objectively measurable task type.

Minimum useful eval set: 5 replayed questions + 3 synthetic questions = 8 tasks,
each run in both modes = 16 total runs. Feasible in a single day.

**Q37.2: [satisfied]**

**Q37.3 — Infrastructure**

`[inferred]` Minimal eval module:

```elixir
defmodule Vaglio.Eval do
  @moduledoc "Compare multi-agent vs single-model output on the same question."

  defstruct [:question, :brief_context, :vaglio_output, :single_output,
             :metrics, :judge_preference]

  @doc "Run a question through the full vaglio protocol and capture output."
  def run_vaglio(question, brief_context, opts \\ [])

  @doc "Run the same question through a single model with equivalent context."
  def run_single(question, brief_context, model \\ :claude, opts \\ [])

  @doc "Extract metrics from both outputs."
  def extract_metrics(eval)

  @doc "Present blind comparison for human judgment."
  def blind_compare(eval)

  @doc "Persist eval result to state dir."
  def persist(eval)
end
```

`run_single/4` uses `RunCliAgent` with a structured prompt that includes the
same BRIEF.md context and instructions (use satisfaction markers, typed
provenance, etc.) — the single model gets the same *protocol* instructions,
just no multi-agent iteration.

`extract_metrics/1` uses a judge model (separate Claude call) to extract
consideration counts and self-consistency checks.

`blind_compare/1` writes two files (`output_a.md`, `output_b.md`) with
randomised assignment, for the owner to read and judge.

**Q37.3: [satisfied]**

**Q37.4 — Statistical validity**

`[inferred]` With 8 tasks and binary preference judgments, we cannot reach
traditional statistical significance (p < 0.05 requires ~16 concordant
preferences out of 20 under a binomial test). This is fine for a directional
eval. The goal is not to publish a paper — it is to inform the owner's
decision about when to use vaglio.

A sequential approach: run 4 tasks first. If all 4 show the same direction
(vaglio preferred or single preferred), that is a strong directional signal.
If mixed, run the remaining 4. Budget: 8 tasks × ~$0.50 vaglio cost + ~$0.12
single cost ≈ $5.00 total. Negligible.

**Q37.4: [satisfied]**

**Q37.5 — Falsifiability**

`[inferred]` Pre-registered hypotheses:

- **H1:** Vaglio produces more unique considerations than single-model
  (coverage). Expected: yes, by 1.5–3x, because four models have different
  training distributions.
- **H2:** The owner prefers vaglio output in blind comparison (preference).
  Expected: yes, for design questions; uncertain for factual/code questions.
- **H3:** Vaglio costs 4–6x a single-model run (cost). Expected: yes,
  by definition (four agents + IC).
- **H-null:** A single model with the vaglio prompt structure (satisfaction
  markers, premise challenge, structured format) produces output
  indistinguishable from vaglio. If true: the value is in the protocol's
  structure, not in the multi-agent aspect. This would suggest the protocol
  should be packaged as a structured prompt, not a multi-agent system.

If H-null is confirmed: the project is still valuable — the structured prompt
is the product. The multi-agent orchestrator becomes optional infrastructure.

**Q37.5: [satisfied]**

---

### Gemini — Q37

**Premise challenge:** Subjective assessment is valid for a personal tool but
it is vulnerable to confirmation bias — the owner designed the protocol and
wants it to work. A blind comparison is the minimum necessary to rule out
"vaglio feels better because I built it."

**Q37.1 — Metrics**

`[inferred]` I agree with Codex's four metrics but want to add one:

5. **Dissent surfacing** (automatable): Count instances where an agent raises
   a consideration that *contradicts* a prior agent's position. In vaglio,
   this should happen naturally (agents see prior turns). In single-model
   mode, a single model may not generate self-contradicting considerations.
   Metric: number of explicit disagreements or counter-considerations.

This measures what vaglio claims to add: structured dissent. If vaglio does
not produce more dissent than a single model prompted to "consider multiple
perspectives," the multi-agent architecture is not earning its cost.

**Q37.1: [satisfied]**

**Q37.2 — Task design**

`[inferred]` I agree with Codex's task set but want to add a crucial control:

**Structured single-model baseline:** The single model should be tested in
*two* configurations:
- **Naive single:** "Answer this question." (Minimal prompt)
- **Protocol-structured single:** "Answer this question. Use satisfaction
  markers. Challenge your own premises. Tag claims as [observed], [inferred],
  or [testimony]. Consider at least one perspective you might be wrong about."

If vaglio beats naive-single but not protocol-structured-single, the value
is in the protocol, not in the multiple agents. This is Codex's H-null and
it is the most important thing to test.

**Q37.2: [satisfied]**

**Q37.3 — Infrastructure**

`[inferred]` Codex's module API is correct. I'd add:

```elixir
defmodule Vaglio.Eval.Judge do
  @moduledoc "LLM-as-judge for automated metrics extraction."

  @doc "Extract unique considerations from an output."
  def extract_considerations(text)

  @doc "Check for internal contradictions."
  def check_consistency(text)

  @doc "Count explicit disagreements/counter-considerations."
  def count_dissent(text)
end
```

The judge model should be a model *not in the roster* to avoid circularity.
Options: Claude (already the IC — some circularity), GPT-4o (external,
reduces bias), or a local model via Ollama. Recommendation: use Claude as
judge but in a separate context with no knowledge of which output is vaglio
vs. single-model.

**Q37.3: [satisfied]**

**Q37.4 — Statistical validity**

`[inferred]` For the blind preference metric, a sign test (binomial test
under null hypothesis p=0.5) with 8 tasks requires 8/8 or 7/8 concordance
for p < 0.05. This is strict. With 12 tasks and 10/12 concordance, p ≈ 0.02.

Recommendation: plan for 12 tasks (5 replayed + 5 synthetic + 2 code review).
Run sequentially. If after 8 tasks the direction is 7-1 or stronger, stop
early. Budget: ~$7.50 total. Still negligible.

**Q37.4: [satisfied]**

**Q37.5 — Falsifiability**

`[inferred]` The most important outcome to pre-register:

**If protocol-structured single matches vaglio on all metrics:** Ship the
protocol as a structured prompt template. The multi-agent orchestrator
becomes a power-user option, not the primary product. This is not a failure
— it means vaglio discovered a valuable structured prompt through the
process of building a multi-agent system. The prompt is the product;
the orchestrator is how we found it.

**Q37.5: [satisfied]**

---

### DeepSeek — Q37

**Premise challenge:** `[inferred]` The owner's subjective assessment is
evidence but it is also the weakest form of evidence for the specific claim
("multi-agent is better than single-model"). The owner has never seen what a
single model would produce on the same questions with the same prompt
structure. Without that comparison, "vaglio is good" ≠ "vaglio is better."

Formal eval is not necessary for the tool to be useful. It is necessary for
the tool's *claim* to be credible.

**Q37.1 — Metrics**

`[inferred]` I agree with coverage, preference, cost, and dissent. I'd
add a metric specific to my position in the roster:

6. **Training distribution diversity** (automatable): For each consideration
   in the output, tag which agent(s) raised it. Measure the proportion of
   considerations raised by only one agent vs. raised by multiple agents.
   High unique-to-one-agent proportion = high diversity = the multi-agent
   architecture is adding value. Low proportion = agents are redundant.

This directly tests whether four different models contribute genuinely
different perspectives or merely paraphrase each other.

**Q37.1: [satisfied]**

**Q37.2 — Task design**

`[inferred]` Gemini's two-baseline design (naive single + protocol-structured
single) is critical. I want to strengthen it:

**Three baselines:**
1. Naive single: "Answer this question."
2. Protocol-structured single: Same question + full vaglio prompt structure
3. **Self-debate single:** One model prompted to generate multiple
   perspectives and then synthesize. "First, generate three distinct
   perspectives on this question. Then synthesize them into a recommendation."

If vaglio beats baselines 1 and 2 but not baseline 3, then the value is in
structured self-debate, not in having genuinely different models. This is a
stronger H-null test — it isolates the contribution of training distribution
diversity from the contribution of structured multi-perspective prompting.

**Q37.2: [satisfied]**

**Q37.3 — Infrastructure**

`[inferred]` Agree with Codex and Gemini's module structure. One addition:

The eval harness should record *all intermediate state*, not just final
output. For vaglio runs, this means: each agent's raw turn, the IC synthesis,
satisfaction markers at each stage, and the final DECISION.md entry. For
single-model runs: the full response. This enables post-hoc analysis that
we haven't yet thought of.

```elixir
defmodule Vaglio.Eval.Run do
  @type t :: %__MODULE__{
    id: String.t(),
    question: String.t(),
    mode: :vaglio | :single_naive | :single_structured | :single_debate,
    model: atom(),
    turns: [map()],   # each turn's raw output
    final_output: String.t(),
    metrics: map(),
    tokens_used: non_neg_integer(),
    cost_usd: float(),
    started_at: DateTime.t(),
    completed_at: DateTime.t()
  }
end
```

**Q37.3: [satisfied]**

**Q37.4 — Statistical validity**

`[inferred]` With 12 tasks × 4 modes (vaglio + 3 baselines) = 48 runs.
Token budget: ~$15–20 total. Each task produces a 4-way preference ranking
(or set of pairwise comparisons). A Friedman test can detect differences
across 4 conditions with 12 observations per condition — this is borderline
but sufficient for directional signal.

Pragmatic recommendation: start with 6 tasks × 2 modes (vaglio vs.
protocol-structured single). This is the most informative comparison. If
vaglio wins clearly, add the other two baselines. If vaglio does not win,
the other baselines are moot.

**Q37.4: [satisfied]**

**Q37.5 — Falsifiability**

`[inferred]` Pre-registered expectations:

- **Coverage:** Vaglio produces 1.5–3x more unique considerations. If not:
  the models are more correlated than expected; roster should be revised.
- **Diversity:** ≥40% of vaglio considerations are unique to one agent.
  If not: agents are redundant; consider replacing one with a more
  differentiated model.
- **Preference:** Owner prefers vaglio in ≥7/12 blind comparisons. If not:
  the multi-agent overhead is not justified for this owner's use case.
- **Protocol-structured single matches vaglio:** The structured prompt is
  the product. Ship it as a template. The orchestrator becomes optional.

The most transformative outcome: if self-debate single (baseline 3) matches
vaglio, then the *entire multi-agent architecture is unnecessary* and the
value is purely in the prompting strategy. This would be a significant
finding — and an honest one.

**Q37.5: [satisfied]**

---

## IC Synthesis — Q37 (Round 22, 2026-04-30)

**IC: Claude**

**IC mindguard check:** The strongest dissenting-adjacent position is shared
by all three agents: the possibility (H-null) that a single model with
vaglio's prompt structure matches multi-agent output. All agents explicitly
accept this as a valid and important outcome to test. No dissent was suppressed.

**Double-loop framing check:** The BRIEF.md framing assumes multi-agent is
valuable and asks how to measure it. All three agents correctly challenged
this by designing baselines that could *disprove* the multi-agent value
proposition. DeepSeek's three-baseline design (naive, structured, self-debate)
is the most thorough challenge to the project's own premises.

### Q37.1 — Metrics: closed

**Adopted metrics (6):**

| # | Metric | Type | What it measures |
|---|---|---|---|
| 1 | Consideration coverage | Automated (LLM judge) | Unique substantive points in output |
| 2 | Self-consistency | Automated (LLM judge) | Internal contradictions |
| 3 | Blind preference | Human judge | "Which analysis is more useful?" |
| 4 | Cost ratio | Automated | Token/dollar cost multiplier |
| 5 | Dissent surfacing | Automated (LLM judge) | Explicit counter-considerations |
| 6 | Training diversity | Automated | Proportion of considerations unique to one agent |

Metrics 1, 2, 5, 6 use an LLM judge (Claude in separate context, no knowledge
of which output is which). Metric 3 requires the owner. Metric 4 is computed
from token counts.

### Q37.2 — Task design: closed

**Task set:**
- 5 replayed questions from Q1–Q36 (diverse: architecture, process, tooling,
  hosting, creative)
- 5 synthetic design questions (new, written for eval)
- 2 code review tasks (PR diff + review request)
- Total: 12 tasks

**Baselines (3):**
1. **Naive single:** "Answer this question." (Claude, minimal prompt)
2. **Protocol-structured single:** Same question + vaglio prompt structure
   (markers, provenance, premise challenge, warrant requirements)
3. **Self-debate single:** One model generates 3 perspectives then synthesizes

**Phased approach:** Start with 6 tasks × vaglio vs. protocol-structured single
(the most informative comparison). If vaglio wins clearly, expand to full 12
tasks × all baselines. Budget: $7–20.

### Q37.3 — Infrastructure: closed

**Module: `Vaglio.Eval`**

```
Vaglio.Eval          — top-level: run_vaglio/3, run_single/4, blind_compare/1
Vaglio.Eval.Run      — struct for a single eval run (all intermediate state)
Vaglio.Eval.Judge    — LLM-as-judge: extract_considerations, check_consistency,
                       count_dissent
Vaglio.Eval.Metrics  — compute coverage, diversity, cost from Run data
```

All intermediate state recorded (each agent turn, IC synthesis, markers).
Results persisted to `state/eval/` as JSON. Integration via existing
`RunCliAgent` infrastructure.

### Q37.4 — Statistical validity: closed

Phased sequential approach:
1. Run 6 tasks (vaglio vs. structured single)
2. If 5/6 or 6/6 concordant in blind preference: strong directional signal,
   stop or expand to full baselines
3. If mixed (3-3 or 4-2): run remaining 6 tasks for 12 total
4. Sign test for preference; Friedman test if all 4 conditions are run

Budget: $7–20 total. Feasible in a single day.

### Q37.5 — Falsifiability: closed

**Pre-registered hypotheses:**

| Hypothesis | Expected | If falsified |
|---|---|---|
| H1: Coverage 1.5–3x | Yes | Models more correlated than expected; revise roster |
| H2: ≥40% unique-to-one-agent | Yes | Agents redundant; replace weakest |
| H3: Owner prefers vaglio ≥7/12 | Yes | Multi-agent overhead not justified |
| H-null: Structured single matches vaglio | No | Protocol prompt is the product; orchestrator optional |
| H-null-2: Self-debate single matches vaglio | No | Multi-agent architecture unnecessary; prompting strategy is sufficient |

**Most transformative outcome:** If H-null-2 holds, the value is purely in
structured multi-perspective prompting, not in having genuinely different
models. This would be an honest and important finding.

---

**Q37 closure status:** All sub-questions satisfied. Ready for work item
decomposition.
