# ADR-0006: Idempotency key in `external_id`; reuse completed results

**Status:** Proposed
**Date:** 2026-11-27

## Context

The brief requires that the same (SHA, gate, harness version) gives the same result. Webhooks
can be duplicated, and gate-warden may restart. It holds no state outside GitHub. Check commands
also read the default-branch tree (ADR-0002), so a change to the default branch can change a
result even when the harness version is unchanged.

## Decision

- Encode the key in the check run's `external_id` as
  `gw1:<sha>:<gate>:<harness_version>:<config_sha>`.
- Before executing, look up gate-warden's check runs on the SHA by name and `external_id`:
  - If a matching run is **completed** with outcome category `verdict`, `check-error` or
    `configuration`, reuse it and do not re-execute.
  - If the category is `infrastructure`, `superseded` or `not-evaluated`, or a run is stuck
    `in_progress` past the gate's maximum run time, re-execute.
- Make execution reproducible:
  - digest-pinned images;
  - an immutable harness commit;
  - a clean workspace;
  - the dependency proxy only;
  - deterministic truncation and ordering of feedback;
  - deterministic seeds at L3 (GW-REQ-135).

## Consequences

- Duplicate and redelivered events are cheap: one API lookup.
- Every merge to the default branch makes new evaluations use a new `config_sha`. That is
  intended, because results depend on default-branch content.
- `config_sha` is an extension of the brief's key. It is recorded in
  [assumptions.md](../spec/assumptions.md) (A-10) for confirmation.
- Non-deterministic checks (flaky tests) can still break idempotency. Reuse hides this for
  repeated requests, and testing.md requires a determinism test.

## Alternatives considered

- **Key without `config_sha`:** a result could be reused after the default-branch policy
  changed, so a PR could pass under rules that no longer apply.
- **Always re-execute:** wastes compute and still does not guarantee the same result.
- **External result cache:** that is state outside GitHub.
