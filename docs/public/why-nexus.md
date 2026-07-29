# Why we built Nexus

Two problems arrived with coding agents at the same time, and most organisations are currently
solving neither.

---

## 1. Cost

### The problem

Agent-assisted delivery converts engineering effort into a variable, per-engineer, per-hour bill that
nobody owns. The reporting that exists is provider-shaped: a workspace total, perhaps a per-seat
split. It answers "how much" and nothing else. It cannot tell you which repository is expensive,
which kind of work is expensive, whether a given spend produced a merged change or a reverted one, or
whether the same money would have bought the same result on a cheaper model.

Worse, a large fraction of the spend is structurally avoidable and invisible. An agent re-reads a
file it already read. It pipes the whole output of a build command into the model's context to find
one error line. It re-derives an approach a colleague solved last week. It runs a planning model on
a mechanical task. None of that shows up as a line item, because none of it is a distinct event to
the provider — it is just more tokens.

### What Nexus does about it

**Attribution first.** Every turn's cost is computed from the agent's own usage record and priced
against the published rate card — uncached input, cache writes, cache reads and output accounted
separately — so the figure is exact rather than a blended estimate. It is attached to a session, a
terminal, a repository, a commit, an engineer, and the prompt that caused it. That makes the ordinary
questions answerable: which repositories cost the most, which engineers are spending on what, what a
feature actually cost to build.

**Then reduction, on four independent fronts:**

- *Context compression in flight.* Request context is compressed before it reaches the model. The
  saving is measured on the wire, not inferred.
- *Tool-output filtering.* Command output is trimmed before the model is charged to read it. This is
  where a surprising amount of waste lives — the model rarely needs the 400 lines a build emits.
- *Context recall.* When a new prompt resembles a past episode that ended well in this repository,
  Nexus injects a zero-token card pointing at the files, the commit and the approach that worked,
  instead of paying the model to rediscover it.
- *Sub-agent output discipline.* Sub-agent responses are the expensive, invisible half of an agentic
  run. Nexus instructs them to answer with maximum signal per token, while the developer's own
  conversation is untouched.

**And then removal of work entirely.** Repeating tasks can be codified into scripts that run with no
agent turn and no tokens. Recurring command sequences are mined from a repository's own history and
promoted into reusable macros, gated on evidence that they would have produced the same outcome.
A stuck agent repeating an identical call is caught and paused rather than left to burn budget. A
file re-read at an unchanged hash is recorded as pure waste so it can be surfaced.

**Finally, model selection that learns.** Nexus classifies each prompt's complexity from content-free
shape signals, records the prediction, and joins it against what actually happened — did the change
hold, did the tests pass, did the developer revert it. From that it derives per-repository rules
about which class of work genuinely succeeds on a cheaper tier. It only ever downgrades, only with
evidence above a configured bar, and never overrides an explicitly chosen model. The default mode is
advisory: it tells you what it would have done and what that would have saved, and you decide.

### Why the number is trustworthy

`nexus savings` reports what you paid, what you would have paid, and the split by mechanism — and
labels each line *measured*, *list price* or *estimate*. Estimates are stated as upper bounds, and
provider-native cache savings are shown separately and explicitly **not** claimed as a Nexus saving.

That discipline is the point. A savings figure that quietly mixes measurement with estimation is
worthless in a finance conversation, and everyone in that conversation knows it.

---

## 2. Visibility, control and governance

### The problem

An agent with write access to a production repository is an actor. It edits authentication code,
adds dependencies, rewrites CI pipelines, touches migrations, and reads whatever files it decides it
needs. In most organisations today, the complete record of that activity is a chat transcript on
someone's laptop.

This fails in three separate directions:

- **Attribution.** When a change is questioned six weeks later, "an agent did it" is not an answer.
  Who instructed it, what were they trying to achieve, what else did that instruction change, and was
  it verified?
- **Prevention.** Reviewing an agent's actions afterwards is the wrong control point for the actions
  that should never have happened. A destructive command, a write to a credentials file, or an
  install from an unrecognised registry needs to be stopped, not reported.
- **Assurance.** Approving agent adoption means being able to state what agents can and cannot do,
  demonstrate it, and produce evidence on request. Policy that exists only in a wiki page is not a
  control.

### What Nexus does about it

**Every change is traceable to an instruction.** The causal link from prompt to tool call to file
change to outcome is captured as structure, not reconstructed from prose. A reviewer can start from
a line of code and walk back to the sentence a human typed, the commit it landed on, and whether the
repository's own tests passed afterwards.

**Policy is enforced before the action, on the machine.** The decision is synchronous and local, so
it cannot be bypassed by a network failure, and it is delivered to the developer as a refusal with a
reason rather than a silent failure. Detection is structural rather than a blocklist of strings —
commands are parsed and evaluated as verb-and-target, so obfuscated forms do not slip through, and a
destructive verb nobody thought to enumerate is still checked against its target.

**Policy can be measured before it is imposed.** `audit` mode records exactly what would have been
blocked while allowing it. An organisation can quantify the disruption of a rule before turning it
on, which is the difference between a policy that is adopted and one that is worked around.

**Sensitive surfaces raise findings without blocking.** Authentication code, payment paths,
migrations, container builds and CI pipelines are watched by default. Touching them is normal work,
so it is not blocked — it is made visible, which is what a reviewer actually wanted.

**Supply chain is treated as a first-class event.** Dependency installs across the common package
managers are recognised, recorded with their package names, and refused when the registry is not on
the allow-list. "An agent added a package from somewhere" is exactly the class of event that should
never be reconstructed from a diff months later.

**The record separates fact from explanation.** Every stored signal is classified as a verifiable
behaviour trace, the model's rationalisation, or the interaction narrative. A model's account of its
own reasoning is genuinely useful to a reviewer — and it is never presented as evidence. Nexus also
records the human's real verdict, inferred from what they did next: a revert, a correction, a re-ask,
or acceptance.

**Divergence from the ask is measured, not assumed.** Nexus snapshots the objective at the start of a
task, the model's restatement of it mid-flight, and the version reflected in its closing summary, and
reports the drift — scope expansion, intent change, a plan that changed substantially, or a concrete
out-of-scope action such as an unrequested dependency install. "Did the agent do what was asked" is a
question with an answer.

**The configuration boundary is a real trust boundary.** A repository-local config file arrives with
the repository and is treated as untrusted. It can tighten controls and add protections; it cannot
disable enforcement, change the privacy tier, remove a protected path, force verbatim capture of a
developer's prompts, or redirect where events are sent. Those decisions belong to the organisation,
and the code enforces that.

**Control extends to a live session, safely.** An authorised teammate can approve a pending action,
send instructions, or halt a running session from the dashboard — without opening an inbound port on
the developer's machine, without a remote approver being able to mint a standing rule for a
destructive command, and without the session hanging or being denied if the control plane is
unreachable.

---

## 3. Why a wrapper, and not a dashboard

Everything above depends on being inside the loop at the moment of action.

A dashboard fed by an API can tell you what an agent did. It cannot stop a command, cannot see the
tool output that never left the developer's machine, cannot know which commit the work was anchored
to, cannot compress the request that is already in flight, and cannot ask the developer to confirm.

The cost of that position is that Nexus must never get in the way. That constraint shaped the whole
implementation: capture fails open and never breaks a session; hooks always exit cleanly no matter
what fails inside them; the local ledger is durable so a network outage costs latency rather than
data; the efficiency layer is skipped rather than blocking when it is not healthy; and the one path
that is deliberately synchronous — the policy decision — is local, dependency-free, and on a
millisecond budget.

---

## 4. What it is not

- It is not a model provider or a proxy you must send your code through to a third party. Captured
  data goes to your workspace and nowhere else.
- It is not a keystroke or screen recorder. At the default privacy tier it retains no prompt text, no
  diff content and no file contents.
- It is not a replacement for code review, CI, or your existing security controls. It is the layer
  that makes agent activity legible to them.

---

Questions, security review, or a pilot: **sales@swfte.com** · **[swfte.com](https://swfte.com)**
