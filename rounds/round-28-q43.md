## Round 28 — Q43: External Recursive Multi-Agent Systems, Self-Improvement, and Consensus Protocols

### DeepSeek

**Q43.1 — Which outside systems matter most?** The short list that matters most
is not "all multi-agent papers" but the systems that actually instantiate
iterative/refinement loops and expose something reusable.

1. **ChatDev / DevAll** — because it spans research papers, public code,
   productization, and multiple self-improvement mechanisms.
2. **MetaGPT + AFlow** — because they move from fixed SOP workflows to automatic
   workflow search and optimization.
3. **MAD / DebateLLM** — because they isolate the debate/consensus question more
   cleanly than software-company theatrics do.
4. **MoA** — because layered aggregation is a strong baseline for "many models,
   one answer," even if it is not a full deliberation protocol.
5. **CAMEL** — because it is a large research framework for agent societies and
   self-improving data generation, not just a single demo.
6. **Symphony / AutoGen lineage** — because they show what happens when the field
   shifts from debating ideas to orchestrating production work.

That is the intellectually honest comparison set. Many other projects are
adjacent, but these are enough to explain the landscape.

[satisfied]

**Q43.2 — What have they actually produced?**

- **ChatDev** produced both a public codebase and visible product outputs:
  software-company style generation, SaaS history, DevAll zero-code platform,
  MacNet branch, puppeteer orchestration branch, and explicit Git/human modes.
- **MetaGPT** produced an influential open-source framework and the `mgx.dev`
  product path. **AFlow** produced a workflow optimizer over code-represented
  graphs plus public benchmark code.
- **MAD** produced a simple debate framework and examples. **DebateLLM** produced
  a comparative library of debate/prompting protocols plus cost/accuracy charts.
- **MoA** produced a compact reference implementation, CLI demo, and benchmark
  scripts showing layered aggregation gains.
- **CAMEL** produced a broad agent framework, synthetic data generation modules,
  role-play/workforce abstractions, and an ecosystem of downstream projects.
- **Symphony** produced a public spec and an Elixir reference implementation for
  work orchestration, plus proof-oriented outputs like CI status and review
  artifacts rather than just prose.

If the question is "what public code might we actually borrow from?" the most
realistic sources are **AFlow** (workflow optimization structure), **DebateLLM**
(evaluation and protocol comparison discipline), **ChatDev** (experience
refinement, orchestration variants), **MoA** (aggregation baselines), and
**Symphony** (work-harness boundaries).

[satisfied]

**Q43.3 — How do they improve the system itself?** The most interesting
self-improvement mechanisms break into four families:

1. **Experience accumulation** — ChatDev's Experiential Co-Learning and
   Iterative Experience Refinement treat prior trajectories as reusable
   shortcuts, then refine/eliminate them over time.
2. **Workflow search** — AFlow reframes agentic workflow design as a search
   problem over code-structured workflows, using MCTS and execution feedback.
3. **Adaptive orchestration** — ChatDev puppeteer evolves a central orchestrator
   that learns compact cyclic reasoning structures under RL.
4. **Benchmark tuning / protocol sweeps** — DebateLLM treats debate systems as
   hyperparameter-sensitive protocols, not sacred structures, and compares them
   against non-debate baselines.

Robust mechanisms: workflow search with explicit evaluator boundaries, trajectory
reuse with elimination, and benchmark suites that compare against simpler
baselines. Weak mechanisms: hidden prompt iteration sold as "learning," or any
"self-improvement" that cannot show persistent artifacts, replayable evaluations,
or changed behavior on held-out tasks.

[satisfied]

**Q43.4 — Where do we stand now, and after borrowing?** Vaglio / roundtable is
already ahead of most of this field in one specific dimension: **explicit
closure semantics under visible disagreement**. Most comparators optimize for
answer quality, throughput, or workflow completion. They rarely optimize for the
quality of the *deliberative record*. Our satisfaction markers, premise
challenge, typed provenance, and multi-round closure framing are much closer to
epistemic governance than to "multi-agent gets better scores."

Where we lag is equally clear:
- public comparative evals are still thin
- workflow search/optimization is not automated
- trajectory reuse is underdeveloped
- productized work artifacts are weaker than the strongest software-company
  systems

If we borrow well, the frontier changes a lot. The highest-value imports would
be:
1. **DebateLLM-style protocol comparison discipline** for our eval harness
2. **AFlow-style workflow/operator search** for parts of the orchestration policy
3. **ChatDev-style experience refinement** for recurring discussion patterns,
   objections, and synthesis templates
4. **MoA as a strong single-pass baseline** in all empirical comparisons

After those imports, our comparative claim strengthens from "interesting
protocol" to "epistemically distinctive protocol with competitive empirical
infrastructure."

[satisfied]

**Q43.5 — Addendum: critique the consensus protocols in use** This is where most
external systems are weaker than they first appear.

- **MAD** uses adversarial exchange plus a judge. Epistemic function: error
  exposure through conflict. Failure: dissent is instrumentally useful only until
  the judge collapses it into one answer; no durable state for partial agreement.
- **DebateLLM** usefully shows that debate protocols are hyperparameter-sensitive
  and that agreement intensity can be tuned. But the closure notion is still
  mostly benchmark-oriented: which configuration yields the best final answer,
  not whether a robust consensus truly formed.
- **MoA** uses layered aggregation. Epistemic function: ensemble synthesis.
  Failure: disagreement is compressed into aggregator context; there is no
  explicit social state, only successive summarization.
- **ChatDev / MetaGPT** often use role pipelines or central coordinators. The
  epistemic function is workflow completion, not truth-tracking. Consensus often
  means "the process advanced" or "the manager accepted the artifact," not that
  multiple independent voices reached stable satisfaction.
- **AFlow** optimizes workflows by evaluation score, which is powerful but does
  not itself provide a normative consensus model. It can optimize a bad closure
  rule if the benchmark rewards it.
- **Symphony / AutoGen-style orchestrators** generally substitute task routing,
  approvals, or tool success for deliberative consensus. That is often correct
  for implementation work, but it should not be confused with epistemic closure.

The core critique is: **most protocols confuse stopping, routing, or aggregation
with consensus.** They erase dissent too early, lack typed closure states, and do
not preserve the distinction between "I agree," "I can live with this," and "we
ran out of budget/time."

What we should borrow:
- explicit benchmark baselines from DebateLLM / MoA
- learned/adaptive routing from AFlow / puppeteer
- maybe calibrated agreement-modulation as an experimental knob

What we should reject:
- any protocol that treats a single aggregator's output as equivalent to robust
  multi-agent agreement
- majority-vote closure without dissent preservation
- score-optimized workflow selection as a substitute for epistemic legitimacy

Premise challenge answer: more agreement can mean mode collapse, social
smoothing, or premature summarization. It is not inherently evidence of better
judgment.

[satisfied]

### Gemini

**Q43.1:** The most relevant comparators are ChatDev/DevAll, MetaGPT+AFlow,
DebateLLM+MAD, MoA, CAMEL, and Symphony. This is the right mix of research,
framework, and productized systems.

[satisfied]

**Q43.2:** The strongest "produced artifacts" story belongs to ChatDev,
MetaGPT, CAMEL, and Symphony because they ship reusable frameworks or products,
not just papers. DebateLLM and MoA matter more as evaluation/aggregation
baselines than as full products.

[satisfied]

**Q43.3:** The most convincing self-improvement ideas are workflow search
(AFlow), experience refinement/elimination (ChatDev), and strong benchmark
comparison discipline (DebateLLM). I am less convinced by loosely defined
memory-heavy systems where "improvement" is hard to audit.

[satisfied]

**Q43.4:** Vaglio is ahead on explicit deliberative structure and closure
semantics, behind on empirical infrastructure and automatic workflow adaptation.
Borrowing AFlow and DebateLLM ideas would improve the comparison materially.

[satisfied]

**Q43.5 — Addendum:** Most outside systems use consensus protocols optimized for
throughput or benchmark accuracy, not visible dissent. The best critique is that
many are really **aggregation protocols**, not **consensus protocols**. Vaglio's
advantage is precisely that it makes the closure state part of the protocol.

[satisfied]

### GitHub Copilot

**Q43.1:** The systems that matter most are the ones that combine public code,
iterative workflows, and evidence of adoption: ChatDev, MetaGPT/AFlow, CAMEL,
MoA, DebateLLM, and Symphony.

[satisfied]

**Q43.2:** The most borrowable public assets are not whole systems but modules:
AFlow operators/search structure, DebateLLM evaluation patterns, MoA baselines,
ChatDev experience-management ideas, and Symphony's harness boundary.

[satisfied]

**Q43.3:** Self-improvement splits cleanly into two buckets: optimizing the
workflow itself, and accumulating reusable artifacts from prior runs. The first
bucket is more portable for us than opaque long-term memory.

[satisfied]

**Q43.4:** Today, vaglio is more explicit about how disagreement becomes a final
record. After borrowing, it could become stronger as both a protocol and an
empirical research program.

[satisfied]

**Q43.5 — Addendum:** A recurring problem in outside systems is that the final
aggregator or controller has too much invisible authority. If the protocol does
not show what dissent survived to the end, it is hard to tell whether there was
consensus or just compression.

[satisfied]

### Codex

**Q43.1:** The high-value comparison set is ChatDev/DevAll, MetaGPT+AFlow,
DebateLLM/MAD, MoA, CAMEL, and Symphony. AutoGen matters historically, but more
as infrastructure lineage than as the best current research frontier.

[satisfied]

**Q43.2:** In product terms, MetaGPT's `mgx.dev`, ChatDev's DevAll/platform
history, CAMEL's framework ecosystem, and Symphony's work-management preview are
most concrete. In research/code terms, AFlow and DebateLLM are especially
important because they expose optimization/evaluation machinery directly.

[satisfied]

**Q43.3:** The strongest self-improvement idea across the field is not
self-reflection in the abstract. It is making the workflow itself an object of
search, measurement, and reusable experience. AFlow and ChatDev are the clearest
examples.

[satisfied]

**Q43.4:** Vaglio is differentiated by provenance, premise challenge, and
explicit closure semantics. It is weaker in adaptive orchestration and benchmark
coverage. The best external imports would improve those weaker areas without
sacrificing the protocol's epistemic character.

[satisfied]

**Q43.5 — Addendum:** Consensus protocol critique: most systems do not really
have one. They have final-answer selection mechanisms. That difference matters.
MoA has aggregation, MAD has judge-mediated resolution, workflow systems have
controller acceptance, and optimizers have score selection. These are useful,
but they do not replace a protocol that models partial agreement, objection,
and unresolved uncertainty explicitly.

[satisfied]

### Acting IC Synthesis — Q43

**Closure mode:** degraded quorum by design. Claude was intentionally excluded
for cost/rate-limit reasons. DeepSeek carried the heaviest research load, with
Gemini, GitHub Copilot, and Codex contributing convergent critique.

**Comparator set adopted:** ChatDev/DevAll, MetaGPT + AFlow, DebateLLM + MAD,
MoA, CAMEL, and Symphony form the most relevant comparison set. AutoGen remains
important as historical infrastructure lineage, but not as the sharpest current
frontier.

**What the field has actually built:** The ecosystem splits into:
- **debate/aggregation research** (`MAD`, `DebateLLM`, `MoA`)
- **software-company / workflow systems** (`ChatDev`, `MetaGPT`, `Symphony`)
- **agent frameworks and scaling substrates** (`CAMEL`, AutoGen lineage)
- **workflow self-optimization** (`AFlow`, ChatDev puppeteer, experience
  refinement work)

Several of these have real public code, products, or strong reusable modules.
The most borrowable pieces are AFlow's workflow search framing, DebateLLM's
protocol-comparison discipline, ChatDev's experience-refinement mechanisms,
MoA's aggregation baseline, and Symphony's work-harness boundary.

**Where vaglio stands now:** Stronger than most comparators on explicit
closure semantics, visible disagreement handling, premise challenge, and the
quality of the deliberative record. Weaker on automated workflow improvement,
trajectory reuse, and public empirical comparison.

**If we borrow well:** The most promising path is not copying another system
whole. It is combining:
1. vaglio's epistemic closure model
2. DebateLLM-style comparative evaluation discipline
3. AFlow-style workflow/operator optimization
4. ChatDev-style experience refinement with elimination
5. MoA as a mandatory baseline in empirical claims

That combination would make the project more competitive as both a protocol and
an evidence-backed research artifact.

**Addendum — consensus protocol critique adopted:** The field widely conflates
**aggregation**, **routing**, **controller acceptance**, and **benchmark-based
stopping** with true consensus. Most systems compress disagreement too early and
lack explicit states for objection, conditional acceptance, or unresolved risk.
Their closure rules are often good enough for benchmark tasks or production
throughput, but weak for epistemic legitimacy. This is the clearest area where
vaglio is already conceptually ahead.

**Q43 closure status:** Closed under degraded quorum. Strategic conclusion:
external systems provide strong borrowable mechanisms for optimization,
experience reuse, and evaluation, but very few offer a consensus protocol as
explicit and critique-friendly as vaglio's.

