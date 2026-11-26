# Behavioural assertions

These are testable Given/When/Then statements. Each one names the requirements it covers
([requirements.md](requirements.md)). L1 assertions are binding for the M1 build. L2/L3
assertions are outlines (planned).

## L1

### 1. Config comes from the default branch

- **Given** a PR whose head changes `.factory/pin.yaml` to harness version `v9.9.9`
- **And** the default branch pins `v1.2.0`
- **When** gate-warden evaluates the PR head
- **Then** the gate definitions of `v1.2.0` are used
- **And** the check-run output records `harness_version: v1.2.0` and the default-branch
  `config_sha`.
- *Covers:* GW-REQ-010, GW-REQ-011, GW-REQ-053

### 2. Missing pin fails closed

- **Given** a repository whose default branch has no `.factory/pin.yaml`
- **When** a PR is evaluated
- **Then** the check run completes with conclusion `failure` and category `configuration`
- **And** no sandbox request is made.
- *Covers:* GW-REQ-014

### 3. Harness tag and commit mismatch

- **Given** a pin with `harness.version: v1.2.0` and a `harness.commit` that `v1.2.0` does not
  resolve to
- **When** a PR is evaluated
- **Then** the outcome is `failure`/`configuration` with a reason that names the mismatch.
- *Covers:* GW-REQ-012

### 4. Unsupported feature is rejected, not ignored

- **Given** a G6 definition that contains `thresholds`
- **When** gate-warden at L1 evaluates a PR
- **Then** the outcome is `failure`/`configuration`.
- *Covers:* GW-REQ-014

### 5. All checks pass, so the gate passes

- **Given** a G6 definition with 7 checks
- **And** every check exits 0 and writes a valid result with `status: pass`
- **When** the gate run completes
- **Then** the check run `gate-warden / G6 Build` completes with conclusion `success`.
- *Covers:* GW-REQ-040, GW-REQ-050

### 6. Any non-pass fails the gate

- **Given** any assignment of {pass, fail, error, missing result, timeout} to a gate's checks
  where at least one is not a valid pass
- **When** the verdict is computed
- **Then** the verdict is `fail` and the conclusion is `failure`.
- *Covers:* GW-REQ-040 (property test)

### 7. Exit code and result disagree

- **Given** a check that exits with code 1 and writes `status: pass`
- **When** its result is interpreted
- **Then** its status is `error` with `error_kind: exit-mismatch`.
- *Covers:* GW-REQ-037

### 8. Invalid output is an error

- **Given** a check whose result file is one of:
  - missing;
  - larger than 1 MiB;
  - not valid JSON;
  - schema-invalid;
  - carrying a `check_id` different from the declared one
- **When** its result is interpreted
- **Then** its status is `error` with `error_kind: invalid-output`.
- *Covers:* GW-REQ-035

### 9. Timeout is an error

- **Given** a check with `timeout_seconds: 10` that runs for longer
- **When** the timeout expires
- **Then** the check is terminated and its status is `error` with `error_kind: timeout`
- **And** the remaining checks still run.
- *Covers:* GW-REQ-033, GW-REQ-036

### 10. Every check runs

- **Given** a gate whose second check fails
- **When** the gate runs
- **Then** results for all declared checks appear in the feedback, in declared order.
- *Covers:* GW-REQ-033

### 11. Never a fail-open conclusion

- **Given** any evaluation outcome
- **When** a check run is completed
- **Then** its conclusion is one of `success`, `failure`, `action_required`.
- *Covers:* GW-REQ-051, GW-REQ-052

### 12. Holdouts appear as REQ-IDs only

- **Given** a check result with a finding
  `{visibility: holdout, req_id: REQ-021, location.path: tests/holdout_x.rs, message: "assert_eq!(...)"}`
- **When** feedback is produced
- **Then** `holdout_failures` contains `REQ-021`
- **And** the strings `tests/holdout_x.rs` and `assert_eq!` appear nowhere in the check-run
  output or annotations.
- *Covers:* GW-REQ-061

### 13. Logs are never published

- **Given** a check that prints a unique marker string to stdout and stderr
- **When** the check run is completed
- **Then** the marker appears nowhere in the check-run output or annotations.
- *Covers:* GW-REQ-061

### 14. Finding text is inert

- **Given** a finding message containing `<img src=x>`, `[link](http://x)` and `@someone`
- **When** it is rendered
- **Then** the rendered output contains no active HTML, link or mention.
- *Covers:* GW-REQ-062

### 15. Output stays within limits deterministically

- **Given** a check result with 1000 findings
- **When** feedback is produced twice from the same inputs
- **Then** both outputs are byte-identical
- **And** `output.text` is at most 65 535 characters
- **And** `truncated.findings_dropped` equals the number of findings omitted.
- *Covers:* GW-REQ-063, GW-REQ-070

### 16. Same key, same result

- **Given** a completed `gate-warden / G6 Build` run on SHA `S` with `external_id`
  `gw1:S:G6:v1.2.0:C` and category `verdict`
- **When** a duplicate `synchronize` event for `S` arrives and the default branch is still at `C`
- **Then** no sandbox request is made and the existing check run is unchanged.
- *Covers:* GW-REQ-070, GW-REQ-071

### 17. New default-branch config means a new evaluation

- **Given** a completed run for SHA `S` under `config_sha` `C1`
- **When** the gate is re-requested after the default branch moved to `C2`
- **Then** a new evaluation runs under `C2`
- **And** its `external_id` ends with `:C2`.
- *Covers:* GW-REQ-071, GW-REQ-072

### 18. Infrastructure failures are retried, then fail

- **Given** sandbox-runner is unavailable for longer than the retry budget
- **When** a gate run is attempted
- **Then** the check run completes with `failure` and category `infrastructure`
- **And** a later `rerequested` event re-executes it.
- *Covers:* GW-REQ-039, GW-REQ-072

### 19. Fork PRs are not evaluated

- **Given** a PR whose head repository is a fork
- **When** an `opened` event arrives
- **Then** `gate-warden / G6 Build` completes with `action_required`, category `not-evaluated`
- **And** no sandbox request is made.
- *Covers:* GW-REQ-003

### 20. Unsigned webhook is rejected

- **Given** a delivery with an invalid `X-Hub-Signature-256`
- **When** it is received
- **Then** the response is 401, no GitHub API call is made, and no check run is created.
- *Covers:* GW-REQ-002

### 21. Older SHAs are superseded

- **Given** an in-progress G6 run for SHA `S1` on PR #7
- **When** SHA `S2` is pushed to PR #7 and evaluated
- **Then** the `S1` run completes with `failure`, category `superseded`
- **And** its sandbox work is cancelled.
- *Covers:* GW-REQ-073

### 22. Stale events are ignored

- **Given** PR #7's current head is `S2`
- **When** a delayed event for `S1` arrives
- **Then** no gate run starts for `S1`.
- *Covers:* GW-REQ-005

### 23. Recovery after restart

- **Given** gate-warden was down while PR #8 received SHA `S3`
- **When** gate-warden starts
- **Then** reconciliation evaluates `S3` without any new event.
- *Covers:* GW-REQ-074

### 24. No secrets in the sandbox

- **Given** any gate run
- **When** the sandbox request is built
- **Then** it contains no token, private key or webhook secret in its environment, mounts or
  arguments
- **And** its egress policy allows only the dependency proxy.
- *Covers:* GW-REQ-030, GW-REQ-031, GW-REQ-083

### 25. Trusted trees are read-only

- **Given** a gate run
- **When** a check tries to write under `GATE_DEFAULT_BRANCH_DIR` or `GATE_HARNESS_DIR`
- **Then** the write fails.
- *Covers:* GW-REQ-032

### 26. No discipline-specific code

- **Given** the gate-warden source tree, excluding tests, fixtures and docs
- **When** it is searched for `cargo`, `clippy`, `nextest`, `agent-gate`, `oracle-guard`
- **Then** there are no matches.
- *Covers:* GW-REQ-038, GW-REQ-100

### 27. G9 is never published

- **Given** a harness gate definition with `gate: G9`
- **When** it is validated
- **Then** validation fails with `ConfigUnresolvable`
- **And** gate-warden never creates a check run whose name contains `G9`.
- *Covers:* GW-REQ-105

### 28. Ruleset audit detects drift

- **Given** a repository whose ruleset requires `gate-warden / G6 Build` from any source
- **When** the audit runs
- **Then** it reports the repository as non-compliant, naming the missing source restriction.
- *Covers:* GW-REQ-081

## L2 (planned)

- **29. Unauthorised `state:design` label:** given `state:design` applied by a bot or by a user
  outside the authorised team, gate-warden reports `UnauthorisedTransition` as a failing verdict
  (GW-REQ-113).
- **30. T3 needs two approvers:** given a T3 spec PR with one authorised approval, the approval
  verification fails; with two distinct authorised approvers, it passes (GW-REQ-114).
- **31. Uncovered change:** given G1 applies to `docs/spec/**` and G6 to `src/**`, a PR that
  changes `build.rs` alone fails with `UncoveredChange` (GW-REQ-021).
- **32. Results are signed:** every completed gate result has a chain-chronicler signature that
  verifies (GW-REQ-116).

## L3 (planned)

- **33. Threshold breach:** given T2 requires mutation score ≥ 0.8 and the check reports 0.75,
  the verdict is `fail` (GW-REQ-130).
- **34. Holdout injection is invisible:** holdout files are present in the gate workspace and
  absent from all published output (GW-REQ-131).
- **35. Deterministic seed:** the same (SHA, gate, harness version, check) gets the same
  `GATE_SEED` (GW-REQ-135).
- **36. G9 inputs:** for a SHA with completed G1–G8 runs, signed results are available to
  release-ratifier, and no G9 check run exists from gate-warden (GW-REQ-134).
