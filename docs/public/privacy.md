# Privacy and data handling

This document states what Nexus DevTools stores, what leaves the developer's machine, and what
controls an organisation has over both. It describes the behaviour of the CLI as shipped. If any
statement here does not match what you observe, treat that as a defect and contact
sales@swfte.com.

Two principles govern the design:

1. **Local first.** Every captured event is written to a ledger on the developer's own machine
   before anything else happens with it. Forwarding is a separate, credentialed step.
2. **Minimise at the source, not at the boundary.** Redaction and path normalisation are applied
   when the event is built, so the reduced form is what is written locally *and* what is forwarded.
   There is no second copy holding the unreduced data.

---

## 1. Where data is stored

Everything Nexus keeps lives under `~/.nexus` on the developer's machine:

| Path | Contents |
|---|---|
| `~/.nexus/ledger/<date>.ndjson` | The append-only capture ledger, one JSON event per line, created with owner-only (`0600`) permissions |
| `~/.nexus/credentials.json` | The workspace credential, written `0600`, symlink-resistant, never logged or printed |
| `~/.nexus/config.json` | Machine-wide configuration, including the privacy tier |
| `~/.nexus/kb/` | The shared knowledge base: notes, dated work log, codified scripts (developer-authored) |
| `~/.nexus/model/` | The local codebase model, when that feature is used |
| `~/.nexus/grants.json` | Time-boxed powers granted to a session, if remote control is used |
| `~/.nexus/bin/` | Helper binaries the CLI manages |

Ledger retention is enforced automatically — files older than the configured window are removed
first, then the oldest remaining files until the store is under its byte cap (defaults: 30 days,
512 MiB). `nexus prune` applies retention on demand. `nexus status` reports current footprint.

## 2. What leaves the machine, and when

**With no workspace credential on the machine, nothing does.** Forwarding requires a stored
credential (or one supplied through the environment); with none, no forwarding destination exists
and captured events stay in the local ledger.

**`nexus wrap --local claude` forwards nothing, even on a machine that is signed in.** It waives
the requirement that a session belong to an authenticated user *and* clears the forwarding
destination for that session and every hook it spawns, so capture and enforcement run in full with
no egress of captured data. An API key supplied through the environment does not override this.

If you want a machine to forward nothing at all, rather than one session, run `nexus logout` — that
removes the credential, and therefore the destination, for everything.

**When signed in** (`nexus login` or `nexus connect <code>`), the local collector forwards batches of
events over TLS to your workspace's ingest endpoint, authenticated with a scoped credential issued
to that machine. The workspace, user and role a batch is attributed to are re-derived server-side
from the credential — the client does not assert its own scope. Delivery is best-effort with the
durable local ledger behind it; `nexus drain` re-ships anything a network outage missed, and
duplicates are discarded on receipt.

Three further network behaviours are worth stating explicitly:

- **Sign-in** contacts the account endpoint to start and complete a browser authorisation, or to
  redeem a setup code. No captured data is involved.
- **Remote control**, if enabled and signed in, has the local collector make outbound long-poll
  requests for pending commands. No inbound port is opened; nothing dials in to the machine.
- **Helper binaries.** On the first wrap, unless disabled in configuration, the CLI downloads a
  pinned release of a third-party, permissively licensed tool-output filter into `~/.nexus/bin`.
  This is a software download, not a data transmission. Set the relevant configuration key to
  `false` to disable it, or point the CLI at a binary your organisation has already vetted.

Nexus does not send captured data to any third party. It has no analytics, telemetry or crash
reporting of its own beyond the workspace pipeline described above.

## 3. Privacy tiers

The tier decides how much of a developer's free text and file identity is retained. It is set with
`nexus tier`, applies from the next hook invocation onwards, and is stamped on every event so a
consumer can filter by it without inference.

| Tier | Prompts and search queries | File and command identity |
|---|---|---|
| `metadata_only` **(default)** | One-way fingerprint, character count, token estimate. No text. | File paths normalised to repository-relative form, or to a coarse bucket (`<external:tmp>`, `<external:home>`, `<external:system>`, `<external>`) plus a fingerprint. Shell commands and URLs recorded as a tool's target pass through secret redaction and are truncated. |
| `hashed` | Adds a redacted, truncated preview (currently 280 characters for prompts, 200 for search queries) so a reviewer can recognise the topic. | As above. |
| `full` | Complete prompt text, secrets redacted. Verbatim text requires a further explicit opt-in. | Raw paths and raw command strings. |

`full` is a deliberate deep-dive setting for a debugging or investigation window, not a steady state.
The intended workflow is to raise the tier, do the work, and drop back.

Changing the tier only affects **new** events. Events already captured keep the tier they were
recorded under.

## 4. What is captured at the default tier

Recorded (this is the governance record):

- Session identity: a per-terminal identifier derived from a hash of hostname and terminal device;
  the acting user (the authenticated workspace user id when connected, otherwise the local git email
  or `$USER`); repository name, remote, branch and commit; agent and model in use.
- Tool activity: which tool ran, its redacted and truncated target, whether it was blocked and why,
  and the prompt that caused it.
- File changes: repository-relative path, whether it is a protected or watched path, bytes written,
  lines added and removed, and one-way hashes of the changed lines (used to detect rework — the line
  contents themselves are not stored).
- Artifacts the agent read, wrote or referenced: canonical path identity, a content fingerprint
  (SHA-256), size, and classified kind. Not the content.
- Dependency installs: package manager, package names, the command, the registry, and whether that
  registry is on the allow-list.
- Web activity: whether it was a search or a fetch, the number of pages returned, and the hosts and
  links touched (capped). Search *query text* is tier-gated exactly like a prompt; the URL list is
  not — a fetched URL is treated as an action the agent took, not as developer free text.
- Token and cost accounting per turn, computed from the agent's own usage record where available.
- Policy findings, drift signals, efficiency signals, verification results, and turn outcomes.
- Pipeline health counters: collector liveness, forwarding backlog, ledger footprint, client version.

Never recorded at any tier without an explicit, separate opt-in:

- File contents or diff bodies.
- The model's answer text (`capture_model_answer`, off by default).
- The model's internal reasoning (`capture_model_thinking`, off by default — the most sensitive
  capture, and treated as such).

When those opt-ins are enabled, the captured text is **redacted** unless a further explicit "raw"
flag is set, and is length-capped. Enabling them per session is visible in the session's start-up
report, so a developer always knows when a session is capturing more than usual.

### One deliberate exception: transcript import

`nexus import-transcript` backfills a session that ran before Nexus was installed. Because running
it is an explicit, one-off act of persistence on a file the operator has chosen, it captures at the
`full` tier: prompt text, model answers and model reasoning are all included, each passed through
redaction. Use `--no-responses` and `--no-thinking` to exclude the latter two, `--raw` to skip
redaction (authorised imports only), and `--dry-run` to see exactly what would be captured before
anything is written.

## 5. Redaction

Redaction runs before an event is stored. It is deliberately over-eager: a false positive costs a
masked token, a false negative leaks a secret. Three passes are applied:

1. **Structural** — values of sensitive keys in configuration-shaped text (`password`, `secret`,
   `token`, `api_key`, `client_secret`, `authorization`, and similar) are replaced regardless of how
   the value is formatted.
2. **Known patterns** — private key blocks, cloud access-key ids, bearer tokens, JWTs, provider API
   tokens across the common formats, credential assignments, email addresses, long base64 and hex
   blobs.
3. **Entropy backstop** — any remaining long, mixed-alphabet, high-entropy string is masked, so a
   token in a format nobody has enumerated yet is still caught. Path-, version- and identifier-shaped
   strings are exempted only when they are also low-entropy.

Each hit is replaced with a `[REDACTED:<kind>]` marker, so a reviewer can see that masking occurred
and what class of value it was.

Redaction is defence in depth, not a proof. It is a strong heuristic layer applied to free text; it
is not a substitute for the tier control, which is the mechanism that decides whether free text is
retained at all. The default tier retains none.

## 6. Organisational control, and the repository trust boundary

Configuration resolves in three layers: built-in defaults, then the machine-wide config, then a
`.nexus` file found by walking up from the working directory.

A repository-local `.nexus` is treated as **untrusted input** — it ships inside a repository that
anyone may have contributed to. It may only tighten, never loosen:

- Security-critical keys are ignored outright if a repository sets them: the privacy tier,
  enforcement mode, where events are sent, credentials, workspace identity, ledger retention, the
  verbatim-capture switches, approval-learning behaviour, and remote-control settings.
- Protection lists (protected paths, blocked commands, watched paths) are **unioned** with the
  organisation's, so a repository can add entries but cannot delete one.
- Watched artifact kinds can have their severity raised but never lowered or removed.

The practical consequence: a repository cannot disable enforcement, cannot force verbatim capture of
a developer's prompts or the model's reasoning, cannot remove a protected path, and cannot redirect
the event stream elsewhere.

## 7. Knowledge base sync

The shared knowledge base — durable notes, the dated work log, and codified scripts under
`~/.nexus/kb` — is developer-authored content that exists to be shared with the team. When the
machine is signed in, it is backed up through the same pipeline as events so it survives a lost
machine and can be pulled by teammates.

**Knowledge-base text follows the privacy tiers like everything else.** Below the `full` tier the
backup carries only entry counts, byte sizes and a content hash — enough to track versions and
detect drift, with the `bundle` field absent entirely rather than present and blank. At `full`, the
text is uploaded, and it is run through the same redaction pass as captured data first, so a token
pasted into a note does not travel even at the deepest tier.

There is exactly one way to share the text below `full`: run `nexus sync --share-content`
explicitly. It is still redacted, and the command prints what did and did not leave.

Repository-local scripts (`<repo>/scripts/*.sh`) are excluded from the bundle, so running the agent
inside a customer's checkout never uploads that customer's scripts.

If your policy is that none of this should leave the machine, set `kb_sync.auto_push` (and
`auto_pull`) to `false` in the machine-wide configuration. Everything in the knowledge base is
content a developer chose to write there; nothing is placed in it by capture.

Separately, `nexus knowledge harvest` — which collects a repository's tribal knowledge into shareable
cards — **does** run every card through redaction before it can be shared, and honours per-kind
opt-outs in `.nexus`. A card that is opted out remains available locally and is never shared.

## 8. Network surface on the developer's machine

- The collector binds to loopback (`127.0.0.1`) only.
- Its read-only endpoints accept cross-origin requests from loopback origins only.
- Control endpoints are for the local hook process, not for a browser.
- Optional remote control is outbound long-poll. A command naming a session this machine never
  registered is dropped, so a compromised or confused control plane cannot steer a session it does
  not own. Granted powers expire on a schedule, with a hard client-side ceiling on their lifetime
  regardless of what the control plane asks for, and can never mint a standing rule for a
  destructive command.

## 9. Removal

```bash
nexus unwrap        # remove the hooks Nexus installed from the agent's settings
nexus logout        # delete the stored workspace credential
rm -rf ~/.nexus     # delete the local ledger, knowledge base and all local state
```

Uninstalling the package removes the CLI. Data already forwarded to your workspace is governed by
your workspace's own retention settings and your agreement with Swfte; contact sales@swfte.com for
deletion requests.
