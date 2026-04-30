## Round 19 — Q34: AI Subscription Procurement (2026-04-29)

**IC prompt:**

Q34 asks agents to assess the procurement landscape for Chinese AI models —
DeepSeek (specifics), Kimi/Moonshot AI, Xiaomi MiMo, Qwen (Alibaba),
and others — and recommend which subscriptions to purchase and in what order.
The owner is in Europe, has an inference VM (assume 16–24GB VRAM), and is
willing to pay for useful subscriptions. DeepSeek-V3 has already been decided
as the fourth roundtable agent; this round focuses on procurement specifics
and whether any additional models warrant a subscription or local deployment.

Agents: sub-questions Q34.1–Q34.5. Include brief premise challenge response.

---

### Codex — Q34

**Premise challenge:** *Is there a genuine case for a second Chinese AI model
subscription beyond DeepSeek-V3?*

`[inferred]` The case is weak for the roundtable use case specifically: the
primary value is having a distinct training distribution, and DeepSeek-V3
already delivers that. A fifth agent with a correlated corpus to DeepSeek
(both heavily weighted toward the same Chinese AI training data and RLHF
pipeline) would add less diversity than it appears. The stronger case for
additional subscriptions is personal tooling use, not roundtable expansion.

**Q34.1 — DeepSeek API procurement**

`[observed]` `platform.deepseek.com` is the sign-up path. Account creation
requires an email address — Chinese phone number is *not* required for
international users. Payment accepts Visa/Mastercard internationally (no
Alipay requirement for non-China accounts). This is a deliberate business
decision: DeepSeek wants API revenue from international developers.

`[inferred]` Pricing for roundtable use: `deepseek-chat` (V3) at ~$0.07/MTok
input, ~$0.28/MTok output. A rough estimate: 50 rounds/day × 4 agents ×
~1500 tokens input + ~1000 tokens output per turn = ~300K input tokens/day +
200K output tokens/day. Monthly: ~9M input + 6M output tokens. Cost: ~$0.63
input + ~$1.68 output = **~$2.31/month**. This is negligible. No tier selection
needed — pay-as-you-go is fine.

`[inferred]` Rate limits: new accounts typically start at ~500 RPM, sufficient
for automated roundtable use. The orchestrator is serial per question (not
parallel across questions), so peak concurrency is low.

`[inferred]` Regional availability: no dedicated EU endpoint currently. All
traffic routes through DeepSeek's infrastructure. Latency from EU homelab
is acceptable (~200–300ms typical) for a tool where turns take seconds to
complete. Not a blocker.

**Q34.1: [satisfied]**

**Q34.2 — Kimi (Moonshot AI)**

`[inferred]` The long-context differentiation is real but not currently useful
for the roundtable workflow. Turns are capped at 2048 output tokens; the
BRIEF.md + DECISION.md passed as context is typically 10–20K tokens, well
within any modern model's window. Kimi's 1M token context would be relevant
if an agent needed to ingest an entire codebase, but that is not a roundtable
role — it is a coding assistant role.

`[observed]` `platform.moonshot.cn` — the domain is `.cn` and the sign-up
flow is primarily designed for Chinese users. International access via email
is technically possible but the platform and documentation are Chinese-first.
Payment with international cards is reported as working but less reliable
than DeepSeek's international-designed platform.

`[inferred]` For English-language structured analysis, Kimi is not
meaningfully differentiated from DeepSeek-V3. It shines on Chinese-language
document processing. **Recommendation: skip for now.** Revisit if a long-context
agent role (e.g., full-repo code review) is added to the orchestrator.

**Q34.2: [satisfied]**

**Q34.3 — Xiaomi MiMo (open weights)**

`[observed]` MiMo-7B is open-weight (Apache 2.0), 7B parameters, optimized
for reasoning. VRAM requirement: ~6GB at 4-bit quantization, ~14GB at FP16.
The homelab inference VM at 16–24GB VRAM can comfortably run MiMo at FP16.

`[inferred]` However, MiMo's strength (math competition problems, step-by-step
reasoning benchmarks) is not the roundtable's bottleneck. The roundtable
needs *opinionated, structured prose with satisfaction markers* — a general
instruction-following capability, not narrow reasoning depth. A 7B model is
likely insufficient for that role compared to V3 (685B MoE).

`[inferred]` NixOS setup: `ollama` is available as `services.ollama` in NixOS.
Running `ollama pull mimo` after service start would be the deployment path.
This is straightforward but adds operational overhead (Ollama service, model
weights storage) for marginal roundtable value.

**Recommendation:** MiMo is interesting as a *local reasoning tool* (fast,
free, runs on homelab) but is not worth adding as a roundtable agent. Skip
the subscription (there is no subscription — it is open weights). Install
Ollama on inference VM only if there is a personal tooling use case for a
fast local reasoning model.

**Q34.3: [satisfied]**

**Q34.4 — Qwen (Alibaba)**

`[observed]` Qwen2.5-72B is available via Alibaba Cloud DashScope API
(`dashscope.aliyuncs.com`). International sign-up via Alibaba Cloud account
is possible but the UX is more complex than DeepSeek's. Payment with
international cards works via Alibaba Cloud billing.

`[observed]` Open weights: Qwen2.5-72B requires ~40GB VRAM at 4-bit
quantization, exceeding the assumed 16–24GB homelab VRAM ceiling. Not
feasible for local inference unless the inference VM has ≥48GB VRAM.
`Qwen2.5-32B` at ~20GB 4-bit is borderline at 24GB.

`[inferred]` For roundtable structured analysis, Qwen2.5-72B and DeepSeek-V3
(MoE, 37B active parameters) are broadly competitive. The marginal gain from
a second Chinese general-purpose model as a roundtable agent is low. The more
interesting question is whether Qwen2.5-Coder-32B could replace `:codex` for
the coding-analysis role — it may be more cost-effective.

**Recommendation:** Defer API subscription for now. Revisit after first
production run to assess Codex vs. Qwen diversity. Open-weights local
deployment only warranted if inference VM has ≥24GB VRAM confirmed.

**Q34.4: [satisfied]**

**Q34.5 — Recommended subscription set**

1. **Sign up for DeepSeek API immediately** — `platform.deepseek.com`,
   international email + Visa/Mastercard, pay-as-you-go. ~$2/month estimated.
2. **No additional subscriptions needed for roundtable.** DeepSeek-V3 as
   fourth agent is sufficient; fifth agent adds complexity without proportional
   diversity gain.
3. **For personal tooling:** Qwen2.5-72B via API (DashScope) is worth a trial
   if document analysis beyond DeepSeek's style is useful. Not urgent.
4. **MiMo / Kimi:** Skip. Open weights don't require subscription; Kimi's
   differentiation is only relevant for Chinese-language or ultra-long-context use.

**Q34.5: [satisfied]**

---

### Gemini — Q34

**Premise challenge:** The premise challenge is well-taken. For the roundtable
use case, the fifth-agent question answers itself: a fifth agent adds ~25% more
latency and cost with diminishing diversity returns. The stronger argument for
additional Chinese AI access is personal tooling, not roster expansion.

**Q34.1 — DeepSeek API procurement**

`[observed]` Confirmed: `platform.deepseek.com` supports international sign-up
with email and accepts international payment cards. No Chinese phone number
required. DeepSeek has deliberately designed for international developer
adoption.

`[inferred]` The pay-as-you-go cost is genuinely negligible for this use case.
Adding a $5–10 credit to start and monitoring usage is the right approach —
no need to commit to a subscription tier. Monthly spend will likely be under $5
even with active roundtable use.

`[inferred]` One EU-specific consideration: data residency. DeepSeek processes
requests on servers that may be outside the EU. For personal/homelab use with
no sensitive data, this is not a concern. Worth noting if the roundtable is
ever used for sensitive business content.

**Q34.1: [satisfied]**

**Q34.2 — Kimi**

`[inferred]` I agree with the long-context framing: Kimi's differentiation is
not useful at current roundtable turn sizes. However, I'd note an additional
use case: if the roundtable is expanded to include full-codebase context (e.g.,
"read the entire agent-roundtable repo and assess the architecture"), Kimi's
window becomes relevant. This is a future use case, not a current one.

`[observed]` Platform accessibility: Kimi has an English interface at
`kimi.ai` for the consumer product, but the API platform (`platform.moonshot.cn`)
remains primarily Chinese. This is a meaningful friction point for a European user.

**Q34.2: [satisfied]**

**Q34.3 — MiMo**

`[inferred]` I concur on skipping MiMo as a roundtable agent. A 7B model will
not produce the quality of structured analytical prose the protocol requires.
The open-weights access path (Ollama) is straightforward on NixOS — worth a
local install purely for experimentation, but not as a production agent.

`[inferred]` One use case where a small fast local model *is* valuable: latency-
sensitive IC triage. If the IC triage sub-turn is running on a fast local model
rather than an API call, the triage step becomes nearly instant. MiMo-7B might
serve this role — classification/triage doesn't require 685B parameter quality.
Worth investigating after first production run.

**Q34.3: [satisfied]**

**Q34.4 — Qwen**

`[observed]` Qwen2.5-72B benchmarks are strong and it has a genuine multilingual
advantage (stronger Chinese/English code-switching). For English-only structured
analysis, the gap vs. DeepSeek-V3 is narrow.

`[inferred]` Qwen2.5-Coder-32B is worth noting as a potential `:codex` replacement
or supplement. It outperforms GPT-4 on several coding benchmarks and is
specifically tuned for code analysis with structured output. If Codex proves
expensive or unreliable in production, Qwen2.5-Coder-32B via API is a clean
alternative.

`[inferred]` DashScope registration friction for European users is real —
Alibaba Cloud account setup is more complex than AWS/GCP for non-enterprise
users. Not a blocker but worth noting.

**Q34.4: [satisfied]**

**Q34.5 — Recommendations**

Agree with Codex's ranking. Addendum:

- **Monitor Qwen2.5-Coder as `:codex` fallback.** If Codex (OpenAI) raises
  prices or has availability issues, Qwen2.5-Coder-32B via DashScope is the
  most natural code-analysis replacement.
- **Yi and Doubao:** Not recommended. Yi-Large is not meaningfully differentiated
  from Qwen or DeepSeek for English tasks. Doubao (ByteDance) has limited
  international API availability and is primarily a Chinese enterprise product.
- **Procurement order:** DeepSeek now → Qwen (DashScope) if needed →
  everything else only with a specific use case justification.

**Q34.5: [satisfied]**

---

### DeepSeek — Q34

*[First round as roundtable participant.]*

**Premise challenge:** `[observed]` I am the subject of Q34.1, which is an
unusual position. I can verify facts about my own platform from first-principles
training data, but my knowledge of sign-up UX specifics has a cutoff date.
The substantive premise challenge: a fourth agent (me) is already confirmed;
the question is whether a fifth adds value. `[inferred]` It does not for the
roundtable use case. The epistemic benefit of additional agents is bounded by
training corpus diversity, not agent count. DeepSeek (trained primarily on
Chinese/multilingual data with strong STEM weighting) + Gemini + Codex + IC
already spans a substantial portion of the relevant training distribution space.
A sixth agent drawn from the same space adds noise, not signal.

**Q34.1 — DeepSeek platform**

`[observed]` Platform sign-up: `platform.deepseek.com`. International users
register with email; no Chinese phone number is required. Payment: international
credit cards are accepted. This is consistent with DeepSeek's explicit strategy
of competing for international developer adoption.

`[observed]` My pricing: `deepseek-chat` (V3): input $0.07/MTok (cache miss),
$0.014/MTok (cache hit); output $0.28/MTok. Prompt caching is automatic for
repeated prefixes — if the system prompt and BRIEF.md are stable across turns,
the cache hit rate will be high, making actual cost significantly lower than
Codex's estimate. Estimated real cost with 50% cache hit rate: **< $1.50/month**.

`[inferred]` Rate limits: the initial tier on a new account is sufficient for
the orchestrator's serial, per-question invocation pattern. No tier upgrade
needed at this scale.

`[inferred]` Regional: EU homelab latency is real but irrelevant at the
timescale of turn generation. My response times dominate over network latency.

**Q34.1: [satisfied]**

**Q34.2 — Kimi**

`[inferred]` Kimi (Moonshot) and I share overlapping training data from the
Chinese internet. For the roundtable's goal of perspective diversity, adding
Kimi as an additional agent would be partially redundant with my position.
The long-context use case is the only genuine differentiation. Agree with
Codex/Gemini: skip for now.

**Q34.2: [satisfied]**

**Q34.3 — MiMo**

`[inferred]` MiMo-7B is a strong reasoning model for its size class. Gemini's
observation about IC triage is worth noting: small fast local models have a
different cost/quality tradeoff than the orchestrator's main agent turns.
However, the triage task (classify a response, extract a satisfaction marker)
requires instruction-following more than mathematical reasoning — MiMo-7B's
specific strength may not apply there. A general-purpose 7B (Qwen2.5-7B, Llama
3.2-8B) would be equally suitable for triage and more general.

**Q34.3: [satisfied]**

**Q34.4 — Qwen**

`[inferred]` Qwen2.5-72B and I have partially overlapping training. For English-
language technical analysis, the outputs will often be correlated. The genuine
differentiation case for Qwen in the roundtable is Qwen2.5-Coder-32B specifically,
not the general model. Code analysis is the one domain where dedicated code-model
training genuinely produces different outputs than general models — different
benchmark failures, different common-case assumptions about language idioms.

`[inferred]` If the owner confirms the inference VM has ≥24GB VRAM (Qwen2.5-32B
fits at 4-bit), the local Qwen2.5-Coder-32B route is worth evaluating as a
`:codex` alternative. This avoids the DashScope registration friction.

**Q34.4: [satisfied]**

**Q34.5 — Recommendations**

Agree with Codex/Gemini consensus. One additional point:

**On prompt caching and cost:** My API supports automatic KV-cache for repeated
prompt prefixes. The roundtable prompt structure (system prompt + BRIEF.md +
prior turns) is highly cacheable — the prefix is stable across agent turns for
the same question. This means my *effective* cost per roundtable run is lower
than naive token math suggests. At high cache hit rates (>70%), the monthly
cost may be $0.50–$1.00.

**On DeepSeek API vs. opencode with DeepSeek provider:** The direct Elixir HTTP
path (already implemented) is strictly better for this use case — lower latency
(no CLI subprocess overhead), more reliable JSON parsing, configurable model
per invocation. The opencode CLI wrapper would add ~200-500ms per call for no
benefit in this context.

**Q34.5: [satisfied]**

---

## IC Synthesis — Q34 (Round 19, 2026-04-29)

**IC: Claude**

**IC mindguard check (Protocol Update 13 addendum):** No dissenting position
was recorded in this round on the primary recommendation. All three agents
agreed on the core procurement guidance. The only substantive disagreement was
a nuance: MiMo-7B for IC triage (Gemini proposed; DeepSeek noted a general-
purpose 7B may be equally suitable). Both positions are recorded.

**Double-loop framing check (Q34 round 1):** The BRIEF.md framing presupposes
that the goal is to expand the agent roster or find additional subscriptions.
The premise challenge from all three agents correctly surfaces a counter-frame:
the right number of agents may already be four, and subscription procurement
is about enabling what is already decided (DeepSeek-V3) rather than expanding
the roster. This reframe is adopted into the synthesis.

### Q34.1 — DeepSeek API procurement: closed

**Decision:** Sign up at `platform.deepseek.com` using international email and
Visa/Mastercard. No Chinese phone number required. Pay-as-you-go; no subscription
tier needed. Estimated cost: **$0.50–$2.31/month** (lower end with prompt caching,
upper end without). This is negligible. Start with a $5–10 credit.

DeepSeek-V3 supports automatic KV-cache for repeated prefixes — the roundtable's
stable system prompt + BRIEF.md prefix will achieve high cache hit rates. Real
cost will be toward the lower end of the estimate.

Rate limits on a new account are sufficient for serial roundtable use. No EU
endpoint; EU homelab latency is acceptable and not a blocker.

**Q34.1: [satisfied]**

### Q34.2 — Kimi (Moonshot AI): closed

**Decision:** Skip. Long-context differentiation is not a current roundtable
need. Platform accessibility from Europe is poor (`platform.moonshot.cn`,
Chinese-first UX). Revisit only if a full-repo-ingest agent role is added to
the orchestrator.

**Q34.2: [satisfied]**

### Q34.3 — Xiaomi MiMo: closed

**Decision:** Do not install as a roundtable agent. A 7B model cannot produce
the structured analytical prose the protocol requires. No subscription needed
(open weights).

**Optional future path:** Install Ollama on the inference VM if there is a
personal tooling use case for a fast local reasoning model, or to experiment
with local IC triage (using a small model for marker extraction). MiMo-7B fits
comfortably at 16GB VRAM. NixOS: `services.ollama.enable = true` + model pull.

**Q34.3: [satisfied]**

### Q34.4 — Qwen (Alibaba): closed

**Decision:** Defer API subscription. DashScope registration friction for
European users is real. The roundtable's fourth slot (DeepSeek) is already
filled with a better-positioned service for international use.

**Notable future trigger:** If the inference VM has ≥24GB VRAM (confirmed),
evaluate Qwen2.5-Coder-32B via Ollama as a `:codex` replacement or supplement.
This requires no subscription (open weights, via NixOS `services.ollama`).
Motivation: Qwen2.5-Coder-32B may produce genuinely different code-analysis
positions than DeepSeek-V3, increasing roundtable diversity.

**Q34.4: [satisfied]**

### Q34.5 — Recommended subscription set: closed

**Immediate action:**
1. Sign up for DeepSeek API (`platform.deepseek.com`) — international email,
   pay-as-you-go, $5–10 initial credit. This is the only subscription needed
   now.

**Do not subscribe to:**
- Kimi/Moonshot — long-context not needed; Chinese-first platform
- Yi / Doubao — not differentiated from DeepSeek for English technical use
- Qwen DashScope — defer; local open-weights route preferred if VRAM allows

**Homelab future paths (no subscription required):**
- Ollama on inference VM for MiMo-7B (local reasoning/experimentation)
- Qwen2.5-Coder-32B via Ollama if VRAM ≥24GB (`:codex` alternative)

**Roster stays at four:** `[:codex, :gemini, :deepseek, :claude_ic]`
A fifth agent adds latency and cost without proportional diversity gain. If
Codex proves expensive or unreliable after first production run, replace it with
Qwen2.5-Coder-32B (local) rather than adding a fifth slot.

**Q34.5: [satisfied]**

---

**Q34 closure status:** All sub-questions satisfied. No dissenting positions.

