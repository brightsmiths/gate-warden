# Configuration

gate-warden has two kinds of configuration:

1. **Service configuration:** how a gate-warden deployment runs. It is owned by the operator.
2. **Per-repository inputs:** what gates a repository gets. They are read from the repository's
   **default branch** only ([ADR-0002](../adr/ADR-0002-gate-config-from-default-branch.md)).

## 1. Service configuration

Supplied through a configuration file and/or environment variables at start-up. Secrets are
never stored in the configuration itself. It holds only references for the SecretProvider.

| Key (conceptual) | Required | Default | Meaning |
| --- | --- | --- | --- |
| `github.app_id` | yes | — | gate-warden App id. It is also the integration id the ruleset audit expects |
| `github.api_url` | no | `https://api.github.com` | GitHub Enterprise Server support |
| `github.private_key_ref` | yes | — | SecretProvider reference to the App private key |
| `github.webhook_secret_ref` | yes (webhook intake) | — | SecretProvider reference |
| `intake.mode` | no | `webhook` | `webhook` (L1) or `nats` (L2, planned) |
| `intake.listen_addr` | webhook mode | — | HTTP listen address |
| `intake.nats.*` | nats mode | — | Stream and consumer names *(L2)* |
| `trust.factory_bot_logins` | no | empty | Bot logins (factory role Apps, for example `…[bot]`) whose PRs are eligible (GW-REQ-003) |
| `sandbox.provider` | yes | — | `sandbox-runner`. `local-dev` is accepted only when `dev_mode` is true |
| `sandbox.endpoint` | yes | — | How to reach sandbox-runner (form set by its interface) |
| `sandbox.dependency_proxy` | yes | — | The only egress destination allowed for gate VMs |
| `dev_mode` | no | `false` | Enables the dev sandbox adapter. With it on, check runs get a `[dev]` name prefix, so they can never satisfy a ruleset |
| `harness.repository` | yes | — | The default harness repository (`owner/name`) for this deployment |
| `harness.allowed_repositories` | no | `[harness.repository]` | Harness repositories that a pin may select |
| `limits.max_concurrent_gate_runs` | no | `4` | Per instance (GW-REQ-092, proposed – to be revisited) |
| `limits.max_result_bytes` | no | `1048576` | Result file cap ([gate-result-format.md](interfaces/gate-result-format.md)) |
| `limits.max_annotations` | no | `50` | Per check run |
| `retry.max_attempts` / `retry.max_elapsed` | no | `5` / `10m` | Infrastructure retries (GW-REQ-039) |
| `reconcile.interval` | no | `10m` | Reconciliation period (GW-REQ-074) |
| `telemetry.otlp_endpoint` | no | none | OpenTelemetry export |
| `secrets.provider` | yes | — | `vault`, a KMS adapter, or `file` (dev only) |

The service configuration contains **no gate content**: no checks, commands or images. All of
that comes from the harness.

## 2. Per-repository inputs (default branch only)

### `.factory/pin.yaml` (human-owned; schema owned by the harness)

gate-warden reads only these fields. Their exact names are confirmed with the harness spec
([assumptions.md](assumptions.md), A-29):

| Field | Required | Meaning |
| --- | --- | --- |
| `harness.version` | yes | Semver tag of the harness, for example `v1.2.0` |
| `harness.commit` | recommended | 40-hex commit the tag must resolve to (GW-REQ-012) |
| `harness.repository` | no | Selects a harness repository other than the default. It must be in `harness.allowed_repositories` |
| `discipline_pack` | yes | Pack whose gate definitions apply, for example `software` |

Any other fields of the pin are ignored by gate-warden. If one of them changes gate behaviour,
it must be added to this spec and to the schema first.

### Harness gate definitions

- Read from the harness repository at the resolved commit, for the pinned discipline pack.
- Where in the harness they live is defined by the harness spec. Proposed:
  `disciplines/<pack>/gates/*.yaml` (handbook open question 2).
- Each definition is validated against
  [`schemas/gate-definition/v1.schema.json`](../../schemas/gate-definition/v1.schema.json)
  ([gate-definition-schema.md](interfaces/gate-definition-schema.md)).

### `risk-map.yaml` *(L2/L3, planned)*

The tier per path glob, read from the default branch. It is used for approval rules (L2) and
thresholds (L3). It is not read at L1.

### Authorised humans *(L2, planned)*

The mapping from tier and transition to GitHub teams is held in human-owned configuration on the
default branch (pin or harness, to be decided at L2). Membership is read from GitHub
(GW-REQ-113/114).

## Resolution order and precedence

1. Read the default-branch HEAD. That commit is `config_sha`.
2. Read and validate the pin at `config_sha`.
3. Resolve the harness repository (pin override if it is on the allow-list, otherwise the service
   default) and the version → commit. Check that it matches `harness.commit` if that is given.
4. Load and validate the pack's gate definitions.
5. Build the `ConfigSnapshot`.

Any failure fails closed with `ConfigUnresolvable` (GW-REQ-014). There is no fallback to cached
or default configuration, and never to the PR.

## Validation summary

| Input | Validation |
| --- | --- |
| Service configuration | At start-up. The service refuses to start if it is invalid, or if `local-dev` is set without `dev_mode` |
| Pin | Required fields present; version is semver; commit is 40 hex characters |
| Gate definitions | JSON Schema v1 plus the extra rules in [gate-definition-schema.md](interfaces/gate-definition-schema.md#validation-beyond-the-schema) |
