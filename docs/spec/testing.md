# Testing strategy

gate-warden is a security-boundary repository and is treated as **T2+**: every PR gets a human
review. Its tests must show that the forge-proof guarantee holds, not only that the happy path
works.

## Test layers

| Layer | Scope | Doubles | Tools (Rust) |
| --- | --- | --- | --- |
| Unit | Business logic: eligibility, config rules, planning, interpretation, verdict, redaction, conclusion mapping, `external_id` | None needed (pure) | `cargo test` / cargo-nextest |
| Property | Verdict over all status combinations; redaction never leaks marked content; truncation is deterministic and bounded; `external_id` round-trips | Generators for results and findings | proptest |
| Contract | Both JSON schemas: golden valid and invalid examples; the Rust types agree with the schemas, round-tripped against them | — | JSON Schema validator in tests |
| Adapter | GitHub adapter against recorded or fake HTTP; webhook HMAC; sandbox adapter request building (no secrets, egress policy) | Fake HTTP server | — |
| Use case | Evaluation orchestrator with fake ports: fake GitHub, fake sandbox returning scripted execution records, fake clock | In-memory port fakes | — |
| End-to-end (M1) | A real App on a test organisation, a real sandbox-runner, and the pilot repository (commit-gate) with a real G6 definition | None | Manual or scripted |
| Red-team | The scenarios in [security.md](security.md#red-team-an-agent-cannot-produce-a-green-required-check) | Fakes, plus E2E where rulesets matter | — |
| Determinism | Evaluate the same key twice end to end and compare the output byte for byte | Real sandbox | — |
| Mutation | Business-logic crates | — | cargo-mutants |

Test files follow the Brightsmiths Rust convention: sibling `*_tests.rs` files declared with
`#[cfg(test)] #[path = "..._tests.rs"] mod tests;`; public-API tests go in `tests/`.

## Fake check commands

A small set of discipline-neutral **fake checks** is used by the use-case and E2E tests. They
are test fixtures, never shipped. Each one:

- writes a fixed result (pass, fail, error, holdout finding);
- writes invalid output (missing, oversize, bad JSON, wrong `check_id`, extra fields);
- exits non-zero with `pass`;
- sleeps past its timeout;
- prints marker strings to stdout and stderr;
- attempts to write into the read-only trees;
- (red-team) spawns a background process that writes into a later check's result path.

These let every assertion in [assertions.md](assertions.md) run without the software pack.

## Mapping assertions to layers

| Assertions | Layer |
| --- | --- |
| 1–4 (config) | Use case with fake GitHub and fake harness |
| 5–11 (verdict, interpretation) | Unit plus property |
| 12–15 (feedback) | Unit plus property (no-leak property: a generated marker in holdout or log content never appears in the output) |
| 16–18, 21–23 (idempotency, supersession, restart) | Use case with fake CheckRunStore and Clock |
| 19–20 (eligibility, HMAC) | Adapter plus use case |
| 24–25 (sandbox) | Adapter (request inspection) plus E2E (read-only mounts) |
| 26 (no discipline code) | CI grep test |
| 27 (G9) | Contract (schema) plus unit |
| 28 (audit) | Adapter with recorded ruleset responses |

## Red-team suite

- Every scenario in [security.md](security.md#red-team-an-agent-cannot-produce-a-green-required-check)
  has a named test.
- Scenario 1 (forged check from another source) needs a real GitHub organisation with
  rulesets. It runs as part of the M1 acceptance, and again whenever the ruleset guidance
  changes.
- Scenario 8 (forged JUnit) documents the current residual risk (T-09). It is expected to show
  that the attack is **not** caught at L1. It is tracked, not hidden.

## Coverage and quality targets (proposed)

| Target | Value |
| --- | --- |
| Line coverage, business-logic crates | ≥ 90% |
| Mutation score (cargo-mutants), business-logic crates | ≥ 80%, with no surviving mutants in the verdict, conclusion mapping or redaction |
| Adapter crates | Every port operation has success and failure tests |
| Schemas | Every `required`, `pattern`, `maxLength` and `additionalProperties` rule has a negative example |

## What is not tested here

- The software pack's checks (cargo, clippy, nextest wrappers, agent-gate, oracle-guard). They
  are tested in their own repositories, plus one E2E run with the real G6 definition.
- sandbox-runner's isolation guarantees, which are tested in sandbox-runner. gate-warden's E2E
  only spot-checks the read-only mounts and the absence of network access.
