# gate-warden specification

gate-warden decides pass or fail for the Brightsmiths factory. It re-runs every gate it owns
from a commit SHA, in a clean sandbox, under its own GitHub App, and publishes the result as a
check run. It is the only App configured as a required-check source, so no agent can produce a
green required check. From L2 it also verifies that human-only transitions were made by
authorised humans.

The system-level architecture lives in the handbook:
[brightsmiths/handbook](https://github.com/brightsmiths/handbook), under `docs/`. Until it is
merged to `main`, it is on the `feat/initial` branch. The pages that matter most here are:

- `docs/architecture/workflow.md`: gates G1–G9 and human touchpoints H1–H7
- `docs/architecture/oracles.md`: anti-faking controls
- `docs/architecture/state-model.md`: GitHub as the system of record
- `docs/architecture/isolation-and-security.md`: microVM isolation and identity
- `docs/adr/ADR-0004-gate-warden-is-the-sole-source-of-verdicts.md`

## The story in one paragraph

A PR is pushed. gate-warden receives the event, checks that the PR is eligible (not a fork,
trusted author), and reads its gate configuration from the **default branch**, never from the
PR. It asks sandbox-runner for a fresh microVM that holds a clean checkout of the SHA and has no
network except the dependency proxy. In that microVM it runs the discipline pack's check
commands, each of which writes a **standard gate result**. It computes a deterministic verdict.
Finally it publishes one check run per gate, carrying structured, coarse feedback that never
reveals test source or holdouts. The same (SHA, gate, harness version) under the same
default-branch config always gives the same result.

## Levels

| Level | Milestone | Content |
| --- | --- | --- |
| **L1** | M1 (human-led) | G6 Build with the Rust software-pack checks; the two discipline-pack interfaces; check runs; feedback; ruleset guidance and audit; idempotency |
| L2 (planned) | M3 | G1, G3, G4; human-transition verification; commit-gate, todo-gate, ratchet; signing; queue-keeper intake; baselines |
| L3 (planned) | M6 | Tier thresholds; holdouts; G5; G8; G9 **inputs** for release-ratifier |

G9 is owned and published by release-ratifier
([ADR-0011](../adr/ADR-0011-g9-owned-by-release-ratifier.md)).

## Documents

Suggested reading order:

| File | Purpose |
| --- | --- |
| [overview.md](overview.md) | System context, scope per level, non-goals |
| [vocabulary.md](vocabulary.md) | Domain concepts and error concepts |
| [requirements.md](requirements.md) | GW-REQ requirements (EARS), with acceptance criteria for L1 |
| [gates.md](gates.md) | Gate catalogue, ownership, check-run naming, the G9 tension |
| [interfaces/gate-result-format.md](interfaces/gate-result-format.md) | **Discipline-pack interface:** standard gate result format v1 |
| [interfaces/gate-definition-schema.md](interfaces/gate-definition-schema.md) | **Discipline-pack interface:** gate definition schema v1 |
| [interfaces/github-integration.md](interfaces/github-integration.md) | App permissions, events, check-run contract |
| [interfaces/sandbox-integration.md](interfaces/sandbox-integration.md) | What gate-warden needs from sandbox-runner |
| [interfaces/feedback-format.md](interfaces/feedback-format.md) | `gate-feedback/v1` and the redaction rules |
| [configuration.md](configuration.md) | Service configuration and per-repository inputs |
| [architecture.md](architecture.md) | Business logic, ports, adapters; main flow |
| [responsibilities.md](responsibilities.md) | CRC cards per component |
| [security.md](security.md) | Threat model and red-team scenarios |
| [assertions.md](assertions.md) | Given/When/Then behavioural assertions |
| [edge-cases.md](edge-cases.md) | Unusual flows and failure modes |
| [testing.md](testing.md) | Test layers, fakes, red-team suite, targets |
| [operations.md](operations.md) | Deployment, App and ruleset setup, observability, runbooks |
| [constraints.md](constraints.md) | Rules for implementers: types, errors, boundaries, performance |
| [tradeoffs.md](tradeoffs.md) | Alternatives considered |
| [assumptions.md](assumptions.md) | Challenged assumptions, open cross-repo items |

Machine-readable schemas:

- [`schemas/gate-result/v1.schema.json`](../../schemas/gate-result/v1.schema.json)
- [`schemas/gate-definition/v1.schema.json`](../../schemas/gate-definition/v1.schema.json)

## Architecture decisions

| ADR | Decision |
| --- | --- |
| [ADR-0001](../adr/ADR-0001-rust-stateless-service.md) | Stateless Rust service; state only in GitHub |
| [ADR-0002](../adr/ADR-0002-gate-config-from-default-branch.md) | Gate config only from the default branch at evaluation time |
| [ADR-0003](../adr/ADR-0003-standard-gate-result-format.md) | Standard gate result format v1 (JSON file) |
| [ADR-0004](../adr/ADR-0004-gate-definition-schema-ownership.md) | gate-warden owns the gate definition schema; the harness owns instances |
| [ADR-0005](../adr/ADR-0005-one-check-run-per-gate.md) | One check run per gate; only safe conclusions |
| [ADR-0006](../adr/ADR-0006-idempotency-key-and-check-run-reuse.md) | Idempotency key in `external_id`; reuse completed runs |
| [ADR-0007](../adr/ADR-0007-checks-run-via-sandbox-runner.md) | Checks run in sandbox-runner microVMs |
| [ADR-0008](../adr/ADR-0008-feedback-in-check-run-output.md) | Feedback in the check-run output, with redaction |
| [ADR-0009](../adr/ADR-0009-event-intake-port.md) | Intake port: webhook at L1, queue-keeper later |
| [ADR-0010](../adr/ADR-0010-ruleset-guidance-and-audit.md) | Ruleset guidance plus read-only audit; no admin permission |
| [ADR-0011](../adr/ADR-0011-g9-owned-by-release-ratifier.md) | G9 owned by release-ratifier; gate-warden supplies inputs only |

ADRs follow the handbook template ([ADR_TEMPLATE.md](../adr/ADR_TEMPLATE.md)).

## Open items

The items that need action outside this repository are listed in
[assumptions.md](assumptions.md):

- A-12: G9 enforcement, to be decided in release-ratifier's spec.
- A-16 and A-28: sandbox-runner's interface and workspace preparation.
- A-27: residual in-VM tampering risk.
- A-29: pin fields and harness layout.
- A-30: pack wrappers.
- Handbook update (part of A-12): `docs/architecture/workflow.md` says every G-gate is run
  by gate-warden, which no longer holds after [ADR-0011](../adr/ADR-0011-g9-owned-by-release-ratifier.md).

Items to be confirmed:

- A-10: `config_sha` in the idempotency encoding.
- A-13: performance targets.
- A-14: baseline writing.

Items deferred to L2:

- A-15: how a required check is concluded when its gate does not apply to a PR.
- A-33: which check runs carry a configuration failure once more gates than G6 exist.

## Workflow for the interface designer

1. **Start from the business logic** in [architecture.md](architecture.md) and the types in
   [vocabulary.md](vocabulary.md). Turn them into Rust types, respecting
   [constraints.md](constraints.md):
   - newtypes;
   - closed enums for `Conclusion`, `CheckStatus` and `OutcomeCategory`;
   - `Result`-based errors.
2. **Define the ports** (EventSource, RepositoryReader, HarnessReader, CheckRunStore, Sandbox,
   SecretProvider, Clock) as traits. Leave L2 and L3 ports for later.
3. **Derive Rust types from the two JSON schemas** and add contract tests against them.
4. **Lay out workspace crates** so the dependency rule is mechanically enforced. Business-logic
   crates must have no adapter dependencies.
5. **Map each L1 assertion** in [assertions.md](assertions.md) to a test location.
6. **Report spec gaps back.** Edit [edge-cases.md](edge-cases.md),
   [assertions.md](assertions.md) or [vocabulary.md](vocabulary.md), or add a
   `clarifications.md`.
