## Round 10 — Q18: Mobile Agent Supervision Interface

**IC prompt (Round 10, 2026-04-28):**

We have built the web dashboard (item 10) which removes the laptop relay
bottleneck. The next friction point is supervision from mobile devices —
the owner wants to watch rounds, inject questions, and get notified when
consensus is reached or human review is needed, without needing a full
browser session.

Q18 has four sub-questions:

- **Q18.1** — Survey: what do developers actually use today for mobile CLI agent
  supervision? (Termius/Tailscale, Prompt/Blink/ShellFish, web dashboards,
  emerging native apps)
- **Q18.2** — Phoenix Channels + native clients: can a Swift/Kotlin app connect
  to the LiveView WebSocket protocol directly, or do we need a REST/SSE
  companion API? What client libraries exist and what do they cover?
- **Q18.3** — Minimum feature set: of the four supervision tasks (watch, alert,
  inject, trigger), which need real-time push vs. polling? Can ntfy.sh /
  Pushover / Shortcuts cover the alerting half without native code?
- **Q18.4** — Recommended path: PWA, companion JSON/SSE API, Phoenix Channels
  Swift client, or push-notification-first? What is the minimum useful step
  vs. the ideal end state?

Agents: address all four sub-questions. Bring concrete evidence — existing
libraries (with repo links and last-commit dates), documented Phoenix Channels
client support, real-world reports from the community on mobile agent
supervision workflows. The IC will synthesise after all voices have posted.

---

## Codex Position — Q18 — 2026-04-28

### Q18.1 — Survey: what developers actually use today

The current mobile supervision pattern is not "native agent control app" first.
It is still **remote shell plus lightweight notifications**.

The strongest evidence is where the mature mobile tools are investing:

- **Blink Shell** is explicitly optimized for mobile remote work: Mosh, SSH,
  Files.app integration, persistent sessions across network transitions, and
  even remote VS Code / Codespaces connectivity. That is a mobile shell stack,
  not a bespoke agent UI. (`https://blink.sh/`)
- **Termius** positions itself as a cross-device SSH workspace with synced
  hosts, snippets, port forwarding, and iOS/Android parity. Again: mobile
  supervision is framed as "reach the remote machine and drive it there."
  (`https://termius.com/`)
- **Secure ShellFish** is the clearest "agent supervision is emerging on
  mobile" signal I found. Its 2026 release notes mention lock-screen **Live
  Activity** for terminals, shell-side `notify` / `widget` commands, explicit
  Tailscale hostname handling, and even a Claude Code UX fix. That is already a
  proto-supervision loop. (`https://apps.apple.com/us/app/ssh-client-secure-shellfish/id1336634154`)
- **Tailscale SSH** fits the access layer around these tools: the docs say you
  can connect "from any device running Tailscale," while the server component
  lives on Linux/macOS targets. That matches the practical setup people use:
  mobile device as control surface, real work on a remote host.

So the real survey answer is:

1. mobile SSH client + VPN/mesh (`Blink`, `Termius`, `ShellFish`, `Tailscale`)
2. browser dashboard if one exists
3. simple push notifications for completion / failure alerts
4. only then a dedicated native app

That is also why Q18 should not assume a native mobile client is the first
useful surface. The current market says the opposite.

Q18.1: [satisfied]

### Q18.2 — Phoenix Channels + native clients

For **Phoenix Channels**, the ecosystem is good enough. For **raw LiveView
protocol**, it is the wrong abstraction for v1 mobile supervision.

Phoenix's own docs still list only the JavaScript client as official and list
Swift/Kotlin clients as third-party. That matters: there is no first-party
Swift/Kotlin Phoenix Channels stack to build on, but there are usable community
clients.

Concrete client evidence:

- `davidstump/SwiftPhoenixClient`
  - repo: `https://github.com/davidstump/SwiftPhoenixClient`
  - `pushed_at`: 2026-02-12T14:38:39Z
- `liveview-native/liveview-client-swiftui`
  - repo: `https://github.com/liveview-native/liveview-client-swiftui`
  - `pushed_at`: 2026-01-22T15:09:24Z
- `com.glia:phoenix-channels-client` Kotlin Multiplatform artifact
  - package page: `https://mvnrepository.com/artifact/com.glia/phoenix-channels-client`
  - latest release: 1.1.3 on 2026-01-16

But the key distinction is protocol scope:

- **Phoenix Channels** is a stable enough target for app-specific events.
- **LiveView** is not just "a channel with nicer widgets." It carries diffing,
  lifecycle semantics, and client behavior that a generic supervision app does
  not need.

LiveView Native is also informative here. The `liveview-native/live_view_native`
core repo was archived on **2026-02-10**, which is a warning against assuming
there is a broad, settled native-client path for generic LiveView consumption.
Its SwiftUI client remains active, but that project is for building native apps
from LiveView, not for a thin supervision console.

My recommendation:

- **Do not** have a Swift/Kotlin supervision app speak raw LiveView protocol.
- If native clients are added, give them a **companion JSON API** for commands
  and a **simple SSE feed** for watch/status events.
- If duplex native interactivity is later needed, add a **small Phoenix
  Channels topic** for operator actions, not full LiveView protocol support.

That keeps mobile clients cheap to build and decouples supervision from web UI
internals.

Q18.2: [satisfied]

### Q18.3 — Minimum feature set: push vs polling

Of the four supervision tasks:

- **Watch**: can start as polling, but SSE materially improves it
- **Alert**: needs push
- **Inject**: request/response is enough; real-time not required
- **Trigger**: request/response is enough; real-time not required

So the minimum split is:

- **push** for "consensus reached", "human review needed", "agent failed",
  "round stalled"
- **poll or SSE** for timeline / transcript / current state
- **plain POST** for operator injections and manual triggers

The alerting half can absolutely be solved without native code in v1.

Concrete evidence:

- **ntfy** is explicitly built for "send push notifications to your phone or
  desktop via scripts from any computer, using simple HTTP PUT or POST
  requests." That is almost a perfect match for round completion and review
  alerts. (`https://docs.ntfy.sh/`)
- **Pushover** exposes a straightforward HTTPS POST API at
  `https://api.pushover.net/1/messages.json`, which is enough for production
  alerts with priorities, sounds, and device targeting. (`https://pushover.net/api`)
- **Apple Shortcuts** already supports `GET`, `POST`, `PUT`, `PATCH`, and
  `DELETE` through the "Get Contents of URL" action. That means iPhone-side
  trigger/inject workflows can exist before a native app does. (`https://support.apple.com/guide/shortcuts-mac/request-your-first-api-apd58d46713f/mac`)

So yes: **ntfy / Pushover / Shortcuts can cover most of the alerting and some
manual control surface for v1**. The only thing they do not replace is a rich,
continuously updating transcript view.

Q18.3: [satisfied]

### Q18.4 — Recommended path

The minimum useful step and the ideal end state are different.

**Minimum useful step (v1.1):**

1. keep the web dashboard as the primary rich UI
2. add push notifications via `ntfy` or `Pushover`
3. add a tiny authenticated JSON surface for:
   - `POST /inject_question`
   - `POST /trigger_round`
   - `POST /ack_review`
4. add an SSE feed for:
   - round status
   - active speaker
   - satisfaction changes
   - review-needed events
5. make the web dashboard installable as a PWA for "open phone, glance, tap
   through" usage

That path gets mobile supervision with minimal architecture risk.

**Ideal end state (v2):**

- native Swift/Kotlin shell over the same JSON/SSE contract
- optional Phoenix Channels topic for low-latency operator actions
- platform-native notifications and lock-screen / widget surfaces

What I would explicitly *not* do:

- no raw LiveView-protocol mobile client in v1
- no "build native first and figure out API later"
- no assumption that operator watching requires full-duplex realtime from day
  one

There is a good precedent for this shape outside Phoenix too: OpenCode exposes a
headless HTTP server with an OpenAPI spec plus SSE event streams. That is the
same architectural lesson here. The mobile surface should talk to a small,
purpose-built supervision API, not to the internal UI protocol.

Q18.4: [satisfied]

### Q18.5 — OpenCode fork for iOS/TestFlight

The OpenCode fork/TestFlight path is **credible as a v2 accelerator**, but it
should not replace the lower-risk v1 mobile plan.

Why it is credible:

- OpenCode has a documented **client/server split**. `opencode serve` exposes an
  OpenAPI endpoint plus SSE/global event streams, which is exactly the kind of
  architecture a mobile client wants.
- The upstream repo is extremely active. The repo metadata fetched today for
  `anomalyco/opencode` shows `pushed_at` `2026-04-28T04:24:31Z` and ~150k stars,
  which is a strong maintenance signal.
- There are already third-party mobile-adjacent proofs that this is a real use
  case:
  - `grinev/opencode-telegram-bot`
    - explicitly positioned as "run and monitor AI coding tasks from your phone
      while everything runs locally on your machine"
    - `pushed_at`: `2026-04-26T11:29:01Z`
  - `grapeot/opencode_ios_client`
    - native iOS client repo
    - `pushed_at`: `2026-04-24T03:51:46Z`

That means the "mobile surface" is not hypothetical. But it is still not the
same as "Roundtable should fork and ship it now."

Pros:

- faster than inventing a native client from zero
- aligned with an existing agent-first ecosystem the owner is already watching
- inherits a mature API/event model rather than forcing Roundtable to invent all
  mobile semantics itself

Cons:

- it inherits another product's assumptions and release cadence
- OpenCode's UX is optimized for "coding agent client" more than "supervise a
  roundtable orchestrator"
- a forked client can become its own maintenance burden quickly
- TestFlight distribution solves delivery, not product fit

My recommendation:

- **v1 / v1.1:** do not fork OpenCode for the mobile supervision path
- **v2 experiment:** if the PWA + push + companion API path proves insufficient,
  then an OpenCode-derived iOS client is a strong shortcut worth testing
- if pursued, keep the contract boundary clean: the app should talk to a stable
  supervision API, not to internal LiveView protocol

So the OpenCode fork is a good **acceleration option**, not the minimum useful
step.

Q18.5: [satisfied]

**Statuses:**
- Q18.1: [satisfied]
- Q18.2: [satisfied]
- Q18.3: [satisfied]
- Q18.4: [satisfied]
- Q18.5: [satisfied]

---

---

## Round 10 — Q18 Agent Positions (summary)

**Codex (Q18.1–Q18.4):** Terminal-first still dominant (Termius 4.7★/18K ratings, Blink 3.1★, ShellFish 4.8★, Prompt 4.3★). Emerging orchestration-first apps: Onepilot, OpenCode Telegram bot (540★), unofficial iOS OpenCode client on App Store. LiveView Native archived Feb 10 2026 — rules out native LiveView binding. SwiftPhoenixClient (530★, Jan 2025) viable for plain Channels. Recommends companion JSON/SSE API + ntfy/Pushover + PWA. Against OpenCode fork as primary path: "couples us to another product's UX and assumptions."

**Gemini (Q18.1–Q18.4):** Same terminal-first observation. Highlighted Moshi app (HITL push notifications from mobile). Grapeot/OpenCodeClient in TestFlight as OpenCode iOS reference. Originally recommended LiveView Native as Phase 3 — RETRACTED after fact check (archived Feb 10 2026 confirmed). Recommends ntfy.sh Phase 1, PWA Phase 2, purpose-built app Phase 3. $5 Pushover one-time vs. self-hostable ntfy.

**Both (Q18.5):**
- OpenCode architecture: HTTP + SSE (`opencode serve`, OpenAPI 3.1 spec at `/doc`, `/event` SSE stream). iOS clients connect to your own server. SSH tunnel support (grapeot/OpenCodeClient uses Citadel).
- Repo moved from `sst/opencode` → `anomalyco/opencode`. 779 releases; latest April 27 2026. Fast-moving upstream.
- Gaps for roundtable: satisfaction labels not native; round triggering not a first-class concept.
- Cost: $99/year Apple Dev + 90-day TestFlight build refresh.
- Codex verdict: [satisfied] — companion API + PWA is better primary path; OpenCode fork only if goal is "mobile coding agent control generally."
- Gemini verdict: [satisfied-conditional] — OpenCode fork is a valid "Admin Console / Pro tool" complement, not the primary path. PWA wins for v1.

---

## IC Synthesis — Q18 (Round 10, 2026-04-28)

**IC: Claude**

### Resolved fact conflicts

LiveView Native (`liveview-native/live_view_native`) was archived February 10, 2026 — confirmed by both agents independently. Gemini's original Q18.2 recommendation of LiveView Native as Phase 3 is withdrawn. This rules out the "bind native app directly to LiveView protocol" path for any new project as of today.

### Q18.1 — State of the art: closed

The current market is two-tier:
- **Terminal-first:** Termius, Blink Shell, ShellFish, Prompt — SSH + Mosh + Tailscale is the dominant real pattern. Excellent for session persistence, reconnect, and high-density output. Poor for structured supervision tasks.
- **Orchestration-first (emerging):** Moshi (HITL push alerts), Onepilot (agent dashboard + terminal on iOS), OpenCode mobile clients (grapeot TestFlight + unofficial App Store builds), OpenCode Telegram bot (540★). These are not mature but prove the demand.

The hardest things to replicate without a terminal: streaming stdout density, session recovery, interactive approval prompts, and arbitrary shell escape hatches. Our orchestrator does not need most of these for the supervision use case.

**Q18.1: [satisfied]**

### Q18.2 — Phoenix Channels native clients: closed

LiveView protocol binding → ruled out (archived). Plain Phoenix Channels → viable via SwiftPhoenixClient (530★, v5.3.5 stable, v6.x in dev for Swift 6 concurrency) and JavaPhoenixClient (71★). However, the right contract for mobile is not LiveView protocol internals — it is a companion REST + SSE API that exposes only what a supervisor needs:

```
GET  /api/state           — current questions, labels, running status
GET  /api/events          — SSE stream: agent_done, round_start, consensus, human_review
POST /api/questions       — inject question
POST /api/rounds/trigger  — start a round for open questions
```

This is smaller than LiveView, has no wire-format coupling to Phoenix internals, and works equally well for a PWA, a Shortcut, or a future native app.

**Q18.2: [satisfied]**

### Q18.3 — Minimum feature set: closed

| Task | Mechanism | Real-time needed? |
|---|---|---|
| Watch | SSE (`/api/events`) | Yes — polling feels wrong during an active round |
| Alert | Push (ntfy.sh or Pushover) | Yes — must fire when app is closed |
| Inject | HTTP POST | No — fire and forget |
| Trigger | HTTP POST | No — fire and forget, watch SSE for progress |

ntfy.sh (self-hostable, iOS app, Elixir `Req.post!` trivial integration) covers alerting without native code. Pushover ($5 one-time, 3K★/4.8★) is the stable commercial fallback. Apple Shortcuts can wrap the POST endpoints for inject and trigger with zero app development.

**Q18.3: [satisfied]**

### Q18.4 — Recommended path (non-fork): closed

Consensus path, in order:

1. **Immediate:** Add ntfy.sh push notifications to orchestrator events (`consensus_reached`, `needs_human_review`). Cost: ~2 hours. Unblocks phone supervision today.
2. **Short-term:** Expose the companion REST/SSE API above. Cost: ~1 day. Enables Shortcuts and future native clients with a stable contract.
3. **Medium-term:** Ship the existing LiveView dashboard as a PWA (manifest + service worker). iOS 16.4+ supports Web Push for home-screen apps. Cost: ~1 day. Covers iPad well; acceptable on iPhone for actions.
4. **Future:** Purpose-built iOS app using SwiftPhoenixClient against the SSE/REST API. Only warranted if PWA UX proves insufficient.

**Q18.4: [satisfied]**

### Q18.5 — OpenCode fork: closed

OpenCode's architecture is HTTP + SSE (`opencode serve`), OpenAPI 3.1 spec public, SSH tunnel supported in existing iOS clients. This is technically sound. However, for this project's supervision use case, the fit is partial:

| Supervision need | OpenCode native support |
|---|---|
| Streaming agent turns | Yes (SSE TextDelta) |
| Satisfaction label display | No — not in data model |
| Question injection | Partial (message send, not roundtable question) |
| Round triggering | No — not a first-class concept |

The repo is extremely fast-moving (779 releases, latest April 27 2026, already migrated from `sst/opencode` → `anomalyco/opencode`). A fork tracking that velocity plus maintaining a Swift client plus $99/year Apple Dev plus 90-day TestFlight build refreshes is a non-trivial ongoing cost for partial fit.

**Decision:** The OpenCode fork is not the v1 path for this orchestrator's supervision needs. It is potentially valuable as a future "Pro layer" if the product expands toward general coding-agent control, or if the owner decides to invest in a richer iPad experience beyond supervision. For now, the companion API + PWA path has better fit/cost ratio.

**Q18.5: [satisfied]**

### Protocol updates

**Protocol Update 6 — Mobile supervision architecture**

- Orchestrator MUST emit push notifications to ntfy.sh on `consensus_reached` and `needs_human_review` events (configurable topic, `NTFY_TOPIC` env var).
- A companion REST + SSE API (`/api/*`) is the canonical mobile contract. LiveView dashboard remains the primary browser UI and is not duplicated.
- LiveView dashboard SHOULD be served with PWA manifest for home-screen install.
- OpenCode fork is deferred to v2 and only warranted if supervision scope expands beyond roundtable-specific needs.
- LiveView Native is ruled out: archived February 10, 2026.


---

