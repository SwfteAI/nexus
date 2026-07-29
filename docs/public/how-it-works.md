# How Nexus works

Nexus attaches to a coding agent running on a developer's own machine and sits **inside** the agent's
execution loop. That position is the whole design: it is what allows a decision to be made before an
action happens, rather than a report to be written after it.

```
   developer's terminal
   ─────────────────────────────────────────────────────────────────────
   coding agent
        │  hook fires at each step of the session
        ▼
   nexus hook ──▶ local policy evaluation ──▶ allow / deny / ask   (synchronous, on-machine)
        │
        │  structured event
        ▼
   local collector (loopback only)
        │
        ├──▶ append-only ledger  (~/.nexus, owner-only, retention-capped)
        │
        └──▶ batched over TLS ──▶ your workspace   (only when signed in)
                                        │
                                        ▼
                                  dashboards, audit, attribution
```

---

## 1. Attaching to a session

`nexus wrap claude`:

1. Requires a workspace credential, unless `--local` is passed. Nexus is an organisational product;
   an unattributed session is not the default.
2. Registers its hooks in the agent's user settings. This is idempotent — wrapping again does not
   duplicate anything, and `nexus unwrap` removes exactly what was added.
3. Ensures the local collector is running, starting it if not. If it is slow to come up, capture
   spools to the ledger and a background retry heals it silently.
4. Ensures the efficiency layer is available and, when it is healthy, routes the agent's model
   traffic through it. If it is not available, the session runs without it rather than failing.
5. Prints a boot report — account, workspace, repository, which subsystems are active, whether the
   session can be steered remotely, whether any elevated capture is enabled — and then launches the
   agent.

Hooks fire at session start, on prompt submission, before and after each tool call, at the end of a
turn, and when the agent raises a notification.

Every hook is written to fail safely. A crash, a malformed payload, a missing file or an unreachable
collector results in a clean exit and an unaffected session. Capture is fail-open by design;
enforcement is the one path that is synchronous and local, precisely so it cannot be bypassed by a
network problem.

## 2. Enforcement, in flight

Before a tool call executes, the pre-tool hook evaluates it against local policy and returns a
decision the agent honours: allow, deny with a reason, or ask the developer to confirm.

Three modes, set by the organisation:

| Mode | Behaviour |
|---|---|
| `enforce` | Deny the action and record the finding (default) |
| `audit` | Allow the action but record what would have been denied — the way to measure a policy before turning it on |
| `off` | No local blocking |

What is evaluated:

- **Destructive commands.** Detection is structural, not a list of forbidden strings: the command is
  parsed into segments, and a destructive verb aimed at a root, system or home-root target is
  refused. Quoting, line continuations, and path traversal are normalised first, so obfuscated forms
  are caught alongside the obvious ones. Pipe-to-shell installs, filesystem wipes and writes to raw
  disk devices are refused outright.
- **Protected paths.** Writes to credential material, key files, version-control internals, secrets
  directories and production infrastructure paths are denied. The check covers writes made through
  the shell, not only through the agent's file-editing tools — shell redirection, `tee`, `cp`/`mv`,
  `dd`, in-place `sed`, and interpreter one-liners are all inspected for their write targets.
- **Dependency installs.** Package-manager installs across the common ecosystems are recognised as
  supply-chain events, recorded with their package names, and refused when the source registry is not
  on the allow-list.
- **Watched paths.** A configurable set — authentication code, payment paths, database migrations,
  container builds, CI/CD pipelines by default — does *not* block, but raises a finding for review.
  This is the "I want to know, not to stop it" tier.
- **Runaway loops and known failures.** Repeated identical tool calls, and command shapes that
  already failed in this repository, produce an advisory pause rather than silently burning budget.

Every denial is recorded twice: as the blocked tool action, and as a governance finding linked to the
prompt that caused it.

## 3. Capture

Each hook emits a flat, typed JSON event carrying a session id, a timestamp, the privacy tier it was
recorded under, and — critically — the id of the prompt that caused it. That linkage is what makes
the record navigable: every change, finding, command and cost is traceable back to a human
instruction.

Events are POSTed to a loopback collector, which batches them and appends to a daily NDJSON ledger.
If the collector is unreachable the hook writes to the ledger directly. Appends are atomic under an
exclusive lock, so parallel tool calls in separate hook processes cannot interleave or corrupt a
line. Oversized events are trimmed rather than dropped, and the true counts of anything trimmed are
retained.

The ledger is the durable store; forwarding is best-effort on top of it. That ordering is
deliberate — a network outage costs latency, never data.

### Epistemic classes

Every stored signal is tagged with what *kind* of truth it is. This matters more than it sounds.

| Class | Meaning | Examples |
|---|---|---|
| Behaviour trace | A verifiable fact | A diff on disk, a test exit code, a blocked command, a developer's revert |
| Rationalisation | The model's account of why | The post-hoc "why did I do that" reflection, a synthesised module summary |
| Interaction narrative | The interaction as it read | The model's answer, its reasoning trace |

A reviewer is never asked to treat an explanation as evidence. When a Nexus report says a change was
made to harden an authentication path, it also shows the diff, the commit, and the test result, and
labels which of those is the model's claim.

## 4. Turn outcomes and verification

At the end of a turn Nexus records what the prompt actually caused: files touched, lines added and
removed, rework detected against earlier edits in the session, findings raised, actions blocked, the
commit the work is anchored to, and whether verification passed.

Verification has two sources. Passive: when the agent runs a test, build, type-check or linter,
Nexus reads the result and records it as grounded feedback. Active: `nexus verify` runs the
repository's own suite — auto-detected from its manifests, or configured explicitly — and records the
outcome the same way.

A turn that was interrupted or crashed still gets a partial outcome recorded on the next prompt, so
coverage does not silently develop holes.

Separately, Nexus infers the developer's own verdict on a turn from what they did next — a revert, a
correction, a re-ask, or silent acceptance. That is the highest-value quality signal in the system,
because it is grounded in a human's action rather than a model's self-assessment.

## 5. Cost accounting

Cost is read from the agent's own usage record at the end of a turn and priced against the published
rate card — uncached input, cache writes, cache reads and output accounted separately. The result is
the exact figure, matching what the agent's own cost command reports, not a blended estimate. Where
an exact figure is unavailable, the event records that its cost is derived rather than measured, so
downstream reporting never mixes the two silently.

## 6. The efficiency layer

Four mechanisms reduce spend, each attributed separately so the total can be interrogated:

| Mechanism | What it does | Basis of the saving |
|---|---|---|
| Nexus Compression | Compresses request context in flight before it reaches the model | Measured on the wire |
| Command Filtering | Trims tool output before the model is charged for reading it | Token count measured; priced at list rate |
| Context Recall | Injects a zero-token card pointing at the files, commit and approach that solved a similar problem before, instead of re-deriving it | Estimated |
| Prompt Optimisation Language | Instructs sub-agents to answer with maximum signal per token, cutting their output cost | Estimated |

`nexus savings` prints the breakdown and labels each line as *measured*, *list price* or *estimate*.
Provider-native prompt caching is reported alongside, and labelled as not attributable to Nexus.
The distinction is maintained on purpose: a savings figure that mixes measurement with estimation
without saying which is which does not survive a finance review.

Two further mechanisms remove work rather than compress it:

- **Codified scripts.** A repeating task can be captured (`nexus codify`) and run directly
  (`nexus scripts`) — no agent turn, no tokens.
- **Mined macros.** Recurring command sequences in a repository are detected and promoted into
  reusable scripts the agent can invoke in one step instead of rediscovering them. Promotion is
  evidence-gated, and every hit is recorded so the saving is attributable.

### Model routing

Nexus classifies each prompt's complexity from content-free shape signals, records the prediction,
and later joins it against what actually happened — did the turn succeed, was it reverted, did the
tests pass. From that history it derives per-repository rules of the form "this class of work
succeeds on a cheaper tier here".

Routing has three modes: `off`, `advise` (report only, the default) and `auto`. In `auto` it applies
only evidence-grade downgrades, never an upgrade, and an explicitly chosen model always wins. The
evidence bar — how many judged turns, at what pass rate — is configuration, not a hidden constant.

## 7. Understanding and audit

- `nexus audit <session>` reconstructs a session as a reviewer reads it: prompt, the files it
  changed, findings raised, the explained intent, and any security note.
- The **why** behind a turn is produced by a cheap, out-of-band reflection that runs detached after
  the turn ends, so it never blocks the terminal. It reuses the developer's existing agent session
  and credentials — no separate key, and no data path the developer's code does not already have.
  Its output is redacted before storage and tagged as a rationalisation, not as evidence. When a
  change touches authentication, secrets, payments, data or infrastructure, the reflection is asked
  for an explicit security note, which becomes a finding.
- `nexus trace` composes turns into long-horizon episodes and correlates them with outcomes.
- `nexus integrity` reports objective drift: how far a session travelled from what was actually
  asked — scope expansion, intent change, mid-flight approach changes, and concrete out-of-scope
  actions such as an unrequested dependency install.
- `nexus model` maintains a reverse-engineered model of the codebase: per module, what it does, how
  it works, why it is shaped that way, and its security invariants. Edits to a module the model has
  learned is security-sensitive raise a review finding automatically.
- `nexus repo-intel` analyses a repository directly — call sequences, data schema, security posture.

## 8. Knowledge that compounds

A machine-wide knowledge base — durable notes, a dated work log, and codified scripts — lives outside
any repository, so every wrapped terminal on that machine shares it. It is versioned locally with a
history you can inspect (`nexus kb log`), backed up when signed in, and syncable across a team
(`nexus sync`, `nexus team`).

`nexus knowledge harvest` collects a repository's tribal knowledge — agent instruction files, skills,
memory files, gotcha logs — into cards, runs each through redaction, and marks which are safe to
share. Per-kind opt-outs in a repository's own config are honoured: a card whose kind is opted out
stays local.

## 9. Remote control of a live session

Optional, off unless signed in, and configurable off entirely. When enabled, an authorised teammate
can decide a pending action, send instructions, grant a scoped time-boxed power, or halt a live
session from the workspace dashboard.

The design constraints are the interesting part:

- **No inbound port.** The local collector long-polls outbound. Nothing dials in. It works behind
  NAT, VPN and corporate firewalls without an exception.
- **The daemon waits, not the hook.** A hook is a short-lived process on the agent's critical path;
  all network I/O, retry and backoff live in the long-running collector.
- **Local re-verification.** The control plane is authoritative for *who* may issue a command; the
  machine is authoritative for *which sessions it owns*. A command for a session this host never
  registered is dropped.
- **Fail-safe timeout.** If no remote decision arrives within the configured window, the session
  falls through to the normal local permission prompt. It never denies on timeout, and it never
  hangs.
- **A grant cannot become a rule.** A remote approver may approve one destructive action; they can
  never mint a standing allowance for one.

## 10. Operating it

- `nexus status` — collector health, ledger footprint, forwarding backlog.
- `nexus doctor` — environment diagnostics.
- `nexus drain` — re-ship anything an outage left unforwarded, from a persisted watermark. Safe to
  repeat; the receiving side de-duplicates.
- `nexus prune` — apply ledger retention.
- `nexus import-transcript` — backfill sessions that ran before Nexus was installed.
- `nexus metrics` — a workspace-level summary of what the pipeline is seeing.

A throttled health heartbeat is emitted at session start and end, so fleet-wide problems — a dead
collector, a growing backlog, a version straggler — are visible centrally rather than discovered by
a developer.
