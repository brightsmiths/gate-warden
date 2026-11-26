# Interface: GitHub integration

gate-warden acts on GitHub only as **its own GitHub App**. This page fixes:

- the App's permissions;
- the events it subscribes to;
- how gate-warden reads repository state;
- the check-run contract.

## App permissions

| Permission | L1 | L2 (planned) | Why |
| --- | --- | --- | --- |
| `checks` | write | write | Publish gate verdicts |
| `contents` | read | read (write only if the L2 baseline decision needs it) | Default-branch pin and tree; PR commits for the workspace |
| `pull_requests` | read | read | PR head and base, author association, changed files, reviews (L2) |
| `metadata` | read | read | Mandatory |
| `members` (organisation) | — | read | Team membership for authorised humans (GW-REQ-113/114) |
| `issues` | — | read | Label events on issues (GW-REQ-113) |
| `administration` | **never** | **never** | [ADR-0010](../../adr/ADR-0010-ruleset-guidance-and-audit.md) |

Every permission added at L2 or later needs an ADR (GW-REQ-119).

Harness access:

- If the harness repository is public, it is read anonymously or with the installation token.
- If it is private, it is read through a separate installation of the gate-warden App with
  `contents: read`.

## Subscribed events

| Event | Actions | Level | Handling |
| --- | --- | --- | --- |
| `pull_request` | `opened`, `synchronize`, `reopened` | L1 | Evaluate the head SHA (GW-REQ-001) |
| `check_run` | `rerequested` (gate-warden's own runs only) | L1 | Re-evaluate the gate (GW-REQ-072) |
| `check_suite` | `rerequested` | L1 | Re-evaluate all gates on the SHA |
| `pull_request` | `labeled`, `unlabeled` | L2 | Transition verification |
| `issues` | `labeled`, `unlabeled` | L2 | Transition verification |
| `pull_request_review` | `submitted`, `dismissed` | L2 | Approval verification |

All other events are ignored. `merge_group` is not handled at L1.

Rules for intake:

- Webhook payloads are untrusted. gate-warden uses them only to decide *what to look at*: repo,
  PR number, SHA. All decision inputs are re-read from the GitHub API with the installation
  token. That covers head and base repos, author association, the current head SHA and changed
  files.
- Event fields are never interpolated into anything that is executed or rendered unescaped.

## Trusted actor determination

A PR is eligible (GW-REQ-003) when both of these hold:

1. `head.repo.full_name == base.repo.full_name` (not a fork).
2. The PR's `author_association` is `OWNER`, `MEMBER` or `COLLABORATOR`, **or** the PR author's
   login is in the configured list of factory App bot accounts
   ([configuration.md](../configuration.md)).

## Reading the default-branch configuration

1. Get the repository's `default_branch` and the commit at its head. That commit is
   `config_sha`.
2. Read `.factory/pin.yaml` at `config_sha`.
3. Resolve the harness version and read the gate definitions (HarnessReader).
4. Compute the merge base of the head SHA and `config_sha`. It becomes `GATE_BASE_SHA`.

Nothing is read from `refs/pull/<n>/head` or `refs/pull/<n>/merge` for configuration
(GW-REQ-011).

## Check-run contract

| Aspect | Value |
| --- | --- |
| `name` | `gate-warden / <GateId> <name>`, e.g. `gate-warden / G6 Build` |
| `head_sha` | The evaluated SHA |
| `external_id` | `gw1:<sha>:<gate>:<harness_version>:<config_sha>` ([ADR-0006](../../adr/ADR-0006-idempotency-key-and-check-run-reuse.md)) |
| `status` lifecycle | Created `in_progress` when execution starts, then `completed`. There is no separate `queued` phase. A run that waits for capacity is created `in_progress`, and its summary says it is queued |
| `conclusion` | `success`, `failure` or `action_required` only ([ADR-0005](../../adr/ADR-0005-one-check-run-per-gate.md)) |
| `details_url` | Optional; points to operator documentation. It never points to logs |
| `output.title` | `<GateId> <name>: <passed / failed / not evaluated> (<outcome category>)` |
| `output.summary` | A short human summary: counts per status, plus the provenance lines from GW-REQ-053 |
| `output.text` | The `gate-feedback/v1` fenced JSON block ([feedback-format.md](feedback-format.md)) |
| `output.annotations` | Up to the configured cap (default 50, sent in batches of 50 per request) of redacted, visible findings that have a location |

Rules for writing check runs:

- **Limits:** `output.summary` and `output.text` are each at most 65 535 characters. Truncation
  is deterministic (GW-REQ-063).
- **Updates:** completing a run is a PATCH on the same check run id. If the id was lost, for
  example after a restart, gate-warden finds the run by name and `external_id`.
- **Superseded runs (GW-REQ-073):** when a new head SHA is evaluated, gate-warden lists its own
  `in_progress` runs on the PR's previous head SHAs, cancels their sandbox work if it is local,
  and completes them with `failure` and category `superseded`.

## Rate limits and retries

- Use conditional requests and cache **within one evaluation** only.
- Treat these as transient and retry them with exponential backoff and jitter, up to a bounded
  total (default 5 attempts or 10 minutes):
  - HTTP 5xx;
  - secondary rate limits;
  - network errors.
- When the attempts are used up:
  - if the failure happened before a check run existed, gate-warden relies on reconciliation
    (GW-REQ-074);
  - otherwise it completes the run as `failure` with category `infrastructure` (GW-REQ-039).
- When the primary rate limit is exhausted, pause until the reset time. Never complete a run as
  a pass.

## Required-check configuration

See [operations.md](../operations.md#ruleset-setup) for the guidance and the audit command
(GW-REQ-080/081).
