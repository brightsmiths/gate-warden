# Interface: structured feedback (`gate-feedback/v1`)

Feedback tells CogWorks, and the humans standing in for it at M1, **what failed, why it matters
and what is allowed**. It is coarse by design. It is carried in the check run's `output.text`
([ADR-0008](../../adr/ADR-0008-feedback-in-check-run-output.md)), because services do not hand
work to each other directly.

The machine-readable contract is
[schemas/gate-feedback/v1.schema.json](../../../schemas/gate-feedback/v1.schema.json). The
table below explains it; if the two disagree, the schema wins. gate-warden validates every
block it publishes against the schema, and failing that validation is a bug, not a gate
outcome.

## Placement

- `output.text` contains exactly one fenced code block with the info string
  `json gate-feedback/v1`.
- The block holds a single JSON object.
- A short human-readable preamble may come before the block. Consumers must parse only the
  block.

## Structure

| Field | Type | Meaning |
| --- | --- | --- |
| `format` | `"gate-feedback/v1"` | Version |
| `sha` | string | Evaluated SHA |
| `gate` | string | Gate id |
| `harness_version` / `harness_commit` | string | Pinned harness |
| `config_sha` | string | Default-branch commit used |
| `verdict` | `pass` / `fail` / `not-evaluated` | Gate outcome |
| `category` | `verdict` / `check-error` / `configuration` / `infrastructure` / `superseded` / `not-evaluated` | Why ([vocabulary.md](../vocabulary.md#outcome-category)) |
| `reason` | string ≤ 500 | Set for `configuration`, `infrastructure`, `superseded` and `not-evaluated`. A gate-warden-authored message, never check output |
| `checks[]` | array | One entry per declared check, in declared order |
| `checks[].check_id` | string | |
| `checks[].status` | `pass` / `fail` / `error` | Interpreted status |
| `checks[].error_kind` | `timeout` / `invalid-output` / `exit-mismatch` / `reported` | Set only when status is `error` |
| `checks[].summary` | string ≤ 200 | From the result, capped and escaped |
| `checks[].why_it_matters` | string | The check's `purpose` from the gate definition. Included only for non-passing checks |
| `checks[].allowed` | string | The check's `allowed_responses` from the gate definition. Included only for non-passing checks |
| `checks[].findings[]` | array | Redacted visible findings: `{path?, start_line?, end_line?, rule?, severity, message, req_id?}` |
| `checks[].holdout_failures[]` | array of REQ-IDs | Deduplicated and sorted. **Nothing else is published for holdouts** |
| `checks[].metrics[]` | array | `{name, value, unit?}` |
| `truncated` | object | `{findings_dropped: n, checks_affected: [ids]}`, if anything was dropped |

## Redaction rules (enforced by gate-warden)

1. **No logs.** stdout and stderr are never included, quoted or linked.
2. **Holdouts:** for any finding with `visibility: holdout`, only `req_id` is published, in
   `holdout_failures`.
   - A holdout finding without a `req_id` is counted as `holdout_failures: ["UNSPECIFIED"]`.
   - Its path, message and rule are dropped.
3. **Text caps:** messages are capped at 300 characters and summaries at 200, with a `…` suffix
   when cut.
4. **Escaping:** finding text is plain data. When it is rendered (summary, annotations):
   - Markdown and HTML control characters are escaped;
   - `@` mentions are neutralised;
   - URLs are not made into links;
   - control characters are stripped.
5. **Paths:** only repository-relative paths that pass the result schema's path rule are
   published. Others are dropped from the finding.
6. **Texts from trusted sources:** `why_it_matters` and `allowed` come from the gate
   definition, which is human-owned and taken from the default branch. They never come from
   check output.
7. **Deterministic truncation:** checks keep their declared order. Within a check, findings are
   sorted by (severity, path, line, rule, message) and dropped from the end until the output
   fits. The result is identical for identical inputs (GW-REQ-070).

**Not covered by these rules:** a pack check that copies test source *into a finding message*.
gate-warden cannot detect that in general. It only caps length. Packs must keep messages coarse
([security.md](../security.md), T-06).

## Example

````text
G6 Build failed: 1 of 7 checks failed.

```json gate-feedback/v1
{
  "format": "gate-feedback/v1",
  "sha": "4f9c...e1",
  "gate": "G6",
  "harness_version": "v1.2.0",
  "harness_commit": "a1b2...c3",
  "config_sha": "9d8e...f7",
  "verdict": "fail",
  "category": "verdict",
  "checks": [
    { "check_id": "agent-gate.scope-check", "status": "pass" },
    {
      "check_id": "rust.test",
      "status": "fail",
      "summary": "2 tests failed",
      "why_it_matters": "Locked tests encode the approved acceptance criteria.",
      "allowed": "Change code under src/. Do not edit locked tests; raise a dispute if you believe a test is wrong.",
      "findings": [
        { "path": "src/billing/invoice.rs", "start_line": 42, "rule": "test-failure",
          "severity": "error", "message": "locked test for negative amounts failed", "req_id": "REQ-017" }
      ],
      "holdout_failures": ["REQ-021"],
      "metrics": [ { "name": "tests.failed", "value": 2, "unit": "count" } ]
    }
  ]
}
```
````

## Versioning

- `format` carries the major version. CogWorks must reject unknown majors.
- Adding optional fields is a minor change within v1. Consumers must ignore fields they do not
  know.

The rule differs from the gate result format on purpose. Here gate-warden is the trusted
producer; for gate results it is the consumer of untrusted input.
