# Responsibilities (CRC)

Each component is described by what it **knows**, what it **does**, who it **collaborates** with,
and its **roles**. These are logical components. The interface designer decides how they map to
modules and crates. See [architecture.md](architecture.md) for the layer each one belongs to.

## Event Intake (adapter)

- **Knows:**
  - the webhook secret (via SecretProvider);
  - the event types it accepts (L1: `pull_request` opened/synchronize/reopened,
    `check_run` rerequested).
- **Does:**
  - verifies the HMAC before parsing;
  - normalises events into intake requests (repo, PR number, head SHA, reason);
  - drops everything else.
- **Collaborators:** SecretProvider, Evaluation Orchestrator.
- **Roles:** gatekeeper for inbound traffic.

## Eligibility Policy (business logic)

- **Knows:**
  - the trusted-actor rules (same-repo head; author association `OWNER` / `MEMBER` /
    `COLLABORATOR`; configured factory App bots).
- **Does:**
  - classifies a PR as eligible or not-evaluated;
  - never trusts eligibility claims from the event alone; it works on PR data freshly fetched
    from GitHub.
- **Collaborators:** Evaluation Orchestrator (supplies PR data).
- **Roles:** guard against running untrusted code.

## Gate Config Resolver (business logic, with readers)

- **Knows:**
  - the pin fields it reads;
  - the gate-definition schema versions it supports;
  - which features the running level does not support.
- **Does:**
  - reads the pin and gate definitions at default-branch HEAD and the pinned harness commit;
  - validates them;
  - produces a `ConfigSnapshot` or `ConfigUnresolvable`.
- **Collaborators:** RepositoryReader, HarnessReader.
- **Roles:** the only source of gate configuration; fail-closed.

## Gate Planner (business logic)

- **Knows:** applicability rules; (L2) the coverage rule.
- **Does:** selects and orders the applicable gates for the PR's branch and changed paths.
- **Collaborators:** Gate Config Resolver (snapshot), RepositoryReader (changed paths).
- **Roles:** decides which gates apply.

## Idempotency Guard (business logic)

- **Knows:**
  - the `external_id` format;
  - which outcome categories may be reused;
  - the stale-run threshold.
- **Does:**
  - decides whether to reuse, execute or supersede;
  - identifies superseded runs for older SHAs.
- **Collaborators:** CheckRunStore, Clock.
- **Roles:** deduplication; restart safety.

## Check Executor (adapter-facing use case)

- **Knows:** how to build a sandbox request from a gate definition and a `ConfigSnapshot`.
- **Does:**
  - prepares the workspace outside the VM;
  - asks for a fresh microVM;
  - runs the checks in order with a fresh `GATE_RESULT_FILE` each;
  - enforces the timeouts;
  - collects the execution records;
  - retries infrastructure failures with bounds.
- **Collaborators:** Sandbox port (sandbox-runner).
- **Roles:** the bridge to isolation; it does not judge.

## Result Interpreter (business logic)

- **Knows:**
  - the gate-result v1 schema;
  - the size limit;
  - the exit-code rules.
- **Does:** turns each execution record into a `CheckOutcome` with status `pass`, `fail` or
  `error`.
- **Collaborators:** none (pure).
- **Roles:** the boundary where untrusted output becomes trusted, validated values.

## Verdict Engine (business logic)

- **Knows:** the verdict rule; (L3) tier thresholds.
- **Does:** computes the gate verdict and outcome category.
- **Collaborators:** Result Interpreter.
- **Roles:** the deterministic judge. No LLM, ever.

## Feedback Redactor (business logic)

- **Knows:**
  - the `gate-feedback/v1` structure;
  - the redaction rules;
  - GitHub's output limits;
  - the truncation order.
- **Does:**
  - composes "what failed / why it matters / what is allowed";
  - reduces holdout findings to REQ-IDs;
  - caps and escapes text;
  - selects annotations.
- **Collaborators:** Verdict Engine, Gate Config Resolver (purpose and allowed-responses texts).
- **Roles:** guard against information leakage.

## Check Run Publisher (adapter)

- **Knows:**
  - the check-run naming;
  - the conclusion mapping;
  - the GitHub API limits and retry policy.
- **Does:**
  - creates `in_progress` runs;
  - completes them with conclusion, output and annotations;
  - marks superseded runs.
- **Collaborators:** CheckRunStore (GitHub).
- **Roles:** the only writer of gate verdicts to GitHub.

## Evaluation Orchestrator (use case)

- **Knows:** the evaluation sequence.
- **Does:** coordinates intake → eligibility → config → planning → idempotency → execution →
  interpretation → verdict → feedback → publish. Also runs reconciliation.
- **Collaborators:** all of the above.
- **Roles:** application service. It holds in-flight handles only.

## Ruleset Auditor (CLI, read-only)

- **Knows:** the expected required check names; the gate-warden App's integration id.
- **Does:** reads repository rulesets with the **operator's** credentials and reports
  discrepancies. It changes nothing.
- **Collaborators:** GitHub (via the operator's token).
- **Roles:** detects drift in the forge-proof configuration.

## Planned components

| Component | Level | Knows / does |
| --- | --- | --- |
| Transition Verifier | L2 | Knows the human-only transitions and the team → tier mapping. Verifies the actor of label events and spec-PR approvals (two at T3) |
| Result Signer | L2 | Signs gate-result envelopes through chain-chronicler |
| Baseline Writer | L2/L3 | The only writer of `.factory/baseline/`. Mechanism deferred (proposed – to be revisited) |
| Threshold Evaluator | L3 | Reads the tier from `risk-map.yaml` and applies gate-definition thresholds |
| Holdout Provisioner | L3 | Arranges `oracle-guard holdout-inject` from the private holdouts repository into the clean workspace |
| G9 Evidence Provider | L3 | Exposes the signed G1–G8 results per SHA to release-ratifier. Never a G9 verdict |
