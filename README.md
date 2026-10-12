# aws-iam-security-lab
Hands-on AWS IAM lab using the CLI: an explicit-deny guardrail policy, an assumable role with a trust policy, redacted screenshots, deploy/cleanup scripts, and a technical write-up. aws, iam, aws-cli, cloud-security, devsecops, least-privilege, security-lab, learning-in-public

# AWS Multi-Cloud IAM Security Architecture: Guardrails & Cross-Account Roles

A hands-on lab that uses the **AWS CLI** to build two core IAM security building blocks: an explicit-deny **guardrail policy** for destructive actions, and an **assumable role** governed by a separate **trust policy**. Everything was done from the terminal, and IAM itself carries no cost, so the lab is free to reproduce.

> **Author's note:** I am early in my Cloud & DevSecOps journey and building toward AWS certification. This repository documents what I built and what I learned, including the limits of the lab.

---

## Table of Contents

1. [Repository Structure](#repository-structure)
2. [Architecture Overview](#architecture-overview)
3. [Key Concepts: Permission Policy vs. Trust Policy](#key-concepts-permission-policy-vs-trust-policy)
4. [Prerequisites](#prerequisites)
5. [Step-by-Step CLI Implementation](#step-by-step-cli-implementation)
6. [Verification](#verification)
7. [Security Best Practices](#security-best-practices)
8. [Lab Scope & Limitations](#lab-scope--limitations)
9. [Cleanup](#cleanup)
10. [Key Takeaways](#key-takeaways)

---

## Repository Structure

```
aws-iam-security-lab/
├── README.md
├── assets/                      # Redacted terminal screenshots (account IDs hidden)
├── policies/
│   ├── deny-sensitive-deletes.json
│   └── trust-policy.template.json   # Replace ACCOUNT_ID, or let deploy.sh do it
├── scripts/
│   ├── deploy.sh                # Create policy + role, attach, verify
│   └── cleanup.sh               # Remove everything the lab created
└── docs/
    └── AWS-IAM-Hands-On-Lab-Report.pdf   # Full technical write-up
```

For the long-form, first-person technical write-up, see the [Hands-On Lab Report (PDF)](docs/AWS-IAM-Hands-On-Lab-Report.pdf).

**Quick start with the scripts:**

```bash
git clone https://github.com/xhaikhfahad/aws-iam-security-lab.git
cd aws-iam-security-lab
./scripts/deploy.sh     # creates policy and role in the account your CLI is using
./scripts/cleanup.sh    # removes them again
```

The scripts read your account ID from `aws sts get-caller-identity`, so no real account ID is stored in this repository.

---

## Architecture Overview

```
                  ┌──────────────────────────────┐
                  │  IAM User (admin, CLI access) │
                  └──────────────┬───────────────┘
                                 │ sts:AssumeRole
                                 ▼
        ┌─────────────────────────────────────────────────┐
        │  CrossAccountSecurityRole                        │
        │                                                  │
        │  Trust Policy      → WHO may assume this role    │
        │  Permission Policy → WHAT the role may/may not do│
        │    └─ DenySensitiveDeletesPolicy (explicit Deny) │
        └─────────────────────────────────────────────────┘
                 Denied: iam:DeleteAccessKey, s3:DeleteBucket
```

| Component | Purpose |
|---|---|
| `DenySensitiveDeletesPolicy` | Customer-managed policy with an **explicit Deny** on `iam:DeleteAccessKey` and `s3:DeleteBucket` |
| `trust-policy.json` | Defines which principal is allowed to call `sts:AssumeRole` on the role |
| `CrossAccountSecurityRole` | IAM role that combines the trust policy and the attached deny guardrail |

---

## Key Concepts: Permission Policy vs. Trust Policy

Every IAM role has **two different kinds of policy**, and mixing them up is a common beginner mistake.

| | Permission Policy | Trust Policy |
|---|---|---|
| **Question answered** | *What can this role do?* | *Who can become this role?* |
| **Attached via** | `attach-role-policy` (or inline) | Set at role creation (`--assume-role-policy-document`) |
| **Key element** | `Action` + `Resource` | `Principal` + `sts:AssumeRole` |
| **In this lab** | `DenySensitiveDeletesPolicy` | `trust-policy.json` |

**Explicit Deny always wins.** In IAM policy evaluation, an explicit `Deny` overrides any `Allow`, regardless of where the Allow comes from. That is what makes a Deny statement a reliable guardrail.

---

## Prerequisites

- An AWS account and an IAM identity with permission to create policies and roles
- AWS CLI v2 configured (`aws configure`)
- A Linux/macOS terminal

Confirm which identity you are using before changing anything:

```bash
aws sts get-caller-identity
```

---

## Step-by-Step CLI Implementation

### Step 1: Write the guardrail policy document

Create `deny-sensitive-deletes.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ExplicitDenySensitiveDeletes",
      "Effect": "Deny",
      "Action": [
        "iam:DeleteAccessKey",
        "s3:DeleteBucket"
      ],
      "Resource": "*"
    }
  ]
}
```

![Step 1: Policy JSON](assets/screenshot1.png)

### Step 2: Create the IAM policy

```bash
aws iam create-policy \
  --policy-name DenySensitiveDeletesPolicy \
  --policy-document file://deny-sensitive-deletes.json
```

The response confirms the policy was created with `AttachmentCount: 0`, meaning it exists but is not attached to anything yet.

![Step 2: Create the IAM policy](assets/screenshot2.png)

### Step 3: Write the trust policy and create the role

Create `trust-policy.json` (a template is provided in `policies/trust-policy.template.json`). Replace `<ACCOUNT_ID>` with your own 12-digit account ID. **Do not keep the angle brackets**, because `<` and `>` are not valid inside an ARN.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create the role with this trust policy:

```bash
aws iam create-role \
  --role-name CrossAccountSecurityRole \
  --assume-role-policy-document file://trust-policy.json
```

![Step 3: Trust policy and role creation](assets/screenshot3.png)

### Step 4: Attach the guardrail to the role

```bash
aws iam attach-role-policy \
  --role-name CrossAccountSecurityRole \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/DenySensitiveDeletesPolicy
```

![Step 4: Attach policy and verify](assets/screenshot4.png)

---

## Verification

Confirm the attachment:

```bash
aws iam list-attached-role-policies --role-name CrossAccountSecurityRole
```

Expected output:

```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "DenySensitiveDeletesPolicy",
            "PolicyArn": "arn:aws:iam::<ACCOUNT_ID>:policy/DenySensitiveDeletesPolicy"
        }
    ]
}
```

Optionally, test the guardrail without making real changes by using the IAM policy simulator:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<ACCOUNT_ID>:role/CrossAccountSecurityRole \
  --action-names s3:DeleteBucket iam:DeleteAccessKey
```

Both actions should return `explicitDeny`.

---

## Security Best Practices

1. **Least privilege.** Grant only the permissions a role needs. Use deny statements as a safety net on top of that, not as a replacement.
2. **Prefer roles over long-lived keys.** Assuming a role issues short-lived credentials from STS, which reduces the risk of leaked static access keys.
3. **Scope trust policies tightly.** Trusting an account `:root` delegates the decision to that account's IAM policies. Where possible, name a specific role or user as the principal.
4. **Add conditions for real cross-account access.** Use `sts:ExternalId` for third-party access and `aws:MultiFactorAuthPresent` to require MFA when assuming the role.
5. **Narrow `Resource` where you can.** This lab uses `"*"` for simplicity. In production, scope denies to specific buckets or prefixes where that fits your design.
6. **Use organization-level guardrails for scale.** Service Control Policies (SCPs) and permissions boundaries enforce limits that identity policies alone cannot.
7. **Never commit identifiers carelessly.** Redact account IDs, user IDs, policy IDs, and ARNs from screenshots, logs, and code before publishing. Account IDs are not secrets on their own, but they help attackers map your environment.
8. **Never commit credentials.** Keep access keys out of repositories and use a secret scanner (for example `gitleaks`) in CI.
9. **Monitor.** Enable CloudTrail so role assumptions and denied actions are recorded.

---

## Lab Scope & Limitations

Being precise about what this lab does and does not prove:

- The deny policy is **identity-based** and attached to one role. It restricts sessions that assume that role. It does **not** restrict other users or roles in the account.
- The trust policy in this lab names the **same account** that owns the role, so the role is assumable within one account. A true cross-account setup would use a different account ID as the principal, ideally with ExternalId or MFA conditions.
- This lab demonstrates the building blocks and the evaluation logic. Org-wide enforcement would need SCPs or permissions boundaries (see best practice 6).

---

## Cleanup

Remove the resources to leave the account clean:

```bash
aws iam detach-role-policy \
  --role-name CrossAccountSecurityRole \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/DenySensitiveDeletesPolicy

aws iam delete-role --role-name CrossAccountSecurityRole

aws iam delete-policy \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/DenySensitiveDeletesPolicy
```

---

## Key Takeaways

- Explicit Deny overrides Allow, which makes it a dependable guardrail.
- A role needs two policy types: a **trust policy** (who) and a **permission policy** (what).
- The AWS CLI makes IAM work repeatable and scriptable, and the same steps can be turned into Infrastructure as Code later (for example Terraform).
- Sanitize everything before sharing your work publicly.

---

## About

Built as part of my learning path in Cloud Computing, AWS, and DevSecOps. Feedback and suggestions are welcome through issues or pull requests.

**Tags:** `aws` `iam` `cloud-security` `devsecops` `aws-cli` `least-privilege`
