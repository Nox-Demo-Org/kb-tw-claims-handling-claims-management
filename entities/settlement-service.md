---
type: Component
title: Settlement Service
description: The SettlementService in the claims-management service is responsible for managing claim settlement agreements, applying authorization controls on payouts, initiating payment transactions with the external payments-gateway, and updating…
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-claims-management/blob/main/entities/settlement-service.md
tags:
- claims-management
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/claims-management/blob/HEAD/src/main/java/com/tidewell/claims/settlement/SettlementService.java
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:10Z'
---

<!-- anchor: src/main/java/com/tidewell/claims/settlement/SettlementService.java:L1-L1 -->

# Settlement Service

The `SettlementService` in the `claims-management` service is responsible for managing claim settlement agreements, applying authorization controls on payouts, initiating payment transactions with the external `payments-gateway`, and updating claim lifecycle states.

## Responsibilities

- **Settlement Finalization**: Transitions the [[entities/claim]] status to `settled` once an agreed settlement figure has been reached with the policyholder or third party (see [[concepts/claim-lifecycle]]).
- **Dual Approval Enforcement**: Enforces dual-approval controls on settlements with payouts exceeding £25,000 (introduced following Audit finding 2025-04 via TWCLM-9). Payouts above this threshold must be approved by a second approver (or the Claims Operations manager if the approver is away) before payout execution can proceed. See [[concepts/payout-and-approvals]].
- **Event Publishing**: Emits the `claims.claim.settled` event upon settlement agreement and authorization, alerting downstream systems and initiating reconciliation tracking (see [[decisions/event-driven-claim-lifecycle]] and [[summaries/api-spec]]).
- **Payout Execution**: Calls the external `payments-gateway` via `POST /v1/payouts` to disburse funds. All payment requests must maintain their original `idempotency_id` during retries to prevent duplicate financial transactions.
- **Payout Confirmation Handling**: Consumes `payments.payout.sent` events from `payments-gateway` to confirm fund delivery and transition the claim status from `settled` to `paid`.

## Settlement and Payout Workflow

```
[Claim in Review / Agreed] 
           │
           ▼
[Settlement Finalization]
           │
   Amount > £25,000?
    ├── Yes ──> [Dual Approval Required] ──> [Approved by Second Approver / Claims Ops]
    │                                                      │
    └── No ────────────────────────────────────────────────┘
                               │
                               ▼
            [Publish: claims.claim.settled]
                               │
                               ▼
            [Invoke: POST /v1/payouts (with idempotency_id)]
                               │
                               ▼
            [Consume: payments.payout.sent]
                               │
                               ▼
                 [Claim Status: paid]
```

### High-Value Payout Approvals (£25,000 Threshold)
- Settlements under or equal to £25,000 can proceed directly to payout dispatch once settled by the assigned handler.
- Settlements strictly exceeding £25,000 enter a pending approval state requiring sign-off by a secondary authorized approver or the Claims Operations manager before `POST /v1/payouts` is invoked.

### Operational Idempotency and Reconciliation
- **Idempotency**: Payout retries must reuse the initial `idempotency_id`. Generating a new ID on retrying a stuck payout risks executing duplicate payments.
- **Pending Payout Monitoring**: Payouts remaining in `pending` for more than 2 hours are reviewed during daily 09:00 checks for missing `payments.payout.sent` events.
- **Daily Reconciliation**: Evening reconciliation compares emitted `claims.claim.settled` events against received `payments.payout.sent` events. Any settled claim without a confirmed payout after 24 hours is flagged for operational investigation in `#tw-claims`.

## Dependencies

- **Outbound REST Services**:
  - `POST /v1/payouts` (`payments-gateway`): Triggers the payment transfer using payment details and an `idempotency_id`.
- **Event Messaging**:
  - Emits `claims.claim.settled`: Published when a claim settlement is finalized and ready for payout.
  - Consumes `payments.payout.sent`: Ingested from `payments-gateway` when the payout has been successfully sent.
- **Internal Domain Models & Workflows**:
  - [[entities/claim]]: Claim domain model for tracking financial reserves, settlement amounts, and transitions between `settled` and `paid`.
  - [[concepts/payout-and-approvals]]: Payout governance rules, dual approval policies, and operational runbook workflows.
  - [[concepts/claim-lifecycle]]: Broader state machine transitions across intake, review, settlement, and closure.
