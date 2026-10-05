---
type: Interface Reference
title: API Specification
description: This document details the exposed REST endpoints, consumed external REST APIs, asynchronous messaging topics (AsyncAPI 2.6.0), and database interfaces used by the claims-management service.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-claims-management/blob/main/summaries/api-spec.md
tags:
- claims-management
- summaries
sources:
- resource: https://github.com/Nox-Demo-Org/claims-management/blob/HEAD/src/main/java/com/tidewell/claims/api/ClaimController.java
- resource: https://github.com/Nox-Demo-Org/claims-management/blob/HEAD//Users/akash/Documents/Projects/Project-NoX/demo/tidewell/sources/uploads/claims-events-asyncapi.yaml
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:10Z'
---

<!-- anchor: src/main/java/com/tidewell/claims/api/ClaimController.java:L1-L17 -->
<!-- anchor: /Users/akash/Documents/Projects/Project-NoX/demo/tidewell/sources/uploads/claims-events-asyncapi.yaml:L1-L15 -->

# API Specification

This document details the exposed REST endpoints, consumed external REST APIs, asynchronous messaging topics (AsyncAPI 2.6.0), and database interfaces used by the `claims-management` service.

---

## Exposed REST Endpoints

### Get Claim Details

Retrieves claim status, assigned handler information, timeline, and reserve values. Used primarily by internal tools and customer-facing interfaces such as the customer portal.

* **Path**: `GET /v1/claims/{id}`
* **Controller**: `com.tidewell.claims.api.ClaimController`
* **Related Entity**: [[entities/claim]]

#### Path Parameters
| Parameter | Type | Description |
|---|---|---|
| `id` | `string` | Unique identifier of the claim. |

#### Response (`200 OK`)
Returns a `ClaimView` object:
```json
{
  "id": "CLM-10293",
  "policyId": "POL-99201",
  "status": "assigned",
  "reportedAt": "2026-03-01T10:15:30Z",
  "handlerName": "Jane Doe",
  "handlerAssignedAt": "2026-03-01T14:22:00Z",
  "reservePence": 150000
}
```

#### Field Specifications
| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique claim identifier. |
| `policyId` | `string` | Associated policy identifier. |
| `status` | `string` | Current claim lifecycle status (e.g., `submitted`, `assigned`, `in_review`, `settled`, `paid`, `declined`, `withdrawn`). See [[concepts/claim-lifecycle]]. |
| `reportedAt` | `string` | ISO-8601 timestamp of when the claim was first reported. |
| `handlerName` | `string` | Name of the assigned claims handler (populated once assigned via [[entities/handler-assignment]]). |
| `handlerAssignedAt` | `string` | ISO-8601 timestamp of when handler assignment occurred. |
| `reservePence` | `long` | Current financial reserve allocated for the claim in pence. |

---

## Consumed External REST Endpoints

### Create Payout

* **Service**: `payments-gateway`
* **Path**: `POST /v1/payouts`
* **Triggered By**: [[entities/settlement-service]] / [[concepts/payout-and-approvals]]
* **Description**: Initiates disbursement of funds to the claimant once a settlement is agreed and necessary approvals are granted.
* **Controls**:
  * Payouts exceeding £25,000 require dual approval within `claims-management` before invoking this endpoint.
  * Invocations require an `idempotency_id` to prevent duplicate money transfers upon retry.

---

## Asynchronous Message Events

AsyncAPI specification version: `2.6.0` (`Tidewell claims events v1.4.0`). For architecture context, see [[decisions/event-driven-claim-lifecycle]].

```
+------------------+         claims.claim.reported         +-------------------+
|  claims-intake   | ------------------------------------> | claims-management |
+------------------+                                       +-------------------+
                                                              |         ^
       +------------------------------------------------------+         |
       | claims.handler.assigned                                        | payments.payout.sent
       | claims.claim.settled                                           |
       v                                                                |
+-------------------+                                      +-------------------+
| Downstream / Hubs |                                      | payments-gateway  |
+-------------------+                                      +-------------------+
```

### Consumed Events

#### [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]]
* **Producer**: [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]]
* **Consumers**: `claims-management`, `fraud-scoring`
* **Description**: Emitted when a new claim is reported by a customer or intake agent.
* **Payload Schema**:
```yaml
type: object
properties:
  claim_id:
    type: string
  policy_id:
    type: string
  customer_id:
    type: string
  peril:
    type: string
  incident_date:
    type: string
  description:
    type: string
  excess_amount:
    type: integer
```

#### `payments.payout.sent`
* **Producer**: `payments-gateway`
* **Consumer**: `claims-management`
* **Description**: Confirms the money has been disbursed to the claimant. Upon receipt, `claims-management` transitions the claim status to `paid`. See [[concepts/claim-lifecycle]].

#### `fraud.score.flagged`
* **Producer**: `fraud-scoring`
* **Consumer**: `claims-management`
* **Description**: Published when a fraud score exceeds `0.8`, routing the claim to the counter-fraud team for manual investigation. See [[concepts/fraud-handling]].
* **Payload Schema**:
```yaml
type: object
properties:
  claim_id:
    type: string
  score:
    type: number
```

---

### Published Events

#### `claims.handler.assigned`
* **Producer**: `claims-management`
* **Description**: Published immediately when a claim handler is assigned to a claim (target SLA: 1 working day). See [[entities/handler-assignment]].
* **Payload Schema**:
```yaml
type: object
properties:
  claim_id:
    type: string
  policy_id:
    type: string
  customer_id:
    type: string
  handler_name:
    type: string
  assigned_at:
    type: string
```

#### `claims.claim.settled`
* **Producer**: `claims-management`
* **Consumers**: `payments-gateway`, `notifications-hub`
* **Description**: Emitted when a settlement agreement is finalized with the claimant. Triggers payout processing and notification dispatch. See [[entities/settlement-service]].
* **Payload Schema**:
```yaml
type: object
properties:
  claim_id:
    type: string
  amount_pence:
    type: integer
  settled_at:
    type: string
```

---

## Direct Database Interfaces

| Table | Source Database | Access Type | Description |
|---|---|---|---|
| `policy_cover` | `policy-admin` | Direct Read (`SELECT`) | Used by [[entities/cover-check]] to evaluate excess amounts and peril coverage. Introduced as a temporary optimization during the November 2023 storm surge. See [[decisions/direct-database-access-for-cover-check]]. |
