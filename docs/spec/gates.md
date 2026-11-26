# Gate catalogue

The handbook's workflow (`docs/architecture/workflow.md` in
[brightsmiths/handbook](https://github.com/brightsmiths/handbook)) defines gates G1–G9. This page
says which of them gate-warden runs, at which level, and how they map to check runs.

gate-warden knows gates only as **ids, names and ordered lists of opaque checks** taken from gate
definitions. The check lists below are the **software pack's** examples. They live in the harness
and are not part of gate-warden.

## Ownership and levels

| Gate | Name | Owner | Level | Notes |
| --- | --- | --- | --- | --- |
| G1 | Specify | gate-warden | L2 (planned) | GW-REQ-110 |
| G2 | Architecture | gate-warden | After factory role complete | Not in the brief's increments ([assumptions.md](assumptions.md), A-11) |
| G3 | Contracts | gate-warden | L2 (planned) | GW-REQ-111 |
| G4 | Plan | gate-warden | L2 (planned) | GW-REQ-112 |
| G5 | Oracles | gate-warden | L3 (planned) | GW-REQ-132 |
| G6 | Build | gate-warden | **L1** | The L1 checks (GW-REQ-038); L2 adds GW-REQ-115, L3 adds GW-REQ-130/131 |
| G7 | Refactor | gate-warden | After factory role complete | Not in the brief's increments (A-11) |
| G8 | Assurance | gate-warden | L3 (planned) | GW-REQ-133 |
| G9 | Release | **release-ratifier** | n/a | gate-warden supplies inputs only (GW-REQ-134, [ADR-0011](../adr/ADR-0011-g9-owned-by-release-ratifier.md)) |

## G6 Build at L1

At L1 there is exactly one active gate: **G6 Build**. It applies to every PR in an installed
repository and is published as the check run **`gate-warden / G6 Build`**.

The software pack's G6 definition (in the harness) declares the following checks. The order
matters: checks that do not execute PR code come first ([security.md](security.md), T-09).

| Order | Example check id | Command (pack-supplied) | Executes PR code? |
| --- | --- | --- | --- |
| 1 | `agent-gate.scope-check` | agent-gate `scope-check` over the merge-base..SHA commit range | No |
| 2 | `oracle-guard.lock-verify` | oracle-guard `lock --verify` | No |
| 3 | `rust.fmt` | `cargo fmt --check` (wrapped) | No |
| 4 | `rust.build` | build (wrapped) | **Yes** (`build.rs`, proc macros) |
| 5 | `rust.clippy` | clippy (wrapped) | **Yes** |
| 6 | `rust.test` | cargo-nextest emitting JUnit (wrapped) | **Yes** |
| 7 | `oracle-guard.inventory` | oracle-guard `inventory`, reading the JUnit output | No, but it reads output that PR code produced |

"Wrapped" means the pack supplies a thin wrapper that turns the tool's native output into a
gate result ([interfaces/gate-result-format.md](interfaces/gate-result-format.md)). Who writes
these wrappers is a cross-repo item ([assumptions.md](assumptions.md), A-30).

## Check-run naming

- **Name:** `gate-warden / <GateId> <Gate name>`, for example `gate-warden / G6 Build`. Rulesets
  require these names, so renaming a gate is a breaking change that needs a ruleset update.
- **`external_id`:** `gw1:<sha>:<gate>:<harness_version>:<config_sha>`
  ([ADR-0006](../adr/ADR-0006-idempotency-key-and-check-run-reuse.md)).
- **Detail:** one check run per gate. Per-check detail goes in the output
  ([ADR-0005](../adr/ADR-0005-one-check-run-per-gate.md)).

## Applicability

Each gate definition has `applies_to` rules: branch glob patterns and changed-path globs. They
are evaluated against the default-branch config snapshot.

- **L1:** G6 has no restricting rules, so it applies to every PR.
- **L2 (planned):** when spec-PR gates (G1, G3, G4) and G6 coexist, applicability must never let
  a change avoid a gate. The **coverage rule** (GW-REQ-021) requires every changed path to be
  covered by at least one applicable gate. Applicability may not be decided by branch name
  alone. Branch names such as `cogworks/<issue>/<stage>` are chosen by the factory and are not
  a security boundary.
- **Open L2 item:** when a gate does not apply to a PR, its required check still needs a
  conclusion or the PR cannot merge. One option is a `success` check run with "not applicable"
  as the reason. It is safe only together with the coverage rule. This must be decided when G1
  is specified (A-15).

## G9 and the single-required-check-source tension

The handbook's principle is that gate-warden is the **only** App allowed as a required-check
source. The user has decided (option A) that **release-ratifier owns the G9 verdict and
publishes it**. These two statements are in tension:

- If G9 is enforced as a **required check**, a second App becomes a required-check source. That
  weakens the property that only one identity can produce a green required check. Every
  required-check source becomes something an attacker can try to forge or compromise.
- If release-ratifier instead **refuses to tag or publish** when G9 fails, no second
  required-check source is needed. Enforcement then sits in release-ratifier's own release path,
  and GitHub cannot enforce it on the release PR's merge.

**This decision is not taken here.** It belongs to release-ratifier's spec. gate-warden's part
(L3, GW-REQ-134) is limited to:

- making the signed G1–G8 results for a SHA available as evidence;
- never publishing a check run named for G9.

The item is tracked in [assumptions.md](assumptions.md) (A-12) and
[tradeoffs.md](tradeoffs.md#g9-ownership).

## Tier variation (L3, planned)

Gate definitions may carry `thresholds` per tier (for example a mutation score or holdout
ratio). At L1 and L2 a definition that contains `thresholds` is rejected with
`ConfigUnresolvable`, so thresholds are never silently ignored (GW-REQ-014).
