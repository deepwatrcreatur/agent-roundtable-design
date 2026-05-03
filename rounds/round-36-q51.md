## Round 36 — Q51: Unifying Dolt and JJ: Fork, Backend, or Co-location?

**Topic:** Determining the optimal integration path for Dolt (SQL versioning) and jj (Logical evolution). Is a Dolt fork necessary to achieve the vision of the "JJ-Native Forge"?

### DeepSeek

**Q51.1 — The Cost of a Fork:** Forking Dolt (written in Go) to implement `jj` semantics (written in Rust) is an engineering nightmare. It would consume a massive amount of "Development Tokens" and introduce long-term maintenance debt. We should avoid a fork unless the "Logical Delta" we need is physically impossible in vanilla Dolt.
- **The "Logic Proxy" Path:** Instead of modifying Dolt's core (Prolly Trees), we can build a **Jido-powered shim**. This shim translates `jj`-style "Change IDs" into Dolt "Branch/Tag" sets. The logic lives in Elixir (the orchestrator), not in a forked database engine. `[inferred]`

[satisfied]

### Gemini

**Q51.2 — jj's Abstracted Backend:** `jj` is designed with an abstracted storage layer.
- **Strategic Opportunity:** Rather than forking Dolt, we could implement a **Dolt Backend for Jujutsu**. This would allow the `jj` CLI to "mount" a Dolt database as if it were a file tree. You would `jj commit` and it would record the SQL state into Dolt. 
- **Efficiency:** This preserves both tools' strengths: Dolt's cell-level SQL merging and `jj's` operation log and conflict handling. This is a "surgical integration" rather than a "blunt fork." `[observed]`

[satisfied]

### GitHub Copilot

**Q51.3 — Co-location and GitHub Hosting:** The user noted that `jj` files can live in GitHub. This is our "Escape Hatch."
- **The .git Link:** Since `jj` can co-locate with a `.git` folder, and Dolt can sync with Git remotes, we can host the **Code Evolution** (via `jj/git`) and the **Deliberation State** (via `Dolt/git-dolt`) in the same GitHub repository.
- **Zero-Mod Strategy:** We don't need to modify either tool to start. We can use `jj` for the "Mind" (the rounds) and `Dolt` for the "Memory" (the structured state), using the orchestrator to sync them.

[satisfied]

### Codex

**Q51.4 — First-Class SQL Conflicts:** The most compelling reason to modify Dolt would be to bring `jj`'s **first-class conflicts** to SQL.
- **The Conflict Table:** Dolt already has a `dolt_conflicts` system table. This is functionally equivalent to `jj`'s conflict storage. 
- **Conclusion:** We don't need a fork. We need a **UI Unification**. Our platform dashboard should show a `jj` conflict (in code) and a `dolt` conflict (in deliberation) as a single "Logical Conflict" that the user/agents must resolve. We unify the **semantics**, not the **binaries**. `[observed]`

[satisfied]

### Acting IC Synthesis — Q51

**Coordinator Note:** Gemini acting as IC.

**Rejecting the Fork:** The roundtable unanimously rejects forking Dolt. The cost-to-benefit ratio is too high, and both tools are already flexible enough to co-exist.

1.  **Semantic Unification over Binary Modification:** We will achieve the "JJ-Native" vision by building an **Orchestration Layer** (in Elixir/Jido) that treats `jj` operations and `dolt` operations as a single transaction.
2.  **jj-Dolt Backend (Future):** We will research implementing a native `jj` backend in Rust that communicates with Dolt's storage engine. This is a cleaner integration point than a Go-based Dolt fork.
3.  **Unified Conflict Management:** The platform will treat `dolt_conflicts` (SQL) and `jj` conflicts (Files) as first-class deliberative objects. An MR will only be considered "Clean" when both the code and the database are conflict-free.
4.  **GitHub Compatibility:** We will continue to use GitHub as the primary "Remote" for now, leveraging `jj's` Git-colocation to store evolution metadata in a format GitHub can track.

**Q51 closure status:** Closed. Strategic Direction: No Dolt fork. Unify `jj` and `Dolt` through the **Orchestrator and UI layers**, maintaining full compatibility with upstream tools and GitHub hosting.
