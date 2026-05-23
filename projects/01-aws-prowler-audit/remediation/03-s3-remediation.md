# 🪣 S3 Remediation

## Findings Addressed

| Severity | Finding |
|----------|---------|
| 🟠 High | S3 bucket publicly accessible |

---

## Fix 1 — Block All Public Access on S3 Bucket 🟠

### Why It Matters
A publicly accessible S3 bucket means anyone on the internet can read (or potentially write) your data. This is one of the most common causes of AWS data breaches. AWS provides a **Block Public Access** setting that overrides any bucket policy or ACL that would otherwise allow public access.

### Step 1 — Block Public Access at the Account Level (Recommended)
This applies to all current and future buckets in your account — the strongest protection.

```bash
aws s3control put-public-access-block \
  --account-id <your-account-id> \
  --public-access-block-configuration \
    BlockPublicAcls=true,\
    IgnorePublicAcls=true,\
    BlockPublicPolicy=true,\
    RestrictPublicBuckets=true
```

### Verify Account-Level Setting
```bash
aws s3control get-public-access-block --account-id <your-account-id>
```

Expected output:
```json
{
  "PublicAccessBlockConfiguration": {
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  }
}
```

---

### Step 2 — Block Public Access at the Bucket Level
Apply to each individual bucket as well for defense in depth.

```bash
aws s3api put-public-access-block \
  --bucket <your-bucket-name> \
  --public-access-block-configuration \
    BlockPublicAcls=true,\
    IgnorePublicAcls=true,\
    BlockPublicPolicy=true,\
    RestrictPublicBuckets=true
```

### Verify Bucket-Level Setting
```bash
aws s3api get-public-access-block --bucket <your-bucket-name>
```

> ✅ **Expected result:** All four settings show `true`.

---

## Fix 2 — Remove Public Bucket Policy (if present)

### Check Existing Bucket Policy
```bash
aws s3api get-bucket-policy --bucket <your-bucket-name>
```

### If the Policy Allows Public Access — Remove It
```bash
aws s3api delete-bucket-policy --bucket <your-bucket-name>
```

Or update the policy to restrict access to specific principals only:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<your-bucket-name>/*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

Apply the updated policy:
```bash
aws s3api put-bucket-policy \
  --bucket <your-bucket-name> \
  --policy file://bucket-policy.json
```

---

## Fix 3 — Enable S3 Bucket Encryption at Rest

### Why It Matters
Encrypting data at rest ensures that even if someone gains physical or unauthorized access to the storage, the data is unreadable without the key.

```bash
aws s3api put-bucket-encryption \
  --bucket <your-bucket-name> \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "alias/aws/s3"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

### Verify
```bash
aws s3api get-bucket-encryption --bucket <your-bucket-name>
```

> ✅ **Expected result:** SSEAlgorithm shows `aws:kms`

---

## Fix 4 — Enable S3 Access Logging

### Why It Matters
Access logging records every request made to your bucket — who accessed it, from where, and when. Essential for auditing and incident investigation.

```bash
# First create a separate bucket for logs
aws s3api create-bucket \
  --bucket <your-bucket-name>-access-logs \
  --region <your-region>

# Enable logging on your main bucket
aws s3api put-bucket-logging \
  --bucket <your-bucket-name> \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "<your-bucket-name>-access-logs",
      "TargetPrefix": "access-logs/"
    }
  }'
```

> ✅ **Expected result:** Access logs appear in the log bucket within a few minutes of activity.

---

## Verify All Fixes with Prowler

After remediating, re-run Prowler on S3 only to confirm all findings are resolved:

```bash
prowler aws --services s3
```

> ✅ **Expected result:** S3 shows 0 High/Critical failures.

---

## Summary

| Finding | Action Taken | Status |
|---------|-------------|--------|
| S3 bucket publicly accessible | Enabled Block Public Access (account + bucket level) | ✅ Done |
| Public bucket policy | Removed / replaced with restrictive policy | ✅ Done |
| No encryption at rest | Enabled SSE-KMS encryption | ✅ Done |
| No access logging | Enabled S3 access logging to separate bucket | ✅ Done |

---

> 📝 *All three service remediations complete. See the [findings summary](../findings/findings-summary.md) for the full audit overview.*
