
# IAM Least Privilege & Cloudsplaining

## Overview
This project demonstrates how to build a least-privilege IAM policy from scratch
and use Cloudsplaining to identify privilege escalation paths in an AWS account.

---

## What is Least Privilege?
The principle of least privilege means granting only the minimum permissions
required to perform a task — nothing more.

The Golden Rule:
> Grant the minimum permissions needed to do the job. Nothing more.

---

## Tools Used
| Tool | Purpose |
|---|---|
| AWS CLI | Create and manage IAM roles and policies |
| Cloudsplaining | Scan IAM policies for escalation paths |
| IAM Access Analyzer | Audit and generate least-privilege policies |

---

## Part 1 — Verify Identity (Never Use Root)

Always confirm you are logged in as an IAM user, not root.

### Command
\`\`\`bash
aws sts get-caller-identity
\`\`\`

### Expected Output
\`\`\`json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "YOUR_ACCOUNT_ID",
    "Arn": "arn:aws:iam::YOUR_ACCOUNT_ID:user/Administrator"
}
\`\`\`

### UserId Prefix Reference
| Prefix | Meaning |
|---|---|
| AIDA... | IAM User - Safe to proceed |
| AROA... | IAM Role |
| ASIA... | Temporary STS session |
| root in ARN | Root user - Stop immediately |

---

## Part 2 — Create the Trust Policy

A trust policy defines who can assume the role (EC2, Lambda, another user etc).

### trust-policy.json
\`\`\`json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
\`\`\`



---

## Part 3 — Create the Least Privilege DynamoDB Policy

This policy grants read-only access to a specific DynamoDB table only.
Zero wildcards. Zero write permissions.

### policy.json
\`\`\`json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBReadOnlyXYZ",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:BatchGetItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:DescribeTable",
        "dynamodb:ListTagsOfResource"
      ],
      "Resource": [
        "arn:aws:dynamodb:us-east-1:YOUR_ACCOUNT_ID:table/xyz",
        "arn:aws:dynamodb:us-east-1:YOUR_ACCOUNT_ID:table/xyz/index/*"
      ]
    }
  ]
}
\`\`\`

### Actions Explained
| Action | Purpose |
|---|---|
| dynamodb:GetItem | Fetch single item by primary key |
| dynamodb:BatchGetItem | Fetch multiple items in one request |
| dynamodb:Query | Query by partition key |
| dynamodb:Scan | Full table scan |
| dynamodb:DescribeTable | Read table metadata |
| dynamodb:ListTagsOfResource | Read tags on the table |

### Intentionally Excluded
| Excluded Action | Reason |
|---|---|
| dynamodb:PutItem | Would allow writes |
| dynamodb:UpdateItem | Would allow edits |
| dynamodb:DeleteItem | Would allow deletes |
| dynamodb:CreateTable | Infrastructure change |
| dynamodb:* | Never use wildcards |



---

## Part 4 — Create IAM Role via AWS CLI

### Step 1: Create the Role
\`\`\`bash
aws iam create-role --role-name DynamoDB-ReadOnly-xyz-Role --assume-role-policy-document file://trust-policy.json
\`\`\`

### Step 2: Create the Policy
\`\`\`bash
aws iam create-policy --policy-name DynamoDB-ReadOnly-xyz --policy-document file://policy.json
\`\`\`

### Step 3: Attach Policy to Role
\`\`\`bash
aws iam attach-role-policy --role-name DynamoDB-ReadOnly-xyz-Role --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/DynamoDB-ReadOnly-xyz
\`\`\`

### Step 4: Verify Role
\`\`\`bash
aws iam get-role --role-name DynamoDB-ReadOnly-xyz-Role
\`\`\`

### Step 5: Verify Policy Attached
\`\`\`bash
aws iam list-attached-role-policies --role-name DynamoDB-ReadOnly-xyz-Role
\`\`\`

### Step 6: Verify Policy Permissions
\`\`\`bash
aws iam get-policy --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/DynamoDB-ReadOnly-xyz
aws iam get-policy-version --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/DynamoDB-ReadOnly-xyz --version-id v1
\`\`\`

---

## Part 5 — Run Cloudsplaining

### Step 1: Install Cloudsplaining
\`\`\`bash
pip install cloudsplaining
\`\`\`

### Step 2: Export IAM Authorization Details
\`\`\`powershell
\$json = aws iam get-account-authorization-details --output json
[System.IO.File]::WriteAllText("\$PWD\authorization-details.json", \$json)
\`\`\`

> Note: Use [System.IO.File]::WriteAllText on Windows PowerShell 5.x
> to avoid UTF-16 BOM encoding issues that break Cloudsplaining.

### Step 3: Create Output Directory
\`\`\`powershell
mkdir cloudsplaining-report
\`\`\`

### Step 4: Run the Scan
\`\`\`bash
cloudsplaining scan --input-file authorization-details.json --output cloudsplaining-report
\`\`\`

### Step 5: Open the Report
\`\`\`powershell
start cloudsplaining-report\index.html
\`\`\`

### Step 6: Scan Your Specific Policy Only
\`\`\`bash
cloudsplaining scan-policy-file --input policy.json
\`\`\`

---

## Part 6 — Cloudsplaining Findings

### Summary
| Risk | Instances | Severity | Source |
|---|---|---|---|
| Privilege Escalation | 1 | High | AWS Managed Policy |
| Data Exfiltration | 1 | Medium | AWS Managed Policy |
| Resource Exposure | 5 | High | AWS Managed Policy |
| Credentials Exposure | 2 | High | AWS Managed Policy |
| Infrastructure Modification | 5 | Low | AWS Managed Policy |

### Key Finding
All risks originated from AWS Managed Policies — NOT the custom
DynamoDB-ReadOnly-xyz policy. The custom policy was clean with zero findings.

### What Each Finding Means
| Finding | Description |
|---|---|
| Privilege Escalation | Role can create/attach policies or pass roles to become admin |
| Data Exfiltration | Can read S3, Secrets Manager without resource constraints |
| Resource Exposure | Can make resources public (S3 ACLs, security groups) |
| Credentials Exposure | Can call sts:AssumeRole or iam:CreateAccessKey broadly |
| Infrastructure Modification | Broad unconstrained EC2/RDS/Lambda actions |

---

## Key Learnings

1. Never use wildcards (*) in production IAM policies
2. Always scope Resource to specific ARNs — never use Resource: *
3. Use IAM Roles over IAM Users for all AWS services
4. AWS Managed Policies like AdministratorAccess contain escalation paths
5. PowerShell 5.x uses UTF-16 encoding by default — always use
   [System.IO.File]::WriteAllText for AWS CLI compatible JSON files
6. Always verify identity with aws sts get-caller-identity before any IAM work
7. Run Cloudsplaining after every policy change to catch regressions

---

## Files in This Project
| File | Description | Safe to Commit |
|---|---|---|
| policy.json | Least privilege DynamoDB read-only policy | Yes |
| trust-policy.json | Trust policy for EC2 service principal | Yes |
| README.md | Full documentation | Yes |
| authorization-details.json | IAM account export | NO - contains sensitive data |
| cloudsplaining-report/ | HTML scan report | NO - contains account details |

---

## References
- [Cloudsplaining GitHub](https://github.com/salesforce/cloudsplaining)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html)
- [DynamoDB IAM Actions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/api-permissions-reference.html)

