---
type: Architecture Decision
title: 'ADR: Direct Database Access for Cover Check'
description: To maintain throughput and avoid claim registration delays during the storm surge, claims-management bypassed the REST API and introduced direct read access against policy-admin's internal policycover database table via CoverCheckReposit…
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-claims-management/blob/main/decisions/direct-database-access-for-cover-check.md
tags:
- claims-management
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:10Z'
---

# ADR: Direct Database Access for Cover Check

## Status
Deprecated / Pending Remediation (Temporary exception approved in November 2023, expired March 2024; remediation tracked under `TWCLM-7`)

## Context
During the storm surge of November 2023, the platform experienced high claim intake volumes. At the time, evaluating peril coverage and excess amounts required calling `policy-admin`'s REST endpoint (`GET /v1/policies/{id}`). The endpoint was too slow to handle the surge load, creating severe latency bottlenecks in the triage and cover evaluation stages of the [[concepts/claim-lifecycle|claim lifecycle]].

To maintain throughput and avoid claim registration delays during the storm surge, `claims-management` bypassed the REST API and introduced direct read access against `policy-admin`'s internal `policy_cover` database table via `CoverCheckRepository`. This bypass violated standard service boundary architecture (ADR-0003), but was approved as a temporary measure with an agreed sunset date of March 2024.

## Decision
1. **Direct Database Querying**: Allowed `CoverCheckRepository` in `claims-management` to maintain direct read-only access to `policy-admin`'s `policy_cover` table.
2. **Functional Scope**: Restricted access to reading peril coverage and determining applicable excess amounts during the [[entities/cover-check|cover check]] process.
3. **Temporary Nature**: Bound the exception to a sunset target of March 2024, after which `claims-management` was required to revert to HTTP/REST consumption.

## Consequences

### Positive / Immediate Benefits
- Eliminated API latency bottlenecks during the November 2023 storm surge, sustaining high claim triage throughput.
- Provided low-latency read access to peril and excess configuration data.

### Negative / Technical Debt
- **Architectural Violation**: Direct access across service database boundaries violates service encapsulation (ADR-0003) and introduces tight schema coupling.
- **Blocking Upstream Refactoring**: The direct database connection currently blocks the `policy-admin` team from splitting and refactoring the `policy_cover` table.
- **Overdue Expiration**: Direct database access persisted past the March 2024 deadline and remains in production.

### Planned Remediation (`TWCLM-7`)
- Performance on `policy-admin` has improved significantly, with p95 response times reaching ~80 ms.
- Under Jira ticket `TWCLM-7`, `CoverCheckRepository` will be refactored to stop reading `policy_cover` directly.
- The repository will migrate to consume `policy-admin`'s `GET /v1/policies/{id}` or a newly exposed cover-specific REST endpoint, decoupling the database dependencies as documented in [[summaries/api-spec]].

## Related Artifacts
- Repository class: `src/main/java/com/tidewell/claims/cover/CoverCheckRepository.java`
- Consumed database table: `policy_cover` (owned by `policy-admin`)
- Remediation Jira issue: `TWCLM-7`
- Entity documentation: [[entities/cover-check]]
- System overview: [[index]]
