# Trade-offs

This page summarises the alternatives that were weighed. The binding decisions are in the ADRs
under [`docs/adr/`](../adr/).

## Result emission: file versus stdout versus exit code

| Option | Pros | Cons |
| --- | --- | --- |
| **File at `GATE_RESULT_FILE` (chosen)** | Clean separation from tool noise; easy to cap and validate | Native tools need wrappers |
| stdout JSON | No path plumbing | Anything that prints can corrupt or spoof the result |
| Exit code only | Trivial | No findings, so no useful feedback |
| SARIF / JUnit directly | Existing tooling | Discipline-flavoured; no holdout visibility or metrics; large |

Decision: [ADR-0003](../adr/ADR-0003-standard-gate-result-format.md).

## Check-run granularity

| Option | Pros | Cons |
| --- | --- | --- |
| **One per gate (chosen)** | Stable ruleset names; packs can add checks without ruleset changes | Per-check status visible only in the output |
| One per check | Fine-grained UI | Ruleset churn; dropping a check from a pack silently removes a required check |

Decision: [ADR-0005](../adr/ADR-0005-one-check-run-per-gate.md).

## Conclusion for infrastructure errors

| Option | Pros | Cons |
| --- | --- | --- |
| **`failure` with category (chosen)** | Fail safe | Looks like a code failure in the UI unless the category is read |
| `neutral` | Signals "not your fault" | **Satisfies required checks, so it fails open** |
| No conclusion (leave pending) | Blocks merge | Stuck checks; poor signal for CogWorks |

## Intake mode

| Option | Pros | Cons |
| --- | --- | --- |
| **Port with webhook at L1, NATS later (chosen)** | Unblocks M1; one pipeline | Missed deliveries at L1 rely on reconciliation |
| queue-keeper only | Durable delivery | Not available until M3 |
| Polling only | No webhook exposure | Latency; API cost |

Decision: [ADR-0009](../adr/ADR-0009-event-intake-port.md).

## Ruleset automation versus guidance

| Option | Pros | Cons |
| --- | --- | --- |
| **Guidance plus read-only audit (chosen)** | No admin permission on a security-boundary App | Manual steps; drift between audits |
| gate-warden automates rulesets | Always consistent | `administration: write` on every repository; a compromise could disable the protections |

Decision: [ADR-0010](../adr/ADR-0010-ruleset-guidance-and-audit.md).

## Feedback channel

| Option | Pros | Cons |
| --- | --- | --- |
| **Check-run output (chosen)** | Already the system of record for verdicts; no new state | 65 535-character limit; visible to all readers |
| PR comments | Visible in the conversation | Second state location; noise |
| Direct call to CogWorks | Rich | A direct hand-off, which is forbidden |

Decision: [ADR-0008](../adr/ADR-0008-feedback-in-check-run-output.md).

## Idempotency: reuse versus recompute

| Option | Pros | Cons |
| --- | --- | --- |
| **`external_id` lookup and reuse (chosen)** | Cheap duplicates; stable results despite flaky checks | `external_id` must include `config_sha` to stay correct |
| Always recompute | Simple | Expensive; flaky checks flip results |
| External cache | Fast | State outside GitHub |

Decision: [ADR-0006](../adr/ADR-0006-idempotency-key-and-check-run-reuse.md).

## Sandbox granularity

| Option | Pros | Cons |
| --- | --- | --- |
| **One microVM per gate run, checks sequential (chosen)** | Build once per gate; later checks can use earlier outputs (JUnit) | PR code that runs in one check can tamper with later checks' inputs (T-09) |
| One microVM per check | Strong isolation between checks | Repeated builds; hard to pass outputs safely between checks |
| Phased VMs (integrity checks, then code-executing checks, then result collection outside the VM) | Closes most of T-09 | Needs sandbox-runner support and a richer gate-definition model. Candidate for after M1 (A-27) |

Decision: [ADR-0007](../adr/ADR-0007-checks-run-via-sandbox-runner.md).

## Sandbox interaction: synchronous versus callback

| Option | Pros | Cons |
| --- | --- | --- |
| **Await completion through the port (default)** | Simple; fits a stateless service with in-memory handles | Long-held connections; a restart loses the handle (reconciliation recovers it) |
| Callback or event from sandbox-runner | Survives restarts | More moving parts; must not become a direct hand-off of workflow state |

To be fixed with sandbox-runner's interface (A-16).

## G9 ownership

The user chose **option A**: release-ratifier owns and publishes the G9 verdict. gate-warden
supplies inputs only ([ADR-0011](../adr/ADR-0011-g9-owned-by-release-ratifier.md)).

| Option | Pros | Cons |
| --- | --- | --- |
| **A: release-ratifier owns and publishes G9 (chosen)** | Release logic and release provenance stay in one service with one purpose; gate-warden stays smaller | Conflicts with "gate-warden is the only required-check source" if G9 is enforced as a required check |
| B: gate-warden runs G9 using release-ratifier's checks | Keeps a single required-check source | Mixes release concerns into gate-warden. Rejected |

**Open cross-repo item:** how G9 is enforced is decided in release-ratifier's spec:

- **Required check from a second App.** Risk: a second required-check source weakens the
  forge-proof property. There are two identities to protect, and rulesets accept two sources.
- **release-ratifier refuses to tag or publish.** gate-warden remains the only required-check
  source. GitHub merge protection does not enforce G9.

Tracked in [assumptions.md](assumptions.md) (A-12).
