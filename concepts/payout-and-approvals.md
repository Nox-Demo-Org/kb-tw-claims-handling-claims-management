---
type: Concept
title: Payouts and Approvals
description: This page outlines the financial approval controls, payment dispatch mechanics, idempotency rules, and reconciliation processes governing claim settlement payouts within claims-management.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-claims-management/blob/main/concepts/payout-and-approvals.md
tags:
- claims-management
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:10Z'
---

# Payouts and Approvals

This page outlines the financial approval controls, payment dispatch mechanics, idempotency rules, and reconciliation processes governing claim settlement payouts within `claims-management`.

---

## Dual Approval Policy (> £25,000)

Following an audit finding (Audit 2025-04, tracked under TWCLM-9), `claims-management` enforces a dual-approval requirement for high-value claims:

- **Threshold**: Any settlement payout exceeding **£25,000** requires a second approver before the service can dispatch the payout.
- **Execution Gate**: `POST /v1/payouts` cannot be called against `payments-gateway` until both primary and secondary approval signatures are recorded.
- **Escalation & Fallback**: If the assigned secondary approver is unavailable, the **Claims Operations manager** (available on rota via `#tw-claims`) is authorized to provide the secondary approval.

See [[entities/settlement-service]] and [[concepts/claim-lifecycle]] for settlement workflow transitions.

---

## Payout Execution & Idempotency Rules

When a claim is finalized and approved:
1. `claims-management` issues a request to `payments-gateway` via `POST /v1/payouts` (see [[summaries/api-spec]]).
2. `claims-management` publishes the `claims.claim.settled` event.
3. Upon successfully processing the payment, `payments-gateway` publishes `payments.payout.sent`.
4. `claims-management` consumes `payments.payout.sent` and updates the [[entities/claim]] status to `paid`.

### Idempotency Requirements
- **Critical Rule**: When investigating, retrying, or resending a failed or stuck payout, engineers **must never generate a new `idempotency_id`**.
- Resending a payout request with a new `idempotency_id` bypasses payment gateway deduplication and will result in moving money twice.
- Always reuse the original `idempotency_id` associated with the claim settlement attempt.

---

## Operational Monitoring and Daily Reconciliation

### Morning Payout Checks (09:00 Daily)
On-call engineers for Claims Handling perform operational checks:
- **Dashboard Review**: Inspect the payouts dashboard daily at 09:00 for payouts remaining in `pending` status for more than **2 hours**.
- **Troubleshooting**: A payout stuck in `pending` typically indicates that `payments-gateway` failed to publish `payments.payout.sent`. Check `payments-gateway` service logs filtering by the corresponding `claim_id`.

### Evening Reconciliation
Every evening, an automated/operational comparison is performed between:
- Settled claims event stream (`claims.claim.settled`)
- Completed payout event stream (`payments.payout.sent`)

**Escalation Trigger**: Any settled claim that has not received a corresponding `payments.payout.sent` confirmation after **24 hours** must be escalated immediately in `#tw-claims`.
