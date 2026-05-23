# 🪵 CloudTrail Remediation

## Findings Addressed

| Severity | Finding |
|----------|---------|
| 🟠 High | CloudTrail not enabled in all regions |
| 🟠 High | CloudTrail log encryption not enabled |
| 🟠 High | CloudTrail log file validation disabled |
| 🟠 High | No CloudWatch Logs integration |
| 🟠 High | S3 bucket storing logs not versioned |

---

## Fix 1 — Enable Multi-Region CloudTrail Logging 🟠

### Why It Matters
Single-region CloudTrail only captures API activity in one region. Attackers can operate in other regions completely undetected. Multi-region logging ensures every AWS API call across every region is recorded.

### Via AWS CLI
```bash
# Create a new multi-region trail
aws cloudtrail create-trail \
  --name my-security-trail \
  --s3-bucket-name <your-cloudtrail-bucket> \
  --is-multi-region-trail \
  --enable-log-file-validation

# Start logging
aws cloudtrail start-logging --name my-security-trail
```

### If trail already exists — update it
```bash
aws cloudtrail update-trail \
  --name my-security-trail \
  --is-multi-region-trail \
  --enable-log-file-validation
```

### Verify
```bash
aws cloudtrail describe-trails --query 'trailList[*].{Name:Name,MultiRegion:IsMultiRegionTrail}'
# IsMultiRegionTrail should be: true
```

> ✅ **Expected result:** `IsMultiRegionTrail: true` for your trail.

---

## Fix 2 — Enable CloudTrail Log Encryption with KMS 🟠

### Why It Matters
Without encryption, anyone with access to the S3 bucket can read your CloudTrail logs. KMS encryption ensures logs are only readable by authorized principals.

### Step 1 — Create a KMS Key (if you don't have one)
```bash
aws kms create-key --description "CloudTrail Encryption Key"

# Note the KeyId from the output, then create an alias
aws kms create-alias \
  --alias-name alias/cloudtrail-key \
  --target-key-id <key-id>
```

### Step 2 — Add KMS Key Policy for CloudTrail
CloudTrail needs permission to use the key. Add this to your KMS key policy:

```json
{
  "Sid": "Allow CloudTrail to encrypt logs",
  "Effect": "Allow",
  "Principal": {
    "Service": "cloudtrail.amazonaws.com"
  },
  "Action": [
    "kms:GenerateDataKey*",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

### Step 3 — Update Trail to Use KMS Encryption
```bash
aws cloudtrail update-trail \
  --name my-security-trail \
  --kms-key-id alias/cloudtrail-key
```

### Verify
```bash
aws cloudtrail describe-trails \
  --query 'trailList[*].{Name:Name,KMSKey:KMSKeyId}'
# KMSKeyId should show your key ARN
```

> ✅ **Expected result:** KMS key ARN is associated with the trail.

---

## Fix 3 — Enable Log File Validation 🟠

### Why It Matters
Log file validation uses SHA-256 hashing to create a digest file for every log. This lets you verify that logs have not been modified, deleted, or tampered with after delivery.

### Via AWS CLI
```bash
aws cloudtrail update-trail \
  --name my-security-trail \
  --enable-log-file-validation
```

### Verify
```bash
aws cloudtrail describe-trails \
  --query 'trailList[*].{Name:Name,LogValidation:LogFileValidationEnabled}'
# LogFileValidationEnabled should be: true
```

> ✅ **Expected result:** `LogFileValidationEnabled: true`

---

## Fix 4 — Integrate CloudTrail with CloudWatch Logs 🟠

### Why It Matters
Sending CloudTrail logs to CloudWatch allows you to set up **metric filters and alarms** — for example, alerting when the root account is used or when someone disables CloudTrail.

### Step 1 — Create a CloudWatch Log Group
```bash
aws logs create-log-group --log-group-name CloudTrail/SecurityAudit
```

### Step 2 — Create an IAM Role for CloudTrail → CloudWatch
```bash
# Create role trust policy (save as trust-policy.json)
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "cloudtrail.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name CloudTrailToCloudWatchRole \
  --assume-role-policy-document file://trust-policy.json
```

### Step 3 — Update Trail with CloudWatch Integration
```bash
aws cloudtrail update-trail \
  --name my-security-trail \
  --cloud-watch-logs-log-group-arn arn:aws:logs:<region>:<account-id>:log-group:CloudTrail/SecurityAudit \
  --cloud-watch-logs-role-arn arn:aws:iam::<account-id>:role/CloudTrailToCloudWatchRole
```

> ✅ **Expected result:** CloudTrail logs appear in CloudWatch Log Group within minutes.

---

## Fix 5 — Enable Versioning on CloudTrail S3 Bucket 🟠

### Why It Matters
Versioning protects your CloudTrail logs from accidental or malicious deletion. Even if a log file is deleted, the previous version is retained.

### Via AWS CLI
```bash
aws s3api put-bucket-versioning \
  --bucket <your-cloudtrail-bucket> \
  --versioning-configuration Status=Enabled
```

### Verify
```bash
aws s3api get-bucket-versioning --bucket <your-cloudtrail-bucket>
# Status should be: Enabled
```

> ✅ **Expected result:** `"Status": "Enabled"`

---

## Summary

| Finding | Action Taken | Status |
|---------|-------------|--------|
| Single-region logging | Updated trail to multi-region | ✅ Done |
| No log encryption | Created KMS key and attached to trail | ✅ Done |
| Log file validation disabled | Enabled via `update-trail` | ✅ Done |
| No CloudWatch integration | Created log group and linked trail | ✅ Done |
| S3 bucket not versioned | Enabled versioning on log bucket | ✅ Done |

---

> 📝 *Next: See [03-s3-remediation.md](./03-s3-remediation.md) for S3 fixes.*
