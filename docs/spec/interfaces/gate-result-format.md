# Interface: standard gate result format (v1)

**Status:** v1, discipline-pack interface.
**Schema:** [`schemas/gate-result/v1.schema.json`](../../../schemas/gate-result/v1.schema.json)
(JSON Schema draft 2020-12).
**Decision record:** [ADR-0003](../../adr/ADR-0003-standard-gate-result-format.md).

Every check that gate-warden runs is a command supplied by a discipline pack. The command
reports its outcome by writing **one gate result** in this format. The format is the only thing
gate-warden knows about a check's output. Logs and exit codes are secondary signals
(see [Exit code rules](#exit-code-rules)).

## Emission contract

1. gate-warden sets these environment variables for the check command:

   | Variable | Meaning |
   | --- | --- |
   | `GATE_RESULT_FILE` | Absolute path of a fresh, non-existent file. The command must write its result here. |
   | `GATE_ID` | The gate id, e.g. `G6` |
   | `GATE_CHECK_ID` | The check id from the gate definition |
   | `GATE_HEAD_SHA` | The SHA under evaluation |
   | `GATE_BASE_SHA` | The merge base of the head SHA and the default branch at `config_sha` |
   | `GATE_CONFIG_SHA` | The default-branch commit the config was read from |
   | `GATE_WORKSPACE` | The clean checkout of the head SHA (the working directory) |
   | `GATE_DEFAULT_BRANCH_DIR` | Read-only tree of the default branch at `config_sha` |
   | `GATE_HARNESS_DIR` | Read-only tree of the harness at the pinned version |
   | `GATE_SEED` | *(L3, planned)* Deterministic seed (GW-REQ-135) |

2. The command writes a single JSON document, UTF-8 encoded, of at most **1 MiB**, to
   `GATE_RESULT_FILE`. Then it exits.
3. Checks that need policy, keys or other trusted inputs must read them from
   `GATE_DEFAULT_BRANCH_DIR` or `GATE_HARNESS_DIR`, **never** from `GATE_WORKSPACE`. The
   workspace is the PR content under judgement ([security.md](../security.md), T-02).
4. stdout and stderr are collected by the sandbox for operator diagnostics only. They are
   size-capped and **never published** (GW-REQ-061).

Why a file rather than stdout: tools print freely to stdout, and mixing the result with tool
noise invites parse errors and injection.

## Fields

| Field | Type | Required | Meaning |
| --- | --- | --- | --- |
| `format` | const `"gate-result/v1"` | yes | Format and major version |
| `check_id` | string `^[a-z0-9][a-z0-9._-]{0,63}$` | yes | Must equal `GATE_CHECK_ID` |
| `status` | `pass` / `fail` / `error` | yes | See [Status semantics](#status-semantics) |
| `summary` | string ≤ 500 | no | One-line coarse summary |
| `findings[]` | ≤ 1000 items | no | See below |
| `findings[].location.path` | repository-relative path | no | No leading `/`, no `..` segments |
| `findings[].location.start_line`, `end_line`, `start_column`, `end_column` | integer ≥ 1 | no | `end_*` requires the matching `start_*` |
| `findings[].rule` | string ≤ 200 | no | Rule or lint code |
| `findings[].severity` | `error` / `warning` / `notice` | yes | Presentation only. The check's `status` decides pass/fail |
| `findings[].message` | string 1–1000 | yes | Coarse message. Never test source or holdout content |
| `findings[].req_id` | REQ-ID token | no | Requirement this relates to |
| `findings[].visibility` | `visible` (default) / `holdout` | no | `holdout` findings are reduced to `req_id` before publishing |
| `metrics[]` | ≤ 100 items: `{name, value, unit?}` | no | Numeric measurements (thresholds at L3) |
| `tool` | `{name, version?}` | no | Underlying tool identity, for provenance |
| `seed` | string | no | *(L3)* Seed used |

No additional properties are allowed at any level. Unknown fields make the result invalid.
This keeps the format strict and lets it be versioned explicitly.

## Status semantics

- **`pass`:** the check examined the workspace and found it acceptable.
- **`fail`:** the check examined the workspace and found it unacceptable. Findings should
  explain why.
- **`error`:** the check could not reach a judgement, for example because of a tool crash or a
  missing input. It counts as not passing.
- **There is no `skip`.** A check that does not apply should not be declared in the gate, or it
  should report `pass` with a summary saying why. Whether it applies is a configuration
  decision; the check does not decide it at run time.

## Exit code rules

| Exit code | Result file | Recorded status |
| --- | --- | --- |
| 0 | valid, `pass` | `pass` |
| 0 | valid, `fail` / `error` | as reported |
| non-zero | valid, `fail` / `error` | as reported |
| non-zero | valid, `pass` | `error` (GW-REQ-037) |
| any | missing / invalid / too large / wrong `check_id` | `error` (`CheckOutputInvalid`, GW-REQ-035) |
| killed by timeout | any | `error` (`CheckTimeout`, GW-REQ-036) |

## Redaction-relevant rules

- The pack is responsible for keeping messages coarse.
- gate-warden **also** enforces the rules on its side, as defence in depth
  ([feedback-format.md](feedback-format.md)):
  - `holdout` findings are reduced to their `req_id`;
  - text is capped and escaped;
  - logs are never published.

## Versioning

- `format` carries the major version.
- **Breaking changes** create `gate-result/v2` with a new schema file. Breaking changes are
  removing or renaming fields, tightening constraints, or adding a required field.
- **Additive changes** need a minor schema revision of v1 that gate-warden accepts first. Packs
  may emit the new field only once the pinned gate-warden version accepts it. Optional fields
  count as additive. Because v1 forbids unknown properties, the schema must be updated before
  any pack emits a new field.
- gate-warden supports every major version that a pinned harness may still declare. The
  supported versions are listed in the gate-warden release notes.

## Examples

A passing clippy check:

```json
{
  "format": "gate-result/v1",
  "check_id": "rust.clippy",
  "status": "pass",
  "summary": "no clippy warnings",
  "tool": { "name": "clippy", "version": "0.1.83" }
}
```

A failing test check with a holdout finding:

```json
{
  "format": "gate-result/v1",
  "check_id": "rust.test",
  "status": "fail",
  "summary": "2 tests failed",
  "findings": [
    {
      "location": { "path": "src/billing/invoice.rs", "start_line": 42 },
      "rule": "test-failure",
      "severity": "error",
      "message": "locked test for negative amounts failed",
      "req_id": "REQ-017"
    },
    {
      "severity": "error",
      "message": "holdout failed",
      "req_id": "REQ-021",
      "visibility": "holdout"
    }
  ],
  "metrics": [
    { "name": "tests.passed", "value": 118, "unit": "count" },
    { "name": "tests.failed", "value": 2, "unit": "count" }
  ]
}
```

What gate-warden publishes for the second finding is only `REQ-021`
([feedback-format.md](feedback-format.md)).
