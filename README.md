# Nexus

**Nexus wraps your coding agent, lives inside its execution loop, and turns AI-assisted development
into something you can afford, govern, and trust** — cutting token spend, stopping risky actions
before they run, and recording a provable trail of everything the agent does.

Nexus works with **any coding agent**. It starts with **Claude Code** today — **Codex and the rest
are coming soon**.

```bash
npm i -g @swfte/nexus
nexus wrap claude
```

That's it. From the first prompt you're capturing, governed, and saving — nothing else to configure.

![Nexus wrapping Claude Code — one command boots the agent under capture, governance, and savings](https://raw.githubusercontent.com/SwfteAI/nexus/main/docs/assets/demo.gif)

Home: **[www.swfte.com](https://www.swfte.com)**

---

## Why we built it

Coding agents changed the cost and the risk of shipping software overnight — and the tooling didn't
catch up.

The **cost** became a large, variable, per-developer bill nobody could attribute. Vendor dashboards
show you a total; they can't tell you which repo, which engineer, or which prompt spent it — and
they can't cut it. The same context gets re-sent every turn. The same files get re-read and
re-searched. The same boilerplate burns full agent turns. Flagship models run work a cheaper one
would finish identically. The waste is invisible, so it never gets fixed.

The **risk** is that an agent editing your production repo is an actor with write access — and most
teams genuinely cannot say what it did, why, or whether it was allowed to. The record lives in
throwaway chat logs, the reasoning is tangled up with the actions, and there's no line between "the
model *says* it did X" and "X provably happened."

Both problems have one root cause: **nothing was watching the loop itself.** Sit inside the agent's
execution loop, capture the ground truth once, and you can save money *and* govern the work from the
same place — locally, attributably, without asking anyone to trust a screenshot. That's Nexus.

---

## It makes agents cheaper — and proves it

![How Nexus cuts token spend](https://raw.githubusercontent.com/SwfteAI/nexus/main/docs/assets/savings.png)

Nexus treats every token as a cost to justify, and attacks it four ways. Every mechanism's
contribution is **measured on the wire**, not estimated — `nexus savings` shows exactly what each one
saved, so the number holds up in front of finance.

| | How it saves |
|---|---|
| **Compress context** | Shrinks the context window in flight before it's sent — you pay for meaning, not bulk. |
| **Filter tool output** | A 5,000-line build log or `ls -R` never reaches the model at full price; only what matters does. |
| **Reuse prior searches** | Nexus indexes everything the agent has already searched, read, and derived — and **serves it back instead of paying to re-discover it.** The biggest waste in agent work, turned into a lookup. |
| **Codify repeats** | Recurring command sequences replay as scripts with **zero agent turns and zero tokens.** |
| **Route by outcome** | Proposes the cheapest model your repo's *own results* justify — most turns don't need the flagship, and Nexus knows which ones from your history. |

```bash
nexus savings     # what you saved this week, by mechanism
```

### The algorithm: index once, reuse everywhere — and learn as you go

Most agent spend is **rediscovery** — re-reading the same files, re-running the same searches,
re-deriving what it knew a moment ago. Nexus builds a **centralised source of references**:
everything an agent indexes, searches, or learns is captured once and served back on demand, so it
**stops paying to re-index and re-search content it already has.** Same budget, far more work.

And it compounds — the longer Nexus runs, the more it saves:

- **Continuous learning.** Every turn's real outcome — did it work, did your checks pass, was it
  reworked — feeds back, so recall, routing, and macros sharpen against *your* repositories over time.
- **Dynamic model shifting.** Nexus routes each turn to the cheapest model your own results prove is
  sufficient, and re-evaluates as evidence accumulates. Routine turns stay cheap; only the ones that
  genuinely need the flagship get it.
- **A shared reference graph.** What one engineer's agent indexes becomes an instant lookup for the
  whole team's agents — nobody re-pays to discover what the team already knows.

The net effect: **do more with fewer tokens**, and the gap widens the longer you run it.

The rule throughout: **spend less without changing what the developer sees.** Nexus never silently
degrades a result to save money, and every saving is attributable so you can audit the trade-off.

---

## It governs what agents do — before they do it

Put agents near production with confidence.

![How Nexus enforces policy before a tool call runs](https://raw.githubusercontent.com/SwfteAI/nexus/main/docs/assets/enforcement.png)

**Enforcement happens in flight, on-machine, and can't be bypassed.** Before any tool call runs,
Nexus judges it against your policy and returns allow / deny / ask — synchronously, entirely on the
developer's machine, so a network problem can't route around it. Destructive commands, out-of-bounds
paths, things that should never happen unattended: stopped, with the reason shown to the developer
and recorded. Roll it out safely with `audit` mode first, then flip to `enforce`.

**Everything is traceable — what happened, not what was claimed.** Every turn, Nexus records the
*facts*: files touched, lines changed, rework, findings raised, whether your checks passed — anchored
to the commit, with the exact token cost from the agent's own usage. It keeps the model's account of
**why** separate from the verifiable record of **what**, so a reviewer never has to take an
explanation as evidence.

```bash
nexus audit <session>     # the full prompt → change → finding → rationale trail
```

Dependency installs are logged as supply-chain events. Sensitive-code touches raise findings. And
from the dashboard you can require a minimum client version, freeze a team or the whole fleet, revoke
a machine, or halt a live session — all through a signed policy the client verifies.

---

## It maps your codebase — so you can see it

Nexus doesn't just watch the agent; it builds an understanding of the code and lets you *look* at it.

- **A model of your codebase** — what each module does, **why** it exists, and its security
  invariants, reverse-engineered and kept honest (claims stay marked as claims until a human confirms
  them).
- **Repository intelligence** — **sequence diagrams**, a **database-schema map**, and a **security
  posture**, generated from the code and kept current.
- **A visual knowledge graph** — your architecture, module relationships, and team knowledge export
  into a graph you can render and explore, not read as prose.
- **A shared, versioned knowledge base** — what one engineer's agent learns becomes something the
  whole team (and their agents) build on.

Your codebase, categorised and visual — modules, data model, call flows, and the reasoning behind
them — instead of tribal knowledge that walks out the door.

---

## Who it's for

- **Engineering leaders** who adopted agents and can't yet answer "what did they change, what did it
  cost, and who's accountable?"
- **Security & compliance** who need agent activity attributable and bounded by policy — not
  reconstructed from chat logs after an incident.
- **Platform & FinOps** who own the model bill and need per-repo, per-engineer attribution and a
  savings number that survives scrutiny.
- **Individual engineers** who just want local cost visibility with nothing leaving the laptop —
  `nexus wrap --local claude` does exactly that: full capture and enforcement, zero egress.

---

## Get started

```bash
npm i -g @swfte/nexus     # macOS (Apple Silicon + Intel) and Linux (glibc + musl)
nexus wrap claude         # install hooks, start capture + enforcement, launch the agent
```

`npm install` signs you in through your browser and installs the agent hooks automatically — a fresh
machine is connected and capturing without reading another line. It's best-effort and **never fails
an install** (it stays quiet on CI, Docker, and headless setups); the first `nexus` command you run
finishes setup if needed. For fleets, set `NEXUS_SETUP_CODE` for zero-touch, no-browser onboarding.

Same tool via pip (coming soon): `pip install swfte-nexus`. Either way you get one command, `nexus`,
on your `PATH`. It's a single self-contained binary — **no Python or runtime deps**, macOS builds
Developer-ID-signed and notarized. (Windows support is on the way.)

```bash
nexus wrap claude    # wrap Claude Code (Codex and others coming soon)
nexus savings        # what the efficiency layer saved
nexus audit <id>     # the prompt → change → outcome trail
nexus model          # the reverse-engineered map of your codebase
nexus doctor         # environment check
```

Full command reference: **[docs/public/commands.md](docs/public/commands.md)**.

---

## Private by default

- **Local first.** Every event lands in an append-only ledger under `~/.nexus` (owner-only).
  Forwarding needs a workspace credential; with none, nothing leaves the machine. `nexus wrap --local`
  guarantees **zero egress** per session even when signed in.
- **You set the privacy tier.** Default `metadata_only` keeps **no prompt text, no diffs, no file
  contents** — just fingerprints, counts, and secret-redacted metadata. `hashed` and `full` opt in to
  more, deliberately. A repo-local config can *tighten* the tier but never *loosen* it.
- **Redaction before storage**, not at the network edge — keys, tokens, JWTs, emails, and
  high-entropy secrets are masked, over-redacting rather than risking a leak.
- **The model's answers and reasoning are never captured by default.**
- **No inbound network surface** — the collector is loopback-only; remote control is outbound
  long-poll. Nothing dials in.

Full statement: **[docs/public/privacy.md](docs/public/privacy.md)**.

---

## The Open Alexandria Project — our pledge

Nexus exists to make AI-assisted development dramatically cheaper. We've made a pledge about what to
do with that: **a share of every token Nexus saves goes to building the Open Alexandria Project — a
public, freely available knowledge archive for everyone.**

The same efficiency layer that trims your bill helps fund a permanent, open commons of engineering
knowledge — searchable and free to all, not locked inside any one company. The more Nexus saves, the
more we give back.

Participation contributes only **anonymised, aggregated, secret-redacted** knowledge — **never your
code, your prompts, or proprietary content.** The [privacy model](#private-by-default) governs it in
full, and it's on by default. You can **opt out at any time:**

```bash
nexus alexandria off      # opt out of the Open Alexandria Project
nexus alexandria on       # rejoin  ·  nexus alexandria  shows your current status
```

We think tools that profit from the world's code owe something back to it. The Open Alexandria
Project is how we pay that debt — openly, and for everyone.

---

## Coming soon: open-source & self-hostable

We're releasing an **open-source version of the entire Nexus stack** so you can **self-host and
centralise it all in-house** — the collector, the ingest and telemetry backend, the codebase-model
and knowledge graph, and the governance and policy control plane — running inside your own network,
under your own control, with nothing leaving your infrastructure. If you need agent savings and
governance but can't send anything to a hosted service, this is for you.

Register interest: **[www.swfte.com](https://www.swfte.com)** · **sales@swfte.com**

---

## Licensing

Nexus is proprietary software, **free to download and use** under the [End User Licence
Agreement](LICENSE) and the [Swfte Terms of Service](https://www.swfte.com/terms) — installing or
using it accepts both. The compiled binary is distributed openly; the source is not published. Run
`nexus licenses` for third-party notices.

- Product: **[www.swfte.com](https://www.swfte.com)**
- Licensing, procurement, security review, pilots, and open-source early access: **sales@swfte.com**
