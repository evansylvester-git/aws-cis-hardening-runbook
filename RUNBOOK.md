# RUNBOOK — CIS AWS Foundations Benchmark v1.2.0 Hardening

Step-by-step remediation log, in the order the controls were actually worked through in the AWS Console. Each section says what Security Hub flagged, what was changed, and which controls it clears.

**Account / environment constants used throughout:**
- Region: `ap-south-1` (Asia Pacific — Mumbai)
- CloudTrail trail: `cis-org-trail`
- CloudTrail log group: `aws-cloudtrail-logs-894759051479-758c27c5` *(replace `894759051479` with your own account ID if reusing this runbook)*
- CloudWatch metric namespace: `CISBenchmark`
- Alarm naming convention: `CIS-3.x-<ShortName>` (e.g. `CIS-3.1-RootAccountUsage`, `CIS-3.6-UnauthorizedAPICalls`)
- SNS topic for all CIS alarms: `cis-cloudtrail-alarms`

---

## 0. Baseline

Before starting, Security Hub's overall posture management dashboard (across *all* enabled standards, not just CIS) showed **85%** (317 of 371 controls passed, 54 failed). This is the starting reference point — not the CIS-specific score, which is tracked separately in each section below and in the final screenshot.

📷 `evidence/01.png`

---

## 1. IAM — Password Policy (IAM.11–17)

**Findings:** Account password policy didn't meet the benchmark's minimum length, complexity, reuse, and expiration requirements.

**Fix:** IAM → Account settings → Password policy → set:
- Minimum length ≥ 14 characters
- Require at least one uppercase, lowercase, number, and symbol
- Prevent password reuse (last 24)
- Expire passwords after 90 days
- Allow users to change their own password

This single change clears all seven sub-controls (IAM.11–17) at once.

📷 `evidence/02.png`–`evidence/05.png`

---

## 2. IAM — MFA for the Splunk service account

**Findings:** The `splunk-ec2-user` IAM user (used by the home-lab Splunk EC2 instance) had no MFA device enrolled.

**Fix:** IAM → Users → `splunk-ec2-user` → Security credentials → Assign MFA device (virtual MFA).

📷 `evidence/06.png`–`evidence/07.png`

---

## 3. IAM — Move from user-attached policies to a group

**Findings:** Best-practice/least-privilege guidance flags policies attached directly to an IAM user rather than inherited through a group.

**Fix:**
1. Created `splunk-ec2-user-group`.
2. Attached the same policies to the group.
3. Removed the directly-attached policies from `splunk-ec2-user`.
4. Added `splunk-ec2-user` to the new group.

📷 `evidence/08.png`–`evidence/09.png`

---

## 4. IAM.18 — Dedicated Support Access Role

**Findings:** No role existed scoped specifically for AWS Support case access.

**Fix:** IAM → Roles → Create role → AWS service → attach `AWSSupportAccessRole` policy → name it to match, so support access never requires root or full-admin credentials.

📷 `evidence/11.png`–`evidence/12.png`

---

## 5. KMS.4 — Key rotation

**Findings:** The customer-managed KMS key (`cis-cloudtrail-key`) used to encrypt CloudTrail logs did not have automatic annual rotation enabled.

**Fix:** KMS → Customer managed keys → `cis-cloudtrail-key` → Key rotation → enable automatic rotation.

📷 `evidence/13.png`–`evidence/14.png`

---

## 6. EC2.2 — Remove the default VPC

**Findings:** A default VPC existed in the region; the benchmark recommends removing default VPCs that aren't in active use, since their default security group and default subnets are commonly over-permissive.

**Fix:** VPC console → confirmed no resources (EC2 instances, ENIs, RDS, etc.) depended on the default VPC → deleted the default VPC and its subnets/internet gateway in `ap-south-1`.

📷 `evidence/15.png`–`evidence/16.png`

---

## 7. CloudTrail → CloudWatch Logs integration

**Findings:** `cis-org-trail` was logging to S3 with SSE-KMS encryption and log file validation enabled, but had **no CloudWatch Logs integration** — which meant none of the CloudWatch.2–.14 metric filter controls could ever pass, since they all read from a log group populated by CloudTrail.

**Fix:** CloudTrail → Trails → `cis-org-trail` → CloudWatch Logs → Edit → create/select log group `aws-cloudtrail-logs-894759051479-758c27c5` → let AWS create the `CloudTrail_CloudWatchLogs` IAM role for delivery.

**Before** (no CloudWatch Logs configured):
📷 `evidence/10.png`

**After** (log group wired up, IAM role visible):
📷 `evidence/20.png`

This step unblocks every control in the next section.

---

## 8. CloudWatch.2–14 — Log metric filters and alarms

This is the bulk of the project: 13 controls, each following the identical pattern below, just with a different filter pattern and alarm name.

**Per-control steps (repeated 13×):**
1. CloudWatch → Log groups → `aws-cloudtrail-logs-894759051479-758c27c5` → Create metric filter.
2. Paste the control-specific filter pattern (see table below).
3. Assign it to the `CISBenchmark` metric namespace, metric value `1`.
4. Create an alarm on that metric: threshold ≥ 1 for 1 datapoint within 5 minutes, statistic = Sum.
5. Set alarm actions → notify `cis-cloudtrail-alarms` SNS topic.
6. Name the alarm `CIS-3.x-<ShortName>` per the convention above.

| Control | Alarm name | Detects |
|---|---|---|
| CloudWatch.2 | `CIS-3.6-UnauthorizedAPICalls` | Unauthorized API calls |
| CloudWatch.3 | `CIS-3.2-ConsoleSigninNoMFA` | Console sign-in without MFA |
| CloudWatch.4 | `CIS-3.1-RootAccountUsage` | Root account usage |
| CloudWatch.5 | `CIS-3.4-IAMPolicyChanges` | IAM policy changes |
| CloudWatch.6 | `CIS-3.5-CloudTrailConfigChanges` | CloudTrail configuration changes |
| CloudWatch.7 | `CIS-3.3-ConsoleSigninFailures` | Console sign-in failures |
| CloudWatch.8 | `CIS-3.7-DisableOrDeleteCMK` | Disabling/scheduled deletion of CMKs |
| CloudWatch.9 | `CIS-3.8-S3BucketPolicyChanges` | S3 bucket policy changes |
| CloudWatch.10 | `CIS-3.9-ConfigConfigurationChanges` | AWS Config configuration changes |
| CloudWatch.11 | `CIS-3.10-SecurityGroupChanges` | Security group changes |
| CloudWatch.12 | `CIS-3.11-NACLChanges` | Network ACL changes |
| CloudWatch.13 | `CIS-3.12-NetworkGatewayChanges` | Network gateway changes |
| CloudWatch.14 | `CIS-3.13-RouteTableChanges` | Route table changes |

> The exact `filter_pattern` JSON for each control (from the CIS benchmark document) is straightforward to pull from the [CIS AWS Foundations Benchmark v1.2.0 PDF](https://www.cisecurity.org/benchmark/amazon_web_services) §3.1–3.14 — reproduced here per-control if you want the literal strings: *(fill in from your working notes before publishing, since the exact patterns are long and easy to typo — worth a final diff against the CIS doc.)*

**Mid-progress example** (2 of 13 alarms created — `CIS-3.6-UnauthorizedAPICalls`, `CIS-3.1-RootAccountUsage`):
📷 `evidence/30.png`

**Remaining creation screenshots:**
📷 `evidence/21.png`–`evidence/43.png`

---

## 9. Verifying in Security Hub

Security Hub's CIS AWS Foundations Benchmark v1.2.0 standard view was used to confirm each control flipped from **Failed → Passed** after its alarm was created. Security Hub findings refresh on a **12–24 hour cycle**, so a control could still show *Failed* for up to a day after the underlying alarm was already correctly configured — worth noting if you're reproducing this and the score doesn't update immediately.

**Final state:** 98% (40 of 41 enabled controls passed), with the one remaining `CloudWatch.2` finding expected to clear on the next Security Hub refresh.

📷 `evidence/44.png`

---

## Outstanding / to verify before calling this fully complete

- **EC2.6** — VPC Flow Logs (may already be resolved; re-check Security Hub before publishing final numbers)
- **CloudTrail.7** — S3 access logging on the CloudTrail bucket (same — verify current status)

## Evidence index

All 44 screenshots are in [`evidence/`](./evidence), numbered `01.png`–`44.png` in the order the work was done. This runbook explicitly calls out 01, 02–09, 10, 11–16, 20, 21–43, and 44 above; the remaining numbers in each range are the intermediate console steps (navigating to the right screen, filling in each field, save confirmations) for that same control. If you're adapting this repo, re-caption the ones your own screenshots don't line up with exactly — the structure/order is the reusable part.
