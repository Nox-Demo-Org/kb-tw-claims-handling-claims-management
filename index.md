---
okf_version: '0.2'
title: claims-management
description: claims-management is a core backend service in the Tidewell insurance platform responsible for managing the lifecycle of insurance claims from initial intake through investigation, handler assignment, cover verification, triage, and sett…
generated:
  at: '2026-10-05T12:46:10Z'
---

# claims-management

claims-management is a core backend service in the Tidewell insurance platform responsible for managing the lifecycle of insurance claims from initial intake through investigation, handler assignment, cover verification, triage, and settlement payout processing.

### Core Architecture & Data Flows
- **Claim Intake**: Consumes [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims-intake (claims.claim.reported)]] events emitted by `claims-intake` upon initial claim submission.
- **Handler Assignment**: Assigns claims handlers to active claims and broadcasts `claims.handler.assigned` events.
- **Cover & Excess Verification**: Evaluates policy excess and peril coverage using `CoverCheckRepository` (reading `policy_cover`).
- **Settlement & Payouts**: Handles settlement negotiations, applies dual-approval controls for payouts exceeding £25,000, emits `claims.claim.settled` events, triggers payments via `payments-gateway`, and listens for `payments.payout.sent` to mark claims as paid.
- **External & Customer API**: Serves claim progress, handler status, reserve amounts, and timelines via `GET /v1/claims/{id}`.

### Key Architectural Decisions & Debt
- **Direct Database Access**: Reads `policy_cover` directly from the `policy-admin` database rather than calling policy APIs (temporarily introduced in Nov 2023 to handle storm surge load, pending migration back to REST endpoints).
- **Dual Approval for High Value Settlements**: Requires a second approver before issuing payouts over £25,000.
- **Event-Driven Lifecycle Updates**: Leverages asynchronous messaging for claim intake, assignment notifications, and settlement execution.

<!-- okf:contents -->

## Contents

- [Concepts and flows](/concepts/index.md) — 3 pages. Flows, lifecycles and cross-cutting mechanisms.
- [Architecture decisions](/decisions/index.md) — 2 pages. One ADR per architecture decision the code or documents make evident.
- [Components and data models](/entities/index.md) — 4 pages. One page per significant component and core data model.
- [Interfaces and references](/summaries/index.md) — 1 page. API, event and module references for the application.
- [Change log](/log.md) — every generation and sync, newest first.

<!-- /okf:contents -->
