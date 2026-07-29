# Command reference

Every command is `nexus <command>`. Run `nexus <command> --help` for the exact flags of any command;
this page is the map.

Commands marked **Core** are available on every install. Commands marked **Intelligence** require a
workspace entitled for them; without one they print an explanatory message and exit non-zero without
side effects.

---

## Session

| Command | Tier | Description |
|---|---|---|
| `nexus wrap <agent> [args…]` | Core | Install hooks, ensure the collector, apply policy, and launch the agent |
| `nexus unwrap` | Core | Remove the hooks Nexus installed |

`nexus wrap` flags:

| Flag | Effect |
|---|---|
| `--local` | Run without a workspace account: capture and enforcement only, no egress of captured data |
| `--capture-answer` | Record the model's verbatim answers for this session (redacted) |
| `--capture-answer-raw` | As above, without redaction — for an authorised investigation only |
| `--capture-thinking` | Record the model's reasoning trace for this session (redacted) |

The three capture flags are announced in the session boot report, so elevated capture is never
silent.

## Account

| Command | Tier | Description |
|---|---|---|
| `nexus setup` | Core | Connect this machine and install the Claude Code hooks. Idempotent: a no-op once connected |
| `nexus login` | Core | Browser sign-in linking this machine to your workspace. `--no-browser` prints the URL instead of opening it |
| `nexus connect <code>` | Core | Redeem a setup code issued from the workspace dashboard |
| `nexus whoami` | Core | Show the current identity (never prints the credential) |
| `nexus logout` | Core | Remove the stored credential |

`nexus setup` is what the npm postinstall runs, and what the first interactive `nexus` command runs
if the postinstall could not. It takes three routes in priority order: `NEXUS_SETUP_CODE` from the
environment (no browser, no terminal), an interactive browser sign-in, or — with neither — it defers,
reported but never fatal. `--quiet` suppresses output unless something fails; `--max-wait` caps the
approval wait.

For headless and CI use, a credential can be supplied through the environment instead of a stored
file; see [configuration.md](configuration.md).

## Operations

| Command | Tier | Description |
|---|---|---|
| `nexus status` | Core | Collector health, ledger footprint, forwarding state |
| `nexus doctor` | Core | Environment diagnostics: agent settings, hooks, collector, ledger, helpers, background workers |
| `nexus update [--channel npm\|pip]` | Core | Upgrade to the latest release through whichever channel installed this copy. Keeps working while a client policy blocks everything else |
| `nexus tail [-n N]` | Core | Watch captured events as they land |
| `nexus drain` | Core | Re-ship events left unforwarded by an outage, from a persisted watermark |
| `nexus prune [--days N]` | Core | Apply ledger retention (age and size caps) |
| `nexus collector [--port] [--gateway]` | Core | Run the local collector in the foreground (normally started automatically) |
| `nexus rtk [status\|install]` | Core | Status of, or install, the bundled tool-output filter binary |
| `nexus hook <event>` | Core | Internal hook dispatcher. Not intended to be run by hand |

## Privacy and policy controls

| Command | Tier | Description |
|---|---|---|
| `nexus tier [metadata_only\|hashed\|full]` | Core | Show or set the capture privacy tier. Writes machine-wide config; a repository cannot set this |
| `nexus asks [on\|off]` | Core | Whether advisory findings interrupt the session with a confirmation |
| `nexus approvals [list\|on\|off\|forget\|clear]` | Core | Inspect, disable or forget learned permission rules |
| `nexus remote [status\|on\|off]` | Core | Remote-control state for this host |
| `nexus grants [list\|revoke\|clear]` | Core | List or revoke time-boxed powers granted to a session |

## Cost

| Command | Tier | Description |
|---|---|---|
| `nexus savings [--days N] [--json]` | Core | What was saved over a window, by mechanism, labelled measured or estimated |
| `nexus compress <file> [--engine]` | Intelligence | Measure compression on a given input |
| `nexus route [report\|learn\|mode] [--days N]` | Intelligence | Model routing: report the recommendation, refold outcome history into rules, or set the mode |
| `nexus recall [list\|learn\|match] [--days N]` | Intelligence | The context-recall index: inspect it, rebuild it, or test what a prompt would match |
| `nexus macros [list\|mine] [--days N]` | Intelligence | Mined command macros for this repository |

## Zero-token automation

| Command | Tier | Description |
|---|---|---|
| `nexus scripts [list\|run <name>]` | Core | List or run a codified repeating task — no agent turn, no tokens |
| `nexus codify <name> "<command>"` | Core | Capture a repeating command as a reusable script |

## Knowledge

| Command | Tier | Description |
|---|---|---|
| `nexus memory [--path\|--edit]` | Core | The durable shared knowledge file |
| `nexus diary [list\|add\|show]` | Core | The dated work log |
| `nexus kb [status\|log] [--limit N]` | Core | Knowledge-base version history — what changed and when |
| `nexus sync [push\|pull]` | Core | Two-way team sync of the shared knowledge base |
| `nexus team [list\|show <user>]` | Core | Browse teammates' synced knowledge bases |
| `nexus knowledge [harvest\|share\|export]` | Intelligence | Harvest a repository's knowledge into redacted, shareable cards |

## Audit and analysis

| Command | Tier | Description |
|---|---|---|
| `nexus audit <session> [--json]` | Intelligence | Reconstruct a session: prompt, change, finding, rationale, security note |
| `nexus integrity <session> [--json]` | Intelligence | Objective and drift report for a session |
| `nexus trace [--session] [--correlate]` | Intelligence | Long-horizon episodes and outcome correlation |
| `nexus experience [--session] [--json]` | Intelligence | Scored experiences: efficiency, quality, risk |
| `nexus curriculum [--json]` | Intelligence | Measured-gap analysis: where the agent should improve |
| `nexus explain` | Intelligence | Run the behind-the-scenes rationale reflection for a turn |
| `nexus whys [mine\|review\|list]` | Intelligence | Reverse-engineer missing rationales and confirm the candidates |
| `nexus model [build\|enrich\|show\|list\|security\|review]` | Intelligence | The reverse-engineered codebase model: how, why, and security invariants per module |
| `nexus repo-intel [--cwd] [--out]` | Intelligence | Analyse a repository: call sequences, data schema, security posture |
| `nexus metrics [--days N] [--json]` | Intelligence | Workspace telemetry summary |
| `nexus rollup [--out] [--days N]` | Intelligence | Fold the ledger into the dashboard feed |

## Evidence and feedback

| Command | Tier | Description |
|---|---|---|
| `nexus verify [--kind] [--if-changed]` | Core | Run the repository's verification suite and record the grounded result |
| `nexus feedback <session> --good\|--bad [--note]` | Core | Record a human judgement of a run |
| `nexus import-transcript <file>` | Core | Backfill sessions that ran before Nexus was installed. `--dry-run` shows what would be captured |

`nexus import-transcript` is an explicit act of persistence and therefore captures at the `full`
privacy tier — prompt text, model answers and model reasoning, each redacted. Use `--no-responses`
and `--no-thinking` to exclude the latter two, `--raw` to skip redaction (authorised imports only),
and `--dry-run` to see what would be captured before anything is written. See
[privacy.md](privacy.md).

---

Reference: [how it works](how-it-works.md) · [privacy](privacy.md) · [configuration](configuration.md)
