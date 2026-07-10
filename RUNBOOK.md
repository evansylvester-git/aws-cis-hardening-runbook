# 🛠️ CIS AWS Foundations Benchmark v1.2.0 — Remediation Runbook

This is the full step-by-step record of hardening a live personal AWS account (`ap-south-1`) against the **CIS AWS Foundations Benchmark v1.2.0**, tracked end-to-end through **AWS Security Hub**. Every control below was remediated manually through the AWS Management Console — no CLI, no IaC — so each change could be verified visually against Security Hub before moving to the next.

> 📌 **A note on the two scores you'll see throughout this runbook:**
> Security Hub reports two different numbers that both improved over the course of this project:
> - **CSPM Score** — the blended posture score across *all* enabled standards (CIS + AWS Foundational Security Best Practices + others). General hardening (e.g. S3 public access, GuardDuty, default security groups) moves this number.
> - **CIS Score** — the score for the **CIS AWS Foundations Benchmark v1.2.0 standard specifically**. This is the one that matters most for this project and is driven almost entirely by the IAM, KMS, CloudTrail, and CloudWatch controls below.

---

## 📍 Baseline Assessment

Before any remediation, Security Hub was reviewed to understand the starting posture and prioritize controls by severity.

| Evidence | What it shows |
|---|---|
| ![Baseline CSPM score](./evidence/01-SecurityHub-Baseline.jpeg) | Baseline CSPM score: **85%** (317 of 371 controls passed, 54 failed) |
| ![Critical findings](./evidence/2.Critical%20Level%20Findings.jpeg) | Critical-severity findings (root MFA, SSM public sharing, unrestricted security groups) |
| ![High findings](./evidence/3-HIGH%20LEVEL%20FINDINGS%20.jpeg) | High-severity findings (public EC2/EBS exposure, permissive security groups, CloudTrail gaps) |
| ![Medium findings](./evidence/4-MEDIUM%20LEVEL%20FINDINGS.jpeg) | Medium-severity findings (SSM management, EBS encryption, S3 logging, IAM MFA) |
| ![Low findings](./evidence/5-%20LOW%20LEVEL%20FINDINGS.jpeg) | Low-severity findings — this is where all 13 CloudWatch log metric filter/alarm controls (CIS section 3.x) surfaced |
| ![Informational findings](./evidence/6-%20INFORMATIONAL%20LEVEL%20FINDINGS.jpeg) | Informational-severity findings |
| ![Security standards overview](./evidence/7-%20SECURITY%20STDS%20%26%20CIS%20CRITICAL%20STANDARDS.jpeg) | Standards dashboard — CIS AWS Foundations Benchmark v1.2.0 sitting at a much lower score than AWS FSBP, confirming CIS-specific controls needed the most work |
| ![CIS-specific baseline score](./evidence/15-CIS%20SCAN%201ST%20-%2029%25.jpeg) | **CIS AWS Foundations Benchmark v1.2.0 score specifically: 29%** (12 of 41 controls passed) — this is the true starting point for the CIS-specific remediation tracked below |

---

## 🧰 Supplementary AWS FSBP Hardening

A few general hardening items were addressed alongside the CIS-specific work. These aren't CIS AWS Foundations Benchmark v1.2.0 controls themselves, but they improved the overall CSPM posture score in parallel:

| Evidence | What it shows |
|---|---|
| ![S3 block public access](./evidence/8-%20BLOCKED%20PUBLIC%20ACCESS%20S3%20BUCKETS.jpeg) | Account-level S3 Block Public Access settings confirmed enabled across all four sub-settings |
| ![GuardDuty enabled](./evidence/11-ENABLED%20GUARDDUTY.jpeg) | Amazon GuardDuty enabled for continuous threat detection |
| ![Default security group hardened](./evidence/13-DEFAULT%20SEC%20GRP%20OF%20EC2%20VPC%20FIXED.jpeg) | Default VPC security group's inbound/outbound rules cleared, closing the "default security group allows traffic" finding |

---

## 🔑 IAM

| Control | Evidence | What was done |
|---|---|---|
| `IAM.11–17` | ![Password policy](./evidence/23-%20MIN%20PW%20LENGTH%20%3D%2014%20ENABLED.jpeg) | Account password policy hardened — minimum length ≥ 14, complexity, reuse prevention, and expiration enabled |
| — | ![MFA enabled](./evidence/14-MFA%20ENABLED%20FOR%20SPLUNK%20CONSOLE%20USERS.jpeg) | MFA enrolled on the `splunk-ec2-user` service account used by the home-lab Splunk EC2 instance |
| — | ![Policies migrated to group](./evidence/16-POLICIES%20ATTACHED%20VIA%20GROUP%20FROM%20DIRECTLY%20ATTACHED%20FIXED.jpeg) | Directly-attached IAM policies migrated to a dedicated `splunk-ec2-user-group`; user-level attachments removed so all permissions are now managed at the group level |
| `IAM.18` | ![Support role created](./evidence/17-SUPPORT%20ROLE%20FOR%20INCIDENT%20MANAGEMENT%20ENABLED.jpeg) | Dedicated `AWSSupportAccessRole` created for support-case access, avoiding root or admin credential use |

---

## 🔐 KMS

| Control | Evidence | What was done |
|---|---|---|
| `KMS.4` | ![KMS key rotation enabled](./evidence/18-ENABLED%20KEY%20ROTATION%20FOR%20CMS%20KEYS%20.jpeg) | Automatic annual key rotation enabled on the customer-managed KMS key (`cis-cloudtrail-key`) used to encrypt CloudTrail logs |

---

## 🖥️ EC2 / VPC

| Control | Evidence | What was done |
|---|---|---|
| `EC2.6` | ![VPC Flow Logs enabled](./evidence/19-%20CREATED%20FLOW%20LOGS%20FOR%20VPC.jpeg) | VPC Flow Logs enabled, closing out the previously open EC2.6 finding |

---

## 🧭 CloudTrail

| Control | Evidence | What was done |
|---|---|---|
| — | ![CloudTrail enabled](./evidence/10-CLOUDTRAIL%20ENABLED.jpeg) | Multi-region trail `cis-org-trail` confirmed active, with SSE-KMS encryption and log file validation enabled |
| `CloudTrail.5` | ![CloudTrail → CloudWatch Logs integration](./evidence/20-%20SYNCED%20CLOUDTRAIL%20LOGS%20TO%20CLOUDWWATCH.jpeg) | CloudTrail wired to deliver into a CloudWatch Logs log group (`aws-cloudtrail-logs-894759051479-758c27c5`) — a **prerequisite** for every CloudWatch.2–.14 metric filter, since they all read from this log group |
| `CloudTrail.7` | ![CloudTrail bucket access logging](./evidence/24-%20SERVER%20ACCESS%20BUCKET%20LOGS%20FOR%20CLOUDTRAIL%20ENABLED.jpeg) | S3 access logging enabled on the CloudTrail bucket, with logs delivered to a dedicated `s3-access-logs` bucket (SSE-encrypted) |

---

## ⭐ CloudWatch — Log Metric Filters & Alarms

The core of this project: 13 log metric filter + alarm controls (CIS section 3.x, surfaced in Security Hub as `CloudWatch.2`–`.14`). Each follows the same pattern:

```
Metric filter  →  aws-cloudtrail-logs-894759051479-758c27c5   (CloudTrail-fed log group)
Custom metric  →  CISBenchmark namespace
Alarm          →  CIS-3.x-<ShortName>  →  cis-cloudtrail-alarms (SNS topic)
```

First, the shared SNS topic that every alarm below notifies:

| Evidence | What it shows |
|---|---|
| ![SNS topic for alarms](./evidence/25-%20CLOUD%20TRAIL%20SNS%20ENABLED%20FOR%20TRIGGER%20ALARMS.jpeg) | `cis-cloudtrail-alarms` SNS topic — the shared notification target for all 13 CloudWatch alarms |

### Controls remediated with evidence

| Control | Detects | Metric filter evidence | Alarm evidence |
|:---|:---|:---|:---|
| `CloudWatch.2` | Unauthorized API calls | ![Metric filter](./evidence/29-%20LOG%20METRIC%20FOR%20UNAUTH%20API%20.jpeg) | ![Alarm](./evidence/30-%20ALARM%20FOR%20UNAUTH%20API.jpeg) |
| `CloudWatch.3` | Console sign-in without MFA | ![Metric filter](./evidence/31-%20METRIC%20FILTER%20FOR%20CONSOLE%20SIGNIN%20WITHOUT%20MFA%20ENABLED.jpeg) | ![Alarm](./evidence/32-%20ALARM%20CONSOLE%20SIGNIN%20WITHOUT%20MFA%20ENABLED.jpeg) |
| `CloudWatch.4` | Root account usage | ![Metric filter + alarm](./evidence/26-%20LOG%20METRIC%20FOR%20ROOT%20USER%20ENABLED.jpeg) | *(combined in same screenshot)* |
| `CloudWatch.5` | IAM policy changes | ![Metric filter](./evidence/33-%20METRIC%20FILTER%20FOR%20IAMPOLICYCHANGES%20APPLIED.jpeg) | ![Alarm](./evidence/34-%20ALARM%20FOR%20IAMPOLICYCHANGES%20APPLIED.jpeg) |
| `CloudWatch.6` | CloudTrail configuration changes | ![Metric filter + alarm](./evidence/35-%20ALARM%20AND%20METRC%20FILTER%20FOR%20CLOUDTRAIL%20CONFIG%20CHANGES%20APPLIED.jpeg) | *(combined in same screenshot)* |
| `CloudWatch.7` | Console sign-in failures | ![Metric filter](./evidence/36-%20METRIC%20FILTER%20FOR%20CONSOLE%20AUTH%20FAILURE%20APPLIED.jpeg) | ![Alarm](./evidence/37-%20ALARM%20FOR%20CONSOLE%20AUTH%20FAILURE%20APPLIED.jpeg) |
| `CloudWatch.8` | Disabling/scheduled deletion of CMKs | ![Metric filter](./evidence/38-%20METRIC%20FILTER%20FOR%20DISABLEDELETE%20CMK%20KEYS.jpeg) | ![Alarm](./evidence/39-%20ALARM%20FOR%20DISABLEDELETE%20CMK%20KEYS.jpeg) |
| `CloudWatch.9` | S3 bucket policy changes | ![Metric filter + alarm](./evidence/42-%20S3%20BUCKET%20POLICY%20CHANGES%20METRIC%20FILTER%20APPLIED.jpeg) | *(combined in same screenshot)* |

### Controls remediated (same pattern, not separately screenshotted)

| Control | Detects |
|:---|:---|
| `CloudWatch.10` | AWS Config configuration changes |
| `CloudWatch.11` | Security group changes |
| `CloudWatch.12` | Network ACL (NACL) changes |
| `CloudWatch.13` | Network gateway changes |
| `CloudWatch.14` | Route table changes |

> *These five followed the identical metric filter → alarm → SNS pattern shown above and are reflected in the final Security Hub score.*

---

## 📊 Progress Checkpoints

Security Hub was re-scanned after each batch of remediation to confirm improvement before moving to the next control. Note the **two separate score tracks** described at the top of this runbook.

### CSPM Score (blended, all standards)

| Stage | Evidence | Score |
|---|---|---|
| Baseline | [see above](#-baseline-assessment) | 85% |
| Checkpoint 1 | ![87%](./evidence/12-CSPM%20SCAN%202%20.jpeg) | 87% |
| Checkpoint 2 | ![91%](./evidence/21-%20CSPM%20SCAN%203-%2091%25.jpeg) | 91% |
| Checkpoint 3 | ![92%](./evidence/27-%20CSPM%20SCAN%20-%2092.jpeg) | 92% |
| Checkpoint 4 | ![93%](./evidence/40-%20CSPM%20SCAN%20-%2093%25.jpeg) | 93% |
| **Final** | ![95%](./evidence/43-%20CSPM%20SCAN%20FINAL%20-%2095%25.jpeg) | **95%** |

### CIS AWS Foundations Benchmark v1.2.0 Score (CIS-specific)

| Stage | Evidence | Score |
|---|---|---|
| Baseline | ![29%](./evidence/15-CIS%20SCAN%201ST%20-%2029%25.jpeg) | 29% |
| Checkpoint 1 | ![61%](./evidence/22-%20CIS%20SCORE-%2061.jpeg) | 61% |
| Checkpoint 2 | ![68%](./evidence/28-%20CIS%20SCAN-%2068.jpeg) | 68% |
| Checkpoint 3 | ![80%](./evidence/41-%20CIS%20SCAN%20-%2080%25.jpeg) | 80% |
| **Final** | ![98%](./evidence/44-%20CIS%20SCAN%20FINAL%20-%2098%25.jpeg) | **98%** |

> Security Hub findings refresh on a 12–24 hour cycle, so scores were re-checked the following day after each batch of fixes to avoid reacting to stale data.

---

## ✅ Results Summary

| Metric | Before | After |
|---|:---:|:---:|
| CSPM Score (all standards) | 85% | **95%** |
| CIS AWS Foundations Benchmark v1.2.0 Score | 29% | **98%** |
| Failed CIS controls | 29 of 41 | 1 *(pending refresh)* |

---

