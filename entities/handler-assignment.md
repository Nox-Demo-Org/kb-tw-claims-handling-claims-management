---
type: Component
title: Handler Assignment
description: The handler assignment mechanism in claims-management manages the allocation of reported insurance claims to claims handlers, updates claim lifecycle statuses, and broadcasts assignment events across the platform.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-claims-management/blob/main/entities/handler-assignment.md
tags:
- claims-management
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/claims-management/blob/HEAD/src/main/java/com/tidewell/claims/assignment/HandlerAssignmentService.java
- resource: https://github.com/Nox-Demo-Org/claims-management/blob/HEAD/docs/adr/0002-handler-assignment.md
- resource: https://github.com/Nox-Demo-Org/claims-management/blob/HEAD//Users/akash/Documents/Projects/Project-NoX/demo/tidewell/sources/uploads/claims-events-asyncapi.yaml
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:10Z'
---

<!-- anchor: src/main/java/com/tidewell/claims/assignment/HandlerAssignmentService.java:L1-L1 -->
<!-- anchor: docs/adr/0002-handler-assignment.md:L1-L1 -->
<!-- anchor: /Users/akash/Documents/Projects/Project-NoX/demo/tidewell/sources/uploads/claims-events-asyncapi.yaml:L1-L15 -->

# Handler Assignment

The handler assignment mechanism in `claims-management` manages the allocation of reported insurance claims to claims handlers, updates claim lifecycle statuses, and broadcasts assignment events across the platform.

## Responsibilities

- **Handler Allocation**: Assigns an internal claims handler to newly reported claims received from [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]].
- **Status Progression**: Updates the [[entities/claim]] status from `submitted` to `assigned` once a handler is allocated, before advancing to `in_review` when assessment begins (see [[concepts/claim-lifecycle]]).
- **Event Notification**: Broadcasts the `claims.handler.assigned` event via the asynchronous messaging bus upon assigning a handler.
- **SLA Tracking**: Operates against target operational timelines to ensure claims are picked up promptly after intake.

## Operational SLAs and Status Visibility

- **Target SLA**: The target turnaround time for handler assignment is **1 working day** from initial claim submission.
- **Customer Visibility**: In the claim status lifecycle, while the internal state transitions from `submitted` to `assigned`, the customer-facing portal view remains displayed as `Submitted` until the handler begins active assessment, transitioning the claim to `in_review` (`In review`). Handler status and claim timeline details can be queried via `GET /v1/claims/{id}` (see [[summaries/api-spec]]).

## Event Specification: `claims.handler.assigned`

When a handler is assigned to a claim, `claims-management` publishes a message to the `claims.handler.assigned` channel (as documented in `claims-events-asyncapi.yaml`).

### Payload Schema

| Field | Type | Description |
| :--- | :--- | :--- |
| `claim_id` | `string` | Unique identifier for the claim |
| `policy_id` | `string` | Identifier of the associated policy |
| `customer_id` | `string` | Identifier of the customer filing the claim |
| `handler_name` | `string` | Name of the assigned claims handler |
| `assigned_at` | `string` | ISO timestamp of when the assignment occurred |

*Note: According to the AsyncAPI specification, there are currently no registered external consumers consuming `claims.handler.assigned`.*

## Dependencies

- **Inbound Events**:
  - [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]]: Emitted by [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] to initiate the claim handling workflow within `claims-management`.
- **Outbound Events**:
  - `claims.handler.assigned`: Emitted by `claims-management` upon successful allocation of a handler.
- **Related Entities & Concepts**:
  - [[entities/claim]]: Updates handler assignment details and transitions the claim state.
  - [[concepts/claim-lifecycle]]: Context on the end-to-end lifecycle progression from `submitted` to `assigned` and `in_review`.
  - [[decisions/event-driven-claim-lifecycle]]: Architectural context on using asynchronous messaging for claim stage updates.
  - [[summaries/api-spec]]: API contracts for exposed endpoints (such as `GET /v1/claims/{id}`) and event channels.
