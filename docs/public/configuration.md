# Configuration

Nexus resolves configuration from three layers. Later layers win, subject to the trust rules below.

1. **Built-in defaults** — safe, restrictive, and usable with no configuration at all.
2. **Machine-wide config** — `~/.nexus/config.json`. This is where an organisation's settings land,
   whether placed by an administrator, a managed-device profile, or your own provisioning.
3. **Repository config** — a `.nexus` file, found by walking up from the working directory.

Both files are JSON objects. Unknown keys are ignored; a malformed file is treated as absent rather
than being allowed to break a session.

---

## The repository trust boundary

A `.nexus` file ships inside a repository, so it is attacker-influenced input. It may **tighten**,
never loosen.

**Ignored outright when set by a repository:**

- The privacy tier.
- The enforcement mode.
- Where events are sent, and any credential or workspace identity.
- Ledger retention limits.
- The verbatim capture switches for prompts, model answers and model reasoning.
- Approval-learning behaviour.
- Remote-control settings, including whether it is armed and how long a granted power may live.
- The address of the local optimisation proxy — which decides where model traffic goes.

**Merged so they can only get stricter:**

- `protected_paths`, `blocked_commands`, `watched_paths` — a repository's entries are added to the
  organisation's, never substituted for them.
- `watched_artifact_kinds` — a severity may be raised, never lowered or removed.

**Passed through** — everything else: knowledge-sync opt-outs, per-feature disables, verification
commands, and similar operational settings.

---

## Settings an organisation typically sets

Placed in `~/.nexus/config.json`.

### Privacy

| Key | Default | Meaning |
|---|---|---|
| `privacy_tier` | `"metadata_only"` | `metadata_only`, `hashed` or `full`. Also settable with `nexus tier` |
| `capture_model_answer` | `false` | Record the model's verbatim answers |
| `capture_model_answer_raw` | `false` | …without redaction |
| `capture_model_thinking` | `false` | Record the model's reasoning trace |
| `capture_prompt_raw` | `false` | At `full` tier, keep prompt text unredacted |
| `model_answer_max_chars` | `20000` | Length cap on captured answers |

See [privacy.md](privacy.md) for exactly what each tier retains.

### Enforcement

| Key | Default | Meaning |
|---|---|---|
| `enforcement` | `"enforce"` | `enforce`, `audit` (record what would have been blocked, allow it) or `off` |
| `protected_paths` | credential material, key files, version-control internals, secrets directories, production infrastructure paths | Writes here are denied |
| `blocked_commands` | `[]` | Additional organisation-specific denials. The catastrophic cases are handled structurally and are always active |
| `allowed_registries` | the standard public registries for the major ecosystems | Dependency installs from anywhere else are denied |
| `watched_paths` | authentication code, payment paths, migrations, container builds, CI/CD pipelines | Touching these raises a finding; it does not block |
| `watched_artifact_kinds` | credential material, audio, archives, binaries | Artifact classes that raise a finding when touched |

### Retention

| Key | Default | Meaning |
|---|---|---|
| `ledger_retention_days` | `30` | Age cap on the local ledger |
| `ledger_max_bytes` | 512 MiB | Size cap; the oldest files are removed first |

### Knowledge sync

| Key | Default | Meaning |
|---|---|---|
| `kb_sync.enabled` | `true` | Master switch for knowledge-base backup and team sync |
| `kb_sync.auto_push` | `true` | Back the knowledge base up when it changes (throttled) |
| `kb_sync.auto_pull` | `true` | Mirror teammates' knowledge bases at session start |
| `sync.skills` / `sync.agents_md` / `sync.mastermind` | `true` | Per-kind opt-outs for harvested knowledge cards — agent skills, agent instruction files, and sharing to the central knowledge store respectively. Also settable per repository |

### Efficiency layer

| Key | Default | Meaning |
|---|---|---|
| `routing.mode` | `"advise"` | `off`, `advise` (report only) or `auto` (apply evidence-grade downgrades) |
| `routing.pass_rate_floor`, `routing.min_turns`, `routing.trial_turns` | see `nexus route report` | The evidence bar a cheaper tier must clear |
| `recall.enabled` | `true` | Inject a zero-token card when a prompt resembles a past successful episode |
| `macros.enabled` | `true` | Mine and surface recurring command sequences |
| `pol.enabled` | `true` | Sub-agent output discipline |
| `loop_guard.mode` | `"ask"` | `off`, `advise` or `ask` when an identical call repeats past the threshold |
| `redundant_read_guard.enabled` | `true` | Record a re-read of an unchanged file as waste |

### Remote control

| Key | Default | Meaning |
|---|---|---|
| `remote_control.enabled` | `true` | Inert unless the machine is signed in |
| `remote_control.approval_min_risk` | `"medium"` | Risk floor that sends an otherwise-prompting action to the dashboard |
| `remote_control.wait_s` | `60` | How long to wait for a remote decision before falling back to the local prompt |
| `remote_control.max_grant_ttl_s` | `3600` | Hard ceiling on a remotely granted power, regardless of what is requested |
| `remote_control.pty_host` | `false` | Opt-in: run the agent under a pseudo-terminal so remote prompts can reach an idle session |

### Verification

```json
{
  "verification": {
    "commands": [
      { "kind": "test",      "command": "pytest -q" },
      { "kind": "typecheck", "command": "tsc --noEmit" }
    ]
  }
}
```

Leave it empty to auto-detect from the repository's manifests. `kind` is one of `test`,
`typecheck`, `build`, `lint`, `static_analysis`, `runtime`.

---

## A repository `.nexus` example

```json
{
  "protected_paths": ["deploy/**", "charts/prod/**"],
  "watched_paths": [
    { "path": "**/billing/**", "label": "billing logic", "severity": "HIGH" }
  ],
  "sync": { "mastermind": false },
  "verification": {
    "commands": [{ "kind": "test", "command": "make test" }]
  },
  "disable": []
}
```

This tightens protection, adds a watch, opts the repository out of sharing harvested knowledge
centrally, and declares how the repository is verified. It cannot change the privacy tier or the
enforcement mode; if it tried, those keys would be ignored.

---

## Environment variables

| Variable | Purpose |
|---|---|
| `NEXUS_SETUP_CODE` | Redeem this setup code during setup — a zero-touch fleet install, with no browser and no terminal |
| `NEXUS_NO_SETUP` | Never attempt setup automatically, at install time or on first run |
| `NEXUS_SETUP_DEBUG` | Print why the npm postinstall skipped or failed; it is silent otherwise, because npm swallows postinstall output |
| `NEXUS_SKIP_POLICY` | Bypass the client policy check. Present for our own tests and for incident response; it is not a secret and not a security boundary |
| `NEXUS_GATEWAY_KEY` | Supply a workspace credential from the environment for headless and CI use, instead of a stored file |
| `NEXUS_CAPTURE_ANSWER`, `NEXUS_CAPTURE_ANSWER_RAW`, `NEXUS_CAPTURE_THINKING` | Session-scoped equivalents of the `nexus wrap` capture flags |
| `NEXUS_NO_CAPTURE` | Set on Nexus's own out-of-band calls so they are not captured recursively |

---

Reference: [how it works](how-it-works.md) · [privacy](privacy.md) · [commands](commands.md)
