# ADR-0004: gate-warden owns the gate definition schema; the harness owns instances

**Status:** Proposed
**Date:** 2026-11-27

## Context

Gate definitions say which checks form a gate, when it applies and which sandbox image to use.
Discipline packs in the harness supply them. gate-warden interprets them. The schema is a
discipline-pack interface and must be versioned. The harness repository's structure is still
open (handbook open question 2).

## Decision

- This repository owns the **schema** `gate-definition/v1`
  (`schemas/gate-definition/v1.schema.json`). The harness owns the **instances**.
- The schema is discipline-neutral:
  - gate id (`G1`–`G8`; G9 is excluded, see
    [ADR-0011](ADR-0011-g9-owned-by-release-ratifier.md));
  - name;
  - applicability globs;
  - a digest-pinned sandbox image;
  - resource limits;
  - ordered checks (id, purpose, allowed responses, argv, env, timeout);
  - optional per-tier thresholds (L3).
- Features not yet implemented (`thresholds` before L3) are rejected, not ignored.
- Versioning is semver: the major version is in the `schema` field; breaking changes create a
  new major and a new schema file.

## Consequences

- The harness validates its instances against this schema in its own CI.
- Where in the harness gate-warden finds the definitions for a pack is set by the harness spec.
  gate-warden isolates that behind its configuration adapter.
- Schema changes need coordinated releases: gate-warden must support a new version before any
  harness version uses it.

## Alternatives considered

- **Harness owns the schema:** the consumer that enforces semantics would not own the contract,
  so drift is likely.
- **Gate structure built into gate-warden code:** this breaks discipline neutrality.
- **Image by tag:** not reproducible, which breaks idempotency.
