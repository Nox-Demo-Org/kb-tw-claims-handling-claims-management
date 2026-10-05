---
type: Concept
title: Claim Lifecycle
description: The claims-management service manages the end-to-end lifecycle of an insurance claim from initial reporting through triage, assignment, investigation, settlement approval, payment execution, and closure.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-claims-management/blob/main/concepts/claim-lifecycle.md
tags:
- claims-management
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:10Z'
---

# Claim Lifecycle

The `claims-management` service manages the end-to-end lifecycle of an insurance claim from initial reporting through triage, assignment, investigation, settlement approval, payment execution, and closure. The lifecycle relies on an asynchronous event-driven architecture (see [[decisions/event-driven-claim-lifecycle]]) combined with synchronous endpoints for customer status visibility.

---

## Lifecycle State Machine

The claim transitions through the following core lifecycle statuses:

| Internal Status | Set By | Meaning | Customer-Facing Status (`ClaimView`) |
| :--- | :--- | :--- | :--- |
| `submitted` | `claims-intake` | Claim reported and registered; awaiting triage and handler assignment. | `Submitted` |
| `assigned` | `claims-management` | A claims handler has been assigned to the claim. | `Submitted` |
| `in_review` | `claims-management` | The assigned handler is actively assessing the claim, cover, and evidence. | `In review` |
| `settled` | `claims-management` | Settlement terms and payout amount have been agreed upon. | `Settled` |
| `paid` | `claims-management` | Payment execution confirmed via payment gateway event. | `Paid` |
| `declined / withdrawn` | `claims-management` | Claim has been closed without issuing a payment. | `Closed` |

---

## Detailed Lifecycle Stages

```
[ Intake (claims-intake) ]
           │
           ▼ (claims.claim.reported)
    ┌──────────────┐
    │  submitted   │
    └──────┬───────┘
           │ Assign Handler (target: 1 working day)
           ▼ (claims.handler.assigned)
    ┌──────────────┐
    │   assigned   │
    └──────┬───────┘
           │ Begin Assessment & Cover Check (policy_cover)
           ▼
    ┌──────────────┐
    │  in_review   │ ──(Fraud / Counter-fraud flag)──> [ fraud.score.flagged ]
    └──────┬───────┘
           │
    ┌──────┴─────────────────────────────────┐
    │ Settlement Agreed                      │ Declined / Customer Withdrawn
    ▼ (claims.claim.settled)                 ▼
┌──────────────┐                     ┌──────────────────────┐
│   settled    │                     │ declined / withdrawn │
└──────┬───────┘                     └──────────────────────┘
       │ POST /v1/payouts (Dual approval if > £25,000)
       ▼
[ payments-gateway ]
       │
       ▼ (payments.payout.sent)
┌──────────────┐
│     paid     │
└──────────────┘
```

### 1. Intake and Registration
- Claims are initially reported by the customer or agent via the intake service.
- `claims-management` ingests newly reported claims by consuming the [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] event emitted by `claims-intake` (see [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]]).
- The claim entity is initialized in the `submitted` status with peril, incident date, policy details, and initial reserve amounts (see [[entities/claim]]).

### 2. Handler Assignment & Triage
- Claims are assigned to a claims handler with an operational target SLA of **1 working day** (see [[entities/handler-assignment]]).
- Upon assignment, the claim transitions to `assigned`, setting `handlerName` and `handlerAssignedAt`.
- The service publishes the `claims.handler.assigned` asynchronous event:
  - Payload: `claim_id`, `policy_id`, `customer_id`, `handler_name`, `assigned_at`.

### 3. Cover Verification & Investigation
- The claim transitions to `in_review` as the assigned handler begins assessment.
- **Cover Check & Excess Verification**: The handler verifies peril coverage and excess amounts by querying policy data. As a temporary measure introduced during the November 2023 storm surge, this queries `policy-admin`'s `policy_cover` database table directly rather than calling external REST endpoints (see [[entities/cover-check]] and [[decisions/direct-database-access-for-cover-check]]).
- **Fraud Evaluation**: In parallel, fraud scoring evaluates the claim using internal signals and external vendor scoring. If fraud risk exceeds `0.8`, `fraud.score.flagged` is emitted, routing the claim to the counter-fraud team (see [[concepts/fraud-handling]]).

### 4. Settlement & Approvals
- Once the assessment concludes and the payout amount is finalized, the claim transitions to `settled`.
- The service publishes the `claims.claim.settled` event:
  - Payload: `claim_id`, `amount_pence`, `settled_at`.
- **Dual Approval Workflow**: Payouts exceeding **£25,000** require dual approval (a second handler approval or sign-off by the Claims Operations manager) before payout dispatch (see [[concepts/payout-and-approvals]] and [[entities/settlement-service]]).

### 5. Payout Execution and Completion
- After approval, `claims-management` invokes `POST /v1/payouts` on the `payments-gateway` using a persistent `idempotency_id`.
- The service listens for the `payments.payout.sent` event from `payments-gateway`.
- Upon receipt of `payments.payout.sent`, the claim transitions to its terminal success status: `paid`.

### 6. Decline or Withdrawal
- If the claim is rejected (e.g., peril not covered, excess exceeds claim value, fraud confirmation) or withdrawn by the policyholder, the claim transitions to `declined / withdrawn`.
- The customer tracker displays this terminal state as `Closed`.

---

## Tracking and Customer Visibility

Claim status, timeline updates, and handler details are exposed synchronously to client applications (such as the customer portal) via the `ClaimController` (see [[summaries/api-spec]]):

```http
GET /v1/claims/{id}
```

**Response View Model (`ClaimView`)**:
* `id` (`String`): Unique identifier of the claim.
* `policyId` (`String`): Identifier of the linked insurance policy.
* `status` (`String`): Current claim status.
* `reportedAt` (`String`): Timestamp of initial claim reporting.
* `handlerName` (`String`): Name of the assigned claims handler (populated once assigned).
* `handlerAssignedAt` (`String`): Timestamp when the handler was assigned.
* `reservePence` (`long`): Current reserve amount allocated for the claim in pence.
