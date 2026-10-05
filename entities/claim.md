---
type: Component
title: Claim
description: The Claim entity is the core domain model in claims-management.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-claims-management/blob/main/entities/claim.md
tags:
- claims-management
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/claims-management/blob/HEAD/src/main/java/com/tidewell/claims/api/ClaimController.java
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:10Z'
---

<!-- anchor: src/main/java/com/tidewell/claims/api/ClaimController.java:L1-L17 -->

# Claim

The `Claim` entity is the core domain model in `claims-management`. It represents the lifecycle, status, reserve amounts, and assigned handling ownership of an insurance claim within the Tidewell insurance platform.

## Responsibilities

- **State Management**: Tracks and transitions the claim lifecycle from initial submission through handler assignment, review, settlement, payout, or closure.
- **Reserve and Financial Tracking**: Holds financial representations including `reserve_pence` (monetary amounts are stored as integer pence) and `excess_amount`.
- **Handler Association**: Maintains records of the assigned claims handler (`handler_id`, `handler_assigned_at`).
- **External & Customer Exposure**: Provides a read projection (`ClaimView`) for internal tracking and customer portal visibility through `GET /v1/claims/{id}` via `ClaimController`.

## Domain Model and Data Schema

According to the platform data dictionary, `Claim` is owned by `claims-management` and consists of the following key fields:

| Field | Type / Description |
| :--- | :--- |
| `id` | Unique claim identifier |
| `policy_id` | Identifier of the associated policy |
| `status` | Current lifecycle state of the claim |
| `peril` | Specific peril covered by the claim (evaluated against policy cover) |
| `handler_id` | Identifier of the assigned claims handler |
| `handler_assigned_at` | Timestamp indicating when the claim handler was assigned |
| `reserve_pence` | Financial reserve allocated for the claim, represented in integer pence |
| `excess_amount` | Policy excess applicable to the claim |

### Read Projection (`ClaimView`)
Exposed via `GET /v1/claims/{id}` in `src/main/java/com/tidewell/claims/api/ClaimController.java`:
```java
public record ClaimView(
    String id,
    String policyId,
    String status,
    String reportedAt,
    String handlerName,
    String handlerAssignedAt,
    long reservePence
) {}
```
*Note: `handlerName` is populated once a handler has been assigned to the claim.*

## Entity States and Lifecycle Transitions

A claim transitions through multiple distinct states across its lifecycle:

| Status | Set By | Description | Customer View Status |
| :--- | :--- | :--- | :--- |
| `submitted` | [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake]] | Claim reported and awaiting handler assignment | `Submitted` |
| `assigned` | `claims-management` | A claim handler has been assigned | `Submitted` |
| `in_review` | `claims-management` | Handler is actively assessing and investigating | `In review` |
| `settled` | `claims-management` | Settlement amount is agreed | `Settled` |
| `paid` | `claims-management` | Settlement payout sent via `payments.payout.sent` | `Paid` |
| `declined / withdrawn` | `claims-management` | Claim closed without payout | `Closed` |

### Key State Transition Triggers

1. **Intake to Submitted**: Initiated by `claims-intake` when a user reports an incident, emitting the [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] event.
2. **Assignment (`submitted` -> `assigned`)**: Handled by [[entities/handler-assignment|handler assignment]] logic (SLA target of 1 working day), emitting `claims.handler.assigned`.
3. **Assessment & Cover Check (`assigned` -> `in_review`)**: Handler evaluates cover eligibility and peril excess via [[entities/cover-check|cover check]], reading `policy_cover`.
4. **Settlement (`in_review` -> `settled`)**: Evaluated by [[entities/settlement-service|settlement processing]], triggering [[concepts/payout-and-approvals|dual approval]] if the payout exceeds £25,000, and publishing `claims.claim.settled`.
5. **Payout (`settled` -> `paid`)**: Invokes `POST /v1/payouts` against `payments-gateway`. Upon receiving the `payments.payout.sent` event, the claim transitions to `paid`.

For an end-to-end breakdown of the claim workflow, see [[concepts/claim-lifecycle]].

## Dependencies

- **[[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake]]**: Consumes [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] events to initialize new claim records.
- **[[entities/handler-assignment|Handler Assignment Subsystem]]**: Manages handler allocation, assignment timestamps, and broadcasts `claims.handler.assigned`.
- **[[entities/cover-check|Cover Check]]**: Interacts with the `policy_cover` table directly to verify peril limits and policy excess (see [[decisions/direct-database-access-for-cover-check|Direct Database Access for Cover Check]]).
- **[[entities/settlement-service|Settlement Service]] & `payments-gateway`**: Submits payouts via `POST /v1/payouts`, emits `claims.claim.settled`, and consumes `payments.payout.sent` events (see [[decisions/event-driven-claim-lifecycle|Event-Driven Claim Lifecycle]]).
- **Customer Portal / REST Consumers**: Accesses claim status, timelines, and reserves via [[summaries/api-spec|Claim API endpoints (`GET /v1/claims/{id}`)]].
