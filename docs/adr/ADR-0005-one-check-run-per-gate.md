# ADR-0005: One check run per gate, with a restricted set of conclusions

**Status:** Proposed
**Date:** 2026-11-27

## Context

Required checks are matched by name in repository rulesets. The brief asks for one check run per
gate, with per-check detail in its output. GitHub treats the check-run conclusions `success`,
`neutral` and `skipped` as satisfying a required check. Any path that emits `neutral` or
`skipped` would therefore produce a de facto green.

## Decision

- Publish **one check run per gate per SHA**, named `gate-warden / <GateId> <name>`. Per-check
  results go in the output (summary, `gate-feedback/v1` block, annotations).
- Allow only three conclusions:
  - `success` (verdict pass);
  - `failure` (verdict fail, check error, configuration, infrastructure, superseded);
  - `action_required` (not evaluated: fork or untrusted actor).
- Never use `neutral`, `skipped`, `cancelled`, `stale` or `timed_out`. The conclusion type in the
  domain model has exactly the three variants, so the invalid ones cannot be represented.

## Consequences

- Rulesets list one required check per active gate (G6 at L1). Adding a check to a gate needs no
  ruleset change.
- A gate's internal structure is visible only in the output, not in the PR checks list.
- Infrastructure problems show as `failure`. Operators tell them apart by outcome category.

## Alternatives considered

- **One check run per check:** more ruleset entries, and they change whenever a pack adds a
  check. A pack could also effectively remove a required check by dropping it.
- **`neutral` for infrastructure errors:** it satisfies required checks, so it fails open.
- **`cancelled` for superseded runs:** harmless for required checks, but it is a fourth variant
  with no benefit. The category already explains the outcome.
