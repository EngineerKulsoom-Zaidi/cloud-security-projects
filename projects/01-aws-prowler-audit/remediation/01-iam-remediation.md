# 🔐 IAM Remediation

## Findings Addressed

| Severity | Finding |
|----------|---------|
| 🔴 Critical | Active root account access key |
| 🟠 High | MFA not enabled on root account |
| 🟠 High | IAM user with console access and no MFA |
| 🟠 High | Weak password policy |
| 🟡 Medium | Access keys not rotated in 90+ days |
| 🟡 Medium | Unused IAM credentials not disabled |

---

## Fix 1 — Delete Root Account Access Key 🔴

### Why It Matters
The root account has full, unrestricted access to every AWS service and resource. An active root access key is the most dangerous credential in AWS — if leaked, an attacker can do anything including deleting your entire account.

### Steps (AWS Console)
1. Log in as root → click your account name (top right) → **Security Credentials**
2. Scroll to **Access keys** section
3. Click **Delete** on any active root access key
4. Confirm deletion

### Via AWS CLI (to verify no root keys exist)
```bash
aws iam get-account-summary --query 'SummaryMap.AccountAccessKeysPresent'
# Should return: 0
```

> ✅ **Expected result:** No active root access keys. Value should be `0`.

---

## Fix 2 — Enable MFA on Root Account 🟠

### Why It Matters
Without MFA, anyone who gets the root password has full access to your account. MFA adds a second layer — even if the password is stolen, the attacker cannot log in.

### Steps (AWS Console)
1. Log in as root → **Security Credentials**
2. Scroll to **Multi-factor authentication (MFA)**
3. Click **Assign MFA device**
4. Choose **Authenticator app** (Google Authenticator / Authy)
5. Scan the QR code and enter two consecutive OTP codes to confirm
6. Click **Add MFA**

### Verify via AWS CLI
```bash
aws iam get-account-summary --query 'SummaryMap.AccountMFAEnabled'
# Should return: 1
```

> ✅ **Expected result:** Returns `1` confirming MFA is enabled on root.

---

## Fix 3 — Enable MFA on IAM Users 🟠

### Steps (AWS Console)
1. Go to **IAM** → **Users** → click the username
2. Go to **Security credentials** tab
3. Under **Multi-factor authentication (MFA)** → click **Assign MFA device**
4. Follow the same steps as root MFA setup above

### Via AWS CLI
```bash
# List all IAM users
aws iam list-users --query 'Users[*].UserName'

# Check if a specific user has MFA enabled
aws iam list-mfa-devices --user-name <username>
# Should return a device — if empty, MFA is not set up
```

> ✅ **Expected result:** Every IAM user with console access has an MFA device listed.

---

## Fix 4 — Set a Strong IAM Password Policy 🟠

### Why It Matters
A weak password policy allows users to set simple passwords that are easy to brute-force or guess.

### Via AWS CLI
```bash
aws iam update-account-password-policy \
  --minimum-password-length 14 \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --require-numbers \
  --require-symbols \
  --allow-users-to-change-password \
  --max-password-age 90 \
  --password-reuse-prevention 5
```

### Verify
```bash
aws iam get-account-password-policy
```

> ✅ **Expected result:** Policy shows minimum 14 characters, all complexity rules enabled, 90-day expiry.

---

## Fix 5 — Rotate or Disable Old Access Keys 🟡

### Why It Matters
Access keys older than 90 days are a security risk — they may have been exposed without your knowledge.

### Find Old Keys
```bash
# List all access keys and their last used date
aws iam list-users --query 'Users[*].UserName' --output text | \
  xargs -I {} aws iam list-access-keys --user-name {}
```

### Rotate a Key
```bash
# Create a new key first
aws iam create-access-key --user-name <username>

# Then delete the old one
aws iam delete-access-key --user-name <username> --access-key-id <old-key-id>
```

### Disable Unused Keys
```bash
aws iam update-access-key \
  --user-name <username> \
  --access-key-id <key-id> \
  --status Inactive
```

> ✅ **Expected result:** No active access keys older than 90 days.

---

## Summary

| Finding | Action Taken | Status |
|---------|-------------|--------|
| Active root access key | Deleted immediately via console | ✅ Done |
| No MFA on root | Enabled virtual MFA (Authenticator app) | ✅ Done |
| No MFA on IAM users | Enabled MFA on all console users | ✅ Done |
| Weak password policy | Applied strong policy via AWS CLI | ✅ Done |
| Old access keys | Rotated keys older than 90 days | ✅ Done |

---

> 📝 *Next: See [02-cloudtrail-remediation.md](./02-cloudtrail-remediation.md) for CloudTrail fixes.*
