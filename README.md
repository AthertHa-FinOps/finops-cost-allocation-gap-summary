# Cloud Governance Case Study: AWS Tag Compliance Investigation & Detection Pipeline

> **Project context:** This is a self-directed learning project completed in a personal AWS
> sandbox account. It is not a professional engagement or output of a prior employer. Cost
> figures are intentionally small ($315 in total spend) so I could test the full investigation
> process without financial risk. The techniques and reasoning below are meant to translate to
> real enterprise environments regardless of spend size.

**I traced a cloud cost allocation gap back to its root cause and built a detection pipeline to catch it going forward.** This project applies 
reconciliation and audit-readiness habits from 13+ years in regulated real estate transactions to a cloud governance problem, using AWS-native tools to 
investigate and fix it.

---

## The Short Version

- Found a 17% cost allocation gap ($53 of $315 in monthly spend) invisible to Finance chargeback reporting
- Used CUR and Athena SQL to isolate the affected resources, then CloudTrail to confirm the tags were never submitted at creation, not removed later
- Traced the resource back to console-based provisioning with no attributable identity chain behind it
- Built a detection pipeline (CloudTrail → EventBridge → Lambda) that flags missing tags on new resources and alerts Finance and the resource owner
- Documented the reasoning behind choosing detection over immediate enforcement, given there was no existing compliance baseline

---

## What Happened

The spend was billed correctly — Finance just couldn't tell whose it was. That's a reporting problem, not overspending, but it still breaks chargeback, 
forecasting, and any commitment planning built on that data.

I confirmed the invoice was accurate first, which meant the issue was downstream in attribution.

![Invoice validation confirming $315 total spend](screenshots/01%20-%20invoice-total-315.png)
*Confirming the invoice matched CUR totals before looking further.*

Using CUR and Athena, I found a resource with all three required tags missing at once — not one tag, all three, on every billing line. That pattern 
ruled out someone just forgetting a tag.

![CUR query result showing NULL Environment, Project, and Owner tags](screenshots/02%20-%20cur-resource-isolation-missing-allocation-tags.png)
*The resource verification query, showing all three tags returning NULL simultaneously.*

![CUR resource isolation query and result](screenshots/03%20-%20cloudtrail-forensics-runinstances-missing-tags-cli-launch.png)
*The underlying query used to isolate this resource in Athena.*

I moved to CloudTrail to find out why. The event history for this resource confirmed the launch details, IAM principal, and timestamp.

![CloudTrail event history showing RunInstances event](screenshots/03a%20-%20cloudtrail-event-history-runinstances.png)
*CloudTrail record of the RunInstances event, confirming a console launch.*

The `tagSpecificationSet` on the original API call showed only a `Name` tag was ever submitted. The others were never there to begin with.

![CloudTrail tagSpecificationSet showing only the Name tag present](screenshots/03b%20-%20cloudtrail-runinstances-json-cli-useragent.png)
*The tag specification submitted at creation, confirming Environment, Project, and Owner were never included.*

I also checked whether the principal that created the resource had any traceable identity chain behind it. It didn't — no role assumption, no 
federation context. That's a separate finding from the tagging gap, but it matters for the same reason: without it, there's no way to route 
accountability back to a team.

Expanding the check to S3 confirmed the issue wasn't isolated to one resource.

![S3 tag compliance audit showing NULL Environment tags](screenshots/04%20-%20cur-s3-tag-compliance-audit.png)
*Two S3 buckets missing the required Environment tag, despite having Project and Owner present.*

---

## What I Built

A detection pipeline using CloudTrail, EventBridge, and Lambda. New EC2 and S3 resources get checked against required tags (`Environment`, `Project`, 
`Owner`) shortly after creation, and violations route to a Finance channel and the resource owner.

![Lambda function finops-tag-validator showing REQUIRED_TAGS list, event parsing logic for RunInstances and CreateBucket, missing tag 
detection, and alert payload construction](screenshots/05b%20-%20lambda-finops-tag-validator-code.png)
*End-to-end flow: resource creation → CloudTrail → EventBridge → Lambda → Slack alerts.*

![EventBridge rule configuration](screenshots/05a-eventbridge-rule.png)
*The EventBridge rule matching RunInstances and S3 tagging events.*

EC2 and S3 needed different logic since S3 doesn't accept tags inline at creation the way EC2 does — an early version of this pipeline flagged every 
new S3 bucket as noncompliant before I caught that and fixed it.

![Lambda function code for tag validation](screenshots/05b%20-%20lambda-finops-tag-validator-code.png)
*The Lambda function checking for required tags and constructing the alert payload.*

![Lambda runtime settings](screenshots/05b-ii%20-%20lambda-settings.png)
*Lambda deployment configuration.*

I chose detection over immediate enforcement (like blocking non-tagged resources outright) because there was no existing compliance baseline. Blocking 
deployments before you know what's actually out there tends to break more things than it fixes.

![Lambda test execution result showing the alert firing](screenshots/05c-lambda-test-result.png)
*Test run confirming the pipeline correctly detects and alerts on a missing-tag violation.*

---

## Key Technologies

AWS Cost and Usage Report (CUR), Amazon Athena, AWS CloudTrail, Amazon EventBridge, AWS Lambda (Python), AWS Organizations Tag Policies, Slack alerting.

---

## Full Technical Report

The complete investigation, including SQL queries, full CloudTrail evidence, Lambda logic, and the reasoning behind each decision, is in the [full 
technical report](https://github.com/AthertHa-FinOps/aws-finops-cost-allocation-investigation).
