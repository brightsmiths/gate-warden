# Requirements

Requirements use EARS form and are numbered `GW-REQ-NNN`. Each has a level tag:

- **[L1]** is in scope for the M1 build and has acceptance criteria (AC).
- **[L2] (planned)** and **[L3] (planned)** are outlines. They are refined into full
  requirements when that level is specified.

Terms are defined in [vocabulary.md](vocabulary.md). Gaps are recorded in
[assumptions.md](assumptions.md) and are not filled in here.

Numbering leaves gaps on purpose, so related requirements can be added later without
renumbering.

---

## L1: Trigger and intake

### GW-REQ-001 [L1] Evaluate PR head on push

When gate-warden receives a `pull_request` event with action `opened`, `synchronize` or
`reopened` for a repository where its App is installed, gate-warden shall evaluate every
applicable gate for the PR's head SHA.

- **AC1:** a push to a PR branch produces check runs on the new head SHA.
- **AC2:** events with other actions (for example `labeled` or `edited`) start no gate run at L1.

### GW-REQ-002 [L1] Verify webhook authenticity

When gate-warden receives a webhook delivery, gate-warden shall verify its HMAC signature
(`X-Hub-Signature-256`) against the App's webhook secret before parsing the payload any
further.

- **AC1:** a delivery with a missing or invalid signature is rejected with HTTP 401. No GitHub
  call is made, and the rejection is logged without the payload body.

### GW-REQ-003 [L1] Do not evaluate untrusted PRs

If the PR head repository differs from the base repository, or the PR author is not a trusted
actor, then gate-warden shall not run any check. Instead it shall publish each applicable
gate's check run with conclusion `action_required` and outcome category `not-evaluated`.

- **AC1:** a fork PR gets `action_required` check runs and no sandbox request is made.
- **AC2:** `action_required` does not satisfy a required check, so the PR cannot merge.

### GW-REQ-004 [L1] Intake through a port

gate-warden shall receive events through an event-source port. At L1 there shall be an adapter
for the App webhook. The queue-keeper/NATS adapter is GW-REQ-058.

- **AC1:** the business logic compiles and is tested without any webhook or NATS dependency.

### GW-REQ-005 [L1] Ignore stale events

When gate-warden receives an event whose head SHA is no longer the PR's current head, it shall
not start a gate run for that SHA.

- **AC1:** a delayed `synchronize` event for an older SHA produces no sandbox request.

---

## L1: Configuration resolution

### GW-REQ-010 [L1] Resolve config from the default branch

When gate-warden starts an evaluation, it shall read `.factory/pin.yaml` from the head of the
repository's **default branch** at that moment. It shall then load the gate definitions from the
harness at the pinned harness version.

- **AC1:** the commit SHA of the default branch that was used (`config_sha`) is recorded in every
  check run produced by the evaluation.

### GW-REQ-011 [L1] Never read config from the PR

gate-warden shall not read gate configuration (the pin, gate definitions or the harness
reference) from the PR's head, merge ref or branch.

- **AC1:** a PR that changes `.factory/pin.yaml` is evaluated with the default-branch pin. This
  is shown by a test where the PR's pin points to a harness version that does not exist and the
  run still uses the default-branch version.

### GW-REQ-012 [L1] Pin the harness version immutably

When gate-warden resolves the harness version, it shall record both the version tag and the
harness commit SHA it resolved to. If the pin also gives a commit SHA, gate-warden shall verify
that the tag resolves to that commit.

- **AC1:** if the tag and the pinned commit do not match, the result is `ConfigUnresolvable`.

### GW-REQ-013 [L1] Validate gate definitions

gate-warden shall validate every gate definition against the gate definition schema v1
([`schemas/gate-definition/v1.schema.json`](../../schemas/gate-definition/v1.schema.json))
before running any of its checks.

- **AC1:** an invalid definition gives `ConfigUnresolvable`. No sandbox request is made.

### GW-REQ-014 [L1] Fail closed on configuration errors

If the pin is missing or invalid, the harness version cannot be resolved, a gate definition is
invalid, or a definition uses a feature the running level does not support (for example
`thresholds` at L1), then gate-warden shall complete the affected check runs with conclusion
`failure` and outcome category `configuration`. It shall not fall back to any default or to PR
content.

- **AC1:** each of the listed causes, when injected separately, gives `failure`/`configuration`
  with a message that names the cause.

---

## L1: Gate planning

### GW-REQ-020 [L1] Select applicable gates

gate-warden shall decide which gates apply to a PR by evaluating each gate definition's
`applies_to` rules (branch patterns and changed paths) against the PR. These rules come from
the default-branch config snapshot.

- **AC1:** at L1 the software-pack G6 definition applies to every PR, and exactly one check run
  (`gate-warden / G6 Build`) is published per evaluated SHA.

### GW-REQ-021 [L2] (planned) Every changed path is covered

When more than one gate is active, gate-warden shall fail the evaluation with `UncoveredChange`
if any changed path is covered by no applicable gate. Applicability may not be decided by
branch name alone.

---

## L1: Execution

### GW-REQ-030 [L1] Run each gate in a fresh, isolated sandbox

When gate-warden runs a gate, it shall ask sandbox-runner for a **fresh microVM** containing a
clean workspace of the SHA. The microVM shall have:

- a read-only root filesystem;
- a non-root user;
- no secrets;
- no network egress except the dependency proxy;
- the resource limits from the gate definition.

- **AC1:** the sandbox request built for a gate run contains no credentials and an egress policy
  of "dependency proxy only".
- **AC2:** two gate runs never share a microVM.

### GW-REQ-031 [L1] Prepare the workspace outside the sandbox

gate-warden shall make sure the workspace (the PR's commits from the merge base up to the SHA)
is prepared outside the microVM. Repository credentials shall never enter the microVM.

- **AC1:** a test of the sandbox adapter shows that no token or credential appears in the VM
  environment, mounts or arguments.

### GW-REQ-032 [L1] Provide the default-branch tree read-only

gate-warden shall mount the default-branch tree at `config_sha` and the harness at the pinned
version **read-only** in the microVM. It shall give their locations to checks as
`GATE_DEFAULT_BRANCH_DIR` and `GATE_HARNESS_DIR`, so that checks take policy from these trees
and not from the PR.

### GW-REQ-033 [L1] Run checks in declared order

gate-warden shall run a gate's checks one after another, in the order the gate definition
declares. It shall run every check even if an earlier one fails, unless workspace preparation
itself failed.

- **AC1:** a gate with one failing check still reports results for all of its checks.

### GW-REQ-034 [L1] Check result contract

For each check, gate-warden shall give the command a fresh, empty result file path in
`GATE_RESULT_FILE`. It shall then read exactly one gate result from that path after the command
exits ([interfaces/gate-result-format.md](interfaces/gate-result-format.md)).

### GW-REQ-035 [L1] Invalid output is an error

If the result file is missing, larger than the size limit, not valid JSON, invalid against
gate-result v1, or has a `check_id` that differs from the declared check id, then gate-warden
shall record that check's status as `error` (`CheckOutputInvalid`).

### GW-REQ-036 [L1] Enforce timeouts

If a check runs longer than its `timeout_seconds`, gate-warden shall have the check terminated
and record its status as `error` (`CheckTimeout`).

### GW-REQ-037 [L1] Exit code and result agree

If a check command exits non-zero but its result reports `pass`, then gate-warden shall record
the status as `error`.

- **AC1:** a fake check that exits 1 with a `pass` result gives `error` and a failing gate.

### GW-REQ-038 [L1] L1 software-pack checks come from the pack

gate-warden shall run the L1 software-pack (Rust) checks only as commands declared in the
harness G6 gate definition. Those checks are:

- agent-gate `scope-check`;
- oracle-guard `lock --verify` and `inventory`;
- build;
- `cargo fmt --check`;
- clippy;
- tests via cargo-nextest emitting JUnit.

gate-warden shall contain no code specific to any of them.

- **AC1:** a code search of gate-warden's sources finds no reference to cargo, clippy, nextest,
  agent-gate or oracle-guard outside test fixtures and docs.

### GW-REQ-039 [L1] Retry infrastructure failures, then fail

If sandbox-runner or GitHub fails with a transient infrastructure error, gate-warden shall retry
with bounded exponential backoff. If retries are exhausted, it shall complete the check run with
conclusion `failure` and outcome category `infrastructure`.

---

## L1: Verdict

### GW-REQ-040 [L1] Deterministic verdict

gate-warden shall compute a gate's verdict as `pass` if and only if every check in the gate
produced a valid result with status `pass`. Otherwise the verdict shall be `fail`.

- **AC1:** this holds for all combinations of `pass`, `fail`, `error` and missing results
  (property test).

### GW-REQ-041 [L1] No LLM in the verdict

gate-warden shall not use an LLM or any other probabilistic component to compute a verdict.

---

## L1: Publishing

### GW-REQ-050 [L1] One check run per gate

When a gate run starts, gate-warden shall create or update one check run on the SHA. Its name
shall be `gate-warden / <GateId> <name>` and its status `in_progress`. When the gate run ends,
gate-warden shall complete the check run, with per-check detail in its output.

### GW-REQ-051 [L1] Only safe conclusions

gate-warden shall complete check runs only with the conclusions `success`, `failure` or
`action_required`. It shall never use `neutral`, `skipped`, `cancelled`, `stale` or `timed_out`.

- **AC1:** the conclusion type in the business logic has exactly these three variants.

### GW-REQ-052 [L1] Map outcome to conclusion

gate-warden shall map outcomes to conclusions as follows:

| Outcome | Conclusion |
| --- | --- |
| verdict `pass` | `success` |
| verdict `fail` | `failure` |
| `check-error` | `failure` |
| `configuration` | `failure` |
| `infrastructure` | `failure` |
| `superseded` | `failure` |
| `not-evaluated` | `action_required` |

### GW-REQ-053 [L1] Record provenance in the check run

gate-warden shall record the following in each check run's output:

- the SHA;
- the gate id;
- the harness version and harness commit;
- `config_sha`;
- the gate-warden version;
- the sandbox image digest;
- the gate-result format version.

---

## L1: Feedback

### GW-REQ-060 [L1] Structured coarse feedback

gate-warden shall put `gate-feedback/v1` structured feedback in the check run output. For each
non-passing check it shall state what failed, why it matters (the check's `purpose`) and what is
allowed (the check's `allowed_responses`)
([interfaces/feedback-format.md](interfaces/feedback-format.md)).

### GW-REQ-061 [L1] Never expose test source or holdouts

gate-warden shall not publish:

- raw check logs;
- test source;
- holdout content or file names;
- holdout test names.

For findings with `visibility: holdout`, gate-warden shall publish only their REQ-IDs.

- **AC1:** a fake check that emits a holdout finding with a path and message gives feedback
  that contains only the REQ-ID.
- **AC2:** a fake check that writes source-like text to stdout gives feedback that does not
  contain that text.

### GW-REQ-062 [L1] Treat finding text as untrusted

gate-warden shall cap the length of every finding text field and escape it before rendering,
so that check output cannot inject Markdown, HTML, links or mentions into the check run.

### GW-REQ-063 [L1] Bounded output

gate-warden shall keep the check run output within GitHub's limits (65 535 characters of text;
annotations sent in batches of at most 50). It shall truncate findings deterministically and
record how many were dropped.

---

## L1: Idempotency, supersession and restart

### GW-REQ-070 [L1] Idempotent per key

gate-warden shall give the same verdict and the same feedback for the same
(SHA, gate, harness version) evaluated against the same `config_sha`.

### GW-REQ-071 [L1] Reuse completed results

When an evaluation is requested and a completed gate-warden check run with a matching
`external_id` exists on the SHA with outcome category `verdict`, `check-error` or
`configuration`, gate-warden shall reuse it and shall not re-execute the checks.

### GW-REQ-072 [L1] Re-run on request

When gate-warden receives a `check_run` `rerequested` event for one of its check runs, it shall
re-evaluate that gate. GW-REQ-071 still applies, so only runs with category `infrastructure`,
`superseded` or `not-evaluated` actually re-execute.

### GW-REQ-073 [L1] Supersede older SHAs

When a newer head SHA is evaluated for a PR, gate-warden shall stop in-progress gate runs for
older SHAs of that PR. It shall complete their check runs with `failure` and category
`superseded`.

### GW-REQ-074 [L1] Rebuild after restart

When gate-warden starts, and periodically after that, it shall reconcile the open PRs of
installed repositories. Any head SHA that has no completed gate-warden check run, or whose
`in_progress` run is older than the gate's maximum run time, shall be evaluated again. All the
information needed for this comes from GitHub.

### GW-REQ-075 [L1] No other workflow state

gate-warden shall hold no durable workflow state other than GitHub check runs and, from L2,
baseline files. (In-memory caches are allowed if losing them does not change any outcome.)

---

## L1: Rulesets and identity

### GW-REQ-080 [L1] Ruleset guidance

The repository shall provide ruleset setup guidance. The guidance shall make each gate-warden
check a required status check, restricted to the gate-warden App as its source
([operations.md](operations.md#ruleset-setup)).

### GW-REQ-081 [L1] Ruleset audit

gate-warden shall provide a read-only audit command, run by a human with their own credentials.
It shall report, for a repository, whether every expected gate-warden check is required on the
default branch and restricted to the gate-warden App's integration id.

- **AC1:** the audit flags a repository where the required check accepts any source.

### GW-REQ-082 [L1] Least-privilege App

The gate-warden App shall request only these permissions at L1:

- `checks: write`;
- `contents: read`;
- `pull_requests: read`;
- `metadata: read`.

It shall not request `administration`.

### GW-REQ-083 [L1] Secrets via provider

gate-warden shall obtain the App private key and webhook secret through a secret-provider port
(Vault or any KMS). It shall never log them, and they shall never enter a sandbox.

---

## L1: Non-functional (proposed – to be revisited)

### GW-REQ-090 [L1] Intake latency (proposed – to be revisited)

When an eligible event is received, gate-warden shall mark the check run `in_progress` within
30 s at p95, excluding queue wait due to concurrency limits.

### GW-REQ-091 [L1] Overhead (proposed – to be revisited)

gate-warden's own overhead per gate run, excluding sandbox start-up and check execution, shall
be at most 60 s at p95.

### GW-REQ-092 [L1] Concurrency (proposed – to be revisited)

gate-warden shall run at most a configurable number of gate runs at the same time per instance
(default 4) and queue the rest.

---

## Negative requirements ("must not")

| ID | Level | Requirement |
| --- | --- | --- |
| GW-REQ-100 | L1 | gate-warden shall not contain discipline-specific logic (see GW-REQ-038). |
| GW-REQ-101 | L1 | gate-warden shall not read gate config from the PR it judges (see GW-REQ-011). |
| GW-REQ-102 | L1 | gate-warden shall not expose holdout content or test source in feedback (see GW-REQ-061). |
| GW-REQ-103 | L1 | gate-warden shall not hold workflow state beyond check runs and baseline files (see GW-REQ-075). |
| GW-REQ-104 | L1 | gate-warden shall not use an LLM to decide pass/fail (see GW-REQ-041). |
| GW-REQ-105 | L1 | gate-warden shall not run G9 or publish a G9 check run (see GW-REQ-134). |

---

## L2 (planned, M3)

| ID | Outline |
| --- | --- |
| GW-REQ-110 | **G1 Specify** on spec PRs: schema valid, EARS/Gherkin lint, unique REQ-IDs, no unresolved questions, upstream attestations present and verified. Delivered as pack-supplied checks. |
| GW-REQ-111 | **G3 Contracts** on spec PRs: stubs compile, API lint, breaking-change detection. |
| GW-REQ-112 | **G4 Plan** on spec PRs: plan DAG acyclic, slice size budget, 100% REQ coverage. |
| GW-REQ-113 | **Human-only labels:** when `state:design`, the factory trigger label, a tier-lowering change or a hold-clearing change is made, gate-warden verifies that the actor was an authorised human (not a bot or App). Authorisation comes from GitHub team membership, mapped per tier in default-branch configuration. Where the result is published (which check, which gate) is to be specified at L2. |
| GW-REQ-114 | **Spec PR approvals:** gate-warden verifies that approving reviews come from humans authorised for the tier, with two distinct approvers at T3. |
| GW-REQ-115 | **More checks:** commit-gate CI mode, todo-gate and ratchet, added as pack-supplied checks. |
| GW-REQ-116 | **Signing:** gate-warden signs every gate result through chain-chronicler L1. |
| GW-REQ-117 | **Baselines (proposed – to be revisited):** gate-warden is the only writer of `.factory/baseline/`. The write mechanism (commit to the default branch under a ruleset bypass, or a gate-warden-authored PR) is deferred to L2 and needs its own ADR. |
| GW-REQ-118 | **Path coverage:** see GW-REQ-021. |
| GW-REQ-119 | **Permissions:** the App gains the permissions it needs for L2 verification (for example `members: read` for team membership, and the label and review events). Each new permission needs an ADR. |
| GW-REQ-058 | **queue-keeper intake:** gate-warden consumes PR events from queue-keeper's NATS JetStream queues as an alternative event-source adapter, and refuses events that queue-keeper marked as untrusted. |

## L3 (planned, M6)

| ID | Outline |
| --- | --- |
| GW-REQ-130 | **Tier thresholds:** gate definitions may declare per-tier metric thresholds (mutation score, holdout ratio). gate-warden reads the tier from the default-branch `risk-map.yaml` and compares metrics deterministically. |
| GW-REQ-131 | **Holdouts:** gate-warden makes sure `oracle-guard holdout-inject` adds tests from the private holdouts repository to the clean workspace, outside the agent's reach, and runs acceptance holdouts against the built artifact. Feedback stays REQ-ID only. |
| GW-REQ-132 | **G5 Oracles:** red-check, AC trace, manifest signature, holdout ratio. |
| GW-REQ-133 | **G8 Assurance:** blocking reviewer-agent findings according to tier; every dispute has a decision. LLM findings are inputs only. |
| GW-REQ-134 | **G9 inputs:** gate-warden makes the signed results of G1–G8 for a SHA available as evidence that release-ratifier can consume. gate-warden does **not** run G9 or publish a G9 check run. How G9 is enforced is decided in release-ratifier's spec ([ADR-0011](../adr/ADR-0011-g9-owned-by-release-ratifier.md)). |
| GW-REQ-135 | **Fresh seeds:** checks that need a random seed get `GATE_SEED` = SHA-256 of (SHA, gate, harness version, check id). The seed is recorded in the result, which keeps GW-REQ-070 intact. |

## After factory role complete

These are not in the brief's increments and are kept for gate-warden's own roadmap:

- G2 Architecture
- G7 Refactor
- merge-queue support
