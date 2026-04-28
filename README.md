# FinOps Case Study: AWS Cost Allocation Gap and Governance Remediation

**Restored 100% cost allocation visibility after a 17% attribution gap in AWS billing data.** Cloud billing analyst with 12 years in financial controls and audit, applied to a cloud cost attribution failure using AWS-native forensic and governance tooling.

---

## The 30-Second Version

- Identified a 17% cost allocation gap where $53 of $315 in monthly spend was invisible to Finance chargeback reporting
- Proved root cause at creation time using CUR and CloudTrail `tagSpecificationSet` evidence. Tags were never submitted with the API call, confirming a provisioning time control failure rather than post creation drift
- Isolated failure to console based provisioning outside governance controls, confirmed via IAM principal, UserAgent string, and session attribution chain
- Built an event driven detection pipeline using CloudTrail, EventBridge, and Lambda with EC2 and S3 branching logic
- Restored 100% allocation coverage and reduced detection time from 30 days to low minute-level under normal conditions

---

## Results

| Metric | Before | After |
|---|---|---|
| Allocation coverage | ~83% | 100% |
| Mean time to detect | ~30 days | Low minute-level† |
| Monthly unallocated spend | $53 | $0 |
| Annualized attribution gap | $636 | $0 |
| Chargeback accuracy | Incomplete | Restored to 100% |
| Finance reconciliation effort | ~4 hrs/month | Materially reduced |

†Event driven detection with near real time characteristics. CloudTrail delivery varies by region and load. Treat this as a typical observed range, not a hard SLA.

---

## What This Is

This is a financial reporting integrity problem, not a cost reduction problem. The spend was billed correctly. Finance could not see it, attribute it, or charge it back. Unattributed spend breaks chargeback accuracy, distorts forecast baselines, and produces Savings Plans recommendations sized against incomplete data. Restoring allocation coverage fixes all three.

The investigation started at monthly reconciliation, confirmed billing accuracy, then used CUR and Athena SQL to isolate resources with NULL tags across all three required dimensions. CloudTrail forensics proved the tags were never submitted at creation time. That distinction matters: it separates a provisioning control failure from a configuration drift problem and identifies the exact control point that must be fixed.

---

## Architecture

```
EC2 RunInstances / S3 CreateBucket / S3 PutBucketTagging
             │
             ▼
       AWS CloudTrail
    (all API-level events)
             │
             ▼
    Amazon EventBridge
  (finops-tag-compliance-monitor)
             │
             ▼
        AWS Lambda
   (finops-tag-validator)
   Checks: Environment · Project · Owner
        │              │
        ▼              ▼
  Slack: Finance   Slack: Owner
     Channel        Direct DM
```

Event driven detection validates required tags at or near resource creation and routes violations to Finance and the responsible owner. Service specific logic is applied where tagging behaviour differs across AWS services.

---

## Investigation Flow

**Phase 1: Invoice validation.** Confirmed billing was accurate before investigating allocation. The failure was downstream in the attribution layer, not in the invoice.

**Phase 2: CUR analysis.** Used Athena SQL against CUR to identify resources with NULL tags across all three required columns simultaneously. The pattern of all three tags absent on every billing line item for the same resource ruled out accidental omission and shifted the investigation to provisioning path forensics.

**Phase 3: CloudTrail forensics.** Filtered CloudTrail to the `RunInstances` event for the identified resource. The `tagSpecificationSet` contained only the `Name` tag. `Environment`, `Project`, and `Owner` were never submitted with the API call. A targeted `AssumeRole` query scoped to the principal ARN, bounded to the 30 minute pre incident window, returned zero results. No role assumption chain, no federation context, no enforced identity boundary at the provisioning layer.

**Phase 4: Scope expansion.** Confirmed the issue was systemic across EC2 and S3. RDS was fully compliant. Partial tagging in S3 produced the same Finance invisible outcome as no tagging.

---

## Root Cause

**Primary failure:** No preventive tagging control existed at resource creation time. No SCP, IAM condition key, IaC guardrail, or curated provisioning interface enforced required tags before the resource reached the billing layer.

**Contributing factor:** Use of non attributable principals removed enforceable ownership mapping. Without a traceable identity chain, post incident ownership cannot be reliably assigned and targeted enforcement cannot be designed around known roles.

**Design decision:** Automated tag back remediation was evaluated and rejected. Default tags produce chargeback reports that assign cost to the wrong team, distort showback data, and corrupt forecast baselines. The safer approach was to surface the gap accurately and alert the responsible principal.

---

## Control Approach

Detection was chosen as the first control because no compliance baseline existed. Enforcement before a baseline breaks CI/CD pipelines and managed service roles in ways that are harder to govern than the original problem. The intended progression is:

1. **Detect** (implemented): EventBridge and Lambda catch violations within minutes and alert Finance and the resource owner
2. **Baseline**: Tag compliance tracked as a first class FinOps KPI
3. **Stabilize**: MTTR and repeat violations measured by principal
4. **Enforce**: SCP guardrails applied after compliance exceeds 95%

Scoped enforcement can be introduced early for high risk scenarios. Broad enforcement across existing workloads follows once compliance stabilizes.

---

## Key Technologies

| Layer | Technology |
|---|---|
| Billing data | AWS Cost and Usage Report (CUR) |
| Query engine | Amazon Athena |
| Forensics | AWS CloudTrail |
| Event capture | Amazon EventBridge |
| Tag validation | AWS Lambda (Python 3.12+) |
| Governance | AWS Organizations Tag Policies and SCP |
| Alerting | Slack (Finance channel and resource owner DM) |

---

## Forward KPIs

| Metric | Target | Measured By |
|---|---|---|
| MTTR | Under 24 hrs | Time from alert to tag compliance restored |
| Repeat violation rate | Below 10% in 30 days | Same principal in repeated violations |
| Compliance trend | 95% within 60 days | Tagged spend divided by total spend |
| Tag validity rate | 90% within 60 days | Valid tag values, not just presence |

---

## Full Technical Report

Full investigation details including SQL queries, CloudTrail evidence, Lambda logic, IAM conditions, failure modes, and architecture are available in the main [technical investigation report](https://github.com/AthertHa-FinOps/aws-finops-cost-allocation-investigation).
