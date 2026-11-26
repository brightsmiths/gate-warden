# ADR-0007: Checks run in sandbox-runner microVMs; gate-warden implements no isolation

**Status:** Proposed
**Date:** 2026-11-27

## Context

Gate checks build and test untrusted PR code. Brightsmiths requires every gate run to execute in
a fresh microVM with:

- a read-only root filesystem;
- a non-root user;
- no secrets;
- deny-by-default egress (dependency proxy only).

See handbook ADR-0005 and `docs/architecture/isolation-and-security.md`. sandbox-runner is the
service that provides microVMs. Its exact interface (CLI, library or API) is not defined yet.

## Decision

- gate-warden reaches sandbox-runner through a **sandbox port**. It requests one fresh microVM per
  gate run, with:
  - the clean workspace;
  - the read-only default-branch and harness trees;
  - the gate's image and limits;
  - egress limited to the dependency proxy.
- The workspace is prepared **outside** the microVM. Credentials never enter it.
- Checks run sequentially inside the one microVM, with a fresh result file for each check.
- The adapter for the real sandbox-runner interface is written once that interface exists. A
  local-container adapter may exist **for development and tests only**. It is never selectable
  for real verdicts. Configuration refuses it unless an explicit dev-mode flag is set, and
  results produced in dev mode cannot carry the production check-run names.

## Consequences

- gate-warden depends on sandbox-runner L1 at M1. Delivery order matters.
- PR code that runs in a check (build scripts, tests) can tamper with state that later checks in
  the same microVM read, such as JUnit files. This is a recorded residual risk
  ([security.md](../spec/security.md), T-09). Mitigation depends on check ordering and on
  sandbox-runner capabilities.
- gate-warden never holds a KVM dependency itself.

## Alternatives considered

- **A fresh microVM per check:** stronger isolation between checks, but the build is repeated
  for every check. That multiplies cost, and checks such as `inventory` need the output of an
  earlier check.
- **gate-warden drives microsandbox directly:** duplicates sandbox-runner's responsibilities and
  breaks "small services".
- **Containers:** not hardware isolation, which handbook ADR-0005 rules out.
