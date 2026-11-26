# Interface: sandbox-runner integration

gate-warden runs every gate in a **fresh microVM provided by sandbox-runner**
([ADR-0007](../../adr/ADR-0007-checks-run-via-sandbox-runner.md)). sandbox-runner's concrete
interface (CLI, library or API) is not defined yet. This page states what gate-warden **needs**
from it. That need is the contract gate-warden's sandbox adapter is written against. Where
sandbox-runner cannot provide something, it is an open cross-repo item
([assumptions.md](../assumptions.md)).

## Request: run one gate

| Element | Content | Requirement |
| --- | --- | --- |
| Workspace | A clean checkout of the head SHA, with Git history covering `GATE_BASE_SHA..GATE_HEAD_SHA` (needed for the scope check), mounted read-write at `GATE_WORKSPACE` | Prepared **outside** the microVM (GW-REQ-031) |
| Default-branch tree | The tree at `config_sha`, mounted **read-only** at `GATE_DEFAULT_BRANCH_DIR` | GW-REQ-032 |
| Harness tree | The harness at the pinned commit, mounted **read-only** at `GATE_HARNESS_DIR` | GW-REQ-032 |
| Image | `sandbox.image` from the gate definition (digest-pinned) | Provides the check tools |
| Root filesystem | Read-only | Brightsmiths isolation rules |
| User | Non-root | Same |
| Secrets | **None**: no tokens, keys or credentials in env, mounts or arguments | GW-REQ-030/031 |
| Network | Deny by default; egress only to the configured dependency proxy | GW-REQ-030 |
| Limits | `cpus`, `memory_mib`, `disk_mib`, `wall_clock_seconds` from the gate definition | Enforced by sandbox-runner |
| Checks | Ordered list of `{check_id, argv, env, timeout_seconds}` | Run without a shell |
| Per-check result path | A fresh, empty path per check, exposed as `GATE_RESULT_FILE` | GW-REQ-034 |
| Labels | Repo, SHA, gate id, `external_id`, used only for tracing | Never interpreted inside the VM |
| *(L3)* Holdout injection | A step run before the checks that adds holdouts from the private repository, outside the agent's reach | GW-REQ-131 |

Who prepares the workspace is an open item:

- **Default:** gate-warden prepares it with its installation token (fetch the SHA and history
  down to the merge base) and hands sandbox-runner a directory or volume.
- **Alternative:** sandbox-runner offers workspace preparation itself, using a token gate-warden
  passes to the runner (never to the VM).

## Execution semantics expected from sandbox-runner

1. Boot a **fresh** microVM per gate run. Never reuse one, and never share writable caches
   between gate runs.
2. Run the checks **sequentially** in the given order, with working directory `GATE_WORKSPACE`.
3. **Between checks**, terminate every process the previous check started (its process group,
   or every non-init process). This limits tampering by background processes started by PR code
   ([security.md](../security.md), T-09). If sandbox-runner cannot do this, it is an open item.
4. Enforce each check's timeout: kill the check and report `timed_out`.
5. After each check, copy out the content of its result file (at most 1 MiB plus 1 byte, so
   oversize can be detected) before the next check starts.
6. Destroy the microVM after the last check, on error, or on cancel.

## Response: per-check execution records

| Field | Meaning |
| --- | --- |
| `check_id` | As requested |
| `exit_code` | Integer, or none if the check was killed |
| `timed_out` | Boolean |
| `result_bytes` | The result file content (capped), or none if it is missing |
| `result_truncated` | True if the file exceeded the cap |
| `duration_ms` | For metrics |
| `log_ref` | Optional opaque reference to size-capped stdout/stderr kept for operators. **Never published** |

Run-level fields:

- `image_digest`: the digest actually booted, which must equal the requested one;
- `sandbox_error`: set if the VM failed as a whole.

gate-warden turns the records into check outcomes (Result Interpreter). The records themselves
are not trusted beyond what is listed: an exit code, a timeout flag and opaque bytes.

## Failure modes

| Failure | gate-warden handling |
| --- | --- |
| Provider unavailable, or the VM fails to boot | Retry with bounds, then `failure` with category `infrastructure` (GW-REQ-039) |
| Booted digest differs from the requested one | `failure`, `infrastructure`; this is a security event and is logged |
| VM crash mid-run | Checks with no record become `error`. If no check completed, the whole run is treated as `infrastructure` and retried |
| Workspace preparation failed (SHA unreachable) | `infrastructure` after retries |
| Cancel (superseded) | Request cancellation; the run is completed as `superseded` |

## Dev-only adapter

A local adapter (for example containers) may implement the same port for development and tests.
It must be refused unless dev mode is enabled explicitly. It must never be used for check runs
on repositories whose rulesets require gate-warden checks
([ADR-0007](../../adr/ADR-0007-checks-run-via-sandbox-runner.md)).
