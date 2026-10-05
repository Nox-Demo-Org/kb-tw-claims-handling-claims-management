# Architecture decisions

One ADR per architecture decision the code or documents make evident.

## Pages

- [ADR: Direct Database Access for Cover Check](/decisions/direct-database-access-for-cover-check.md) — To maintain throughput and avoid claim registration delays during the storm surge, claims-management bypassed the REST API and introduced direct read access against policy-admin's internal policycover database table via CoverCheckReposit…
- [Decision: Event-Driven Claim Lifecycle](/decisions/event-driven-claim-lifecycle.md) — Managing claim state transitions via synchronous REST requests introduces tight coupling and operational risks.
