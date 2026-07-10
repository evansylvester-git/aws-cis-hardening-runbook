<div align="center">

# 🛡️ CIS AWS Foundations Benchmark v1.2.0 — Hardening Runbook

**Hands-on remediation of a live AWS account, validated end-to-end through AWS Security Hub.**

![AWS](https://img.shields.io/badge/AWS-Security%20Hub-orange?logo=amazonaws&logoColor=white)
![CIS](https://img.shields.io/badge/CIS-AWS%20Foundations%20v1.2.0-blue)
![Score](https://img.shields.io/badge/CIS%20Score-98%25-brightgreen)
![Method](https://img.shields.io/badge/Method-Console--first-lightgrey)
![Region](https://img.shields.io/badge/Region-ap--south--1-9cf)
![License](https://img.shields.io/badge/License-MIT-yellow)

</div>

---

> **Why this project exists**
> Security Hub tells you *what's* failing. It doesn't tell you *how* to fix it, or *why* the fix is safe to apply on a live account. This runbook closes that gap — the step-by-step guide I wish I'd had, written while doing the remediation myself rather than after the fact.

---

## 📊 Results

<div align="center">

| Metric | Before | After |
|:---|:---:|:---:|
| **CIS Foundations Benchmark score** | 85% | **🟢 98%** |
| **Failed CIS controls** | 54+ | **1** *(pending Security Hub's 12–24h refresh)* |
| **Controls remediated in this project** | — | **13 CloudWatch controls** + IAM, KMS, EC2, CloudTrail |

</div>

📄 **[Read the full walkthrough → `RUNBOOK.md`](./RUNBOOK.md)**
🖼️ **[See screenshot evidence → `evidence/`](./evidence)**

---

## 🎯 Scope

Remediation was performed **manually through the AWS Management Console** (`ap-south-1` — Asia Pacific, Mumbai) — deliberately, not via CLI or IaC — so every change could be verified visually against Security Hub before moving to the next control.

<details open>
<summary><b>🔑 IAM</b></summary>

| Control | What was done |
|---|---|
| `IAM.11–17` | Hardened account password policy — min length, complexity, reuse prevention, expiration |
| — | Enrolled MFA on the `splunk-ec2-user` service account (home-lab Splunk EC2 instance) |
| — | Migrated directly-attached IAM policies to a dedicated `splunk-ec2-user-group`, removing user-level attachments |
| `IAM.18` | Created a dedicated `AWSSupportAccessRole` for support-case access, instead of using root or admin credentials |

</details>

<details open>
<summary><b>🔐 KMS</b></summary>

| Control | What was done |
|---|---|
| `KMS.4` | Enabled automatic annual key rotation on the customer-managed KMS key used for CloudTrail log encryption |

</details>

<details open>
<summary><b>🖥️ EC2 / VPC</b></summary>

| Control | What was done |
|---|---|
| `EC2.2` | Removed the default VPC in the account region |
| `EC2.6` | VPC Flow Logs *(in progress / verify)* |

</details>

<details open>
<summary><b>🧭 CloudTrail</b></summary>

| Control | What was done |
|---|---|
| — | Multi-region trail (`cis-org-trail`) with SSE-KMS encryption and log file validation enabled |
| — | CloudTrail → CloudWatch Logs integration — a **prerequisite** for every CloudWatch.2–.14 control, since they all read from the log group CloudTrail delivers into |
| `CloudTrail.7` | S3 access logging on the CloudTrail bucket *(in progress / verify)* |

</details>

---

## ⭐ CloudWatch — the core of this project

All **13** log metric filter + alarm controls in the benchmark, each following the same repeatable pattern:

```
Metric filter  →  aws-cloudtrail-logs-894759051479-758c27c5   (CloudTrail-fed log group)
Custom metric  →  CISBenchmark namespace
Alarm          →  CIS-3.x-<ShortName>  →  cis-cloudtrail-alarms (SNS topic)
```

| Control | Detects |
|:---|:---|
| `CloudWatch.2` | 🚫 Unauthorized API calls |
| `CloudWatch.3` | 🔓 Sign-in without MFA |
| `CloudWatch.4` | 👑 Root account usage |
| `CloudWatch.5` | 📝 IAM policy changes |
| `CloudWatch.6` | 🧭 CloudTrail configuration changes |
| `CloudWatch.7` | ❌ Console sign-in failures |
| `CloudWatch.8` | 🔑 Disabling/deleting CMKs |
| `CloudWatch.9` | 🪣 S3 bucket policy changes |
| `CloudWatch.10` | ⚙️ AWS Config configuration changes |
| `CloudWatch.11` | 🛡️ Security group changes |
| `CloudWatch.12` | 🌐 Network ACL (NACL) changes |
| `CloudWatch.13` | 🚪 Network gateway changes |
| `CloudWatch.14` | 🗺️ Route table changes |

---

## 🗂️ Repo structure

```
.
├── README.md          ← you are here
├── RUNBOOK.md          ← full step-by-step remediation guide
└── evidence/           ← console screenshots, one set per stage (mapped in RUNBOOK.md)
```

---

## 🧰 Tooling / environment

- **AWS Management Console** (no CLI/Terraform — intentional, for step-by-step traceability)
- **AWS Security Hub** — CIS AWS Foundations Benchmark v1.2.0 standard
- **AWS CloudTrail, CloudWatch Logs, CloudWatch Alarms, SNS**
- **Region:** `ap-south-1` (Asia Pacific — Mumbai)

---

## ⚠️ A note on account IDs / ARNs

All account IDs, ARNs, and resource identifiers shown in `RUNBOOK.md` and the screenshots come from a **personal lab AWS account** used solely for this project. If you fork or adapt this runbook, replace any real account ID with a placeholder (e.g. `123456789012`) before publishing.

---

## 🔜 Related / next in this portfolio

- **Splunk SSH brute-force detection** — home-lab detection project (Splunk on EC2), being extended with MITRE ATT&CK `T1110` mapping and a Lambda-based auto-blocking response via SNS.
- **Threat & Vulnerability Management** project with custom risk scoring *(planned)*.

---

## 📄 License

MIT — feel free to reuse the runbook structure and metric filter patterns for your own AWS account hardening.

<div align="center">

*Built to document real remediation work — not a checklist, a walkthrough.*

</div>