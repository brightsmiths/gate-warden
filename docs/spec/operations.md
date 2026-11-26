# Operations

## Deployment

- **Form:** one stateless binary or container image. It runs anywhere: cloud or on-prem, with no
  cloud-specific dependency.
- **Inputs at start-up:** service configuration ([configuration.md](configuration.md)), and the
  secrets that the SecretProvider resolves.
- **Network needs:**
  - inbound HTTPS from GitHub (webhook mode), or NATS (L2);
  - outbound to the GitHub API, the harness repository and sandbox-runner.
- **KVM:** none needed. Isolation belongs to sandbox-runner.
- **Instances:** one is enough at M1. Several instances are safe (idempotency), with duplicate
  execution as the worst case.
- **Upgrades:** rolling. In-flight runs on a stopped instance are recovered by reconciliation.

## GitHub App setup (L1)

1. Create the **gate-warden** App with the permissions in
   [github-integration.md](interfaces/github-integration.md#app-permissions):
   - `checks: write`;
   - `contents: read`;
   - `pull_requests: read`;
   - `metadata: read`.
2. Subscribe to `pull_request`, `check_run` and `check_suite`.
3. Set the webhook URL and secret. Store the secret and the private key in the secret store.
4. Install the App on the pilot repository (commit-gate), and on the harness repository if it
   is private.
5. Note the App id. It is the integration id used in rulesets.

## Ruleset setup

These are the guidance (GW-REQ-080) and the audit (GW-REQ-081), following
[ADR-0010](../adr/ADR-0010-ruleset-guidance-and-audit.md).

On each repository judged by gate-warden, a repository admin creates or updates a branch
ruleset that targets the **default branch**:

1. **Require status checks to pass.** Add each active gate-warden check, at L1
   `gate-warden / G6 Build`, and **set its source to the gate-warden App**. Do not use "any
   source".
2. **Require a pull request before merging**, so that every change to the default branch goes
   through gate-warden.
3. **Restrict bypass** to the smallest possible set of humans. No factory role App and no agent
   identity may be on the bypass list. gate-warden itself is not on it either (until the L2
   baseline decision, A-14).
4. Keep `.factory/`, `risk-map.yaml` and `AGENTS.md`/`CLAUDE.md` human-owned. Use push rulesets
   restricting file paths where the GitHub plan supports them, as the handbook's enforcement
   layer 4.
5. Do **not** add any other App as a required-check source. See
   [gates.md](gates.md#g9-and-the-single-required-check-source-tension) for the G9 open item.

### Audit

Run `gate-warden audit-rulesets --repo <owner/name>` (CLI shape to be fixed by interface
design). It uses the operator's own token and is read-only. It reports:

- every expected gate-warden check (from the default-branch config) that is not required on the
  default branch;
- every required gate-warden check whose source is not restricted to the App's integration id;
- bypass actors on the ruleset, for human review.

Recommended: run it on a schedule (for example daily) across all installed repositories, and
after every ruleset change.

## Observability

- **Traces:** OpenTelemetry, with one span per evaluation and child spans for config
  resolution, the sandbox run, each check and publishing. Attributes: repo, SHA, gate,
  `external_id`, outcome category. Never finding text or logs.
- **Metrics:**
  - gate runs by outcome category;
  - intake-to-`in_progress` latency (GW-REQ-090);
  - gate-warden overhead (GW-REQ-091);
  - queue depth;
  - retries;
  - reused versus executed runs;
  - superseded runs;
  - reconciliation findings.
- **Logs:**
  - structured;
  - no secrets;
  - no check output beyond sizes and statuses;
  - webhook rejections without bodies.
- **Alerts:**
  - sustained `infrastructure` outcomes;
  - image digest mismatches (security event);
  - HMAC rejection spikes;
  - reconciliation repeatedly finding missed SHAs;
  - audit non-compliance.
- **From L2:** telemetry is also emitted through chain-chronicler's pipeline. It is evidence,
  not state.

## Scaling

- The cost is dominated by sandbox execution, so scale sandbox-runner capacity first.
- gate-warden throughput is bounded by `limits.max_concurrent_gate_runs` per instance
  (default 4, proposed – to be revisited) and by GitHub API rate limits.
- Supersession keeps busy PRs from piling up runs.

## Runbooks

| Situation | Action |
| --- | --- |
| Re-run a gate | Use "Re-run" on the check in the GitHub UI. This sends `check_run.rerequested`. Completed verdict results are reused (same key); infrastructure, superseded and not-evaluated results re-execute |
| Force a fresh evaluation after a pack fix | Merge the harness pin change to the default branch. The new `config_sha` means new evaluations on the next push or re-request |
| Rotate the App private key | Generate a new key, add it to the secret store, restart or reload, then revoke the old key. Verdicts are unaffected |
| Rotate the webhook secret | Update the App and the secret store together. Deliveries rejected in the window are recovered by reconciliation |
| Stuck `in_progress` check | Wait for reconciliation (`reconcile.interval`), or re-request |
| Suspected forged green | Run the audit. Inspect the check source in the PR's checks API. Treat as a security incident |
| sandbox-runner outage | PRs show `failure`/`infrastructure` and cannot merge (fail safe). Re-request after recovery |
