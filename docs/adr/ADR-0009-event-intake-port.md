# ADR-0009: Event intake through a port; App webhook at L1, queue-keeper later

**Status:** Proposed
**Date:** 2026-11-27

## Context

The brief allows the trigger to come via queue-keeper (NATS JetStream) or, during M1, directly
from the App webhook. queue-keeper's routing to gate-warden is a later tweak (M3).

## Decision

- Define an **event-source port** that delivers normalised, verified PR events to the business
  logic.
- **L1:** an HTTP adapter for the gate-warden App webhook. It verifies `X-Hub-Signature-256`
  before parsing.
- **L2 (planned):** a NATS JetStream adapter for queue-keeper. It refuses events that
  queue-keeper marked as untrusted.
- Both adapters feed the same idempotent pipeline. Eligibility (fork or trusted actor) is always
  re-checked by gate-warden against the PR data it fetches from GitHub. It is not taken from the
  event alone.

## Consequences

- At L1, deliveries lost while the service is down are recovered by reconciliation (GW-REQ-074).
- Switching to queue-keeper is a configuration change, not a code-path rewrite.

## Alternatives considered

- **Wait for queue-keeper:** blocks M1.
- **Polling only:** higher latency and more API usage. It is kept only as reconciliation.
