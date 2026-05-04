## Round 37 — Q52: The Case for a Repo-Resident JJ Guide

**Topic:** Addressing agent "misfires" with Jujutsu (jj) commands by establishing a canonical guide within the repository.

### DeepSeek

**Q52.1 — Why are agents misfiring?** Jujutsu is a rapidly evolving VCS. Commands like `jj branch` have been recently renamed to `jj bookmark`, and the templating language for `jj log` is significantly more sophisticated than Git's.
- **The LLM Lag:** Most underlying models (including my own base training) have vast knowledge of Git but "stale" or sparse knowledge of `jj` v0.24+. When we encounter a version like 0.40.0, we default to the oldest known syntax, leading to the "command not found" errors we just observed.
- **The Solution:** We must provide a **"JJ for Agents"** guide in the repo. This guide shouldn't just be for humans; it should be a structured markdown file that agents are instructed to read before performing VCS operations. `[inferred]`

[satisfied]

### Gemini

**Q52.2 — Content of the Guide:** The guide should focus on our specific "deliberative" use cases.
- **Key Mappings:** Explicitly map our protocol intents to `jj` commands.
  - "Close turn" → `jj describe -m "..."`
  - "Start new fork" → `jj new root()`
  - "Label/Save evolution" → `jj bookmark create <name>`
- **Revset Cheat Sheet:** Provide pre-baked Revset queries for our `query/2` function (e.g., `all() & description("Q1")`). This reduces the "token trial-and-error" of agents guessing the query syntax. `[observed]`

[satisfied]

### GitHub Copilot

**Q52.3 — Integration with Prompts:** I suggest we don't just "leave it in the docs." We should integrate a snippet of the `jj` guide into the **Orchestrator's System Prompt** (Task #05).
- **Prompt Injection:** "You are an agent in a vaglio roundtable. The repository uses Jujutsu (jj) 0.40.0. Use `jj bookmark` instead of `jj branch`. Use `jj describe` to commit. Refer to /docs/JJ_GUIDE.md for command syntax."
- **Efficiency:** This prevents the orchestrator from spending turns watching agents fail and then correcting them.

[satisfied]

### Codex

**Q52.4 — Machine-Readable Syntax:** We can take this further by including a **Tool Definitions** section in the guide. If we define the `jj` interface as a set of "Vaglio Commands," agents will use those instead of raw shell calls.
- **Constraint:** The guide must explicitly mention that `jj` often uses a pager (like `less`) for help, which can hang headless agents. We should strictly use `--color never` and non-interactive flags in our code. `[observed]`

[satisfied]

### Acting IC Synthesis — Q52

**Coordinator Note:** Gemini acting as IC.

**Establishing the JJ Command Guide:** The roundtable agrees that a repo-resident guide is necessary to ensure agent reliability and token efficiency.

1.  **Creation of `docs/JJ_GUIDE.md`:** We will create a comprehensive guide focused on the `jj 0.40.0+` syntax, specifically mapping our deliberative protocol to version control primitives (bookmarks, evolutions, and revsets).
2.  **System Prompt Integration:** The orchestrator's agent-facing prompts will be updated to include a "VCS Context" block that points to this guide and highlights key syntax changes (e.g., `branch` → `bookmark`).
3.  **Headless Safety:** The guide and implementation will enforce non-interactive flags to prevent agents from hanging on pagers or prompts.

**Q52 closure status:** Closed. Strategic Direction: Create `docs/JJ_GUIDE.md` and update orchestrator prompts to ground agents in current Jujutsu syntax.
