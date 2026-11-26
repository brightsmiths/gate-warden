# ADR-0003: Standard gate result format v1 (JSON written to a file)

**Status:** Proposed
**Date:** 2026-11-27

## Context

gate-warden must contain no discipline-specific logic. Every check is a pack-supplied command,
so gate-warden needs one discipline-neutral result contract that every check honours: check id,
status, findings with location and coarse message, and optional metrics. The format is a
discipline-pack interface and must be versioned.

## Decision

- Define `gate-result/v1` as a strict JSON Schema (draft 2020-12) in
  `schemas/gate-result/v1.schema.json`. It forbids additional properties.
- Statuses are `pass`, `fail` and `error`. There is **no `skip`**.
- The check command writes exactly one result to the path in `GATE_RESULT_FILE` (at most
  1 MiB). stdout and stderr are diagnostics only and are never published.
- A missing, invalid or oversized result, a `check_id` mismatch, or a non-zero exit code
  together with `pass` is recorded as `error`.
- Findings carry `visibility` (`visible` / `holdout`). gate-warden reduces holdout findings to
  their REQ-ID.
- Versioning: the major version is in the `format` field; breaking changes create a new major
  with a new schema file. See
  [gate-result-format.md](../spec/interfaces/gate-result-format.md).

## Consequences

- Native tools (cargo, clippy, nextest) need thin pack wrappers to emit the format.
- A strict schema means packs cannot emit new fields until gate-warden accepts them. This trades
  some flexibility for predictability and a smaller injection surface.
- Every failure to produce a result fails the gate.

## Alternatives considered

- **Result on stdout:** mixed with tool noise, which leads to parse errors and spoofing by
  anything that prints.
- **SARIF / JUnit as the contract:** discipline-flavoured, large, and not covering metrics or
  holdout visibility. Packs may still convert them inside wrappers.
- **Exit code only:** gives no findings, so no useful feedback for CogWorks.
