## Round 15 — Q24–Q28: Messaging Gateway, Auth, Hosting, Repo Discovery, GitHub Alternatives — 2026-04-29

### IC Prompt — Claude — 2026-04-29

Five new questions have been added by the owner. They cluster naturally into two
themes:

**Theme A — Access and interface** (Q24, Q25, Q27): How do people get prompts
into the system, prove who they are, and find the discussions they can interact
with?

**Theme B — Infrastructure** (Q26, Q28): Where does the service run, and does
the dependence on GitHub need a hedge?

These questions land after Protocol Update 10 (Q23), which established the
discussion-repo model. The answers here must be coherent with that model.

**Framing for agents:**

Q24 (messaging gateway) asks whether Telegram or email is a useful inbound
channel alongside the LiveView UI. Hermes and Openclaw use these channels
primarily for authentication bypass — a phone number or Telegram ID is
sufficient identity. Assess whether that model is appropriate here, and whether
it complements or competes with the GitHub-auth approach.

Q25 (authentication) is the most consequential question this round. The owner
has a strong prior toward GitHub OAuth because discussion repos live on GitHub.
The presence of Authentik in the homelab creates an opportunity to proxy
GitHub OAuth through Authentik for single sign-on across homelab services.
Evaluate the direct GitHub OAuth vs. Authentik-as-OIDC-proxy options carefully.
A wrong choice here is expensive to change later.

Q26 (hosting) should be answered concisely. Fly.io has the strongest Elixir
community endorsement. Gigalixir is a valid second option. Vercel is not
suitable for OTP. The honest answer may be "homelab first, Fly.io when you need
to share." Say that if it is true rather than over-engineering a multi-cloud
strategy.

Q27 (repo discovery) is a product-design question as much as a technical one.
Auto-discovery via GitHub topics or `roundtable.toml` scan is elegant, but a
single-owner tool with five repos does not need it on day one. Be honest about
when the complexity pays off.

Q28 (SourceForge and alternatives) should result in a clear ruling-out of
SourceForge and an honest assessment of whether GitLab/Forgejo self-hosting is
worth considering as a GitHub hedge. The owner is already deeply integrated
with GitHub. The question deserves a clean answer, not a hedge.

**Satisfaction markers expected:** `[satisfied]`, `[satisfied-conditional: <condition>]`,
or `[needs more evidence: <what>]` — one per question addressed.

---

## Codex

### Q24 — Messaging Gateway

**Position:** Telegram is the right first messaging channel if one is built.
The case for building it at all is weaker than it appears.

Hermes and Openclaw include messaging channels primarily because their users
lack a native UI, or because the channel *is* the UX (bots embedded in existing
team workflows). Neither reason applies here: the roundtable service is getting
a LiveView dashboard that is the purpose-built interface, and the owner is the
sole primary user. The authentication value Telegram provides — "only my Telegram
user ID can send prompts" — is already handled by the GitHub OAuth flow in Q25.

If the owner wants to inject a prompt from the iPhone without opening a browser,
the most direct path is a mobile-responsive LiveView page, not a Telegram bot.
LiveView over WebSocket works well on mobile; the component that is painful on
mobile is *text input of long prompts*, which the Q21 voice-entry architecture
(whisper.cpp → text → form submit) already addresses.

**Where Telegram does add value:** asynchronous notifications. When a round
completes, the orchestrator could send a Telegram message: "Round 13 closed:
Gemini needs more evidence on Q23.2." This is purely outbound — no inbound
parsing complexity, no auth bypass needed — and it is genuinely useful for a
single user who does not want to poll the dashboard. This use case is much
cheaper to implement than a full bidirectional bot.

**Recommendation:** Build outbound Telegram notifications only. Defer inbound
prompt injection via Telegram until the native mobile UI proves insufficient.
Do not build iMessage or WhatsApp integrations — Apple server API does not
exist, and WhatsApp Business API is not justified for a personal tool.

[satisfied]

---

### Q25 — Authentication Strategy

**Position:** Use Authentik as the OIDC broker with GitHub as the upstream
social provider. Do not implement GitHub OAuth directly in the Elixir app.

The owner's homelab already runs Authentik. The roundtable service should be
an OIDC relying party to Authentik, not to GitHub directly. This gives:

1. **Single sign-on**: one Authentik session covers roundtable plus any other
   homelab service (Gitea, Vaultwarden, whatever). No separate login for each
   app.
2. **Future auth methods without app changes**: adding a second provider
   (email OTP for a collaborator without GitHub?) is an Authentik config change,
   not a code change.
3. **GitHub independence for local access**: if GitHub is down, the owner can
   still log into Authentik with a local account and access homelab services.
   Pure GitHub OAuth would block all local access.

**The Elixir implementation:** Use `assent` with the OpenID Connect provider
pointing at Authentik. Authentik issues OIDC tokens that include GitHub identity
claims (username, email) when the user logged into Authentik via GitHub. The
roundtable app reads the GitHub username from the OIDC claims and calls
`GET /repos/:owner/:repo/collaborators/:username` to check repo-level write
access — this is the only GitHub API call needed for authorization, and it
requires only a service-account token (one GitHub PAT in the server config),
not an OAuth token per user.

**Collaborator flow:** A collaborator authenticates via GitHub OAuth → Authentik
→ roundtable app. The app extracts their GitHub username from the OIDC token,
checks repo membership via the GitHub API with the service PAT, and shows them
the repos they have write access to. No per-user OAuth token storage required.

**Required GitHub PAT scopes:** `repo` (to call the collaborators endpoint on
private repos). For public discussion repos, `public_repo` suffices.

**Telegram identity binding (Q24.3):** Store the Telegram user ID as a custom
attribute in the Authentik user profile. The Telegram bot verifies the update's
`from.id` against this attribute before accepting prompts. This is an Authentik
admin operation the owner performs once per trusted user.

[satisfied]

---

### Q26 — Service Hosting

**Position:** Homelab first. Fly.io as the documented upgrade path.

For a single-owner tool with <50 requests/day and existing Authentik OIDC in
the homelab:

- **Homelab** is the right answer for now. No cold starts, no egress charges,
  Authentik OIDC works natively, Mega S4 backups plug in directly, NixOS
  module or Podman container both work well for an Elixir release.
- **Fly.io** is the right answer if the service is ever exposed to external
  collaborators who shouldn't need homelab VPN access. The Elixir/Phoenix
  support is first-class (`fly launch` generates a `fly.toml` with correct
  Phoenix settings). The free tier includes one shared-CPU VM and 3GB persistent
  volume — sufficient for this workload. LiveView WebSockets work correctly on
  Fly.
- **Gigalixir** is a valid alternative to Fly.io but has a smaller community
  and slower release cadence. Choose Fly.io over Gigalixir unless there is a
  specific reason to prefer it.
- **Vercel** is unsuitable. It is serverless-first and has no concept of a
  long-running OTP process. A LiveView application *cannot* run on Vercel.
  Render and Railway work but offer no Elixir-specific advantages over Fly.

**Auth note:** An externally-hosted Fly.io deployment cannot use the homelab
Authentik directly. Options: (a) expose Authentik externally on a subdomain
with proper TLS (already viable if the homelab has a public IP/domain);
(b) use direct GitHub OAuth in the Fly deployment, Authentik in the homelab
deployment — two config profiles. The service config should make the OIDC
issuer URL a runtime env var so the same binary runs in both contexts.

[satisfied]

---

### Q27 — Discussion Repo Discovery

**Position:** `roundtable.toml` existence is the identity signal. GitHub topic
is an optional convenience layer. Auto-discovery is not needed on day one for
a single-owner tool.

The minimal viable dashboard flow:
1. User pastes `owner/repo` slug into the "Add discussion repo" field
2. App checks for `roundtable.toml` via GitHub contents API
3. If found, parses config and registers the repo
4. Repo appears in the dashboard

Auto-discovery (scan all accessible repos for `roundtable.toml`) adds O(N) API
calls and complexity for a use case the owner does not currently have (many
repos, need to find which are discussions). Add it when there is a concrete need.

**GitHub topic as discoverability signal for others:** A repo owner adding the
`roundtable-discussion` topic makes their discussion discoverable to other users
of the service (public discussions only). This is a social feature — useful when
the service has multiple users who want to browse public discussions — not a
core infrastructure requirement. Note it in the `roundtable.toml` schema as a
documentation convention, not a technical dependency.

**`roundtable.toml` minimal schema:**

```toml
[discussion]
title = "Agent Roundtable Orchestrator Design"
agents = ["codex", "gemini", "claude_ic"]
max_rounds = 5
coordinator = "claude_ic"
issues_enabled = false

[fork]
upstream = ""
fork_of_commit = ""
```

Agent identifiers in `agents` resolve against the service's agent registry
(a config map in the Elixir app that maps `"codex"` → CLI command, model, flags).
Keep `agents` as identifier strings; agent-specific config stays in the service,
not in the discussion repo.

[satisfied]

---

### Q28 — SourceForge and GitHub Alternatives

**Position:** SourceForge is ruled out. GitLab self-hosting is worth noting as
a future hedge. Forgejo/Gitea are the lightest-weight GitHub-compatible
alternatives if the owner ever wants to move off GitHub. No action needed now.

**SourceForge:** The platform has declined significantly from its early-2000s
peak. It suffered a major reputation blow in 2015 when it bundled adware into
downloads of abandoned projects. Current activity is heavily weighted toward
legacy projects. New open-source projects do not choose SourceForge. It does not
offer a modern OAuth-based identity provider comparable to GitHub OAuth. It does
not have an API surface comparable to GitHub's REST or GraphQL APIs. There is no
compelling reason to consider it.

**Forgejo/Gitea:** These are the most credible lightweight GitHub alternatives.
Forgejo (the Codeberg fork of Gitea) has a GitHub-compatible REST API — the same
endpoint paths, same response shapes for repos, contents, and collaborators. In
theory, the roundtable service could work against Forgejo with minimal changes if
the GitHub client module is written behind a behaviour. This is worth keeping in
mind when designing `DiscussionGit` (from Q23's work items): make the GitHub REST
calls go through a module with a clear interface (`@behaviour DiscussionRepo.Backend`)
so a Forgejo implementation can be added later without rewriting the orchestrator.

**GitHub lock-in risk:** The owner is already committed to GitHub across their
homelab NixOS config, other repos, and GitHub auth. The practical risk of a
GitHub outage is low for a personal tool (they poll, not webhook; polling fails
gracefully; state is in files). Account suspension is a non-zero risk for anyone
relying on GitHub, but mitigation is architectural (ensure discussion repos can
be cloned elsewhere) rather than requiring a platform migration.

**Ruling out:** SourceForge — no modern API, declining platform. Radicle —
p2p model does not map to the owner's GitHub-centric auth model. GitLawb /
GitSocial — could not be verified as active platforms during Q22; no evidence
of relevant features. GitLab.com — viable technical alternative but the owner
has no reason to move from GitHub. Self-hosted GitLab — large memory footprint
(4–8GB RAM minimum) makes homelab deployment unattractive versus Forgejo.

[satisfied]

---

## Claude IC

### Round 15 Synthesis — Q24–Q28 — 2026-04-29

Codex has addressed all five questions with consistent reasoning. I find the
positions defensible and will synthesize them into decisions. There are a few
points where I want to add precision or surface the premise challenges that the
brief requires.

---

**Q24 — Messaging Gateway**

Codex's recommendation — outbound Telegram notifications only, no inbound bot —
is correct for the current stage. The framing is right: the question to ask is
not "does Telegram work?" (it does) but "does the native UI already solve this?"
The native LiveView interface plus Q21's voice-entry tier handles prompt
injection. Inbound Telegram adds a second code path that the owner must maintain
without clear benefit over the dashboard.

**Premise challenge response:** Is outbound Telegram notification still justified
given that the LiveView dashboard has real-time updates via WebSocket? Yes,
because the owner is not always watching the dashboard. A round completion
notification pushed to the phone requires no polling and no context switch to
the browser. The signal-to-noise ratio is high (one message per round, not
continuous). This is the one messaging-channel feature that genuinely adds
something the dashboard cannot replicate passively.

Decision: build outbound Telegram notification only. Inbound prompt injection
via Telegram deferred indefinitely. iMessage, WhatsApp, Signal ruled out at
this stage.

**[satisfied]**

---

**Q25 — Authentication Strategy**

The Authentik-as-OIDC-proxy approach (option (c) in the brief) is the right
architecture. Codex's reasoning about GitHub independence for local access is
the decisive factor: if the service uses direct GitHub OAuth, the owner loses
access to their own homelab tool when GitHub is unreachable. That is not
acceptable for a self-hosted service.

The implementation is clean:
- `assent` with OpenID Connect provider → Authentik
- Authentik federates GitHub as upstream social provider
- App reads GitHub username from OIDC claims
- Repo permission check via service PAT (`GET /repos/:owner/:repo/collaborators/:username`)
- No per-user OAuth token stored in the roundtable app

**Premise challenge response:** GitHub OAuth coupling is mitigated by the
Authentik proxy layer. The owner can create a local Authentik account as a
fallback credential for homelab access when GitHub is down. Collaborators who
only have GitHub identities will be blocked when GitHub is down — this is
acceptable because they cannot push to the discussion repo during an outage
either.

Decision: Authentik OIDC proxy with GitHub as upstream provider. Direct GitHub
OAuth used only in production Fly.io deployments without local Authentik
(configured via `OIDC_ISSUER_URL` env var).

**[satisfied]**

---

**Q26 — Hosting**

"Homelab first, Fly.io as documented upgrade path" is the correct and honest
answer. There is no reason to over-engineer a multi-region deployment for a
personal tool in its first year.

The runtime env var for `OIDC_ISSUER_URL` is a concrete action item: the same
Elixir release binary should work in homelab (pointing at Authentik) and on
Fly.io (pointing at GitHub OAuth directly or a cloud-accessible Authentik
instance). This is a config concern, not a code concern.

**Premise challenge response:** The question "is evaluation premature?" is
answered by noting that Fly.io is already the correct answer for external
deployment — we do not need further analysis. The work item is: write a
`fly.toml` and document the two-config-profile pattern. That is small scope,
worth doing before deployment rather than after.

Gigalixir is a valid fallback but not recommended over Fly.io for new projects
in 2026. Vercel is ruled out.

Decision: homelab (NixOS module or Podman container) for personal use.
Fly.io as the standard external deployment path. Document `OIDC_ISSUER_URL`
as the key env var differentiating the two profiles.

**[satisfied]**

---

**Q27 — Repo Discovery**

The "paste `owner/repo` slug" flow is the right day-one UX. Auto-discovery
is a future feature. Codex's schema for `roundtable.toml` is accepted with
one addition: a `version` field to support schema evolution.

```toml
schema_version = 1

[discussion]
title = "Agent Roundtable Orchestrator Design"
agents = ["codex", "gemini", "claude_ic"]
max_rounds = 5
coordinator = "claude_ic"
issues_enabled = false

[fork]
upstream = ""
fork_of_commit = ""
```

**Premise challenge response:** Auto-discovery is a social/discoverability
feature that has value only when the service has multiple independent users
browsing public discussions. For a single owner managing a handful of repos,
the paste-a-slug UX is faster and less surprising than scanning. The right
time to add discovery is when the first external user says "how do I find
public discussions?" — not before.

Decision: `roundtable.toml` existence (checked via GitHub contents API) is
the canonical identity signal. `schema_version = 1` added to schema.
GitHub topic `roundtable-discussion` documented as an optional discoverability
convention. Auto-discovery deferred.

**[satisfied]**

---

**Q28 — SourceForge and GitHub Alternatives**

SourceForge is ruled out. The reasoning does not require elaboration beyond
what Codex stated: declining platform, no modern OAuth API, adware incident
legacy, no new-project adoption.

The `DiscussionRepo.Backend` behaviour observation is the most useful technical
insight from this question. When implementing `DiscussionGit` (Q23 work item 2),
the module should be written as a concrete implementation of a `@behaviour`,
not as a hardcoded GitHub client. The behaviour interface is small:

```elixir
@callback read_file(repo :: t(), path :: String.t()) :: {:ok, binary()} | {:error, term()}
@callback write_file(repo :: t(), path :: String.t(), content :: binary(), message :: String.t()) :: :ok | {:error, term()}
@callback list_files(repo :: t(), path :: String.t()) :: {:ok, [String.t()]} | {:error, term()}
```

A `Forgejo` implementation of this behaviour would be a few hundred lines and
would make migration off GitHub possible without touching the orchestrator.
This is a low-cost hedge worth taking at implementation time.

**Premise challenge response:** Evaluating alternatives is not a distraction —
it produced the useful action item above. The answer is clear: stay on GitHub,
but write the `DiscussionGit` adapter behind a behaviour so the option exists.

Decision: SourceForge and all evaluated GitHub alternatives ruled out for
current use. GitLab.com and Forgejo noted as viable future options. `DiscussionGit`
to be implemented behind `DiscussionRepo.Backend` behaviour (added to Q23 work
items as item 2a).

**[satisfied]**

---

### Protocol Update 11 — Interface, Auth, and Infrastructure Decisions

Adopted from Round 15:

1. **Messaging:** Outbound Telegram notification only. No inbound bot. All other
   messaging channels (iMessage, WhatsApp, Signal, email) deferred indefinitely.
2. **Auth:** Authentik OIDC proxy with GitHub as upstream social provider. Repo
   permission checks via service PAT. `OIDC_ISSUER_URL` env var differentiates
   homelab and external deployment profiles.
3. **Hosting:** Homelab (NixOS/Podman) for personal use. Fly.io as standard
   external deployment path. Gigalixir valid fallback. Vercel ruled out.
4. **Repo discovery:** `roundtable.toml` existence is the identity signal.
   `schema_version = 1` added to schema. Auto-discovery deferred.
5. **Platform:** SourceForge ruled out. GitHub remains the sole supported platform.
   `DiscussionGit` to be implemented behind `DiscussionRepo.Backend` behaviour
   to preserve future optionality for Forgejo/Gitea.

**Updated Q23 work items:**
- Item 2 renamed: `Roundtable.Actions.DiscussionGit` — implement as concrete
  module satisfying `DiscussionRepo.Backend` behaviour (new item 2a: define behaviour)
- Item 6 (Gh adapter demoted): also add `Roundtable.Adapters.GitHub` as the
  first `DiscussionRepo.Backend` implementation
