# 👤 AWS IAM (Identity and Access Management) 

> **Goal:** Create two IAM users with different custom permission policies — one with specific delete/create permissions, and another with a region-restricted (conditional) policy.

---

## 📋 Prerequisites

- An active AWS account with administrator access
- Basic understanding of JSON (for custom policies)

---

## 🗂️ User Requirements

| User | Permissions |
|---|---|
| **shyam** | Delete S3 buckets/objects + Create EC2 instances |
| **ghanshyam** | Create S3 buckets (**only in Tokyo region**) + Full access to EC2 delete actions |

---

## 🔧 Step-by-Step Instructions

### Step 1 — Navigate to IAM

1. In the search bar, type **IAM** and open the IAM service

---

## 👤 Part 1 — Create User: shyam

### Step 2 — Create the User

1. Go to **Users → Create user**

| Field | Value |
|---|---|
| User name | `shyam` |
| Provide user access to AWS Management Console | Optional (check if console login needed) |
| Console password | Custom/Auto-generate |

2. Click **Next**

### Step 3 — Attach Permissions (Custom Policy)

Since shyam needs **specific** permissions (S3 delete + EC2 create — not full access to either), create a **custom policy**.

1. Select **Attach policies directly** → click **Create policy** (opens in new tab)
2. In the Create Policy screen, switch to the **JSON** tab and enter:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3DeletePermissions",
      "Effect": "Allow",
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteBucket"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EC2CreatePermissions",
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    }
  ]
}
```

3. Click **Next**

| Field | Value |
|---|---|
| Policy name | `shyam-s3delete-ec2create-policy` |
| Description | S3 delete + EC2 create permissions for shyam |

4. Click **Create policy**

### Step 4 — Attach the Policy to shyam

1. Go back to the **Create user** tab
2. Click the **refresh icon** in the policy list
3. Search for `shyam-s3delete-ec2create-policy` and select it (checkbox)
4. Click **Next** → **Create user**

---

## 👤 Part 2 — Create User: ghanshyam

### Step 5 — Create the User

1. Go to **Users → Create user**

| Field | Value |
|---|---|
| User name | `ghanshyam` |
| Provide user access to AWS Management Console | Optional |

2. Click **Next**

### Step 6 — Attach Permissions (Custom Policy with Condition)

ghanshyam needs:
- **S3 bucket creation — restricted to Tokyo region (`ap-northeast-1`)**
- **Full access to EC2 delete actions**

> ⚠️ **Note:** S3 is a **global service** — bucket-level actions don't natively use `aws:RequestedRegion` the way regional services do, but AWS IAM still supports a region condition on the `CreateBucket` API call using `aws:RequestedRegion`, since the request itself is sent to a specific regional endpoint.

1. Select **Attach policies directly** → click **Create policy** (opens in new tab)
2. Switch to the **JSON** tab and enter:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3CreateBucketTokyoOnly",
      "Effect": "Allow",
      "Action": "s3:CreateBucket",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "ap-northeast-1"
        }
      }
    },
    {
      "Sid": "EC2DeleteFullAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:TerminateInstances",
        "ec2:DeleteVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteSecurityGroup",
        "ec2:DeleteSubnet",
        "ec2:DeleteVpc",
        "ec2:DeleteNetworkAcl",
        "ec2:DeleteRouteTable",
        "ec2:DeleteInternetGateway",
        "ec2:DeleteNatGateway",
        "ec2:Describe*"
      ],
      "Resource": "*"
    }
  ]
}
```

> 💡 **Tip:** "Full access to EC2 delete" isn't a single AWS-managed action — it means **all delete-type EC2 actions**. The policy above covers the common ones (instances, volumes, snapshots, security groups, networking). `ec2:Describe*` is included so the user can at least view resources before deleting them.

3. Click **Next**

| Field | Value |
|---|---|
| Policy name | `ghanshyam-s3tokyo-ec2delete-policy` |
| Description | S3 bucket creation (Tokyo only) + EC2 delete permissions for ghanshyam |

4. Click **Create policy**

### Step 7 — Attach the Policy to ghanshyam

1. Go back to the **Create user** tab
2. Click the **refresh icon**
3. Search for `ghanshyam-s3tokyo-ec2delete-policy` and select it
4. Click **Next** → **Create user**

---

## ✅ Step 8 — Verify Permissions

### Test shyam:

1. Log in as `shyam` (use the IAM sign-in URL: `https://<account-id>.signin.aws.amazon.com/console`)
2. Try to **launch an EC2 instance** → ✅ should succeed
3. Try to **delete an object/bucket in S3** → ✅ should succeed
4. Try to **delete an EC2 instance** → ❌ should be denied (not in policy)

### Test ghanshyam:

1. Log in as `ghanshyam`
2. Switch region to **Tokyo (ap-northeast-1)** → try to **create an S3 bucket** → ✅ should succeed
3. Switch to any other region (e.g., Mumbai) → try to **create an S3 bucket** → ❌ should be denied
4. Try to **terminate/delete an EC2 instance** (any region) → ✅ should succeed
5. Try to **launch a new EC2 instance** → ❌ should be denied (only delete actions granted)

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **IAM User** | An identity representing a person or application with AWS credentials |
| **Custom Policy** | A user-defined JSON policy specifying exact allowed/denied actions |
| **Action** | The specific API operation being permitted (e.g., `s3:DeleteBucket`) |
| **Resource** | The AWS resource(s) the action applies to (`*` = all resources) |
| **Condition** | Additional constraints on when a policy statement applies (e.g., region) |
| **`aws:RequestedRegion`** | Condition key that restricts actions to a specific AWS region |
| **Effect: Allow/Deny** | Whether the statement permits or blocks the specified actions |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Using AWS-managed policies (like `AmazonS3FullAccess`) when **specific/limited** permissions are required — always use custom policies for granular control
- ❌ Forgetting the `Resource` field — every statement requires it (use `*` for "all resources" if not restricting)
- ❌ Assuming "full access to delete" is a built-in AWS action — it must be built from multiple `Delete*` actions
- ❌ Forgetting to refresh the policy list after creating a new policy in a separate tab
- ❌ Not testing with the actual IAM user login — testing as root/admin won't reveal permission issues

---

## 🧹 Cleanup (Optional)

1. **Delete IAM users**: IAM → Users → select `shyam`/`ghanshyam` → Delete
2. **Delete custom policies**: IAM → Policies → select the custom policies → Delete

---

