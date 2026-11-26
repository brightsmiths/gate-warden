# Edge cases and failure modes

Each case gives the expected behaviour and the requirements involved. The governing principle
is **fail safe**: when in doubt, the required check is not `success`.

## Events and SHAs

| Case | Expected behaviour | Refs |
| --- | --- | --- |
| Duplicate webhook delivery | Reuse via `external_id`; no second execution | GW-REQ-071 |
| Events out of order (old SHA after new) | The old SHA is ignored because it is not the current head | GW-REQ-005 |
| Force-push that replaces the head during a run | The new SHA is evaluated. The old run is superseded (`failure`, `superseded`) | GW-REQ-073 |
| Force-push back to a previously evaluated SHA | A completed result is reused if `config_sha` is unchanged. A superseded result is re-executed | GW-REQ-071 |
| PR closed during a run | The run is completed normally. The result stays on the SHA. No special handling | — |
| PR reopened | Evaluated like `opened`. Reuse applies | GW-REQ-001/071 |
| Empty PR (no changed files) | Still evaluated. G6 applies to every PR at L1 | GW-REQ-020 |
| PR targeting a non-default branch | Still evaluated. Config still comes from the **default** branch, not the base branch | GW-REQ-010 |
| Same SHA in two PRs | One set of check runs on the SHA, shared. The key is per SHA, not per PR. Supersession affects a SHA only if it is no longer the head of **any** open PR it belongs to | GW-REQ-070/073 |
| `merge_group` events | Ignored at L1 (A-08) | — |
| Webhook missed during downtime | Recovered by reconciliation | GW-REQ-074 |

## Configuration

| Case | Expected behaviour | Refs |
| --- | --- | --- |
| Pin missing, unparseable, or missing required fields | `failure`/`configuration` | GW-REQ-014 |
| Harness tag does not exist | `failure`/`configuration` | GW-REQ-014 |
| Harness tag moved (does not match the pinned commit) | `failure`/`configuration` | GW-REQ-012 |
| Harness repository not allow-listed | `failure`/`configuration` | [configuration.md](configuration.md) |
| No gate definitions for the pinned pack | `failure`/`configuration`. A repository with no gates must not pass silently | GW-REQ-014 |
| Two definitions for the same gate id | `failure`/`configuration` | [gate-definition-schema.md](interfaces/gate-definition-schema.md) |
| Definition with `gate: G9` | Rejected by the schema, giving `configuration` | GW-REQ-105 |
| Unsupported gate-definition major version | `failure`/`configuration` | [gate-definition-schema.md](interfaces/gate-definition-schema.md#versioning) |
| Default branch changes mid-run | The run stays bound to its `config_sha`. The next evaluation uses the new one | [ADR-0002](../adr/ADR-0002-gate-config-from-default-branch.md) |
| PR changes `.factory/` | Judged under default-branch config. Takes effect after merge | GW-REQ-011 |
| Repository has the App installed but no rulesets | gate-warden still publishes. The audit reports non-compliance | GW-REQ-081 |
| Which check runs carry a configuration failure | gate-warden cannot know the planned gates when the config is unresolvable. It publishes a configuration failure on every gate check run name it published before for this repository (found via the previous head SHA's runs), and always on `gate-warden / G6 Build` at L1. **Open item for L2:** the generalisation (A-33, A-15) | GW-REQ-014 |

## Execution

| Case | Expected behaviour | Refs |
| --- | --- | --- |
| sandbox-runner unavailable | Bounded retries, then `failure`/`infrastructure` | GW-REQ-039 |
| Workspace preparation fails (SHA garbage-collected, history too shallow for the merge base) | Retry, then `infrastructure` | [sandbox-integration.md](interfaces/sandbox-integration.md) |
| Sandbox image cannot be pulled | `infrastructure` | — |
| Booted image digest differs from the requested one | `infrastructure`, logged as a security event | T-14 |
| VM crashes mid-gate | Checks without records become `error`. If no check finished, the run is retried as `infrastructure` | [sandbox-integration.md](interfaces/sandbox-integration.md) |
| A check exceeds its timeout | `error` (timeout); later checks still run if the gate budget allows | GW-REQ-036 |
| Sum of check timeouts exceeds the gate wall clock | Rejected at config validation | [gate-definition-schema.md](interfaces/gate-definition-schema.md) |
| Gate wall clock reached during a check | That check and every check not yet run become `error` (timeout) | GW-REQ-036 |
| Dependency proxy down | Checks report `fail`/`error` according to the pack. The gate fails. gate-warden does not retry check-level failures | — |
| Check writes its result to the wrong path | The result is missing, so `error` | GW-REQ-035 |
| Check writes a result with a different `check_id` | `error` | GW-REQ-035 |
| Check result contains unknown fields | Schema-invalid, so `error` | [gate-result-format.md](interfaces/gate-result-format.md) |
| Result uses absolute paths or `..` in a location | Schema-invalid, so `error` | Same |
| Flaky check (different results on re-execution) | Reuse prevents flip-flopping for the same key. A determinism test flags it | GW-REQ-071, [testing.md](testing.md) |

## Publishing

| Case | Expected behaviour | Refs |
| --- | --- | --- |
| GitHub API 5xx or secondary rate limit | Backoff and retry | [github-integration.md](interfaces/github-integration.md#rate-limits-and-retries) |
| Check run created, but completion fails | Retry. If it still fails, the run stays `in_progress` until reconciliation finds it stale and re-executes | GW-REQ-074 |
| Output exceeds 65 535 characters | Deterministic truncation, with counts recorded | GW-REQ-063 |
| More findings with locations than the annotation cap | The first N in sort order are annotated; the rest appear only in the feedback | [feedback-format.md](interfaces/feedback-format.md) |
| Check run deleted or altered by someone else | Impossible for other Apps, which cannot edit gate-warden's runs. Humans re-request | — |

## Restart and concurrency

| Case | Expected behaviour | Refs |
| --- | --- | --- |
| Restart mid-run | The sandbox is orphaned and sandbox-runner destroys it at its own limits. The check run stays `in_progress` until reconciliation marks it stale and re-executes | GW-REQ-074 |
| Two instances receive the same event | Both may execute. Both get the same result. The second completion overwrites with identical content | [ADR-0006](../adr/ADR-0006-idempotency-key-and-check-run-reuse.md) |
| Concurrency limit reached | The event is queued in memory. The check run is created `in_progress` with "queued" in the summary | GW-REQ-092 |
