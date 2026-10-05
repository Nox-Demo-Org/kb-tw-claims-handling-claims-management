---
type: Architecture Decision
title: 'Decision: Event-Driven Claim Lifecycle'
description: Managing claim state transitions via synchronous REST requests introduces tight coupling and operational risks.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-claims-management/blob/main/decisions/event-driven-claim-lifecycle.md
tags:
- claims-management
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:10Z'
---

# Decision: Event-Driven Claim Lifecycle

## Status
Accepted

## Context
The claims handling process involves multiple independent services across the Tidewell platform, including `claims-intake`, `claims-management`, `fraud-scoring`, `payments-gateway`, and `notifications-hub`. 

Managing claim state transitions via synchronous REST requests introduces tight coupling and operational risks. For instance, slow upstream/downstream integrations (such as external vendor integrations during high-load storm surge events or slow scoring APIs) can cause synchronous intake or settlement request threads to block and fail.

To support high scalability, decouple service responsibilities, and maintain an extensible audit trail across stage transitions, the architecture mandates an asynchronous, event-driven pattern for claim lifecycle milestones.

## Decision
We adopted an asynchronous event publishing and consumption model using defined message channels specified in `claims-events-asyncapi.yaml` (AsyncAPI version 2.6.0). Stage transitions throughout the [[concepts/claim-lifecycle|Claim Lifecycle]] are coordinated via domain events:

1. **Intake and Registration**:
   - `claims-intake` publishes [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|`claims.claim.reported`]] when a customer or agent submits a claim.
   - Consumers: `claims-management` consumes this event to initialize the [[entities/claim|Claim]] entity and begin triage/cover evaluation; `fraud-scoring` consumes it concurrently to run risk evaluation models.
   - Payload schema: `claim_id`, `policy_id`, `customer_id`, `peril`, `incident_date`, `description`, `excess_amount`.

2. **Counter-Fraud Flagging**:
   - `fraud-scoring` asynchronously flags high-risk submissions by publishing `fraud.score.flagged` for scores above 0.8.
   - Payload schema: `claim_id`, `score`. Refer to [[concepts/fraud-handling|Fraud Handling]].

3. **Handler Assignment**:
   - `claims-management` publishes `claims.handler.assigned` once a claim handler is allocated to an active claim.
   - Payload schema: `claim_id`, `policy_id`, `customer_id`, `handler_name`, `assigned_at`. See [[entities/handler-assignment|Handler Assignment]].

4. **Settlement and Payouts**:
   - `claims-management` publishes `claims.claim.settled` upon completing settlement negotiations and approvals.
   - Consumers: `payments-gateway` (to trigger payment execution) and `notifications-hub` (to notify the customer).
   - Payload schema: `claim_id`, `amount_pence`, `settled_at`. Refer to [[entities/settlement-service|Settlement Service]] and [[concepts/payout-and-approvals|Payout and Approvals]].
   - `claims-management` consumes `payments.payout.sent` from `payments-gateway` to confirm payment completion and mark claims as paid.

### Governance and Event Evolution
Per platform engineering conventions:
- Removing or renaming fields on any published event topic requires explicit cross-team agreement in `#tw-architecture` before deployment.
- Changes must adhere to Tidewell event schema standards.

## Consequences

### Positive
- **Service Decoupling**: Upstream intake services are isolated from downstream delays in cover checks, fraud scoring, and handler assignment.
- **Resilience**: Temporary downtime or latency spikes in consumers (e.g., scoring services or notification senders) do not block claim creation or settlement state writes.
- **Extensibility**: New consumers (e.g., analytics, external regulators, future notification channels) can subscribe to lifecycle channels (`claims.handler.assigned`, `claims.claim.settled`) without modifying `claims-management`.

### Negative & Trade-offs
- **Eventual Consistency**: State transitions across services are eventually consistent, requiring handlers and frontend status endpoints (such as `GET /v1/claims/{id}`) to account for transitional processing states.
- **Contract Rigidity**: Schema evolution requires formal synchronization across multiple teams to prevent breaking consumers on core channels.
- **Operational Complexity**: Tracing a single claim's lifecycle requires distributed tracing and correlation IDs across multiple message topics.

## Related Artifacts
- API Specification: [[summaries/api-spec|API & Message Specification]]
- Domain Entities: [[entities/claim|Claim Entity]], [[entities/handler-assignment|Handler Assignment]], [[entities/settlement-service|Settlement Service]]
- Architecture Context: [[index|claims-management Architecture Overview]]
- Related Decision: [[decisions/direct-database-access-for-cover-check|Direct Database Access for Cover Check]]
