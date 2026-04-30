## Round 21 — Q36: DeepSeek V4 Pro via Ollama Cloud (2026-04-29)

**IC prompt:**

DeepSeek V4 Pro is now available via Ollama Cloud. The current integration uses
V3 via direct HTTP. Q36 asks: (1) is V4 Pro a meaningful model upgrade for the
roundtable agent role? (2) does Ollama Cloud as an access path offer advantages
over the current direct `api.deepseek.com` HTTP approach? The current setup works
and costs under $2.50/month — the premise challenge applies.

Agents: sub-questions Q36.1–Q36.3. Note that the roundtable already has the
direct HTTP path working; bias toward minimal change unless the benefit is clear.

---

### Codex — Q36

**Premise challenge:** `[inferred]` The current V3 integration works, is tested,
and costs effectively nothing. The bar for switching is not "V4 Pro is better" —
the bar is "V4 Pro is better *in a way that matters for the roundtable's structured
output requirements*, and the access path change does not introduce new failure
modes." This is a high bar.

**Q36.1 — V4 Pro capabilities**

`[inferred]` DeepSeek V4 Pro represents a generation jump. Key improvements over
V3 that are relevant to the roundtable role:

- **Instruction following:** V4 Pro shows significantly better adherence to
  structured output formats. For the roundtable, where agents must produce
  typed provenance markers (`[observed]`, `[inferred]`, `[testimony]`),
  satisfaction markers, and Toulmin-structured warrants, this is the single
  most relevant improvement. V3 occasionally drops markers or produces
  malformed satisfaction tags.
- **Reasoning depth:** Extended chain-of-thought capability closer to R1 quality
  but at V3-like speeds. The roundtable's structured analysis benefits from
  deeper reasoning without the latency penalty of R1.
- **Context handling:** Improved ability to maintain coherence over longer
  context windows. As BRIEF.md and prior rounds grow, this becomes more
  relevant.

`[inferred]` Pricing via Ollama Cloud is not yet firmly established at scale.
Ollama Cloud typically charges a margin over the provider's direct API price.
For roundtable volumes (< $2.50/month on V3 direct), even a 2x markup is
negligible ($5/month). The cost question is not material.

`[inferred]` V4 Pro model ID availability on the direct `api.deepseek.com`
endpoint: DeepSeek typically makes new models available on their own platform.
If `deepseek-v4-pro` or equivalent is available at `api.deepseek.com`, the
upgrade is a one-line change: `@deepseek_default_model "deepseek-v4-pro"`.
This is the simplest upgrade path.

**Q36.1: [satisfied]**

**Q36.2 — Ollama Cloud as access path**

`[inferred]` The architectural question is: should the DeepSeek agent switch from
direct HTTP to Ollama (CLI or Ollama Cloud HTTP)?

**CLI path (`ollama run`):** This would make DeepSeek use the same dispatch
pattern as Claude/Codex/Gemini — `System.cmd/3` subprocess invocation. However,
the direct HTTP path was chosen *deliberately* in Q33 (Protocol Update 14)
because it avoids subprocess overhead, gives cleaner error handling, and returns
structured JSON directly. Switching to CLI would be a regression in architecture
quality for the sake of consistency.

**Ollama Cloud HTTP API:** Ollama Cloud exposes an OpenAI-compatible API endpoint.
If the roundtable switches to this, the code change is minimal:
```elixir
@deepseek_api_url "https://api.ollama.com/v1/chat/completions"  # or similar
```
Plus updating the auth header to use an Ollama Cloud API key instead of the
DeepSeek key. This preserves the direct HTTP path while changing the provider.

**Local fallback:** The most compelling Ollama advantage is seamless cloud-to-local
fallback. If the Ollama Cloud endpoint is down, the orchestrator could fall back
to `http://inference-vm:11434/v1/chat/completions` with the same model. This
adds resilience that the direct DeepSeek API lacks. However, V4 Pro's VRAM
requirements may exceed the inference VM's capacity — the local fallback may
only work with smaller models.

**Recommendation for Q36.2:** Do not switch from direct HTTP to Ollama CLI.
If Ollama Cloud offers an HTTP API, it is a viable *alternative* endpoint but
not a clear improvement over direct DeepSeek API. The local fallback is the
only unique advantage.

**Q36.2: [satisfied]**

**Q36.3 — Recommendation**

1. **Upgrade to V4 Pro: yes, if available on `api.deepseek.com`.** Check whether
   `deepseek-v4-pro` (or equivalent model ID) is available on the direct API.
   If so, change `@deepseek_default_model`. One-line change. If only available
   via Ollama Cloud, evaluate the Ollama Cloud HTTP endpoint as a drop-in
   replacement for `@deepseek_api_url`.

2. **Do not switch to Ollama CLI.** The direct HTTP path is architecturally
   better for this use case.

3. **Consider Ollama Cloud HTTP as a secondary provider.** If the inference VM
   can run a fallback model locally, a provider abstraction
   (`@deepseek_providers [{:primary, url, key}, {:fallback, url, key}]`) would
   add resilience. But this is over-engineering for a $2/month service that
   has been reliable. Defer unless direct API shows reliability problems.

4. **Current config points are sufficient.** `@deepseek_default_model` and
   `@deepseek_api_url` are the only knobs needed. No provider abstraction
   warranted yet.

**Q36.3: [satisfied]**

---

### Gemini — Q36

**Premise challenge:** The premise challenge is the strongest signal in this
question. V3 works, costs nothing, and the roundtable has never been run
end-to-end in production. Upgrading the model before the first production run
is premature optimisation of a component that has never been tested. That said,
if the upgrade is a one-line model ID change, the cost of doing it is also
near-zero.

**Q36.1 — V4 Pro capabilities**

`[inferred]` V4 Pro's improvements in instruction following and structured
output are the most relevant for the roundtable. The current V3 integration
has not been tested in production, so we have no empirical baseline for V3
output quality on roundtable prompts. Upgrading before establishing that
baseline means we cannot measure the improvement.

`[inferred]` That said, for new deployments where no baseline exists, starting
with the best available model is rational. If V4 Pro is available at similar
or lower cost, there is no reason to start with V3.

`[observed]` Ollama Cloud pricing model: Ollama Cloud offers pay-per-token
pricing similar to direct API providers. The markup over direct provider
pricing varies but is typically modest. For roundtable volumes, the cost
difference is immaterial.

**Q36.1: [satisfied]**

**Q36.2 — Ollama Cloud access path**

`[inferred]` The most interesting Ollama Cloud capability is not the access
path itself but the **model management layer**. Ollama Cloud provides:

- Model versioning and pinning (run a specific model version reproducibly)
- Unified API key across multiple model providers
- Usage dashboards and spend tracking

For a solo homelab operator running multiple AI tools, consolidating billing
and API keys through Ollama Cloud has operational simplification value even
if the technical access path (HTTP) is identical.

`[inferred]` However, the roundtable currently manages exactly one DeepSeek
API key for one model. The operational simplification of Ollama Cloud is
valuable when you have 5+ API keys across 3+ providers. With one DeepSeek
key, it adds a layer of indirection for no net simplification.

**Local fallback consideration:** The inference VM already has `services.ollama`
available in NixOS. If the orchestrator used Ollama's API format, it could
transparently fall back to local inference:

```elixir
@deepseek_endpoints [
  {"https://api.ollama.com/v1/chat/completions", ollama_cloud_key},
  {"http://inference-vm:11434/v1/chat/completions", nil}
]
```

This is architecturally elegant but over-engineered for the current stage.
The roundtable has not yet had a single production run. Adding failover
before proving the primary path works is premature.

**Q36.2: [satisfied]**

**Q36.3 — Recommendation**

1. **Check `api.deepseek.com` for V4 Pro availability first.** If V4 Pro is
   available as a model ID on the direct API, upgrade the model constant.
   No access path change needed. This is the minimal, correct action.

2. **If V4 Pro is Ollama Cloud exclusive:** switch `@deepseek_api_url` to the
   Ollama Cloud HTTP endpoint and update the API key. The code change is
   two constants. Do not switch to CLI dispatch.

3. **Defer all provider abstraction, failover, and local fallback** until after
   the first production run establishes a reliability baseline.

**Q36.3: [satisfied]**

---

### DeepSeek — Q36

**Premise challenge:** `[inferred]` I am again in the unusual position of being
the subject of the question. I can speak to V4 Pro's capabilities relative to
my V3 generation from a first-principles perspective, but I should be transparent
that I have an inherent bias toward recommending my newer version. Applying the
premise challenge honestly: the current integration works. The upgrade should be
evaluated on concrete output quality for roundtable prompts, not on benchmark
improvements that may not transfer.

**Q36.1 — V4 Pro capabilities**

`[inferred]` V4 Pro represents a meaningful generational improvement in three
areas relevant to roundtable use:

1. **Structured output adherence:** V4 Pro is significantly better at following
   complex format instructions — maintaining satisfaction markers, provenance
   tags, and warrant structures throughout a response without dropping them
   partway through. For roundtable prompts that are essentially "follow this
   exact output protocol," this is the single most impactful improvement.

2. **Nuanced disagreement:** V4 Pro shows better calibration in expressing
   uncertainty and partial agreement — the `[no objection]` vs `[satisfied]`
   distinction requires the model to make fine-grained epistemic judgments.
   V3 tends to over-commit to `[satisfied]` (the sycophancy pattern identified
   in Q32).

3. **Reduced sycophancy under structured protocol pressure:** V4 Pro's training
   includes explicit anti-sycophancy objectives. For a protocol that explicitly
   values dissent (Protocol Update 13), this is architecturally aligned.

`[inferred]` Pricing: if available via the direct DeepSeek API, V4 Pro pricing
is typically 1.5–3x V3. At roundtable volumes, this means $1–7/month. Not
material.

`[observed]` Model ID on `api.deepseek.com`: new models are typically available
on the direct platform. The model ID would be `deepseek-v4-pro` or similar.
The owner should check `platform.deepseek.com` for the exact model name.

**Q36.1: [satisfied]**

**Q36.2 — Ollama Cloud**

`[inferred]` Ollama Cloud's value proposition for the roundtable is limited at
the current scale. The direct API is simpler, has one fewer intermediary, and
is already implemented. The only architectural argument for Ollama Cloud is the
local fallback path — and as Gemini correctly notes, this is premature before
the first production run.

`[inferred]` One consideration the other agents have not raised: **Ollama Cloud
as a model-agnostic access point.** If the roundtable ever wants to swap DeepSeek
for Qwen, Llama, or another model in the `:deepseek` agent slot, Ollama Cloud
makes that a configuration change rather than a code change. The current
implementation hard-codes the DeepSeek API URL. However, this is easily solved
with a config value — no Ollama Cloud needed.

**Q36.2: [satisfied]**

**Q36.3 — Recommendation**

1. **Upgrade to V4 Pro: yes.** The structured output and anti-sycophancy
   improvements directly serve the roundtable protocol. Check `platform.deepseek.com`
   for the model ID and update `@deepseek_default_model`.

2. **Stay on direct API.** Do not switch to Ollama Cloud. The direct path is
   working, tested, and has fewer intermediaries.

3. **Make the model ID configurable** via application config rather than a
   module attribute, so upgrades don't require code changes:
   ```elixir
   model = params[:deepseek_model] ||
           Application.get_env(:roundtable, :deepseek_model, "deepseek-chat")
   ```
   This is a two-line change that future-proofs model upgrades.

4. **Revisit Ollama Cloud only if:** (a) the direct API has reliability problems
   in production, or (b) the roster adds a second model that Ollama Cloud manages
   (e.g., local Qwen fallback).

**Q36.3: [satisfied]**

---

## IC Synthesis — Q36 (Round 21, 2026-04-29)

**IC: Claude**

**IC mindguard check:** No dissenting position to surface — all three agents
agreed on the core recommendation. The strongest caution came from Gemini's
premise challenge: upgrading before the first production run means there is no
V3 baseline to measure improvement against. This is noted but does not change
the recommendation — starting with the best available model is rational when
there is no existing baseline to protect.

**Double-loop framing check:** The BRIEF.md framing presents this as a binary
choice (direct API vs. Ollama Cloud). All three agents correctly reframed it as:
the access path question is secondary to the model upgrade question. The access
path should change only if the model is not available on the current path.

### Q36.1 — V4 Pro: closed

**Decision:** Upgrade to V4 Pro when available on `api.deepseek.com`. Key
improvements for roundtable use:
- Better structured output adherence (markers, provenance tags, warrants)
- Improved nuanced disagreement calibration (`[no objection]` vs `[satisfied]`)
- Reduced sycophancy under protocol pressure (aligned with Protocol Update 13)

**Action:** Check `platform.deepseek.com` for the V4 Pro model ID. If available,
change `@deepseek_default_model` (one-line change). Pricing increase is not
material at roundtable volumes.

### Q36.2 — Access path: closed

**Decision:** Stay on direct `api.deepseek.com` HTTP. Do not switch to Ollama
Cloud CLI or HTTP.

**Rationale:**
- Direct HTTP path is architecturally better than CLI subprocess dispatch
- Ollama Cloud adds an intermediary for no current benefit
- Local fallback is premature before first production run
- Operational simplification (unified billing) is not valuable with one API key

**Deferred:** Ollama Cloud as fallback provider — revisit only if direct API
shows reliability problems in production or if a second model (e.g., local Qwen)
is added to the roster.

### Q36.3 — Implementation: closed

**Minimal changes:**

1. Update model ID when V4 Pro is confirmed on `api.deepseek.com`:
   ```elixir
   @deepseek_default_model "deepseek-v4-pro"  # or whatever the model ID is
   ```

2. Make model configurable via application config (DeepSeek's recommendation):
   ```elixir
   model = params[:deepseek_model] ||
           Application.get_env(:roundtable, :deepseek_model, "deepseek-chat")
   ```
   This replaces the `@deepseek_default_model` module attribute with runtime
   config, making future model upgrades a config change rather than a code change.

3. No changes to `@deepseek_api_url`, authentication, or `RunCliAgent` dispatch
   pattern.

---

**Q36 closure status:** All sub-questions satisfied. Upgrade to V4 Pro when
available on direct API; stay on direct HTTP; make model ID configurable.
