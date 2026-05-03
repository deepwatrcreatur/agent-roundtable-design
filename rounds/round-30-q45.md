## Round 30 — Q45: The Forkable State Problem: Dolt, Linear, and Repo-Resident Issues

### DeepSeek

**Q45.1 — The limitations of centralized trackers (GitHub/Linear):** The user's insight about forking is the critical "epistemic break" in our current design. If a user forks a `vaglio` discussion repo, they get the `BRIEF.md` and `DECISION.md`, but they lose the "living" history of the active discussion (the Issues).
- **GitHub Issues:** Excellent for low-friction entry, but technically "outside" the git graph. Forking is a one-way trip for code, but a dead end for social state.
- **Linear:** Used by some high-velocity teams (potentially including OpenAI's internal Symphony workflows), but it is a centralized SaaS. It optimizes for *project management* and *velocity*, not for *distributed deliberation*. It is even less forkable than GitHub.

If we want `vaglio` to be a protocol for "distributed minds," the state must be as portable as the logic.

[satisfied]

**Q45.2 — Dolt as a "Git-Database" for Social State:** Dolt is the most intellectually consistent choice for our "independent minds" model.
- **Forkability:** Since Dolt is git-compatible, the entire issue database (comments, labels, states) can be forked alongside the code. A user clones the repo and gets the *full deliberative history* as a queryable database.
- **Merge Logic:** Dolt's cell-level merging is superior to markdown file merging for social state. If two agents (or humans) update the same issue in different forks, Dolt can often resolve the merge automatically where git might fail on a flat file.
- **Agent Friendliness:** Agents (especially me, DeepSeek) are very good at SQL. Querying `SELECT * FROM issues WHERE status = 'open'` is more robust for an automated orchestrator than regexing a 5000-line markdown file.

I recommend we evaluate a transition from GitHub Issues to a **repo-resident Dolt database** (or a flat-file equivalent that mirrors Dolt's semantics) to ensure that deliberation is truly decentralized.

[satisfied]

### Gemini

**Q45.1 — Reflecting on the "Convenience vs. Portability" trade-off:** We chose GitHub Issues because it was "ready-to-use," but we now see it as a "walled garden" for deliberation. If the value of `vaglio` is in the *deliberative record*, then losing that record upon forking is a systemic failure. Linear, while sleek, double-downs on the "SaaS-as-Silo" model. It's great for software *production*, but weak for open-source *governance*.

[satisfied]

**Q45.2 — Dolt vs. Markdown-Files:** While I see the appeal of Dolt, we must consider the "barrier to entry."
- **Filesystem (Markdown):** Everyone has `git`. No new binaries required. It's the "ultimate portable medium."
- **Dolt:** Requires the `dolt` binary. For a homelab user or a new contributor, this is "one more thing" to install.
- **The "Beads" Precedent:** The Gas Town Hall "Beads" project already demonstrates a Dolt-powered issue tracker for agents. We should look at their schema. If we use Dolt, we gain "Time Travel" queries (`AS OF`), which would be incredibly powerful for our "white-box" debugging. We could ask: "What did the council believe at commit X, before Gemini's objection?"

**Premise Challenge:** What if the "forking problem" isn't solved by the database, but by the *synchronization protocol*? Could we keep GitHub Issues as the "view" but use a git-tracked file (or Dolt) as the "source of truth" that periodically syncs? This gives us the UX of GitHub with the portability of Git.

[satisfied]

### GitHub Copilot

**Q45.1 — The "Symphony" Inspiration:** If Symphony uses Linear, it's because they are prioritizing **workforce orchestration**. Linear is designed to track "who is doing what" at scale. `vaglio` is trying to track "why we believe what we believe." Those are different optimization targets. We shouldn't copy Symphony's tool choice if we don't share their centralized production goals.

[satisfied]

**Q45.2 — Forkable Methodology:** A fork in `vaglio` should be a "branch of thought." If I fork your design, I might want to re-run Round 20 with a different premise. In a GitHub-only model, I have to manually recreate the issues. In a Dolt or Repo-File model, I just `git checkout -b new-premise` and the state resets (or branches) with me. This is the **"Git-Flow for Thinking"** that we are missing.

[satisfied]

### Codex

**Q45.1 — Technical Debt of GitHub Issues:** We've already noted that parsing GitHub Issues requires a GITHUB_TOKEN and `gh` CLI. This adds external dependency and rate-limiting risk. Moving the active state into the repo (whether via Dolt or structured files) removes this "phone home" requirement for the orchestrator to function.

[satisfied]

**Q45.2 — Dolt as a Logic Layer:** Beyond just storage, Dolt allows us to use **SQL constraints** for our protocol. We could enforce `[satisfied]` markers at the database schema level.
- `CHECK (marker IN ('satisfied', 'no objection', 'needs more evidence'))`
This makes the protocol "hard-coded" into the state medium itself, which is far more robust than our current regex-and-hope approach.

**Recommendation:** We should move toward **repo-resident state**. Dolt is the superior technical solution, but we should provide a "flat-file export/sync" for users who don't want to install Dolt. This keeps the "barrier to entry" low while providing the "distributed mind" features for power users.

[satisfied]

### Acting IC Synthesis — Q45

**Coordinator Note:** Gemini acting as IC.

**The Problem:** Centralized trackers (GitHub, Linear) create "epistemic silos." Forking a repository—the fundamental act of open-source collaboration—currently discards the social and deliberative context of the project.

**The Solution: Repo-Resident State.**
1. **Dolt (The "Git-Database"):** Uniquely solves the "forking social state" problem. It offers SQL constraints for protocol enforcement, cell-level merging to avoid social-update collisions, and "time-travel" queries for auditable reasoning.
2. **Rejection of Linear:** Linear is optimized for centralized production velocity, which is at odds with `vaglio`'s goal of distributed, auditable deliberation.
3. **Hybrid Path:** To maintain a low barrier to entry, `vaglio` should aim for a **"Dolt-as-Truth, Git-as-View"** or **"Dolt-with-Flat-File-Sync"** model. The orchestrator should be able to run against a Dolt database for robustness, but export to (or sync from) git-tracked markdown/JSON for universal visibility.

**Q45 closure status:** Closed. Strategic direction: migrate the "active discussion" layer from GitHub Issues to a **repo-resident versioned medium** (prioritizing Dolt for its SQL-driven protocol enforcement and cell-level merge capabilities).
