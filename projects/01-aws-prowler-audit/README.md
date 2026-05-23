# 🔍 Project 01 — AWS Security Audit with Prowler

## 📌 Overview

In this project, I ran **Prowler** against a test AWS account to perform a real-world cloud security audit. Prowler checks your AWS environment against industry benchmarks like **CIS**, **AWS Well-Architected Framework**, and compliance standards such as SOC 2 and ISO 27001.

The audit uncovered several misconfigurations across **IAM**, **CloudTrail**, and **S3** — all of which were remediated as part of this exercise.

---

## 🛠️ Tool Used

| Tool | Description |
|------|-------------|
| [Prowler](https://github.com/prowler-cloud/prowler) | Open-source AWS security auditing tool |
| AWS CLI | Used for remediation and configuration |
| AWS Console | MFA setup, key management |

---

## 🔍 Findings

### 🔴 Critical

| Check | Finding | Risk |
|-------|---------|------|
| IAM | Active root account access key found | Full account compromise if leaked — root has no permission boundaries |

### 🟠 High

| Check | Finding | Risk |
|-------|---------|------|
| IAM | MFA not enabled on root account | Account takeover via password theft |
| S3 | Bucket publicly accessible | Unintended data exposure |

### 🟡 Medium

| Check | Finding | Risk |
|-------|---------|------|
| CloudTrail | Single-region logging only | Attacker activity in other regions goes undetected |
| CloudTrail | Logs not encrypted with KMS | Log tampering or unauthorized access |

---

## ✅ Remediation

### IAM

- **Deleted the root access key immediately** — root keys are the highest-risk credential in AWS; no use case justifies keeping them active
- Enabled **MFA on the root account** using a virtual authenticator app
- Reviewed IAM users and applied **least-privilege policies**

### CloudTrail

- Enabled **multi-region CloudTrail logging** to capture API activity across all AWS regions
- Configured **KMS encryption** on CloudTrail logs
- Enabled **log file validation** to detect any log tampering

### S3

- Enabled **S3 Block Public Access** at the account level
- Reviewed and restricted **bucket policies** to remove any unintended public access

---

## 💡 Key Learnings

- An **active root access key** is one of the most dangerous AWS misconfigurations — it should be deleted immediately upon discovery
- **Multi-region CloudTrail** is critical; single-region logging creates blind spots attackers can exploit
- Prowler makes it fast and structured to assess your AWS security posture — it's the ideal first step in any AWS security audit
- Remediation is not just about fixing findings — it's about understanding *why* each misconfiguration is risky

---

## 📁 Folder Structure

```
01-aws-prowler-audit/
├── README.md               # This file
├── findings/               # Prowler raw output / reports
└── remediation/            # Remediation scripts or notes
```

---

## 🔗 References

- [Prowler Documentation](https://docs.prowler.cloud/)
- [CIS AWS Foundations Benchmark](https://www.cisecurity.org/benchmark/amazon_web_services)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS CloudTrail Best Practices](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/best-practices-security.html)
- [S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)

---

> 📝 *This project was completed as a hands-on learning exercise using a test AWS account.*
