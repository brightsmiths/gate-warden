# ADR-0008: Structured feedback is carried in the check-run output, under a fixed redaction policy

**Status:** Proposed
**Date:** 2026-11-27

## Context

CogWorks needs structured, coarse feedback: what failed, why it matters, what is allowed.
Brightsmiths forbids direct hand-offs between services, and GitHub is the only system of record.
The feedback must never contain test source, and holdout failures may only appear as REQ-IDs.

## Decision

- Carry feedback in the check run's `output`:
  - a short human summary;
  - a `gate-feedback/v1` JSON block in a fenced code block in `output.text`;
  - up to a configured number of annotations (default 50) for visible findings with locations.
- Apply a fixed redaction policy in gate-warden (defence in depth on top of the packs):
  - never publish logs;
  - reduce `holdout` findings to their REQ-ID;
  - cap the length of all text fields and escape them;
  - truncate deterministically to stay within GitHub limits.
- Take the "why it matters" and "what is allowed" texts from the gate definition (human-owned,
  trusted). They never come from check output.

## Consequences

- CogWorks parses one well-defined block from a GitHub object it already reads.
- Output is bounded at 65 535 characters, so large failures are truncated, and the counts of
  dropped findings are recorded.
- Feedback is visible to anyone who can read the repository's checks. Redaction therefore has to
  be right for every reader, not only for CogWorks.

## Alternatives considered

- **PR comments:** noisy, and they become workflow state in a second place.
- **Direct API call to CogWorks:** a direct hand-off, which is forbidden.
- **Artifacts or log links:** risk exposing raw logs and test source.
