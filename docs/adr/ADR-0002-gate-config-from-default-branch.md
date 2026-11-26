# ADR-0002: Resolve gate configuration only from the default branch at evaluation time

**Status:** Proposed
**Date:** 2026-11-27

## Context

If gate-warden read its gate configuration from the PR it judges, the PR could weaken or remove
its own gates. The Brightsmiths bootstrap rules state that gate-warden reads gate config from the
default branch, never from the PR it judges. Configuration has two parts:

- `.factory/pin.yaml`, which pins the harness version;
- the harness gate definitions at that version.

Check commands also need trusted policy, such as agent-gate's write matrix and oracle trust
roots. These must not come from the PR either.

## Decision

- At the start of each evaluation, read `.factory/pin.yaml` from the **HEAD of the default
  branch at that moment**. Record its commit as `config_sha`.
- Resolve the pinned harness version to an immutable harness commit. Load that pack's gate
  definitions from it. If the pin also gives a commit, the two must match.
- Never read the pin, the gate definitions or the harness reference from the PR head, branch or
  merge ref.
- Mount the default-branch tree at `config_sha` and the harness tree **read-only** in the
  sandbox (`GATE_DEFAULT_BRANCH_DIR`, `GATE_HARNESS_DIR`). Checks take trusted inputs from there.
- If configuration cannot be resolved, fail closed with category `configuration`.

## Consequences

- A PR that changes `.factory/` or gate-affecting policy is judged by the old rules. The change
  takes effect after a human-reviewed merge.
- If the default branch moves during a run, that run stays bound to the `config_sha` it started
  with. The next evaluation picks up the new config.
- `config_sha` becomes part of the check run's `external_id`
  ([ADR-0006](ADR-0006-idempotency-key-and-check-run-reuse.md)).
- Pack checks must be written to read policy from the read-only trees. This is a cross-repo
  requirement on agent-gate and oracle-guard.

## Alternatives considered

- **Config from the PR base SHA:** the base can be stale and may not be the default branch, for
  example in stacked PRs.
- **Config embedded in gate-warden's deployment:** this couples a discipline-neutral service to
  pack content and breaks per-repo pinning.
- **Merge-ref evaluation:** it mixes PR content into the config source.
