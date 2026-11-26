# ADR-0011: G9 is owned by release-ratifier; gate-warden supplies inputs only

**Status:** Proposed
**Date:** 2026-11-27

## Context

The handbook's workflow says every G-gate is run by gate-warden and that gate-warden is the only
required-check source (handbook ADR-0004). The gate-warden brief lists "G9 inputs for
release-ratifier" at L3, and release-ratifier is described as "Release gate G9 and release
provenance". The user decided (option A) that release-ratifier owns the G9 verdict **and
publishes it**.

## Decision

- gate-warden does **not** run G9 and does **not** publish a G9 check run. The gate definition
  schema rejects `G9`.
- gate-warden's L3 responsibility (GW-REQ-134) is to make the **signed results of G1–G8 for a
  SHA** available as evidence that release-ratifier can consume. The results are signed through
  chain-chronicler (from L2). Exact delivery is specified at L3. Default: the signed results are
  referenced from gate-warden's check runs, with attestations stored via chain-chronicler.
- **How G9 is enforced is decided in release-ratifier's spec.** There are two options:
  1. a required check published by a second App (release-ratifier);
  2. release-ratifier refusing to tag or publish a release when G9 fails.

## Consequences

- **Tension, stated explicitly:** option 1 makes a second App a required-check source. That
  weakens the forge-proof property ("only gate-warden can produce a green required check").
  Every required-check source is an identity an attacker can target, and rulesets must then
  allow two sources. Option 2 keeps gate-warden as the only required-check source, but G9 is no
  longer enforced by GitHub on the release PR merge.
- This is an open cross-repo item. It is tracked in
  [assumptions.md](../spec/assumptions.md) (A-12) and
  [tradeoffs.md](../spec/tradeoffs.md#g9-ownership). The handbook's statement that "every
  G-gate is run by gate-warden" should be updated to match.
- gate-warden's ruleset audit checks only gate-warden's gates.

## Alternatives considered

- **gate-warden runs G9 using release-ratifier's checks (option B):** this keeps the single
  required-check source. The user rejected it in favour of release-ratifier's ownership of the
  release verdict.
- **G9 has no required-check enforcement at all:** an unenforced gate.
