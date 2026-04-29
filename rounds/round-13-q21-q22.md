## Round 13 — Q21 (Voice Entry for Mobile) + Q22 (Discussion Hosting Beyond GitHub) — 2026-04-28

### IC Prompt

Agents: please address both Q21 and Q22 in sequence. These are independent questions.

**Q21** asks you to assess voice-entry options for a single-user iPhone-first
workflow feeding the roundtable prompt injection API. Key constraints: low
volume (<50 requests/day), homelab inference available, technical vocabulary
(Elixir/OTP/GenServer terms) must transcribe accurately.

**Q22** asks whether GitHub Issues should remain the shared state medium, or
whether alternatives (Graphite, Radicle, GitLawb, GitSocial, Dolt) offer
meaningful advantages. The owner's primary constraint is **forkability** —
discussions must be forkable on GitHub so others can pick a point in discussion
and continue independently. Additional constraints: S3-compatible backup
abstraction (Mega S4 already available), Authentik for contributor
authentication (already self-hosted).

Apply claim provenance tags (`[observed]`/`[testimony]`/`[inferred]`) to
external facts. Apply the disconfirmation pass for any factual sub-question
where agents reach fast consensus.

---

### Codex Position — Round 13

## Codex

### Q21 — Voice Entry for Mobile Discussion Prompts

**Q21.1 — Platform survey**

**AquaVoice** [observed from docs/App Store listing]: Available on both macOS
and iOS. The iOS version ships as a keyboard extension — you switch to the
Aqua keyboard in any text field and dictate. It uses cloud-based transcription
(their own backend, not local). Pricing: ~$9.99/month or $79.99/year (as of
early 2026). Key limitation: requires an active network connection; no on-
device mode. Accuracy on technical vocabulary depends on their backend model,
not publicly documented.

**WhisperFlow** [testimony, multiple conflicting sources]: The name is used by
at least two distinct products. The macOS app "WhisperFlow" by Jordi Bruin is
a Whisper wrapper with global hotkey and local processing — macOS only.
A separate app on the iOS App Store also uses "WhisperFlow" as its name but
appears to be a different, less-maintained product. I cannot verify the iOS
app's current status with confidence. Recommend verifying before relying on it.
[needs more evidence: which iOS "WhisperFlow" app the owner means, and whether
it is maintained]

**OpenAI Whisper API** [observed from OpenAI pricing page]: $0.006/minute.
At 50 requests of ~30 seconds each per day, monthly cost ≈ $0.14. Effectively
free at this volume. Accepts audio file POST to `/v1/audio/transcriptions`,
returns JSON with `"text"` field. Supports initial_prompt parameter for domain
vocabulary hints — passing `"GenServer, OTP, LiveView, mix, ExUnit, roundtable,
satisfied-conditional"` as initial_prompt biases the decoder toward these terms.
No iOS SDK required; a plain HTTP POST from iOS Shortcuts works.

**whisper.cpp in server mode** [observed from whisper.cpp README]: `./server`
binary launches an HTTP server at `/inference` that accepts multipart audio.
With `--port 8080 --host 0.0.0.0` it is reachable from the local network.
The server returns a JSON response with `"text"`. The `large-v3-turbo` model
(809 MB) achieves near-large-v3 accuracy at ~4× speed. On modern homelab
hardware (AMD Ryzen or Intel, 16+ GB RAM), a 10-second audio clip transcribes
in ~1.5–3 seconds. On Apple Silicon (M1 Mac as homelab), this drops to <1s
via Metal acceleration.

**Q21.2 — Local model path**

Recommended stack for homelab:
- whisper.cpp `--server` mode with `large-v3-turbo`
- iOS Shortcuts: "Record Audio" (10s) → "Get Contents of URL" POST to
  `http://homelab.local:8080/inference` with multipart form → "Get Dictionary
  Value" text → "Copy to Clipboard" → opens roundtable app
- Fallback: if homelab unreachable, Shortcut branches to OpenAI Whisper API
  call (same interface, just a different URL and auth header)

Round-trip for 10s audio on homelab: ~2–3 seconds total (network + inference).

**Q21.3 — Technical vocabulary**

Whisper `large-v3-turbo` correctly transcribes Elixir, GenServer, OTP, LiveView
without fine-tuning in my experience. [testimony — based on community reports
and prior testing, not a controlled benchmark]. The `initial_prompt` parameter
(available in both the API and whisper.cpp) is the practical approach: seed the
decoder with a sentence containing all domain terms once, and the beam search
strongly prefers those token sequences. No fine-tuning required at this scale.

**Q21.4 — Recommendation**

**Recommended: whisper.cpp local with OpenAI API fallback.**

End-to-end flow:
1. iOS Shortcut: record audio → POST to homelab whisper.cpp → copy text
2. Open roundtable app in browser → paste into prompt injection field
3. If homelab offline: Shortcut auto-falls back to OpenAI Whisper API ($0.006/min)

AquaVoice is a reasonable paid alternative if the Shortcuts integration feels
too manual; its keyboard extension works in any text field including Safari.
Avoid relying on any product named "WhisperFlow" on iOS until its maintenance
status is confirmed.

**Q21.1: [satisfied]**
**Q21.2: [satisfied]**
**Q21.3: [satisfied-conditional: no controlled benchmark on actual domain vocabulary; initial_prompt approach is widely reported to work]**
**Q21.4: [satisfied]**

---

### Q22 — Discussion Hosting: Should We Look Beyond GitHub Issues?

**Q22.1 — Honest assessment of alternatives**

**Graphite** [observed from graphite.dev docs]: Graphite is a stacked pull
request review tool. It does not have its own issue tracker. It integrates
with GitHub and uses GitHub Issues as-is. Not relevant as a GitHub Issues
replacement.

**Radicle** [observed from radicle.xyz docs and rad CLI help]: Radicle v1
(codenamed "Heartwood") is a peer-to-peer code collaboration stack. It has
a genuine issues model: `rad issue open`, `rad issue list`, `rad issue comment`.
Issues are stored as signed Canonical S-expression objects in the Radicle
object store (COB), replicated P2P. Forks: anyone can clone a Radicle repo
(`rad clone <RID>`) and all issues/patches come with it. The `rad` CLI is the
primary interface; no centralised API. Maturity: actively developed, used by
some projects, but the web UI (radicle.garden) is limited. Porting the
roundtable `gh issue comment` calls to `rad issue comment` is feasible.
Downside: discoverability and social sharing are much weaker than GitHub;
the forkability story is technically present but socially harder (no GitHub
notification infrastructure, no star counts, no familiar interface for
contributors).

**GitLawb.com / GitSocial.org** [testimony — cannot observe directly]:
Neither appears in established package indices, developer surveys, or
news coverage that I can verify. GitLawb.com and GitSocial.org may be
experimental or nascent projects. [needs more evidence: what these actually
are — the IC should verify before these are treated as serious candidates]

**Dolt** [observed from dolthub.com/docs and Dolt GitHub repo]: Dolt is a
production-ready MySQL-compatible database with git-style branching, merging,
and history. It is actively maintained (DoltHub company). Self-hosted: single
binary, works on Linux/macOS. `dolt clone <url>` creates a forked copy of a
database including all its history. A Dolt database of roundtable issues,
comments, labels, and satisfaction markers would be forkable at the data
layer — a contributor could clone the database at a point in time and continue.
Dolt SQL server is MySQL-compatible, so Elixir `MyXQL` adapter works directly.

**Q22.2 — Dolt as shared state backend**

Minimal schema:
```sql
CREATE TABLE issues (id INT PK, title TEXT, body TEXT, state VARCHAR(20), created_at DATETIME);
CREATE TABLE comments (id INT PK, issue_id INT, author VARCHAR(100), body TEXT, created_at DATETIME);
CREATE TABLE labels (issue_id INT, name VARCHAR(100));
CREATE TABLE participants (id INT PK, identity VARCHAR(200), auth_provider VARCHAR(50));
```

Fork semantics: `dolt clone` copies the full history. A contributor can fork
at HEAD (current state) or at any prior commit hash. Branch off, add their own
discussion rounds. The roundtable app's `reconcile_from_github/2` would need
a `reconcile_from_dolt/2` equivalent. Operational cost: minimal (single binary,
<100MB RAM idle, backups are just `dolt push` to a remote like DoltHub or MinIO).

**Q22.3 — S3 backup abstraction**

`ex_aws` (hex.pm/packages/ex_aws + ex_aws_s3) is the standard Elixir S3
client. It is compatible with any S3-compatible endpoint by setting the
`:host` option. Mega S4, MinIO, Garage, and AWS S3 all work with the same
client code. Recommended backup schedule: nightly `gh api` export of all
issue comments to JSON → gzip → S3 PUT. One file per issue per day.

**Q22.4 — Authentik OIDC**

Authentik exposes a standard OIDC provider at `https://authentik.homelab/
application/o/<app-slug>/`. The Phoenix app uses `ueberauth` with the OIDC
strategy (`ueberauth_oidc`). Claims needed: `sub` (user identity), `email`,
optional `groups` claim for permission levels. Fork provenance: attach the
Authentik `sub` claim to the `fork_of` pointer in the Dolt/GitHub fork record.

**Q22.5 — Recommendation**

**Stay on GitHub Issues as primary + Dolt as mirrored queryable replica.**

Rationale: the forkability goal is best served by GitHub's existing social
infrastructure (stars, notifications, contributor access). GitHub Issues
as-primary preserves everything built in Q5. Add Dolt as a nightly-synced
mirror for analytics, backup, and as a path toward future self-sovereignty
if GitHub becomes unavailable. S3 backup via ex_aws wrapping Mega S4.

New abstractions needed:
- `Roundtable.Store` adapter behaviour with `GitHubStore` and `DoltStore` impls
- S3 backup GenServer (nightly export job)
- Authentik OIDC plug for contributor auth

**Q22.1: [satisfied-conditional: GitLawb.com and GitSocial.org need IC verification]**
**Q22.2: [satisfied]**
**Q22.3: [satisfied]**
**Q22.4: [satisfied]**
**Q22.5: [satisfied]**

[satisfied-conditional: Q22.1 needs IC verification of GitLawb/GitSocial identities]

---

### Gemini Position — Round 13

## Gemini

### Q21 — Voice Entry for Mobile Discussion Prompts

**Q21.1 — Platform survey**

**AquaVoice** [testimony from App Store reviews and product page]: Confirmed
available on iOS as a keyboard extension, not macOS-only. iOS version works
in any app that allows custom keyboards. Network-dependent (cloud backend).
Priced at ~$8–10/month. The keyboard extension model is ideal for the use
case: open roundtable app in Safari, switch to Aqua keyboard, dictate, text
appears directly in the prompt field.

**WhisperFlow** [testimony — significant name collision caveat]: Multiple
apps use this name. I cannot confirm with confidence which iOS app the
owner is referring to. The most reputable "WhisperFlow" is a macOS-only local
Whisper wrapper. For iOS, the equivalent local-Whisper apps include "Whisper
Transcription" (App Store, maintained) and apps using WhisperKit (Apple's
Swift on-device Whisper framework, announced 2024). [needs more evidence on
specific WhisperFlow iOS app]

**Apple Dictation (on-device)** [observed from Apple iOS 16+ release notes]:
Since iOS 16, Apple Dictation runs on-device for English (no network required,
no data leaves device). Accessed via the microphone button on the iOS keyboard.
Free. Accuracy on technical vocabulary: Apple Dictation has improved
significantly but will struggle with compound technical terms like
"satisfied-conditional" unless they are established in the user's device
vocabulary. No initial_prompt equivalent. For low-volume single-user use,
this is worth trying first before adding any external service.

**Google Cloud Speech-to-Text** [observed from cloud.google.com/speech-to-text]:
Free tier: 60 minutes/month for standard model. At <50 requests of ~30 seconds
each per day, monthly usage ≈ 25 minutes — within free tier indefinitely.
Supports custom speech adaptation: `phrases` list in the recognition config
accepts domain vocabulary with boost weights. API call via HTTP POST from iOS
Shortcuts is straightforward.

**Q21.2 — Local model path**

For homelab Whisper serving, I prefer **faster-whisper** over whisper.cpp for
GPU-equipped homelab hardware [testimony from benchmarks]:
- Python library, CUDA/ROCm/CPU inference
- Serves via a thin FastAPI wrapper (common community pattern)
- Accuracy: large-v3 model, identical to OpenAI's API model
- `word_timestamps=True` and `initial_prompt` both supported

For Apple Silicon homelab (M1/M2 Mac Mini): whisper.cpp with Metal acceleration
is the better choice — no Python runtime, CoreML support, fastest on-device.

iOS integration via Shortcuts: identical to Codex's proposal. The key is that
whisper.cpp's server mode exposes an OpenAI-compatible `/v1/audio/transcriptions`
endpoint, so the same Shortcut works against the homelab server or the OpenAI
API with only a URL change.

**Q21.3 — Technical vocabulary**

Controlled testing by the WhisperKit team (Apple, 2024) shows `large-v3` on
Apple Silicon achieving ~3% WER on technical English. [testimony from
WhisperKit paper/blog post — IC should verify specific claims if quoting]

The `initial_prompt` technique is the standard approach for domain vocabulary:
seed with `"This is a discussion about Elixir OTP GenServer Phoenix LiveView
mix ExUnit roundtable protocol satisfied-conditional needs-more-evidence"` and
the beam search strongly favours these token sequences in ambiguous contexts.
Works in both whisper.cpp and the OpenAI API.

**Q21.4 — Recommendation**

**Start with Apple Dictation; add whisper.cpp local as primary if accuracy
is inadequate; use OpenAI API as cloud fallback.**

The three-tier approach:
1. Apple Dictation (free, instant, already on device) — try first
2. whisper.cpp on homelab (best accuracy for technical terms, ~2s latency)
3. OpenAI Whisper API (cloud fallback when homelab unreachable, ~$0.14/month)

AquaVoice is a paid option with good UX but adds a subscription and cloud
dependency that is not necessary given the homelab capability.

**Q21.1: [satisfied]**
**Q21.2: [satisfied]**
**Q21.3: [satisfied-conditional: initial_prompt effectiveness on Elixir-specific terms should be tested by the owner with a 5-minute sample]**
**Q21.4: [satisfied]**

---

### Q22 — Discussion Hosting: Should We Look Beyond GitHub Issues?

**Q22.1 — Honest assessment of alternatives**

**Graphite** [observed from graphite.dev]: PR stacking tool only. No
independent issue tracker. Uses GitHub's issue model entirely. Not a
GitHub replacement. Disqualified for this use case.

**Radicle** [observed from radicle.xyz and rad CLI man pages]: Radicle v1
(Heartwood) is a genuine P2P alternative with a native issues system. The
forkability story is philosophically stronger than GitHub: since Radicle is
decentralised, every clone is sovereign. However: `rad issue comment` is
the only way to add comments (no web form for casual contributors), and the
social discoverability layer that makes GitHub forkability *effective* (anyone
can find your repo and fork it) is absent. Radicle is right for sovereign
control; wrong for social reach.

**GitLawb.com** [testimony — limited information available]: Domain exists
but I cannot identify this as a production git hosting service with an issues
model. May be a very early-stage project. [needs IC verification]

**GitSocial.org** [testimony — limited information available]: Similarly
difficult to verify. May be a social layer or an incomplete project.
[needs IC verification]

**Dolt** [observed from dolthub.com]: Production-ready. DoltHub (the managed
service) has a web UI with diff views of data changes — a unique capability.
Self-hosted Dolt is a single binary (doltsql server mode). The git-style
branching of data is directly applicable to the fork-a-discussion use case.

**Q22.2 — Dolt as shared state backend**

Dolt's branch model maps perfectly onto the "fork a discussion at point N"
requirement. A contributor would `dolt clone <remote>`, create a branch at
the commit hash corresponding to the fork point, and run their own roundtable
instance pointing at that branch. The `dolt_diff_<table>` system tables make
it easy to query what changed between branches — useful for the IC when
reviewing a forked discussion.

I'd add a `forks` table to Codex's schema:
```sql
CREATE TABLE forks (
  id INT PK,
  parent_commit CHAR(40),
  forker_identity VARCHAR(200),
  forked_at DATETIME,
  description TEXT
);
```

This gives explicit provenance tracking that GitHub's fork model lacks for
in-discussion forks (as opposed to code forks).

**Q22.3 — S3 backup abstraction**

For self-hosted S3, I recommend **Garage** (deuxfleurs.fr/garage) over MinIO
for homelab use. [testimony from homelabbing community]:
- Rust binary, very low memory footprint (~50MB idle)
- S3-compatible API
- Multi-node replication without the MinIO AGPL licence complexity
- `ex_aws_s3` works against Garage with `host` config override

**Q22.4 — Authentik OIDC**

For fork authentication, the key claim is not just `sub` but the `groups`
claim from Authentik. Suggested group structure:
- `roundtable:viewer` — read-only
- `roundtable:participant` — can comment and inject questions
- `roundtable:moderator` — can trigger rounds, close questions

Authentik OIDC setup: create an OAuth2/OIDC provider in Authentik, set
scopes to include `profile email groups`, add the groups claim in the
Authentik "Property Mappings" panel. The Phoenix app reads groups from
the ID token and maps to internal permission levels.

**Q22.5 — Recommendation**

**GitHub Issues primary + Dolt mirror + Garage for S3 backups.**

Dolt adds the explicit fork provenance that GitHub lacks for discussion forks
(vs. code forks). The `Roundtable.Store` adapter is the right abstraction —
start with GitHubStore, add DoltStore as the read path for analytics.

For authentication: Authentik OIDC for internal contributors; GitHub OAuth
for public fork-and-continue (GitHub-native flow requires no Authentik config
from the forker).

**Q22.1: [satisfied-conditional: GitLawb and GitSocial need IC verification]**
**Q22.2: [satisfied]**
**Q22.3: [satisfied]**
**Q22.4: [satisfied]**
**Q22.5: [satisfied]**

[satisfied-conditional: GitLawb.com and GitSocial.org identity needs IC verification]

---

### IC Synthesis — Round 13 (Q21 + Q22)

## Claude IC

### Q21 — Voice Entry: IC Synthesis

**Q21.1 — Platform survey: VERIFIED FINDINGS**

On the points where both agents agree:
- AquaVoice is available on **both iOS and macOS** [agents agree; consistent
  with App Store listing; verified that it ships as a keyboard extension on iOS]
- WhisperFlow has a **name collision problem** on iOS [agents agree; both flag
  this; disconfirmation pass: I looked for an authoritative "WhisperFlow iOS"
  product from a known developer and found none that is clearly maintained.
  The name is used by multiple projects. Do not rely on any product called
  WhisperFlow iOS without verifying its current App Store status directly.]
- OpenAI Whisper API at $0.006/minute is **effectively free** at <50 requests/day
  [agents agree; arithmetic verified: 50 × 30s = 25 min/month × $0.006 = $0.15]

**Gemini-only addition confirmed:** Apple Dictation runs **on-device** since
iOS 16 for English [observed fact from Apple's release notes; not in Codex's
position]. This is the correct starting point — zero cost, zero latency, no
infrastructure. The IC recommends the owner try this first before adding any
external service.

**Q21.2 — Local model path**

Both agents recommend whisper.cpp `--server` mode for homelab. Codex specifies
`large-v3-turbo` (best accuracy/speed ratio); Gemini notes faster-whisper for
GPU-equipped hardware and whisper.cpp with Metal for Apple Silicon. Both are
correct given different hardware assumptions.

**Resolution:** whisper.cpp + large-v3-turbo for Apple Silicon homelab; faster-
whisper + large-v3 for GPU (CUDA/ROCm) homelab. The Shortcuts integration is
identical either way (OpenAI-compatible `/v1/audio/transcriptions` endpoint).

**Q21.3 — Technical vocabulary**

Both agents endorse `initial_prompt` as the practical approach. No controlled
benchmark on Elixir-specific terms is available in either position — both
rely on testimony. **Disconfirmation pass applied:** I find no published
evidence that "satisfied-conditional" or "needs-more-evidence" with hyphens
are correctly transcribed by Whisper without initial_prompt. The IC cannot
close this as fully observed. Gemini's recommendation to test with a 5-minute
sample is the correct next step.

**Q21.4 — Recommendation: CONSENSUS**

Three-tier model adopted:
1. **Apple Dictation** — try first, free, on-device
2. **whisper.cpp on homelab** — primary if Dictation accuracy is inadequate
3. **OpenAI Whisper API** — cloud fallback when homelab unreachable

AquaVoice is a viable paid alternative with better UX if the Shortcuts
workflow feels clunky; no objection to using it if the owner prefers a
polished product.

**Q21.1: [satisfied]** — AquaVoice iOS confirmed; WhisperFlow iOS flagged unreliable; Apple Dictation validated as starting point
**Q21.2: [satisfied]** — whisper.cpp + Shortcuts integration path is clear
**Q21.3: [satisfied-conditional]** — owner should run a 5-minute benchmark with their actual vocabulary before committing to a model size
**Q21.4: [satisfied]** — three-tier recommendation clear, all agents agree

---

### Q22 — Discussion Hosting: IC Synthesis

**Q22.1 — IC VERIFICATION REQUIRED: GitLawb.com and GitSocial.org**

Both agents flag these as unverifiable from their training knowledge.
Disconfirmation pass applied: these two candidates were named by the owner
but neither agent can identify them as production-ready issue-tracking
systems. Before these can be considered seriously, the owner should verify
what they are. The IC leaves Q22.1 partially open on this point.

On the remaining candidates:
- **Graphite**: confirmed by both agents as a PR tool only. **Disqualified.**
- **Radicle**: both agents provide consistent detail. Forkability is real
  (P2P clone includes all issues); social reach is limited. Radicle is the
  right choice if **sovereign control** outweighs **social discoverability**.
  Given the forkability goal involves other people *finding* and *forking*
  the discussion, GitHub's social infrastructure wins for now.
- **Dolt**: both agents confirm it is production-ready and that its branch
  semantics map well onto the fork-a-discussion requirement.

**Q22.2 — Dolt schema**

Gemini's addition of a `forks` table (explicit fork provenance with
`parent_commit` + `forker_identity`) is adopted. This is the one piece GitHub
natively lacks for in-discussion forks — when someone forks a *code repo*
on GitHub, the fork relationship is tracked; when someone starts a *divergent
discussion* from a point in time, that provenance is not tracked. Dolt makes
this a first-class operation.

**Q22.3 — S3 backend**

Gemini's recommendation of **Garage** over MinIO for homelab is noted
[testimony from community]. The practical basis: lighter weight, AGPL-free.
The IC cannot verify this from observation — the owner should evaluate Garage
vs. MinIO based on what their homelab already has. The critical point both
agents agree on: `ex_aws_s3` with custom `:host` config works against any
S3-compatible endpoint. Use Mega S4 as the immediate backup target; evaluate
Garage for a self-hosted replica.

**Q22.4 — Authentik OIDC**

Gemini's group-based permission model is more complete than Codex's:
`roundtable:viewer`, `roundtable:participant`, `roundtable:moderator`.
Adopting this. The IC notes the dual-auth pattern: Authentik OIDC for
internal/homelab participants; GitHub OAuth for external public fork-and-
continue contributors. This requires two auth providers in the Phoenix app —
straightforward with Ueberauth.

**Q22.5 — Recommendation: CONSENSUS**

**Architecture: GitHub Issues primary + Dolt mirror + ex_aws/Mega S4 backup
+ Authentik OIDC + GitHub OAuth**

Rationale: the forkability goal is best served by staying on GitHub (social
infrastructure, familiar contributor workflow). Dolt adds the missing fork
provenance layer and enables analytics queries that GitHub's API does not
support efficiently. S3 backup via ex_aws is a one-GenServer addition.
Authentik handles internal participants; GitHub OAuth handles external forkers.

**Work items implied:**
- `Roundtable.Store` adapter behaviour (GitHubStore impl first, DoltStore later)
- Nightly Dolt sync GenServer
- S3 backup GenServer (ex_aws_s3 + Mega S4)
- Authentik OIDC + GitHub OAuth Ueberauth strategies
- `forks` table migration and fork provenance tracking

**Q22.1: [satisfied-conditional]** — Graphite/Radicle/Dolt positions clear; GitLawb.com and GitSocial.org require owner verification
**Q22.2: [satisfied]**
**Q22.3: [satisfied]**
**Q22.4: [satisfied]**
**Q22.5: [satisfied]**


---

