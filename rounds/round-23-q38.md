## Round 23 — Q38: Eval Landscape and Competitive Analysis

### Codex

**Q38.1 — Benchmark landscape:** Assessed 10+ benchmarks. Strongest fits for
vaglio: GPQA Diamond (expert reasoning), BigBench-Hard (causal/logical), MuSR/FOLIO
(multi-step reasoning). No existing benchmark measures multi-agent deliberation
quality. Closest academic precedent: Multi-Agent Debate (MAD) paper evaluations on
TruthfulQA and StrategyQA. SWE-bench, HumanEval, ARC-AGI rated as poor fits.

**Q38.2 — Competitive products:** Sorted into three categories. Different problem:
Bito, Cursor, Devin, SWE-Agent, OpenHands, Aider, Copilot Workspace (all coding
agents). Adjacent: AutoGen (closest framework — GroupChat with speaker selection),
CrewAI (workflow, not deliberation), MetaGPT/ChatDev (code production pipelines).
Same problem: MAD research systems (academic, not productized), MoA/LLM-Blender
(single-pass aggregation, not iterative deliberation). "Vaglio has very few direct
competitors because the combination is genuinely novel. The risk is not competition
— it is that the niche is too narrow."

**Q38.3 — Edge:** Design decisions with tradeoffs, premise validation, calibrated
confidence (disagreement as signal), homelab-specific architecture decisions,
subjective/multi-dimensional evaluation tasks. Does NOT have edge on: single-answer
tasks, latency-sensitive tasks, knowledge retrieval, cost-sensitive tasks.

**Q38.4 — Strategy:** Custom evals first (items 28-32). If positive, run GPQA
Diamond (198 questions), BigBench-Hard subtasks, TruthfulQA. Do NOT run SWE-bench,
HumanEval, MATH, ARC-AGI. Public benchmark requires deterministic reproducibility
and multiple trials.

[satisfied-conditional: items 28-32 must define concrete metrics]

### Gemini

**Q38.1:** No public benchmark measures multi-agent deliberation quality — this is
the core finding. GPQA: moderate (has ground truth; vaglio is for open-ended
analysis). MMLU-Pro: low (multiple choice, single correct answers). MT-Bench:
moderate (multi-turn format closer to vaglio). TruthfulQA: moderate (dissent
mechanism could reduce hallucination). SWE-bench/HumanEval/MATH/ARC-AGI: low to
very low.

**Q38.2:** No direct commercial competitors. AutoGen's GroupChat is closest but
targets general-purpose orchestration, not structured deliberation. CrewAI is
workflow-oriented. MAD research validates the approach but has not produced a tool.
MoA is single-pass aggregation, not iterative deliberation.

**Q38.3:** Consideration coverage (H1: 1.5-3x), perspective diversity ratio (H2:
>=40%), blind preference, premise challenges surfaced, contradiction detection.
Edge does NOT extend to factual QA, code generation, speed-critical tasks.

**Q38.4:** Focus exclusively on custom evals. One targeted addition: TruthfulQA or
FreshQA subset (~$10-15) as secondary validation. "What vaglio should publish
instead of benchmark scores: the eval report from item 32."

[satisfied]

### DeepSeek

**Q38.1:** Agreed no public benchmark measures deliberation quality. MT-Bench and
WildBench rated as best proxies (open-ended quality, LLM-as-judge scoring). GPQA
Diamond: "partially, but risky — underperforming single Opus would be a bad look."
Emphasized the distinction: GPQA has correct answers, vaglio is for questions that
don't.

**Q38.2:** Strongest overlap: AutoGen (configurable framework), MoA (simpler
architecture, similar insight). Unique strategic observation: "users will compare
vaglio's cost against single-agent tools even if the tasks differ — the burden is
on vaglio to demonstrate the deliberation dimension is worth paying for."

**Q38.3:** Best demonstration dimensions: consideration count/diversity ratio, blind
preference on ambiguous questions, error detection in code review, resistance to
anchoring (misleading premises). Flagged: the eval report should include
cost-per-quality-unit.

**Q38.4:** Custom evals (items 28-32) first. If positive, MT-Bench as credibility
check (80 questions, widely cited, measures open-ended quality). Statistical
caution: N=6 in Phase 1 gives p=0.094 on one-tailed binomial for 5/6 preference.
Report effect sizes and confidence intervals, not just win rates.

[satisfied-conditional: items 28-32 must proceed as specified]

### IC Synthesis — Q38

**Consensus:** (1) Comparing vaglio to SWE-bench coding agents is substantially a
category error. (2) No public benchmark exists for multi-agent deliberation quality.
(3) Vaglio has no direct commercial competitors; AutoGen is closest framework, MoA
closest research validation. (4) Custom evals (items 28-32) should be primary focus.
(5) Vaglio's genuine edge is on open-ended design questions.

**Key divergence resolved:** On which public benchmark to try if internal evals are
positive — DeepSeek's MT-Bench recommendation adopted (measures open-ended quality
via LLM-as-judge, low cost, widely cited, no format adaptation needed).

**Recommendations:** Complete items 28-32. If H1/H2 confirm, run MT-Bench as
credibility data point. Optionally TruthfulQA for hallucination resistance. Report
cost-per-quality-unit in all results. Do NOT run SWE-bench/HumanEval/MATH/ARC-AGI.

**Q38 closure status:** All agents satisfied. Consensus reached.

---
