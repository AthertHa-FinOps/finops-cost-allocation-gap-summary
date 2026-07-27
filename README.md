# Cloud Cost Allocation Investigation: Tracing a 17% Chargeback Gap to Its Root Cause

**I traced a cloud cost allocation gap back to its root cause using captured AWS evidence, then built and tested a control to catch it earlier.**

Cloud spend can be billed correctly and still be useless for financial reporting. If you can't tell which team a charge belongs to, you can't bill it 
back to them. 
This project applies the reconciliation and audit-readiness habits I built over 13 years in regulated real estate transactions to that problem. AWS is 
just the 
environment. The transferable work is reconciliation, evidence collection, root cause analysis, and control design.

---

## Scope and Evidence Basis

This is a self-directed project built in a personal AWS sandbox account. It is not a professional engagement or the output of a prior employer. It has 
two layers, and 
I want to be clear about which is which rather than leave you to work it out.

**The forensic layer is real evidence.** The EC2 instance I investigated, `i-0c9cfb67280fe44ee`, was genuinely provisioned in this account, in `us-
east-2`, on 22 
January 2026. CloudTrail recorded the API call. The EC2 console recorded the resulting resource. Both are reproduced below, and the timestamps match 
across the two 
consoles.

**The cost layer is a scenario dataset.** A sandbox running one small instance for two hours produces a few dollars of spend. There is no allocation 
gap to find at 
that scale. So I priced a representative small-production architecture in the AWS Pricing Calculator, built a CUR-shaped dataset from that baseline, 
and loaded it 
into Athena. The figures below are scenario values, not billed spend: $315 total, $55 unallocated, a 17% gap.

The two layers connect through the resource ID. The untagged resource in the cost data is the same instance the CloudTrail evidence describes.

**Evidence key.** Every screenshot below is labelled.

| Tier | Meaning |
|---|---|
| **[Captured]** | Real output from the sandbox account |
| **[Executed]** | Real query run in Athena against the scenario dataset |
| **[Illustrative]** | Constructed output showing query logic and result shape |
| **[Modelled]** | Scenario reasoning, no underlying data |
| **[Methodology]** | Design and provenance artifacts |

---

## The Short Version

- Found a 17% cost allocation gap. That is $55 of $315 in monthly scenario spend, and none of it could be attributed to a team.
- Used Athena against CUR-shaped data to isolate the affected resources. Then used CloudTrail to confirm the tags were never submitted at creation,
rather than 
removed later.
- Corroborated the CloudTrail record against the EC2 console. Same instance, same region, matching timestamps from two independent sources.
- Found a second, separate problem. The provisioning session had no traceable identity behind it, so even correctly tagged spend could not have been
routed to a team.
- Built and tested a Lambda that reads CloudTrail events, checks three required tags, and builds a structured alert.
- Found two defects in that Lambda's S3 path during testing. Diagnosed both and documented the corrected design.
- Wrote up why I chose detection over immediate enforcement, and why I rejected automatic tag remediation.

---

## The Investigation

The spend was billed correctly. Finance simply could not tell whose it was. That is a reporting failure rather than overspending, but it still breaks 
chargeback, forecasting, and any commitment planning built on that data.

I checked the invoice total first. It reconciled, which ruled out a billing error and moved the problem downstream into attribution.

![Scenario invoice validation totalling $315](Screenshots/01-cost-validation.png)
***[Illustrative]** Service totals reconciled against the scenario dataset. Built with a hardcoded `VALUES` clause for presentation. The production-
equivalent query aggregates `SUM(line_item_unblended_cost)` by product code from CUR.*

Querying the cost data, one resource came back NULL across all three required tag columns, on every billing line. One missing tag is plausibly someone 
forgetting. Three missing at once, consistently, means the resource never went through any tagging process at all. That is a process failure rather 
than a mistake, and it is what moved the work from reconciliation into forensics.

![Scenario query result showing NULL Environment, Project, and Owner](Screenshots/02-cur-resource-isolation.png)
***[Illustrative]** The isolated resource. Constructed output. The reasoning applied to the pattern is the transferable part.*

CloudTrail was the right tool for the next step because it records what was actually asked for. The `tagSpecificationSet` field captures the tags 
submitted with the API call. A tool that inspects the resource afterwards cannot reconstruct that. Config tells you what a resource looks like now. 
CloudTrail tells you what was requested at the moment it was created.

![CloudTrail RunInstances event record](Screenshots/03a-cloudtrail-event-history.png)
***[Captured]** Real CloudTrail record. Root principal, MFA-authenticated, `us-east-2`, and a Chrome UserAgent confirming a console launch rather than 
CLI or SDK.*

![CloudTrail tagSpecificationSet showing only the Name tag](Screenshots/03b-cloudtrail-json-console-useragent.png)
***[Captured]** The tag specification submitted at creation. Only `Name` was included. `Environment`, `Project`, and `Owner` were never submitted. The 
tags were absent from the start, not applied and later stripped.*

The EC2 console records the same instance independently. Same region, and a launch time that matches the CloudTrail timestamp to the second once you 
account for the timezone difference.

![EC2 console showing the investigated instance](Screenshots/06-ec2-console-instance.png)
***[Captured]** Instance `i-0c9cfb67280fe44ee` in `us-east-2`, launched 22 January 2026 at 14:00:13 EST. CloudTrail records the same instant as 
19:00:13Z. Two independent sources of record, one event.*

I also checked whether the principal that created the resource had any traceable identity behind it. It did not. No `sessionIssuer`, no role 
assumption, no federation context, and no `AssumeRole` events for that principal in the 30 minutes before the launch. That is a separate finding from 
the tagging gap, but it matters for a related reason. Tagging makes spend allocatable. Identity makes it attributable. This environment was weak in 
both.

Expanding the check to S3 showed the pattern was not limited to one resource.

![Athena query showing S3 buckets missing the Environment tag](Screenshots/07-athena-s3-tag-query.png)
***[Executed]** Real Athena query against the scenario dataset, using `COALESCE` on the environment tag column and `UNION ALL` for the total row. Two 
buckets carry `Project` and `Owner` but not `Environment`. Partial tagging produces the same Finance-invisible result as no tagging at all.*

For contrast, this is what a compliant resource looks like in the same environment. It defines the standard the assessment measures against.

![EC2 Tags tab showing Environment, Project, and Owner applied](Screenshots/08-ec2-tags-baseline.png)
***[Captured]** All three required tags applied to a live instance.*

---

## What I Built and Tested

A Lambda function triggered by an EventBridge rule watching CloudTrail events. It reads `RunInstances` and `CreateBucket` calls, checks them against 
the three required tags, and builds a structured alert containing the missing tags, the IAM principal, the account, and the region. I deployed it and 
it ran successfully.

![EventBridge rule configuration](Screenshots/05a-eventbridge-rule.png)
***[Captured]** The rule as built, matching `RunInstances` and `CreateBucket`.*

![Lambda function code](Screenshots/05b-lambda-tag-validator-code.png)
***[Captured]** The prototype as deployed.*

![Lambda test execution result](Screenshots/05c-lambda-test-execution.png)
***[Captured]** Test run returning `statusCode 200` and correctly identifying all three missing tags. The EC2 path works.*

## What Testing Revealed

Two defects in the S3 path. Either one would have fired a false alert on every new bucket.

**It validated at the wrong point.** S3 cannot accept tags when a bucket is created, the way EC2 can. Tags are applied afterwards through a separate 
`PutBucketTagging` call. Checking at `CreateBucket` flags every bucket as non-compliant even when it gets tagged correctly two seconds later. The check 
belongs on `PutBucketTagging`, with a delayed sweep to catch buckets that never get tagged at all.

**It read the tag fields in the wrong case.** The function used lowercase keys. CloudTrail renders the S3 request body in PascalCase, so the parse 
would have come back empty and reported every bucket as fully untagged.

Both are small bugs. They are also the kind that quietly produce false alerts until engineers stop reading them. At that point the control still runs 
and still reports success, but it has stopped doing anything.

**One more thing I found.** I deployed the rule and function in `us-east-1`, but the resource I was investigating was created in `us-east-2`. These 
rules are regional. As built, the control would not have caught the event it was designed to catch. That is the regional-gap failure mode from my own 
failure-mode analysis, found in my own control.

I lost sandbox access before I could redeploy any of these fixes, so they are documented as corrected design rather than presented as done.

## What Was Designed, Not Built

Slack delivery. The alert payload is built and written to CloudWatch, but nothing routes it to a Finance channel or to the resource owner yet. That is 
the next step, not something already running. In production the webhook would come from Secrets Manager rather than being hardcoded.

![Tag compliance detection pipeline](Screenshots/architecture-cloud-governance-pipeline.png)
***[Methodology]** End-to-end design. Slack delivery is marked pending, matching the prototype's actual state. Impact metrics shown are projected, not 
measured.*

---

## Why Detection, Not Enforcement

Choosing to detect rather than block was a judgment about maturity, not a technical limitation.

A blocking policy would close the gap on day one. It would also break an unknown number of CI/CD pipelines and developer workflows, because nobody has 
mapped the exceptions yet. Some AWS-managed services do not pass required tags through consistently, so they would break too. Enforcement applied 
before you know what you are enforcing against costs you engineering's trust, and you need that trust for everything that comes after.

I also looked at having the function apply default tags automatically when it found something missing, and rejected it. A guessed owner sends the cost 
to the wrong team. It hides the spend from the team that actually incurred it. And it leaves anomaly detection running against a baseline that includes 
invented attribution. A control that creates false confidence is worse than a visible gap, because at least a gap is something Finance can act on.

The sequence I would follow: detect now, add targeted prevention once compliance settles, then broader enforcement with a grace period once the full 
picture is mapped.

---

## What This Demonstrates

Reconciliation, evidence-based investigation, audit-trail analysis, root cause analysis, and control design. It is the same discipline I used for 13 
years reconciling regulated real estate files against contracts and disclosures before they could close. Check the record against reality, find where 
they diverge, work out why, and build something that stops it happening again.

I am not claiming to have run a cost or compliance control in production. This is cost allocation, asset attribution, and reconciliation work applied 
in a cloud context. It is relevant to IT financial analysis, cloud cost analysis, IT asset management, and cloud billing operations.

---

## Limitations

- The cost figures are a scenario dataset, not billed spend. The construction method is documented in the full report.
- Three of the query screenshots are illustrative rather than executed. The SQL is production-equivalent. The data underneath is constructed.
- The detection control is a tested prototype with two known defects in its S3 path. It is not production-grade and was not redeployed with the fixes.
- Slack alert delivery is designed, not implemented.
- No control was run over time, so MTTR, repeat-violation rate, and compliance-trend figures are proposed measurements rather than results.

**Open items:** redeploy in `us-east-2` with the S3 fixes. Re-run the EC2 queries as executed rather than illustrative. Add the delayed sweep for 
buckets that never get tagged. Instrument the compliance KPIs.

---

## Full Technical Report

The complete investigation is in the [full technical report](https://github.com/AthertHa-FinOps/aws-finops-cost-allocation-investigation). It covers 
the SQL, the full CloudTrail evidence chain, control hierarchy reasoning, the corrected Lambda design, failure-mode analysis, and how the scenario 
dataset was built.
