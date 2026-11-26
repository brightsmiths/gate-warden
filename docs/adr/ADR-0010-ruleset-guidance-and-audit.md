# ADR-0010: Ruleset setup by guidance plus a read-only audit; no Administration permission

**Status:** Proposed
**Date:** 2026-11-27

## Context

The forge-proof property depends on repository rulesets that make gate-warden's checks required
and accept them only from the gate-warden App. Automating ruleset changes needs the
`administration: write` permission. gate-warden is a security-boundary App, and granting it admin
rights would widen what an attacker gains by compromising it.

## Decision

- **L1:** ship ruleset setup guidance ([operations.md](../spec/operations.md#ruleset-setup)) and a
  **read-only audit** subcommand. A human runs it with their own credentials. It reports whether
  each expected gate-warden check is required on the default branch and restricted to the
  gate-warden App's integration id.
- The gate-warden App does **not** request `administration`.
- Any future automation is a separate tool, run by a human under their own identity.

## Consequences

- Rulesets can drift without anyone noticing until the audit is run. operations.md recommends
  running it on a schedule.
- No path exists for a compromised gate-warden to disable the very rules that protect its checks.

## Alternatives considered

- **gate-warden configures rulesets automatically:** convenient, but it requires admin
  permission on every repository.
- **Guidance only:** without an audit, drift is likely.
