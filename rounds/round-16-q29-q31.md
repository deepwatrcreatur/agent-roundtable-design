## Round 16 — Q29/Q30/Q31: Co-evolution, Collaborator Permissions, Homelab Revisit

**IC Round 16 Prompt (2026-04-29)**

Three questions added from user context provided mid-implementation:

- **Q29**: Embedded vs. standalone discussion repos. When should the design
  discussion live inside the service repo (`docs/discussion/`) vs. as a
  separate repo? How does retrofit work?
- **Q30**: GitHub collaborator permissions — can contributors be scoped to
  discussion-only without code write access?
- **Q31**: Additional homelab context: `deepwatercreature.com`, unified
  `unified-nix-configuration` NixOS flake, Caddy on `router`. Does this change
  the Q26 hosting recommendation?

Agents: apply typed claim provenance (`[observed]`/`[testimony]`/`[inferred]`).
IC will apply disconfirmation pass to Q31 (homelab facts are testimony-heavy).
Brief premise challenge is required for each question.

---

### Codex — Q29: Embedded vs. Standalone

**Position: standalone-by-default, embedded-on-request.**

`[inferred]` The core tension is permission model vs. discoverability. In the
embedded model the discussion history is in git blame and it is impossible to
give a discussion contributor code-read access without code write access (in
a private repo). The standalone model cleanly separates concerns.

The decision rule I would apply:

- **Use standalone** when: (a) the service repo is private and you want to
  keep discussion participants separate from code contributors, (b) the
  discussion should be public even if the code is not, or (c) the discussion
  predates or outlasts the service repo.
- **Use embedded** when: (a) the repo is public, (b) all discussion participants
  are already code contributors, or (c) the project is solo with no access
  concerns.

`[inferred]` For a solo developer the embedded model is likely fine in practice —
there is no permission problem to solve. But the architecture should not
*require* the embedded model, because collaborators do create permission
complexity.

**Retrofit model**: `[inferred]` Round 0 in the embedded or standalone repo should
be a "current state" document, not a design proposal. A convention like
`round-00-retrofit-snapshot.md` with a header `# Retrofit: state at commit abc123`
makes the provenance clear. The BRIEF.md should note the retrofit context at
the top.

**Cross-link convention**: `[inferred]` A single file in the service root —
`DISCUSSION_REPO.md` with one line pointing to the discussion repo — is enough
for the standalone model. GitHub `homepage` URL field and a `roundtable-discussion`
topic are supplementary discoverability aids, not the primary link.

**Brief premise challenge**: *Is the embedded model actually simpler for a solo
developer?* `[inferred]` Yes, marginally — one less repo to manage. But "marginally
simpler" is not a reason to bake it into the architecture as the primary model.
The standalone model generalises better.

**Q29: [satisfied]**

---

### Codex — Q30: Collaborator Permission Scoping

**Position: standalone model + inject-via-service renders GitHub write access
largely moot for discussion contributors.**

`[observed]` GitHub's collaborator roles (`read`, `triage`, `write`, `maintain`,
`admin`) are documented in GitHub Docs. There is no "comment-only" role at the
repo level. For a public repo, any authenticated GitHub user can comment on
issues without being a collaborator.

`[inferred]` The key insight: if discussion contributions are made via the
roundtable service (LiveView prompt injection or Telegram bot), the service
authenticates with its own GitHub token and writes round files. The
collaborator never needs `write` access to the discussion repo. They only need
to be authenticated (via GitHub OAuth to the service) and the service checks
their GitHub identity against the repo's collaborator list or visibility.

For the fork-and-PR contribution model (collaborator clones discussion repo,
writes a round file, opens a PR): they need at minimum `write` access (or a
fork from a public repo). `[inferred]` Given the service-mediated path above is
sufficient and simpler, I recommend documenting the fork-and-PR path as an
advanced option, not the primary path.

**GitHub Organizations**: `[inferred]` Setting up a GitHub Organization for a
solo project with occasional collaborators is over-engineering. Straightforward
answer: use standalone discussion repos (either public or private with
explicit collaborators); use the service as the write principal for
discussion contributions.

**GitHub Discussions vs. committed files**: `[observed]` GitHub Discussions is a
native feature that allows threaded conversation without write access. However,
content in GitHub Discussions is not in git history and cannot be forked — which
violates Q23's forkability requirement. Ruled out as primary round medium.

**Brief premise challenge**: *If contributions go via the service, does GitHub
permission complexity answer itself?* `[inferred]` Yes, largely. The service
acts as the authenticated GitHub principal; collaborators authenticate to the
service, not directly to the repo. This collapses the permission surface
significantly.

**Q30: [satisfied]**

---

### Codex — Q31: Homelab Infrastructure Revisit

**Position: homelab is clearly the right first-deploy target; Q26 recommendation
confirmed and strengthened.**

`[testimony]` The owner reports: `deepwatercreature.com` domain, unified
`unified-nix-configuration` flake for all machines, `router` machine running
Caddy for TLS and reverse proxying.

`[inferred]` This is a materially different picture from "homelab = bare metal
with manual setup." NixOS + Caddy + unified flake is a well-integrated
deployment environment where adding a new service is:

1. Add a NixOS module to the service host in the flake
2. Add a Caddy virtual host block: `roundtable.deepwatercreature.com → :4000`
3. `nixos-rebuild switch --flake .#homeserver`

`[inferred]` Caddy handles WebSocket upgrades correctly for Phoenix LiveView via its
`reverse_proxy` directive — Caddy preserves `Connection: Upgrade` and
`Upgrade: websocket` headers by default. No special Phoenix-specific Caddy
config is needed beyond `reverse_proxy localhost:4000`.

`[inferred]` The NixOS deployment workflow (atomic, declarative, rollback via boot
entries, Attic binary cache) is already the owner's existing workflow. `fly
deploy` adds a Docker build step and a new deployment mental model. The
"easier deploy story" Fly.io offers is not easier for a NixOS user who already
deploys via `nixos-rebuild switch`.

**Remaining homelab risks vs. Fly.io**:
- `[inferred]` Home internet reliability: the service would be unreachable during
  ISP outages. Acceptable for a personal tool; the primary user (the owner) is
  on the same LAN.
- `[inferred]` Hardware failure: no managed hardware replacement. Mitigated by
  state durability (JSON round files in git, ETS is ephemeral-only for hot
  state).
- `[inferred]` No managed TLS renewal: Caddy automates Let's Encrypt renewal.
  Non-issue.

**Brief premise challenge**: *Does the service need to be publicly reachable?*
`[inferred]` For the owner alone: no. The service could be LAN-only. The public
domain is useful only if collaborators need external access. For a personal tool,
VPN (Tailscale or WireGuard — the homelab likely already has one or can add it)
may be preferable to public exposure.

**Q31: [satisfied-conditional: confirm Caddy WebSocket behaviour in production;
confirm public domain is desired over VPN-only access]**

---

### Claude IC — Round 16 Synthesis

**Disconfirmation pass (Q31)**: The Codex position relies on testimony about the
homelab configuration. Let me push on one claim: Caddy automatic WebSocket
proxying. `[observed]` Caddy's documentation (caddy.community, official docs)
confirms that `reverse_proxy` handles WebSocket upgrades transparently — the
`upgrade` header is forwarded by default. No special Phoenix configuration is
needed. Disconfirmation sought: is there a known failure mode with Phoenix
LiveView + Caddy? `[inferred]` Phoenix LiveView requires that `PHOENIX_HOST` (or
`host` in the endpoint config) matches the public-facing hostname to prevent
CSRF errors. This is a one-line config change, not a Caddy limitation.
Disconfirmation pass: complete. No blocking issue found.

---

**Q29 — Embedded vs. Standalone: [satisfied]**

The decision rule is clear:

- **Standalone by default.** Standalone separates permissions, is forkable
  independently, and generalises to collaborators. It is the primary model.
- **Embedded as opt-in.** Solo project, public repo, all contributors are
  also developers: add `docs/discussion/roundtable.toml` and proceed. The
  service must support both paths (the `source` parameter in
  `CLI.start_discussion/2` already handles both by slug vs. file path).

**Retrofit convention adopted:**
- `round-00-retrofit-snapshot.md` header: `# Retrofit: Current State at <commit>`
- BRIEF.md first section: `**Retrofit note:** This discussion was opened on
  <date> against existing service at commit <sha>.`
- Service root: `DISCUSSION_REPO.md` with one line: `Design discussion: owner/repo`

**`roundtable.toml` addition:**

```toml
[discussion]
# ... existing fields ...
embedded = false          # true if discussion lives in the service repo
service_repo = ""         # for standalone: "owner/service-repo"
service_commit_at_start = ""
retrofit = false          # true if opened against existing codebase
```

---

**Q30 — Collaborator Permissions: [satisfied]**

**Decision:** The service-mediated contribution path (LiveView injection or
Telegram bot) is the primary model for discussion contributors. Under this
model, collaborators authenticate to the roundtable service via GitHub OAuth;
the service holds the write token; collaborators never need GitHub write access
to the discussion repo directly.

For the standalone model with a public discussion repo, any authenticated GitHub
user can read; the service gates write operations. For a private standalone repo,
grant `read` collaborator access; fork-and-PR is available but not the primary
path.

**GitHub Discussions ruled out** as round medium: violates forkability (Q23).
**GitHub Organizations** not required for solo+occasional-collaborator use case.

---

**Q31 — Homelab Infrastructure: [satisfied-conditional]**

Conditions required for full closure:
1. Confirm Phoenix endpoint hostname config (`PHX_HOST=roundtable.deepwatercreature.com`)
   is set in the NixOS module
2. Decide: public (`roundtable.deepwatercreature.com`) vs. VPN-only for collaborator access

Subject to those two config decisions, the recommendation is:

**Homelab is the correct first-deploy target.** The unified NixOS flake + Caddy
+ `deepwatercreature.com` domain eliminates the main disadvantages of self-hosting.
Fly.io remains a documented fallback only if the service needs to be reachable
by collaborators who cannot or will not use a VPN.

**Minimal NixOS module sketch** (to be developed as a work item):

```nix
services.roundtable = {
  enable = true;
  port = 4000;
  secretKeyBaseFile = config.age.secrets.roundtable-secret-key-base.path;
  githubTokenFile  = config.age.secrets.roundtable-github-token.path;
  phoenixHost      = "roundtable.deepwatercreature.com";
};
```

Caddy virtual host:

```
roundtable.deepwatercreature.com {
  reverse_proxy localhost:4000
}
```

---

### Protocol Update 12 — Discussion Repo Co-evolution and Deployment Conventions

**Adopted (2026-04-29):**

**1. Standalone-first with embedded opt-in**
Default: standalone discussion repos. Embedded (`docs/discussion/`) is valid
for solo/public projects. The `embedded` field in `roundtable.toml` makes
the model explicit.

**2. Retrofit conventions**
- Round 0 file named `round-00-retrofit-snapshot.md` for retroactive discussions
- BRIEF.md opens with a retrofit notice block
- Service root contains `DISCUSSION_REPO.md` (standalone) or omits it (embedded)

**3. Service-mediated contributions as primary path**
Discussion contributors authenticate to the roundtable service via GitHub OAuth;
the service is the GitHub write principal. Direct git fork-and-PR is a
documented secondary path. This makes GitHub write access optional for
discussion contributors.

**4. Homelab deployment target confirmed**
NixOS module + Caddy `reverse_proxy` + `deepwatercreature.com` is the
first-deploy target. Fly.io documented as fallback only. `PHX_HOST` must be set
in the NixOS environment.

**5. `roundtable.toml` schema v1.1 additions**
```toml
embedded = false
service_repo = ""
service_commit_at_start = ""
retrofit = false
```

---
