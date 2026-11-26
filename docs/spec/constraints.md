# Implementation constraints

These rules bind the interface designer and the implementers. Technology choices are recorded
in [`.tech-decisions.yml`](../../.tech-decisions.yml).

## Type system (Rust)

- **Newtypes for identifiers:** `Sha` (validated 40-hex), `GateId` (`G1`–`G8`), `CheckId`,
  `HarnessVersion`, `ConfigSha`, `ReqId`, `ExternalId`. Raw `String`s do not cross module
  boundaries for these concepts. Constructors validate.
- **Make invalid states unrepresentable:**
  - `Conclusion` has exactly `Success`, `Failure` and `ActionRequired` (GW-REQ-051).
  - `CheckStatus` has exactly `Pass`, `Fail` and `Error`. There is no skip.
  - `OutcomeCategory` is a closed enum.
  - `GateId` cannot be `G9`.
- **Untrusted input:** results, webhook payloads and check-run outputs read back from GitHub are
  parsed into validated types at the boundary. Business logic never sees unvalidated data.
- **No `unsafe`** in gate-warden crates.
- **No panics on untrusted input.** No `unwrap` or `expect` on values derived from external
  data. Lints enforce this (`clippy::unwrap_used`, `clippy::expect_used` in non-test code).

## Error handling

- Expected errors are values: `Result<T, E>`, with typed error enums per boundary (see the error
  concepts in [vocabulary.md](vocabulary.md#error-concepts)).
- Panics are reserved for programmer errors. A panic in an evaluation must not produce a
  `success`. The evaluation is abandoned and recovered by reconciliation.
- Every error path that ends an evaluation maps to an outcome category and a conclusion through
  one function (GW-REQ-052).

## Module boundaries

- Business-logic crates depend on no adapter crate, and on no HTTP, GitHub, NATS, sandbox or
  secret-store library ([architecture.md](architecture.md#dependency-rule)).
- The rule is enforced in CI with workspace crate boundaries and `cargo-deny` (bans), or an
  equivalent dependency-rule check.
- No discipline-specific identifiers in non-test sources (GW-REQ-038, assertion 26).

## Determinism

- The verdict, feedback and truncation are pure functions of their inputs (GW-REQ-070).
- Sort with a total order. Iterate over ordered maps (`BTreeMap`), never over hash-map order,
  for anything that is published.
- Time affects only staleness and timeouts, never verdict content. Timestamps are not part of
  the feedback JSON.

## No LLM

- No LLM client, SDK or model call anywhere in gate-warden (GW-REQ-041/104). A CI dependency ban
  enforces this as far as practical.

## Security rules

- Secrets only via the SecretProvider. They are never logged, never placed in a sandbox request,
  and never echoed in errors (GW-REQ-083).
- Webhook HMAC is verified with a constant-time comparison before parsing (GW-REQ-002).
- All rendered check-output text is escaped (GW-REQ-062).
- The dev sandbox adapter is unreachable unless `dev_mode` is set
  ([ADR-0007](../adr/ADR-0007-checks-run-via-sandbox-runner.md)).
- The App's permissions are the least needed. Any change needs an ADR (GW-REQ-082/119).

## Testing requirements

See [testing.md](testing.md). Proposed minimums:

- 90% line coverage and 80% mutation score on business-logic crates;
- no surviving mutants in the verdict, conclusion mapping or redaction;
- every assertion in [assertions.md](assertions.md) has at least one test;
- the red-team suite is part of CI where it can run without real GitHub.

## Performance targets (proposed – to be revisited)

| Target | Value | Ref |
| --- | --- | --- |
| Eligible event → check run `in_progress` | p95 ≤ 30 s, excluding concurrency queueing | GW-REQ-090 |
| gate-warden overhead per gate run (excluding sandbox start and check execution) | p95 ≤ 60 s | GW-REQ-091 |
| Concurrent gate runs per instance | Configurable; default 4 | GW-REQ-092 |
| Memory per in-flight gate run (gate-warden side) | Bounded by `max_result_bytes` × number of checks, plus a constant | — |

These numbers are defaults chosen without load data. They are revisited after M1 with
measurements from the pilot repository ([assumptions.md](assumptions.md), A-13).

## Baselines (proposed – to be revisited)

gate-warden does not write `.factory/baseline/` at L1. The write mechanism is decided at L2 with
its own ADR (GW-REQ-117, [assumptions.md](assumptions.md), A-14).
