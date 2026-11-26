# Interface: gate definition schema (v1)

**Status:** v1, discipline-pack interface.
**Schema:** [`schemas/gate-definition/v1.schema.json`](../../../schemas/gate-definition/v1.schema.json)
(JSON Schema draft 2020-12).
**Decision record:** [ADR-0004](../../adr/ADR-0004-gate-definition-schema-ownership.md).

A gate definition tells gate-warden what a gate consists of. This repository owns the
**schema**. The harness repository owns the **instances**, one or more per discipline pack.
gate-warden reads instances only from the harness at the version pinned on the repository's
default branch ([configuration.md](../configuration.md)).

## Format

- YAML 1.2 (JSON-compatible subset) or JSON, validated against the v1 schema.
- One gate per document.
- At most one definition per gate id within the pinned discipline pack. Two or more give
  `ConfigUnresolvable`.

## Fields

| Field | Required | Meaning |
| --- | --- | --- |
| `schema` | yes | const `gate-definition/v1` |
| `gate` | yes | `G1`–`G8`. `G9` is rejected because it is owned by release-ratifier ([ADR-0011](../../adr/ADR-0011-g9-owned-by-release-ratifier.md)) |
| `name` | yes | Display name; forms the check-run name `gate-warden / <gate> <name>` |
| `description` | no | Human documentation |
| `applies_to.branches` | no | Globs matched against the PR head branch |
| `applies_to.paths_any` | no | Globs. The gate applies if any changed path matches. If both lists are given, both must match. If `applies_to` is absent, the gate applies to every PR |
| `sandbox.image` | yes | Image reference **pinned by digest** (`…@sha256:<64 hex>`). It carries the check tools |
| `sandbox.limits.cpus`, `memory_mib`, `disk_mib?`, `wall_clock_seconds` | yes (except `disk_mib`) | Resource limits passed to sandbox-runner |
| `checks[]` | yes, 1–50 | Run in order; every check runs (GW-REQ-033) |
| `checks[].id` | yes | Unique within the gate; must match the result's `check_id` |
| `checks[].purpose` | yes | "Why it matters" text for feedback |
| `checks[].allowed_responses` | yes | "What is allowed" text for feedback, for example "fix code under `src/`; do not edit locked tests; raise a dispute with `task-marshal dispute`" |
| `checks[].command` | yes | Argument vector, run without a shell |
| `checks[].env` | no | Extra non-secret variables. `GATE_*` names are reserved |
| `checks[].timeout_seconds` | yes | Per-check timeout (GW-REQ-036) |
| `thresholds` | no | *(L3, planned)* `{T0..T3: {metric: {min?, max?}}}`. Rejected before L3 (GW-REQ-014) |

## Validation beyond the schema

gate-warden also checks the following and fails with `ConfigUnresolvable` if any does not hold:

- Check ids are unique within the gate.
- The sum of `timeout_seconds` across checks is at most `sandbox.limits.wall_clock_seconds`.
- `thresholds` is absent while gate-warden runs below L3.
- The glob patterns compile.

## Discipline neutrality

Nothing in the schema refers to a language, a tool or a test framework. The software pack's
Rust tooling appears only inside `command` vectors and in the sandbox image. These belong to the
pack.

## Example (software pack, abbreviated)

```yaml
schema: gate-definition/v1
gate: G6
name: Build
sandbox:
  image: registry.example/brightsmiths/rust-gate@sha256:3b1f...e9
  limits: { cpus: 4, memory_mib: 8192, wall_clock_seconds: 2700 }
checks:
  - id: agent-gate.scope-check
    purpose: Every commit only touches paths its author's role may write.
    allowed_responses: Revert out-of-scope changes. Do not modify policy files.
    command: [gate-check-scope]
    timeout_seconds: 120
  - id: oracle-guard.lock-verify
    purpose: Locked oracle tests and test config match the signed manifest.
    allowed_responses: Restore locked files. To disagree with a test, raise a dispute.
    command: [oracle-guard, lock, --verify, --gate-result]
    timeout_seconds: 120
  # ... rust.fmt, rust.build, rust.clippy, rust.test, oracle-guard.inventory
```

The concrete commands and wrapper names belong to the harness and pack tools, not this spec
([assumptions.md](../assumptions.md), A-30).

## Versioning

- `schema` carries the major version. The rules are the same as for the gate result format
  ([gate-result-format.md](gate-result-format.md#versioning)).
- A harness version declares the gate-definition major version it uses. A gate-warden release
  lists the majors it supports.
- If the pinned harness uses an unsupported major, the result is `ConfigUnresolvable`.
