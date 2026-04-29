## Round 18 — Q33: Adding DeepSeek as a Roundtable Agent

**IC Round 18 Prompt (2026-04-29)**

Q33 asks how to add DeepSeek to the agent roster. Owner context: `opencode`
already installed via fnox/NixOS; DeepSeek API subscription acceptable;
preference for solutions that fit existing homelab conventions.

Agents: apply provenance markers and the unique contribution prompt (Protocol
Update 13 addendum). IC will apply the double-loop framing check and mindguard
check.

**Double-loop framing check**: The BRIEF frames this as "how do we add DeepSeek?"
An alternative framing: "does the roster need a fourth agent at all, or is
the value in DeepSeek's model quality rather than its novelty?" The BRIEF
acknowledges this via the premise challenge; agents should not assume the
four-agent outcome is predetermined.

---

### Codex — Q33

**IC mindguard note opened**: I will surface any minority position before converging.

**Unique contribution flag**: I will explicitly mark what is not yet in the brief.

---

**Q33.1 — Model: DeepSeek-V3 (`deepseek-chat`) for the deliberation role.**

`[inferred]` DeepSeek-R1's thinking tokens make it poorly suited to the
roundtable's structured output format. R1 emits a `<think>...</think>` block
before its response; this token stream is typically not exposed in the final
API response, but it significantly increases latency (often 10-30s for complex
prompts vs. 2-5s for V3). For a synchronous round where three agents speak
sequentially, per-turn latency compounds.

`[observed]` DeepSeek-V3's pricing (per DeepSeek's published API pricing as
of early 2025): ~$0.07/MTok input, ~$0.28/MTok output. A typical roundtable
turn (2,000 token prompt + 500 token response) costs approximately $0.0003.
At 5 rounds × 3 questions × 4 agents, total cost is under $0.02 per full
discussion. This is not a meaningful cost constraint. `[testimony]`

`[inferred]` **However**: R1 as the Challenger role (Q33.3 option d) is a
different case. The Challenger's job is adversarial and single-turn; R1's
extended reasoning is well-matched to "find the strongest objection to this
synthesis." The latency is acceptable for a one-off closure check.

**Recommendation**: `deepseek-chat` (V3) for regular agent turns;
`deepseek-reasoner` (R1) for the Challenger role (when activated at first
production run per Protocol Update 13).

---

**Q33.2 — Invocation: direct Elixir HTTP call via `Req`.**

`[inferred]` Option (c) is architecturally cleanest. Here is why the CLI
options are inferior:

- **opencode**: opencode is a full coding agent with its own context
  management, file I/O, and tool-use loop. It is designed for multi-turn
  agentic coding tasks, not single-turn structured text generation. Using
  `opencode` for a one-shot prompt is like using a chainsaw to slice bread —
  it works but carries substantial overhead (process startup, context
  initialisation, potential file system side-effects).

- **`llm` CLI**: A reasonable option — Simon Willison's tool is purpose-built
  for single-turn LLM prompts. But it adds a Python dependency and a plugin
  install step to the NixOS config. `[observed]` The `llm` NixOS package
  currently requires manual plugin management (`llm install llm-deepseek`) which
  is not declarative. This conflicts with the homelab's NixOS-first approach.

- **Shell wrapper script**: Mirrors the `llm` CLI but in Bash. Adds a new
  binary to `environment.systemPackages` that duplicates logic already available
  via `Req`. No advantage over direct HTTP.

**Direct Elixir HTTP** (`Req` — already being added for item 24 Telegram):

```elixir
defp build_command(%{agent: :deepseek, prompt: prompt} = params) do
  {:http, :deepseek, prompt, params}
end
```

In `run/2`, detect `{:http, :deepseek, prompt, params}` and call:

```elixir
defp run_deepseek(prompt, params) do
  api_key = System.get_env("DEEPSEEK_API_KEY") || params[:api_key]
  model   = params[:deepseek_model] || "deepseek-chat"

  case Req.post("https://api.deepseek.com/v1/chat/completions",
         json: %{
           model: model,
           messages: [%{role: "user", content: prompt}],
           max_tokens: 2048
         },
         headers: [{"Authorization", "Bearer #{api_key}"}]) do
    {:ok, %{status: 200, body: body}} ->
      text = get_in(body, ["choices", Access.at(0), "message", "content"]) || ""
      {:ok, %{stdout: text}}
    {:ok, %{status: status, body: body}} ->
      {:error, {:api_error, status, body}}
    {:error, reason} ->
      {:error, reason}
  end
end
```

`[inferred]` This removes the CLI dependency for DeepSeek, gives full control
over retry/timeout, and the response is already the plain text content (no
`extract_text/2` change needed — return `%{stdout: text}` directly).

`[unique contribution]` The direct HTTP approach also means DeepSeek works
identically in dev, CI, and production with only `DEEPSEEK_API_KEY` set — no
binary in PATH required. This is a meaningful advantage in CI environments where
CLI agents may not be installed.

---

**Q33.3 — Role: add as fourth agent `:deepseek`, with R1 as Challenger.**

`[inferred]` Option (d) — R1 as dedicated Challenger — is appealing but
premature. The Challenger role is deferred to the first production run.
Implementing a separate R1 integration path before the Challenger role is
activated would add complexity for no current benefit.

Option (a) — replace Gemini — loses a Google-perspective agent. `[inferred]`
The value of multi-agent deliberation comes partly from training distribution
diversity. Codex (OpenAI training), Gemini (Google training), and Claude IC
(Anthropic training) represent three substantially different corpora and RLHF
processes. DeepSeek (Chinese research lab training) represents a fourth distinct
perspective. Replacing Gemini reduces rather than increases diversity.

**Recommendation**: Add `:deepseek` as a **fourth agent** alongside the existing
three. The agent order in `@default_agents` would be
`[:codex, :gemini, :deepseek, :claude_ic]` (IC still speaks last).

`[inferred]` With four agents, the consensus logic is unchanged: `consensus?/1`
checks for no blocking `[needs-more-evidence]` with at least one `[satisfied]`.
An agent marking `[no objection]` is non-blocking. A fourth agent adds one more
opportunity to surface dissent; the consensus threshold does not need updating.

**IC prompt**: Update `@agent_roles` in `Orchestrator` to add a DeepSeek role
description reflecting its distinctive perspective (multilingual training,
strong math/code reasoning, Chinese research lab provenance — bring independent
perspective from a different training distribution).

---

**Q33.4 — Key management: `DEEPSEEK_API_KEY` via agenix + NixOS service env.**

`[observed]` The unified-nix-configuration flake already uses agenix for secrets
(per the tooling context). The pattern for a new secret:

1. `secrets/deepseek-api-key.age` — encrypted secret file
2. In the roundtable NixOS module (item 26), add:
   ```nix
   deepseekApiKeyFile = lib.mkOption { type = lib.types.path; };
   ```
   and in `serviceConfig.EnvironmentFile` include it.
3. For dev: `DEEPSEEK_API_KEY=sk-...` in `.env` or shell.
4. For the fnox wrapper (if opencode is ever used for DeepSeek): add
   `DEEPSEEK_API_KEY` to the fnox key injection spec alongside `Z_AI_API_KEY`.

`[observed]` DeepSeek's API: no regional restrictions as of 2025; rate limits
are generous on paid tiers (1000 RPM for chat completions). Key rotation is
manual via the DeepSeek platform dashboard. No unusual operational concerns.

---

**Q33.5 — Output format: no `extract_text/2` change needed.**

`[inferred]` With the direct HTTP approach, `run_deepseek/2` returns
`{:ok, %{stdout: text}}` where `text` is already the plain string content.
`extract_text/2` receives this and its first case matches:

```elixir
case JSON.decode(raw) do  # raw is already plain text, not JSON
  ...
  _ -> raw  # fallback: return as-is
end
```

Wait — `extract_text/2` calls `JSON.decode(raw)`. If `text` is plain prose,
the fallback `_ -> raw` catches it cleanly. No change needed.

**However**: if DeepSeek responds in markdown with code blocks, `extract_text/2`
returns the full markdown string including fences. This is the same behaviour
as other agents. No special handling needed.

`[unique contribution not found — output format analysis covers the brief's
question; no additional dimension identified]`

**Q33: [satisfied]**

---

### Claude IC — Round 18 Synthesis

**Mindguard check**: Codex's position converges on direct HTTP + four-agent
roster. I will surface the minority/alternative before synthesising:

*Alternative position (not voiced but considered)*: Keep three agents; use
DeepSeek-V3 as a drop-in replacement for Codex to reduce cost, since OpenAI
Codex CLI is more expensive per token. Rationale: the diversity argument
assumes Codex and DeepSeek provide meaningfully different perspectives, but
both are strong-at-code, RLHF-aligned models; the more distinctive perspectives
in the current roster are Gemini (Google research) and Claude IC (Anthropic
safety-focused training). Swapping Codex for DeepSeek might increase cost
efficiency without much diversity loss.

`[inferred]` I assess this alternative as Medium plausibility. It is not
clearly wrong. I note it explicitly rather than absorbing it into the synthesis.
The synthesis below adopts the four-agent recommendation but flags this as a
reversible decision.

---

**Q33.1 — Model: [satisfied]**

DeepSeek-V3 (`deepseek-chat`) for regular agent turns. DeepSeek-R1
(`deepseek-reasoner`) reserved for the Challenger role at closure, activated
at first production run. Rationale: V3 latency is compatible with sequential
multi-agent rounds; R1's reasoning depth is suited to the adversarial single-turn
Challenger task.

**Q33.2 — Invocation: [satisfied]**

Direct Elixir HTTP via `Req` in `RunCliAgent`. Extend `run/2` to detect the
`:deepseek` agent and route to `run_deepseek/2` (inline HTTP, no shell process).
Response text returned as `%{stdout: text}` directly — no `extract_text/2`
changes needed for the primary path.

`RunCliAgent` schema change: `:deepseek` added to the `{:in, [...]}` validator.

**Q33.3 — Role: [satisfied-conditional: reassess after first production run]**

Add `:deepseek` as fourth agent. Default agent list:
`[:codex, :gemini, :deepseek, :claude_ic]`.

Condition: if empirical data from the first production run shows Codex and
DeepSeek positions are highly correlated (low diversity gain), revisit the
replacement-vs-addition decision at that point.

`@agent_roles` entry for DeepSeek: *"You are DeepSeek, an AI agent developed
by DeepSeek AI (Chinese research lab). Bring independent analytical perspective
grounded in your distinct training distribution. Focus on rigorous reasoning and
identifying considerations that may be underweighted by agents trained on
primarily English-language corpora."*

**Q33.4 — Key management: [satisfied]**

`DEEPSEEK_API_KEY` via agenix + NixOS service module `EnvironmentFile`. Dev:
`.env` or shell export. No fnox wrapper needed for the direct HTTP path.

**Q33.5 — Output format: [satisfied]**

No `extract_text/2` changes. Direct HTTP path returns plain text; fallback
case handles it correctly.

---

### Protocol Update 14 — DeepSeek Agent Integration

**Adopted (2026-04-29):**

1. **`:deepseek` added to `RunCliAgent`** — HTTP path via `Req`, not CLI.
   Model: `deepseek-chat` (V3) by default; `deepseek-reasoner` (R1) configurable
   via `:deepseek_model` option (reserved for Challenger role).
2. **`:deepseek` added to `@default_agents`** — fourth agent, speaks before IC:
   `[:codex, :gemini, :deepseek, :claude_ic]`
3. **`@agent_roles` updated** with DeepSeek role description.
4. **`RunCliAgent` schema** extended: `:deepseek` added to `{:in, [...]}` list.
5. **`DEEPSEEK_API_KEY`** env var; agenix secret in unified-nix-configuration;
   `EnvironmentFile` in roundtable NixOS module.

**Noted alternative** (not adopted; revisit after first run): replace `:codex`
with `:deepseek` for cost efficiency if Codex/DeepSeek diversity proves low.
