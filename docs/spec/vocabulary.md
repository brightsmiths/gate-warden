# Vocabulary

This page defines the domain concepts gate-warden uses. Other spec files use these terms with
exactly these meanings. System-wide terms (tiers, trust zones, H1–H7) are defined in the
handbook ([brightsmiths/handbook](https://github.com/brightsmiths/handbook), `docs/`). Here they
are only restated as far as gate-warden needs them.

Levels: **L1** = M1 (human-led), **L2** = M3, **L3** = M6. A term marked *(L2)* or *(L3)* is
planned and not part of the L1 build.

## Core concepts

### Commit SHA (`Sha`)

- **Description:** the full 40-hex-character Git commit identifier that a gate run judges.
  Normally the head SHA of a pull request.
- **Identifier type:** lowercase hex string, exactly 40 characters (SHA-1 object format).
  SHA-256 object format is not supported at L1.
- **Constraints:** abbreviated SHAs are never accepted. Every verdict is bound to exactly one SHA.
- **Lifespan:** immutable.

### Gate (`GateId`)

- **Description:** a named quality barrier in the Brightsmiths workflow (G1–G9). A gate groups
  one or more checks. It produces one verdict per (SHA, gate).
- **Identifier type:** string matching `^G[1-9]$`.
- **Contents:** id, display name (for example "Build"), ordered list of checks, applicability
  rules, sandbox requirements. All of these come from a gate definition.
- **Constraints:** gate-warden runs G1–G8 only. G9 belongs to release-ratifier
  ([ADR-0011](../adr/ADR-0011-g9-owned-by-release-ratifier.md)). Only G6 is active at L1
  ([gates.md](gates.md)).

### Check (`CheckId`)

- **Description:** one command inside a gate. It examines the workspace and emits one gate
  result.
- **Identifier type:** string matching `^[a-z0-9][a-z0-9._-]{0,63}$`, unique within a gate
  definition. Example: `rust.clippy`.
- **Contents:** id, purpose text, allowed-responses text, argument vector, timeout.
- **Constraints:** gate-warden treats the command as opaque. It has no built-in knowledge of
  any check.

### Check command

- **Description:** the executable and arguments that a discipline pack supplies for a check. It
  runs inside the gate sandbox and writes a gate result to the file named by
  `GATE_RESULT_FILE`.
- **Constraints:** it must not need secrets or network access, other than the dependency proxy.
  Its output (exit code, result file, logs) is **untrusted data**.

### Gate definition

- **Description:** a versioned, declarative document from the harness. It tells gate-warden
  which checks make up a gate, when the gate applies and which sandbox image to use.
- **Format:** YAML or JSON, validated against
  [`schemas/gate-definition/v1.schema.json`](../../schemas/gate-definition/v1.schema.json).
- **Owner:** the schema is owned by this repository. Instances are owned by the harness
  (discipline packs). See [interfaces/gate-definition-schema.md](interfaces/gate-definition-schema.md).
- **Lifespan:** immutable per harness version.

### Harness version (`HarnessVersion`)

- **Description:** the version of the harness that a repository pins in `.factory/pin.yaml` on
  its default branch. It selects which gate definitions apply.
- **Identifier type:** a semver tag (for example `v1.4.0`) plus the commit SHA it resolved to.
  gate-warden records both.
- **Constraints:** if the pin carries a commit SHA, the tag must resolve to it. Otherwise the run
  fails with `ConfigUnresolvable`.

### Pin file

- **Description:** `.factory/pin.yaml` on the repository's default branch. It is human-owned.
  gate-warden reads only the fields listed in [configuration.md](configuration.md). The full pin
  schema is owned by the harness.

### Default-branch config snapshot (`ConfigSnapshot`)

- **Description:** the gate configuration resolved for one evaluation: the pin file, the gate
  definitions at the pinned harness version, and the identity of the default-branch commit they
  were read from.
- **Fields:** `config_sha` (the default-branch commit), `harness_version`, `harness_commit`,
  resolved gate definitions, a digest of the resolved definitions.
- **Constraints:** it is always taken from the default branch HEAD at the time of evaluation, and
  **never** from the PR under evaluation
  ([ADR-0002](../adr/ADR-0002-gate-config-from-default-branch.md)).
- **Lifespan:** one gate run. It is not persisted; its identifiers are recorded in the check run.

### Gate run

- **Description:** one execution of one gate for one SHA under one config snapshot. It consists
  of a fresh microVM, a clean workspace and the gate's checks run in order.
- **Identifier:** the idempotency key plus `config_sha` (see below).
- **Lifespan:** from intake until its check run is completed. No durable state outside the
  GitHub check run.

### Idempotency key

- **Description:** `(SHA, GateId, HarnessVersion)`. The same key with the same default-branch
  config gives the same verdict.
- **Encoding:** carried in the check run `external_id` as
  `gw1:<sha>:<gate>:<harness_version>:<config_sha>`
  ([ADR-0006](../adr/ADR-0006-idempotency-key-and-check-run-reuse.md)).
- **Note:** `config_sha` is part of the encoding because the check commands also read the
  default-branch tree. See [assumptions.md](assumptions.md) (A-10).

### Gate result (`GateResult`)

- **Description:** the standard, discipline-neutral output of one check
  ([interfaces/gate-result-format.md](interfaces/gate-result-format.md)).
- **Fields:** format version, check id, status, optional summary, findings, optional metrics,
  optional tool identity.

### Check status

- **Values:** `pass`, `fail`, `error`. There is no `skip` value. A check that cannot judge
  reports `error`, which fails the gate.

### Finding

- **Description:** one problem reported by a check.
- **Fields:** location (path, line range), rule, severity, coarse message, optional REQ-ID,
  visibility (`visible` or `holdout`).
- **Constraints:** treated as untrusted text. It is size-capped, escaped and redacted before
  publishing ([interfaces/feedback-format.md](interfaces/feedback-format.md)).

### Metric

- **Description:** a named numeric measurement reported by a check, for example the number of
  tests passed. At L1 metrics are reported only. From L3 they can be compared with tier
  thresholds.
- **Fields:** name, value (number), optional unit.

### Verdict

- **Description:** the deterministic pass/fail outcome of a gate run.
- **Values:** `pass` or `fail`.
- **Rule:** a gate passes if and only if every one of its checks produced a valid result with
  status `pass`. Anything else (fail, error, missing result, timeout, infrastructure failure)
  means `fail`. No LLM is involved.

### Outcome category

- **Description:** why a check run ended the way it did. It is shown in the feedback.
- **Values:**
  - `verdict`: all checks produced valid results; the verdict follows from them.
  - `check-error`: at least one check reported `error`, timed out, or produced an invalid or
    missing result.
  - `configuration`: the config snapshot could not be resolved or validated.
  - `infrastructure`: sandbox or GitHub failure after bounded retries.
  - `superseded`: a newer SHA on the same PR replaced this run.
  - `not-evaluated`: the event was not eligible to run (fork or external actor).

### Check run

- **Description:** the GitHub object on a SHA that carries one gate's verdict. It is published
  under the gate-warden App and named `gate-warden / <GateId> <Gate name>`, for example
  `gate-warden / G6 Build`.
- **Allowed conclusions:** `success`, `failure`, `action_required`. `neutral`, `skipped` and
  `cancelled` are never used.
- **Lifespan:** permanent GitHub object. It is the only workflow state gate-warden holds.

### Feedback

- **Description:** the structured, coarse explanation in a check run's output: what failed, why
  it matters, what is allowed. CogWorks and humans read it.
- **Format:** `gate-feedback/v1`, a fenced JSON block plus a human summary
  ([interfaces/feedback-format.md](interfaces/feedback-format.md)).
- **Constraints:** it never contains test source, holdout content or names, or raw logs.
  Holdout failures appear as REQ-IDs only.

### REQ-ID

- **Description:** a requirement identifier from a spec (for example `REQ-042`). gate-warden
  passes it through in findings and feedback. Its syntax is defined by the spec tooling of the
  discipline pack. gate-warden only requires it to be a short token matching
  `^[A-Za-z][A-Za-z0-9_-]{0,63}$`.

### Holdout *(L3)*

- **Description:** a hidden oracle test kept in a private holdouts repository. It is injected at
  gate time by `oracle-guard holdout-inject`.
- **Constraints:** its content, file names and test names never appear in any gate-warden
  output.

### Locked test / manifest

- **Description:** oracle tests written by the oracle identity and hashed into a signed manifest
  (software pack: `tests/locked/MANIFEST`). gate-warden does not interpret manifests. It runs
  pack checks (`oracle-guard lock --verify`, `inventory`) that do.

### Tier *(L2/L3)*

- **Description:** a criticality class T0–T3 per module, assigned by the human-owned
  `risk-map.yaml`. It selects thresholds (L3) and approval requirements (L2). It is not used at
  L1.

### Human-only transition *(L2)*

- **Description:** a GitHub state change that only an authorised human may make: applying
  `state:design`, applying the factory trigger label, lowering a tier, clearing a hold, or
  approving a spec PR.

### Authorised human *(L2)*

- **Description:** a GitHub user who is a member of a team that the default-branch
  configuration maps to the relevant tier and transition. It is never a bot or App account.

### Baseline *(L2/L3, proposed – to be revisited)*

- **Description:** a no-regression reference stored under `.factory/baseline/`. gate-warden is
  the only writer. How it is written is deferred to L2 ([assumptions.md](assumptions.md), A-14).

### Trusted actor

- **Description:** an event originator that may cause a gate run. The PR head must be in the
  same repository (not a fork), and the PR author must have the author association `OWNER`,
  `MEMBER` or `COLLABORATOR`, or be one of the configured factory App bot accounts
  ([configuration.md](configuration.md)).

## Error concepts

| Error | Meaning | Resulting check run | Security note |
| --- | --- | --- | --- |
| `IntakeRejected` | The webhook signature is invalid or the payload is malformed | None. The event is dropped and logged | Never trigger work from unverified payloads |
| `NotEligible` | Fork PR or untrusted actor | `action_required`, category `not-evaluated` | Prevents running untrusted code under gate-warden's authority |
| `ConfigUnresolvable` | Pin missing or invalid, harness version unresolvable, gate definition invalid, or an unsupported feature is present | `failure`, category `configuration` | Fail closed. Never fall back to PR content or defaults |
| `UncoveredChange` *(L2)* | A changed path is covered by no applicable gate | `failure`, category `configuration` | Stops gates being dodged through applicability rules |
| `SandboxFailure` | Provider unavailable, VM crash, or workspace preparation failed | `failure`, category `infrastructure`, after bounded retries | Never treated as a pass |
| `CheckTimeout` | A check exceeded its timeout | The check's status becomes `error`; category `check-error` | Hung checks cannot stall or pass a gate |
| `CheckOutputInvalid` | Result file missing, too large, unparseable, schema-invalid, or its `check_id` does not match | The check's status becomes `error`; category `check-error` | Malformed output never yields a pass |
| `GitHubPublishFailure` | The check run could not be created or updated | Retried. If it still fails, the run is retried on the next event or reconciliation | A missing required check blocks merge (fail-safe) |
| `Superseded` | A newer SHA was pushed to the same PR | `failure`, category `superseded` | Old SHAs never get a late green |
| `UnauthorisedTransition` *(L2)* | A human-only transition was made by a non-authorised actor | `failure` on the relevant gate (planned) | Core of the process-referee role |
