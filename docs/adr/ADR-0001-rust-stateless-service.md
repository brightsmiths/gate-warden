# ADR-0001: Implement gate-warden as a stateless Rust service

**Status:** Proposed
**Date:** 2026-11-27

## Context

gate-warden is a control-plane, security-boundary service. Brightsmiths' principles require:

- GitHub is the only system of record. Any service can restart and rebuild its view from
  GitHub (handbook ADR-0001, `docs/architecture/state-model.md` in
  [brightsmiths/handbook](https://github.com/brightsmiths/handbook)).
- Services run anywhere, with no cloud-specific dependencies.
- The software discipline pack is Rust first. Most Brightsmiths services are Rust, and this
  repository's tooling assumes it.

gate-warden must not hold workflow state beyond GitHub check runs and baseline files.

## Decision

- Implement gate-warden in **Rust** as **one deployable, stateless service**.
- All workflow state lives in GitHub check runs, plus baseline files from L2. In-memory state
  (queues, in-flight run handles, caches) may be lost at any time without changing any outcome.
- Restart recovery is done by reconciling against GitHub (GW-REQ-074).
- Infrastructure is reached only through ports (see [architecture.md](../spec/architecture.md)):
  GitHub, sandbox-runner, secrets, the event source and, from L2, signing.

## Consequences

- Horizontal scaling is possible. Duplicate execution is harmless because of idempotency
  ([ADR-0006](ADR-0006-idempotency-key-and-check-run-reuse.md)), though it costs compute.
- With direct webhooks at L1, events missed during downtime are recovered only by
  reconciliation, not by redelivery.
- The ruleset audit is a subcommand of the same binary, run by humans
  ([ADR-0010](ADR-0010-ruleset-guidance-and-audit.md)).

## Alternatives considered

- **A database for run state:** faster lookups, but it is a second source of truth and violates
  "no workflow state beyond check runs".
- **A job per event (for example a CI workflow):** GitHub Actions in the judged repository would
  run under a repository-controlled workflow. That breaks "never read config from the PR".
- **Another language:** no benefit that outweighs the ecosystem's Rust-first choice.
