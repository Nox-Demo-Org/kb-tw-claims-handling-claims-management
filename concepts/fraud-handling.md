---
type: Concept
title: Fraud Handling and Scoring
description: In the Tidewell claims platform, fraud evaluation operates alongside the initial triage and claim intake flow to detect high-risk claims and escalate suspicious activity to the counter-fraud team before handler settlement or payouts occur.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-handling-claims-management/blob/main/concepts/fraud-handling.md
tags:
- claims-management
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:46:10Z'
---

# Fraud Handling and Scoring

In the Tidewell claims platform, fraud evaluation operates alongside the initial triage and claim intake flow to detect high-risk claims and escalate suspicious activity to the counter-fraud team before handler settlement or payouts occur.

---

## Fraud Scoring Architecture

Fraud assessment is driven by the `fraud-scoring` service, which listens asynchronously to incoming [[ap:kb-tw-claims-intake-claims-intake/summaries/api-spec#claims-claim-reported|claims.claim.reported]] events emitted during intake. 

The composite fraud score is calculated using a weighted composite model:
- **Internal Signals (60%)**: Proprietary indicators calculated across customer claim history, policy status, and submission patterns.
- **External Fraud Data Vendor (40%)**: External data vendor API providing specialized identity, peril, and historical fraud scoring metrics.

```
                  claims.claim.reported
                            │
                            ▼
                    ┌───────────────┐
                    │ fraud-scoring │
                    └───────┬───────┘
            ┌───────────────┴───────────────┐
            ▼                               ▼
    Internal Signals (60%)         External Vendor (40%)
            │                               │
            └───────────────┬───────────────┘
                            │
               Combined Score > 0.8 ?
                     ├── Yes ──► Emits fraud.score.flagged ──► Counter-Fraud Team
                     └── No  ──► Standard Lifecycle Flow
```

---

## Escalation and Flagging (`fraud.score.flagged`)

When the combined fraud score evaluates to **greater than 0.8**, the claim is flagged:
- **Event Publication**: `fraud.score.flagged` is published to message brokers.
- **Counter-Fraud Escalation**: The claim is diverted from standard automated processing or fast-track settlement directly to the counter-fraud investigation team.
- **Lifecycle Integration**: Escalated claims are audited and reviewed prior to advancing through [[entities/claim|Claim]] state transitions or authorizing payouts in [[concepts/payout-and-approvals|Payout and Approvals]].

---

## Vendor Integration & Resilience

The external fraud vendor provides an SLA of **99.5% availability** with a **400 ms median response time**.

### August 2026 Latency Incident
On 27 August 2026, the external vendor experienced severe service degradation, resulting in response times exceeding 30 seconds for 47 minutes. 

Because the downstream HTTP client (`requests.post` in `app/vendor.py`) had no configured request timeout:
- Claim registration threads blocked indefinitely while waiting for vendor responses.
- A backlog of **1,140 claims** accumulated in registration queues.

### Resilience and Fallback Pattern
To prevent vendor downtime or latency from stalling the intake pipeline (see [[concepts/claim-lifecycle|Claim Lifecycle]]):
1. **HTTP Client Timeout**: Implemented a **2-second connection/read timeout** on external vendor HTTP calls.
2. **Fallback Scoring Mode**: If the external vendor fails or times out within 2 seconds, the system falls back to calculating the score based exclusively on **100% internal signals**.
3. **Review Flag**: Claims scored under fallback conditions are flagged as "marked for review" due to the altered scoring threshold dynamics resulting from the missing 40% vendor weighting.
