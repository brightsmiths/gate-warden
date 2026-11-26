# AGENTS.md

Guidance for humans and coding agents working in this repository. This file is human-owned.
Agents must not modify it.

## What this repository is

**gate-warden** (working name) is the Brightsmiths service that decides pass or fail. It
re-runs gates from a commit SHA in a clean sandbox, under its own GitHub App, and publishes
check runs. It is the only allowed required-check source.

- System architecture: [brightsmiths/handbook](https://github.com/brightsmiths/handbook), under
  `docs/` (on `feat/initial` until merged).
- Specification: [`docs/spec/`](docs/spec/README.md).
- Architecture decisions: [`docs/adr/`](docs/adr/), using
  [`ADR_TEMPLATE.md`](docs/adr/ADR_TEMPLATE.md).
- Technology choices: [`.tech-decisions.yml`](.tech-decisions.yml).

## Rules

1. **This is a control-plane, security-boundary repository.** It is treated as T2+ whoever
   builds it. Every PR gets a human review.
2. **Read the spec first.** In particular:
   - [`docs/spec/constraints.md`](docs/spec/constraints.md);
   - [`docs/spec/security.md`](docs/spec/security.md);
   - [`docs/spec/requirements.md`](docs/spec/requirements.md).
3. **Never** add:
   - discipline-specific logic: no knowledge of cargo, clippy, nextest, agent-gate, oracle-guard
     or any other check tool;
   - reading gate config from a PR;
   - an LLM in the verdict path;
   - workflow state outside GitHub check runs and baseline files.
4. **Conclusions** are only `success`, `failure` or `action_required`. Never `neutral` or
   `skipped`.
5. **Never publish** logs, test source or holdout content. Holdout failures appear as REQ-IDs
   only.
6. **Schema changes** to `schemas/` are discipline-pack interface changes. Follow the versioning
   rules in the interface specs and add an ADR.
7. **Committed files must not reference git-ignored scratch folders.** Cite the handbook
   instead.
8. **Never commit secrets** (App private keys, webhook secrets, tokens).

## Rust conventions

- **Tests:** sibling files `foo_tests.rs`, declared with
  `#[cfg(test)] #[path = "foo_tests.rs"] mod tests;`. Public-API tests go in `tests/`.
- **Errors:** expected errors are values (`Result`). No `unwrap` or `expect` on external data.
  No `unsafe`.
- **Formatting and linting:** `cargo fmt --check` and `cargo clippy -- -D warnings` must pass.
