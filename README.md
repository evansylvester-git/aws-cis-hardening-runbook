# CIS AWS Foundations Benchmark v1.2.0 — Hardening Runbook

Hands-on remediation of a personal AWS account against the **CIS AWS Foundations Benchmark v1.2.0**, tracked and validated end-to-end through **AWS Security Hub**. This repo documents the starting posture, every control remediated, the exact console steps taken, and the final compliance score — with screenshot evidence for each stage.

> **Why this project exists:** Security Hub tells you *what's* failing. It doesn't tell you *how* to fix it, or *why* the fix is safe to apply on a live account. This runbook closes that gap — it's the step-by-step guide I wish I'd had, written while doing the remediation myself rather than after the fact.

## Results

| | Before | After |
|---|---|---|
| CIS AWS Foundations Benchmark v1.2.0 score | 85% (baseline, all standards) | **98%** |
| Failed CIS controls | 54+ | 1 (pending Security Hub's 12–24h refresh cycle) |
| Controls remediated in this project | — | **13 CloudWatch metric filter/alarm controls (CloudWatch.2–.14)**, plus IAM, KMS, EC2, and CloudTrail hardening |

See [`RUNBOOK.md`](./RUNBOOK.md) for the full step-by-step walkthrough and [`evidence/`](./evidence) for screenshots of every stage.

## Scope

Remediation was performed manually through the **AWS Management Console** (region: `ap-south-1`), not via CLI or IaC — deliberately, so each change could be verified visually against Security Hub before moving to the next control. Controls covered:

### IAM
- **IAM.11–17** — Account password policy (minimum length, complexity, reuse prevention, expiration)
- MFA enrollment for a service IAM user (`splunk-ec2-user`) used by the home-lab Splunk EC2 instance
- Migrated directly-attached IAM policies to a dedicated group (`splunk-ec2-user-group`), removing user-level policy attachments
- **IAM.18** — Created a dedicated `AWSSupportAccessRole` for support-case access instead of using root or admin credentials

### KMS
- **KMS.4** — Enabled automatic annual key rotation on the customer-managed KMS key used for CloudTrail log encryption

### EC2 / VPC
- **EC2.2** — Removed the default VPC in the account region
- **EC2.6** *(in progress / verify)* — VPC Flow Logs

### CloudTrail
- Multi-region trail (`cis-org-trail`) with SSE-KMS encryption and log file validation enabled
- **CloudTrail → CloudWatch Logs integration** — required *before* any CloudWatch metric filter control can pass, since CloudWatch.2–.14 all read from the log group CloudTrail delivers into
- **CloudTrail.7** *(in progress / verify)* — S3 access logging on the CloudTrail bucket

### CloudWatch — the core of this project
All 13 log metric filter + alarm controls in the benchmark, each following the same pattern: a metric filter on the CloudTrail-fed log group (`aws-cloudtrail-logs-894759051479-758c27c5`), a custom metric in the `CISBenchmark` namespace, and an alarm (`CIS-3.x-<ShortName>`) wired to a shared SNS topic (`cis-cloudtrail-alarms`):

| Control | Detects |
|---|---|
| CloudWatch.2 | Unauthorized API calls |
| CloudWatch.3 | Sign-in without MFA |
| CloudWatch.4 | Root account usage |
| CloudWatch.5 | IAM policy changes |
| CloudWatch.6 | CloudTrail configuration changes |
| CloudWatch.7 | Console sign-in failures |
| CloudWatch.8 | Disabling/deleting CMKs |
| CloudWatch.9 | S3 bucket policy changes |
| CloudWatch.10 | AWS Config configuration changes |
| CloudWatch.11 | Security group changes |
| CloudWatch.12 | Network ACL (NACL) changes |
| CloudWatch.13 | Network gateway changes |
| CloudWatch.14 | Route table changes |

## Repo structure

```
.
├── README.md              ← you are here
├── RUNBOOK.md              ← full step-by-step remediation guide
└── evidence/               ← console screenshots, one set per stage (see RUNBOOK.md for the mapping)
```

## Tooling / environment

- AWS Management Console (no CLI/Terraform — intentional, for step-by-step traceability)
- AWS Security Hub — CIS AWS Foundations Benchmark v1.2.0 standard
- AWS CloudTrail, CloudWatch Logs, CloudWatch Alarms, SNS
- Region: `ap-south-1` (Asia Pacific — Mumbai)

## Notes on the account IDs / ARNs in this repo

All account IDs, ARNs, and resource identifiers shown in `RUNBOOK.md` and the screenshots are from a **personal lab AWS account** used solely for this project. Before publishing, replace any real account ID with a placeholder (e.g. `123456789012`) if you fork or adapt this runbook for your own use.

## Related / next in this portfolio

- **Splunk SSH brute-force detection** — home-lab detection project (Splunk on EC2), being extended with MITRE ATT&CK T1110 mapping and a Lambda-based auto-blocking response via SNS.
- **Threat & Vulnerability Management** project with custom risk scoring (planned).

## License

MIT — feel free to reuse the runbook structure and metric filter patterns for your own AWS account hardening.
