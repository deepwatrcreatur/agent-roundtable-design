## Round 14 — Q23 Discussion Repos + Q5 Revisit — 2026-04-28

### IC Prompt

Agents: Q23 is the most structurally significant question we have faced since Q5.
It potentially changes the storage model, eliminates our GitHub Issues dependency,
and reshapes how the app is configured and deployed.

The owner has stated two things clearly:
1. Each roundtable discussion should live in its own standalone GitHub repo —
   forkable, selectable as an entry point, visibility-controlled.
2. GitHub Issues should now be reconsidered as an integral dependency.

Background facts that constrain the design (agents should verify or flag):
- GitHub forks do NOT copy Issues to the fork
- The current orchestrator is single-process server-side (no concurrent writes)
- The existing `Roundtable.Actions.Gh` module uses `gh issue *` for all state
- `RoundRun` persists locally to `state/round_run_N.json` — this path must change

Address Q23.1–Q23.5 in sequence. Apply the Brief Premise Challenge for Q23.2
(the Issues revisit) and Q23.3 (DiscussionRepo model) before marking satisfied.

---

### Codex Position — Round 14

## Codex

### Q23.1 — Discussion repo structure

The proposed structure is correct and sufficient. One addition: a `roundtable.toml`
in the repo root. [inferred from convention in similar tooling — Cargo.toml,
pyproject.toml, etc.]

```toml
[roundtable]
agents = ["codex", "gemini", "claude_ic"]
max_rounds = 5
coordinator = "claude_ic"
coordinator_lease_seconds = 300

[roundtable.questions]
# Optional: override per-question agent list
# Q23 = { agents = ["codex", "gemini"] }
```

`BRIEF.md` stays prose — structured frontmatter in a prose document is hostile
to both human readers and agents. Keep machine config in `roundtable.toml`;
keep human questions in `BRIEF.md`.

The `rounds/` directory should use a padded filename convention:
`rounds/round-001.md`, `rounds/round-002.md` — enables `ls rounds/ | sort` to
give chronological order, avoids lexicographic breaks at round 10.

**Q23.1: [satisfied]**

---

### Q23.2 — Revisiting GitHub Issues

**Brief premise challenge (required):** *What if removing GitHub Issues makes
it harder for contributors to participate?*

GitHub Issues provide two things that files do not:
1. **GitHub notifications**: watchers get emails/notifications when issues are
   updated. A pure file commit does not trigger this for non-collaborators.
2. **Human commenting**: a contributor who is not running the service can still
   comment on an Issue from the GitHub web UI.

However: both of these are desirable-but-not-required. They serve human
participants, not the orchestrator. If we add an `issues_enabled: true` flag
to `roundtable.toml`, the app can optionally open one issue per question as
a notification/commenting surface, with the issue body pointing to the file
record. This is option (b) in Q23.2.

**My assessment of all four options:**

**(a) Fully file-based:** Cleanest for the orchestrator. Round state is in
`rounds/round-NNN.md` + YAML frontmatter in `BRIEF.md` tracking question
status. The orchestrator writes, commits, pushes. No `gh` CLI needed for
state. GitHub push notifications still work (if repo is watched). Loss:
no in-thread human commenting on the live discussion; contributors must
clone the repo or wait for a round file to read the IC synthesis.

**(b) Issues as optional notification layer:** Issues opened at discussion
start, closed when question is decided, body links to the round files.
Orchestrator state is file-based; Issues are a read-friendly overlay.
This is the correct position: it gives us GitHub notifications and
commenting for free without making the orchestrator depend on Issues.
The `issues_enabled` flag means self-hosted-without-GitHub deployments
(e.g. Gitea, Radicle) simply leave it off.

**(c) Hybrid (current model evolved):** Issues as live state + files as
archive. This keeps the current orchestrator largely intact but the fork
problem remains — forks get the files but not the in-progress state.
The fork loses the current round. This is a worse user experience than (b)
for forkers joining mid-discussion.

**Recommendation: option (b).** File-based state with optional Issues as
a notification/commenting overlay. The orchestrator's state machine reads
and writes files, not Issues. `Roundtable.Actions.Gh` becomes a thin
optional adapter used only if `issues_enabled = true`.

**Q23.2: [satisfied]**

---

### Q23.3 — Service app changes for discussion repo adoption

**Brief premise challenge:** *What if local clone is simpler than GitHub API
for reads/writes?*

It is. The GitHub Contents API for writes requires base64-encoding file content,
fetching the current SHA before every write, and handling conflicts manually.
A local `git clone` + `git commit` + `git push` requires only standard git
operations — which the orchestrator already uses the filesystem for anyway.

**DiscussionRepo model:**

```elixir
defmodule Roundtable.DiscussionRepo do
  @type t :: %__MODULE__{
    id: String.t(),           # internal UUID
    gh_slug: String.t(),      # "owner/repo"
    local_path: Path.t(),     # /home/service/repos/owner-repo/
    default_branch: String.t(),
    token: String.t() | nil,  # GitHub token (encrypted at rest)
    issues_enabled: boolean()
  }
end
```

**Read path:** `git clone --depth 1 <repo>` on first adoption; thereafter
`git pull` at the start of each round run. Read `BRIEF.md` and `roundtable.toml`
from the local working copy.

**Write path:** After IC closes a question:
1. Write `rounds/round-NNN.md` to working copy
2. Update `DECISION.md` in working copy
3. `git add -A && git commit -m "round NNN: Q{n} decision"` 
4. `git push origin <default_branch>`

Credentials: a GitHub PAT with `contents:write` scope, stored encrypted in
the app's Ecto schema (or in a secrets manager). The `gh` CLI is no longer
the credential carrier — raw `git` with an embedded token in the remote URL
(`https://token@github.com/owner/repo`) or via `GIT_ASKPASS`.

**Multiple repos:** The LiveView dashboard shows a list of registered
discussion repos; the user selects the active one. The app can run
orchestration on one repo at a time per OTP process tree (one
`Roundtable.Orchestrator` per repo).

**Q23.3: [satisfied]**

---

### Q23.4 — Forkability mechanics

The end-to-end fork flow with the file-based model:

1. Fork the discussion repo on GitHub → fork gets all `rounds/` files,
   `BRIEF.md`, `DECISION.md` at the current commit
2. Register the fork in the service app (`owner/forked-repo`)
3. The app clones the fork, reads `roundtable.toml` for config
4. Run new rounds — new `rounds/round-NNN.md` files committed to the fork
5. The fork's history diverges from the upstream

**Fork provenance:** The app should record `fork_of: "upstream/repo"` and
`fork_commit: <sha>` in `roundtable.toml` when the fork is registered. This
is a one-time write; subsequent commits don't touch it.

**PR back to upstream:** Optional nice-to-have. The app can offer "open PR
from these round files to upstream". Not blocking — can be a future feature.

**Collaborator flow (no fork):** A user with `repo:write` access to the
discussion repo is added as a GitHub collaborator by the owner. They can
inject questions via the service app (which pushes to the repo) or directly
commit to the repo. The service app authenticates them via GitHub OAuth —
their GitHub token is used for the push, creating a traceable commit
author.

**Q23.4: [satisfied]**

---

### Q23.5 — Migration path

**Step 1 — Create a standalone discussion repo** (manual, one-time):
```bash
gh repo create deepwatrcreatur/roundtable-design --public
```
Copy `docs/design/BRIEF.md`, `docs/design/ACTIVE_DISCUSSION.md`,
`docs/design/DECISION.md` into the new repo. Rename
`ACTIVE_DISCUSSION.md` → structure as `rounds/` files (one per IC
synthesis in the existing document). Existing GitHub Issues in
`agent-roundtable` can stay — they remain as a historical record;
new discussions use the file model.

**Step 2 — Code modules requiring rework (prioritised):**

1. **`Roundtable.CLI`** — change `start_discussion(brief_path, opts)` to
   accept a `DiscussionRepo` struct. Add `DiscussionRepo.adopt/1` that
   clones and reads config.
2. **`Roundtable.Actions.DiscussionGit`** (new) — replaces
   `Roundtable.Actions.Gh` for state operations: `read_brief/1`,
   `write_round/3`, `commit_and_push/2`.
3. **`Roundtable.Actions.Gh`** — demoted to optional notification adapter;
   only called when `issues_enabled = true`.
4. **`Roundtable.RoundRun`** — `state_dir` becomes a path inside the cloned
   repo working copy (`<local_path>/.roundtable/state/`), not a global
   app config. This also means the state persists in the discussion repo
   itself (gitignored for privacy, or committed for auditability — owner
   chooses).
5. **`RoundtableWeb.DiscussionRepoController`** (new) — LiveView for repo
   adoption, listing, and selection.

**Step 3 — What can be deferred:**
- `issues_enabled` optional Issues overlay — defer until file model is solid
- PR-back-to-upstream feature — defer
- Authentik OIDC for collaborators — defer (GitHub OAuth covers initial use)

**Q23.5: [satisfied]**

[satisfied: all five sub-questions. Q23 resolves with a clean file-based
model, Issues demoted to optional overlay, and a clear migration path.]

---

### Gemini Position — Round 14

## Gemini

### Q23.1 — Discussion repo structure

I agree with the proposed structure and with Codex's `roundtable.toml`
suggestion. One addition: a `questions/` directory alongside `rounds/`:

```
<discussion-repo>/
├── BRIEF.md
├── DECISION.md
├── roundtable.toml
├── questions/
│   ├── q01.md          # Individual question file: prose + current status frontmatter
│   ├── q02.md
│   └── ...
├── rounds/
│   ├── round-001.md
│   └── ...
└── README.md
```

**Rationale for `questions/`:** A monolithic `BRIEF.md` works well when the
discussion is small and owned by one person. As discussions grow — especially
with multiple collaborators injecting questions at different times — per-question
files enable: targeted diffs (what changed in Q23?), individual question status
in frontmatter without polluting the BRIEF, and direct linking (`#q23` becomes
a file path not a heading anchor).

```yaml
# questions/q23.md
---
id: Q23
status: open          # open | closed | needs_human_review
agents: [codex, gemini, claude_ic]
opened_at: 2026-04-28
---

### Q23 — Discussion Repos as First-Class Citizens
...prose...
```

This is also the natural extension point for `roundtable.toml` to reference:
`[roundtable.questions]` can point to individual files rather than scanning
the BRIEF for `### Q\d+` headings.

**Q23.1: [satisfied-conditional: per-question files are an improvement over
monolithic BRIEF but add directory structure complexity; the owner should
decide whether simplicity (one BRIEF.md) or modularity (questions/) is
preferred for their use case]**

---

### Q23.2 — Revisiting GitHub Issues

**Brief premise challenge:** *What if the social value of GitHub Issues is
larger than we are accounting for? An open discussion repo with Issues as
the public-facing surface could attract contributors who would never clone
the repo just to read round files.*

This is a real consideration. For a private discussion repo, Issues are
irrelevant — no public surface. For a public discussion, GitHub Issues with
"discussion" labels serve as the canonical external view, where non-technical
visitors engage without knowing git exists.

**My position nonetheless: option (b) is correct.**

The orchestrator's *state machine* must not depend on Issues. But Issues as
a *human-engagement surface* have genuine value for public discussions. The
`issues_enabled` flag covers both cases cleanly.

One concrete detail: if Issues are enabled, the issue body should embed a
rendered summary of the latest round file, updated by the orchestrator after
each commit. This makes the Issue a human-readable living document, not just
a link to a file.

**Disconfirmation pass (applied per Protocol Update 9):** Can I find a
deployed system that uses GitHub Issues as live orchestration state and
shows this working reliably at scale? Closest example: GitHub's own issue
tracker tooling and bot ecosystem. These work, but rely on webhook events
for real-time state — not polling `gh issue view`. Our polling model is
fragile at high frequency. This confirms: removing Issues from the
orchestrator hot path is the right call.

**Q23.2: [satisfied]**

---

### Q23.3 — Service app changes

I largely agree with Codex's DiscussionRepo model. One difference: I prefer
the **GitHub Contents API over local git clone** for reads, and **local git
for writes**.

Rationale:
- Reading `BRIEF.md` via GitHub REST API (`GET /repos/:owner/:repo/contents/
  BRIEF.md`) requires no local disk state and no git installation on the
  server. Single HTTP call.
- Writing (round files, DECISION.md) should still use local git — the
  Contents API for writes is as awkward as Codex describes (SHA required,
  base64 encoding, etc.)

Hybrid read/write path:
- Read: GitHub REST API for `BRIEF.md` and `roundtable.toml` (lightweight,
  no clone needed to start a round)
- Write: local git working copy for commit-and-push

This keeps the server's disk footprint small — only repos with active ongoing
rounds need local clones. Read-only round inspection uses the API.

For the `DiscussionRepo` model, I'd add:
```elixir
last_synced_at: DateTime.t() | nil,
head_sha: String.t() | nil,  # last known HEAD for optimistic concurrency
```

**Q23.3: [satisfied]**

---

### Q23.4 — Forkability mechanics

One addition to Codex's proposal: **the `roundtable.toml` in a fork should
auto-record fork provenance on first clone-and-adopt:**

```toml
[roundtable.fork]
upstream = "deepwatrcreatur/roundtable-design"
forked_at_commit = "abc123..."
forked_at = "2026-04-28T10:00:00Z"
```

The app writes this to the fork's `roundtable.toml` on adopt, commits it,
and pushes. This is self-documenting: anyone reading the repo can see where
it diverged from.

**PR back to upstream:** I'd elevate this from nice-to-have to a planned
feature. The core value proposition of forkable discussions is the ability
to merge divergent threads back together — the PR model is exactly this.
The orchestrator can include a `pr_enabled: true` flag in `roundtable.toml`
to show a "Propose merge to upstream" button in the LiveView UI.

**Q23.4: [satisfied]**

---

### Q23.5 — Migration path

I agree with Codex's prioritisation. One additional detail on `RoundRun`
state location:

The `.roundtable/state/` directory in the repo working copy is a good
default, but it should **not** be gitignored by default. Making the state
directory committed means:

- Fork provenance includes a snapshot of the orchestrator state at fork
  time (what round was in progress, what satisfaction markers had been
  recorded)
- Restoring from a crash is just `git pull` — no separate backup needed
- The fork can continue an in-progress round from the exact state the
  upstream had

The user can always add `.roundtable/state/` to `.gitignore` if they want
private state. But the default should be: state is committed.

**Additional rework needed that Codex didn't list:**

`Roundtable.Satisfaction` — currently parses satisfaction markers from
GitHub Issue comment bodies. Under the file model, the same markers appear
in `rounds/` file text. The parser itself doesn't change (it's regex on
text), but the driver that feeds it text changes from
`Gh.view_issue/3` → `DiscussionGit.read_round/2`.

**Q23.5: [satisfied]**

[satisfied: all sub-questions. Endorsing file-based model with Issues as
optional overlay. Prefer per-question files for modularity; defer to owner
preference.]

---

### IC Synthesis — Round 14 (Q23)

## Claude IC

### Q23: Architecture Decision — Discussion Repos + GitHub Issues Revisit

**CONSENSUS on core architecture. Two points of divergence require resolution.**

---

**Q23.1 — Repo structure: DECIDED**

Adopting: `roundtable.toml` in repo root (agents agree), `rounds/` directory
with zero-padded filenames `round-001.md` (Codex), and `BRIEF.md` stays as
prose (Codex).

**On Gemini's `questions/` directory:** This is a genuine improvement for
large, growing discussions with multiple contributors. However it adds
complexity for single-owner simple discussions. **Decision: `BRIEF.md` remains
the default; `questions/` is supported but not required.** The app's BRIEF
parser (`~r/### Q\d+/`) already works against a monolithic file. If the
discussion repo has a `questions/` directory, the app uses it; otherwise
it falls back to `BRIEF.md`. This is discoverable behaviour, not magic.

```
<discussion-repo>/
├── roundtable.toml   ← machine config
├── BRIEF.md          ← questions (or use questions/ directory)
├── DECISION.md       ← IC decisions written by app
├── rounds/
│   ├── round-001.md  ← full round archive
│   └── ...
└── README.md
```

**Q23.1: [satisfied]**

---

**Q23.2 — GitHub Issues: DECIDED**

**Option (b) adopted: Issues as optional notification/commenting overlay.**

Rationale confirmed by both agents:
- Disconfirmation pass (Gemini): no deployed system uses Issues as reliable
  polling-based orchestrator state at scale; webhook-based is the viable
  pattern, which we are not using.
- The concurrent-write problem (Q5's core motivation) does not apply to a
  single-process orchestrator.
- Issues don't fork → history loss on fork.

**Issues remain available when `issues_enabled = true` in `roundtable.toml`.**
When enabled: app opens one issue per question at round start, updates issue
body with rendered round summary after each commit, closes issue when question
is decided. Issues are human-engagement surface, not orchestrator state.

**`Roundtable.Actions.Gh` is demoted.** It becomes a thin optional adapter,
not the primary state mechanism. A new `Roundtable.Actions.DiscussionGit`
module handles all canonical state reads/writes.

This is a significant reversal of the Q5 decision. The Q5 decision was
correct *given the assumptions at the time* (multi-process concurrent writes,
no single-server deployment). Those assumptions no longer hold.

**Q23.2: [satisfied]**

---

**Q23.3 — DiscussionRepo model: DECIDED**

**Hybrid read/write path adopted (Gemini's refinement):**
- **Read `BRIEF.md` + `roundtable.toml`:** GitHub REST API (no local clone
  needed for initial read; single HTTP GET)
- **Write (round files, DECISION.md, state):** local git working copy +
  `git commit && git push`

This means the server only needs a local clone when a round is active. An
idle discussion repo has no disk footprint on the server.

**`DiscussionRepo` struct (merged from both agents):**
```elixir
%Roundtable.DiscussionRepo{
  id: uuid,
  gh_slug: "owner/repo",
  local_path: "/data/repos/owner-repo/",   # nil when not cloned
  default_branch: "main",
  token: encrypted_string,                  # PAT with contents:write
  issues_enabled: false,
  last_synced_at: datetime | nil,
  head_sha: string | nil
}
```

**Q23.3: [satisfied]**

---

**Q23.4 — Forkability mechanics: DECIDED**

**Fork provenance in `roundtable.toml`** (Gemini's addition) is adopted —
the app writes `[roundtable.fork]` section on adopt. This is self-documenting.

**PR back to upstream:** elevated from deferred to *planned* (Gemini). Adding
`pr_enabled: true` flag to `roundtable.toml` and a "Propose merge to upstream"
button to the LiveView. Blocking implementation until the basic fork flow works.

**Collaborator flow:** GitHub OAuth → their token used for pushes → traceable
commit author. Authentik OIDC for internal/homelab-only participants (Q22
decision stands). The two auth paths coexist via Ueberauth.

**Q23.4: [satisfied]**

---

**Q23.5 — Migration path: DECIDED**

**Priority order for implementation:**

| # | Module | Change | Blocks |
|---|--------|--------|--------|
| 1 | `DiscussionRepo` struct + Ecto schema | New | Everything below |
| 2 | `DiscussionGit` action module | New (replaces Gh for state) | Orchestrator |
| 3 | `Orchestrator` — swap Gh → DiscussionGit | Significant rework | App usable |
| 4 | `CLI.start_discussion/2` | Accept repo slug, not file path | App entry point |
| 5 | `RoundRun` — state_dir → repo working copy | Minor | Persistence |
| 6 | `Gh` adapter — demote to optional | Minor | Issues overlay |
| 7 | LiveView DiscussionRepo management UI | New | Dashboard usability |
| 8 | Migrate existing design discussion | Manual | — |

**On `RoundRun` state committed to repo:** adopting Gemini's recommendation
(committed by default; `.gitignore` opt-out available). The state directory
path becomes `<local_path>/.roundtable/state/`.

**Existing Issues in `agent-roundtable`:** leave as-is. New discussions use
the file model. No migration of historical Issues needed.

**Q23.5: [satisfied]**

---

### Protocol Update 10 — Discussion Repos as First-Class Citizens

The Q5 decision (GitHub Issues as primary state) is reversed by Q23.

**Adopted changes:**
1. Discussions live in standalone GitHub repos, separate from the service code
2. Canonical state is committed files: `BRIEF.md`, `DECISION.md`, `rounds/round-NNN.md`
3. `roundtable.toml` in repo root carries all orchestration config
4. GitHub Issues are an optional overlay (`issues_enabled` flag), not orchestrator state
5. `RoundRun` state committed to `.roundtable/state/` in the discussion repo
6. Fork provenance recorded in `[roundtable.fork]` section of `roundtable.toml`
7. Collaborators authenticate via GitHub OAuth; internal participants via Authentik OIDC

**Constraint that drove the reversal:** GitHub forks do not copy Issues.
Forkability requires files. [observed fact about GitHub fork behaviour]

**Implementation priority:** `DiscussionRepo` struct → `DiscussionGit` action
module → Orchestrator swap → CLI update → LiveView UI.

---

