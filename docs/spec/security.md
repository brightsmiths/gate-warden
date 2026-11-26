# Security: threat model and mitigations

gate-warden is a **security-boundary** service. Its central guarantee is:

> **No agent (or anything other than a correct gate-warden evaluation) can produce a green
> required check.**

This page lists the threats to that guarantee and to the confidentiality of oracles, with their
mitigations and the requirements behind them. The system-wide controls are described in the
handbook (`docs/architecture/isolation-and-security.md`, `docs/architecture/oracles.md` in
[brightsmiths/handbook](https://github.com/brightsmiths/handbook)).

## Assets

| ID | Asset | Why it matters |
| --- | --- | --- |
| A1 | Required-check verdicts on SHAs | The merge decision depends on them |
| A2 | gate-warden App private key and webhook secret | Possession allows forging verdicts |
| A3 | Holdout content and names; locked test source | Leaking them lets builders game the oracle |
| A4 | Default-branch gate configuration and harness gate definitions | Changing them changes the rules |
| A5 | Ruleset configuration | Defines which source of checks is trusted |
| A6 | Signed gate results (L2+) | Evidence for G9 and audit |

## Trust boundaries

1. **GitHub → gate-warden:** webhooks are authenticated by HMAC, but their contents are
   untrusted.
2. **gate-warden → sandbox:** everything inside the microVM is untrusted: PR code, check output
   and logs. The sandbox holds no secrets.
3. **Harness and default branch → gate-warden:** trusted. They are human-reviewed control-plane
   content.
4. **gate-warden → GitHub:** check runs are written under the App identity.

## Threats

| ID | Threat (STRIDE) | Mitigation | Requirements / ADRs |
| --- | --- | --- | --- |
| **T-01** | **Spoofing a green status.** An agent or other App posts a passing check or commit status with the gate's name | Rulesets require each gate-warden check **with the source restricted to the gate-warden App's integration id**. The audit detects drift | GW-REQ-080/081, [ADR-0010](../adr/ADR-0010-ruleset-guidance-and-audit.md) |
| **T-02** | **Tampering with config or policy through the PR.** The PR edits `.factory/pin.yaml`, gate definitions, agent-gate policy or trust roots to weaken its own gates | Config is read from default-branch HEAD only. The default-branch tree and the harness are mounted read-only and given to checks as the trusted policy source. Packs must not read policy from `GATE_WORKSPACE` | GW-REQ-010/011/032, [ADR-0002](../adr/ADR-0002-gate-config-from-default-branch.md) |
| **T-03** | **Fail-open conclusions.** `neutral` or `skipped` satisfy required checks | Only `success`, `failure` and `action_required` exist in the domain type | GW-REQ-051, [ADR-0005](../adr/ADR-0005-one-check-run-per-gate.md) |
| **T-04** | **Time-of-check / time-of-use (TOCTOU).** (a) The head moves during the run; (b) the default branch changes during the run | (a) The verdict is bound to the immutable SHA, and older SHAs are superseded. (b) The run is bound to the `config_sha` it started with, which is recorded in `external_id` and output. Re-evaluation happens on the next event | GW-REQ-053/070/073 |
| **T-05** | **Malicious PR code at build or test time.** Escape, exfiltration, network abuse | A fresh hardware-isolated microVM per gate run, with a read-only root, non-root user, no secrets and dependency-proxy-only egress | GW-REQ-030/031, [ADR-0007](../adr/ADR-0007-checks-run-via-sandbox-runner.md) |
| **T-06** | **Leaking oracles through feedback or logs.** Holdout names, test source | Logs are never published. Holdout findings are reduced to REQ-IDs. Text is capped. **Residual:** a pack check that writes test source into a *visible* finding message cannot be detected in general. Packs must keep messages coarse, and pack review covers this | GW-REQ-061/063, [ADR-0008](../adr/ADR-0008-feedback-in-check-run-output.md) |
| **T-07** | **Injection through check output.** Markdown, HTML, links or `@mentions` rendered in check runs; prompt injection aimed at downstream LLM readers | Finding text is treated as data: escaped, capped, control characters stripped, URLs not linked. Feedback is structured JSON, so consumers must not feed raw text into prompts (system rule: content is data) | GW-REQ-062 |
| **T-08** | **Theft of the App key or webhook secret** | SecretProvider (Vault or KMS). Never logged. Never in a sandbox. Least-privilege App with no `administration`. Key rotation runbook | GW-REQ-082/083, [operations.md](operations.md#runbooks) |
| **T-09** | **In-VM tampering between checks.** PR code running in `rust.build` or `rust.test` (`build.rs`, proc macros, tests) modifies files that a later check reads, such as JUnit output for `inventory`, or leaves a background process that writes a later check's `GATE_RESULT_FILE` | Integrity checks that do not execute PR code run first. Each check gets a fresh result path. sandbox-runner kills all of a check's processes before the next check. The default-branch and harness trees are read-only. **Residual:** checks after a code-executing check may read outputs PR code produced. Closing this fully needs separate VMs per phase, or out-of-VM result collection. Recorded in A-27 | GW-REQ-033/034, [sandbox-integration.md](interfaces/sandbox-integration.md) |
| **T-10** | **Spoofed or replayed webhooks** | HMAC verification before parsing. Payloads are used only as pointers; decisions re-read GitHub. Replays are harmless because of idempotency | GW-REQ-002/070 |
| **T-11** | **Fork PRs or external actors** cause gate-warden to run untrusted code | Not evaluated: `action_required`, no sandbox request | GW-REQ-003 |
| **T-12** | **Ruleset drift or misconfiguration** (check not required, any source accepted, bypass list too broad) | Guidance plus a scheduled read-only audit | GW-REQ-080/081 |
| **T-13** | **A dev sandbox adapter used for real verdicts** | Refused unless `dev_mode` is set. Dev-mode check-run names carry a `[dev]` prefix, so they cannot satisfy rulesets | [ADR-0007](../adr/ADR-0007-checks-run-via-sandbox-runner.md), [configuration.md](configuration.md) |
| **T-14** | **Supply chain of the rules** (moving harness tags, swapped sandbox image) | Harness tag resolved to a commit and checked against the pin. Images pinned by digest, and the booted digest is verified. Harness repository limited to an allow-list | GW-REQ-012, [gate-definition-schema.md](interfaces/gate-definition-schema.md) |
| **T-15** | **Denial of service** (event floods, huge PRs, endless checks) | Bounded concurrency. Supersession of old SHAs. Per-check and per-gate timeouts. Result size caps. A blocked gate is fail-safe (the PR cannot merge) | GW-REQ-036/063/073/092 |
| **T-16** *(L2)* | **Human-only transitions made by bots or unauthorised users** | Actor verification against team membership, mapped per tier on the default branch. Bots and Apps are never authorised humans | GW-REQ-113/114 |
| **T-17** | **Reuse of stale results** after the rules changed | `config_sha` is in `external_id`, so a new default-branch commit means new evaluations | [ADR-0006](../adr/ADR-0006-idempotency-key-and-check-run-reuse.md) |
| **T-18** | **A second required-check source for G9** | See the next section | GW-REQ-105/134, [ADR-0011](../adr/ADR-0011-g9-owned-by-release-ratifier.md) |

## G9 and the forge-proof property

G9 (Release) is owned **and published** by release-ratifier, not gate-warden
([ADR-0011](../adr/ADR-0011-g9-owned-by-release-ratifier.md)). How G9 is enforced is decided in
**release-ratifier's spec**:

- **Required check from a second App:** rulesets on the release path then trust two App sources.
  That widens the attack surface of the guarantee at the top of this page. A compromised or
  misconfigured release-ratifier App could produce a green required check, and audits must
  cover two identities.
- **release-ratifier refuses to tag or publish:** gate-warden stays the only required-check
  source. G9 is enforced in release-ratifier's own release path instead of by GitHub's merge
  protection.

gate-warden's ruleset guidance and audit cover only gate-warden's gates. gate-warden never
publishes a check run named for G9 (GW-REQ-105). This is an open cross-repo item
([assumptions.md](assumptions.md), A-12).

## Red-team: "an agent cannot produce a green required check"

The scenarios below are written as tests ([testing.md](testing.md#red-team-suite)). Each must
end with the PR **not** mergeable, or the attack not reaching a pass.

1. Post a commit status or check run named `gate-warden / G6 Build` from another App or token.
   The ruleset rejects it, because the source is restricted.
2. A PR edits `.factory/pin.yaml` to point at a permissive harness. The default-branch pin is
   used.
3. A PR edits agent-gate policy or oracle trust roots in the workspace. Checks read them from
   `GATE_DEFAULT_BRANCH_DIR` (pack conformance test).
4. A check emits `pass` with a non-zero exit code. The result is `error` and the gate fails.
5. A check emits no result file, or invalid JSON, or the wrong `check_id`. The result is
   `error`.
6. A check hangs. It times out to `error`.
7. `build.rs` writes a forged `pass` into the next check's result path. The path is fresh per
   check and the process is killed between checks. The test asserts that the forged file is not
   read.
8. PR code forges JUnit output to fool `inventory`. This is a residual (T-09). The test
   documents the current behaviour and tracks A-27.
9. A fork PR. The run is `action_required` and no sandbox request is made.
10. Gate-warden infrastructure is down. No check run is posted, so the required check is
    missing and the PR cannot merge.
11. Send a forged webhook. It is rejected with 401.
12. Delayed results for an old SHA after a new push. The old run is superseded and the new SHA
    is evaluated separately.
13. A gate definition containing `thresholds` before L3. The result is `configuration` failure,
    not a silently ignored threshold.
14. Finding text containing Markdown, links or `@mentions`. It is rendered inert.
15. A holdout finding with a path and message. Only the REQ-ID appears.
