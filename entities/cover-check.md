---
type: Component
title: Cover Check
description: The cover check mechanism in claims-management is responsible for verifying policy coverage, validating claim perils, and determining the applicable policy excess during the claim assessment and triage stages of the claim lifecycle.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-claims-management/blob/main/entities/cover-check.md
tags:
- claims-management
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/claims-management/blob/HEAD/src/main/java/com/tidewell/claims/cover/CoverCheckRepository.java
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:10Z'
---

<!-- anchor: src/main/java/com/tidewell/claims/cover/CoverCheckRepository.java:L1-L1 -->

# Cover Check

The cover check mechanism in `claims-management` is responsible for verifying policy coverage, validating claim perils, and determining the applicable policy excess during the claim assessment and triage stages of the [[concepts/claim-lifecycle|claim lifecycle]].

## Responsibilities

* **Peril Coverage Verification**: Evaluates whether the incident peril associated with an incoming [[entities/claim|Claim]] is covered under the customer's policy. Claims submitted without peril validation (such as those originating from phone intake channels) undergo triage verification.
* **Excess Determination**: Retrieves and calculates the specific excess amount applicable to the claim's peril from policy cover records.
* **Cover Data Retrieval**: Provides data access abstractions via `CoverCheckRepository` for retrieving peril and excess details.

## Implementation & Data Access

Cover verification is executed by `CoverCheckRepository`. Rather than consuming policy-admin REST endpoints, `CoverCheckRepository` currently executes direct read queries against the `policy_cover` table in the `policy-admin` database.

### Architectural Context & Technical Debt

* **Direct Database Read**: Direct read access on the `policy_cover` table was introduced as a temporary measure during the November 2023 storm surge to circumvent performance bottlenecks on `GET /v1/policies/{id}`.
* **Remediation Status**: The temporary approval for direct table access expired in March 2024. A planned migration is tracked to transition `CoverCheckRepository` back to REST API communication (either via `GET /v1/policies/{id}` or a dedicated cover endpoint in `policy-admin`).
* For full architectural details and context, see [[decisions/direct-database-access-for-cover-check|Direct Database Access for Cover Check]].

## Dependencies

* **Database Tables**:
  * `policy_cover` (consumed directly from `policy-admin` database)
* **Domain Components & Entities**:
  * [[entities/claim|Claim Entity]]: Consumes cover and excess evaluations during claim triage and review.
  * [[concepts/claim-lifecycle|Claim Lifecycle]]: Relies on cover and excess confirmation before proceeding to settlement.
* **Related Specifications**:
  * [[summaries/api-spec|API Specification]]: Outlines interfaces interacting with claim records and data models.
