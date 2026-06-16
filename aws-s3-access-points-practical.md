# 🎯 AWS S3 Access Points — Practical Guide

> **Goal:** Use S3 Access Points to give different IAM users scoped access to specific "folders" (prefixes) within the same bucket — e.g., a Sales user can only access the `sales/` folder via the `sales-AP` access point, and an HR user only the `hr/` folder via `hr-AP`.

---

## 📋 Prerequisites

- An active AWS account with administrator access
- Basic understanding of IAM policies and S3 bucket policies

---

## 🗂️ Concept: S3 Access Points

**S3 Access Points** are named network endpoints attached to a bucket that simplify managing **data access for applications/users sharing a single bucket**. Instead of writing one giant, complex bucket policy covering every user and use case, each access point can have its **own policy and permissions** — making it easier to control and scale access for **hundreds of applications or users**.

> 💡 **Typical use case:** A single bucket stores data for multiple departments (e.g., `sales/` and `hr/` folders). Instead of giving every user direct bucket-wide permissions, you create **one access point per department**, each scoped to its own folder/prefix — keeping access clean, auditable, and easy to manage.

---

## 🗂️ Architecture Overview

```
Bucket: company-data-bucket
├── sales/        ← accessible via Access Point: sales-AP   (by user-sales)
└── hr/           ← accessible via Access Point: hr-AP       (by user-hr)
```

---

## 🔧 Step-by-Step Instructions

### Step 1 — Create the Bucket

1. Search **S3** → **Create bucket**

| Field | Value |
|---|---|
| Bucket name | e.g., `company-data-bucket` (must be globally unique) |
| Other settings | Leave as default |

2. Click **Create bucket**

---

### Step 2 — Create Folders Inside the Bucket

1. Open the bucket → click **Create folder**

| Folder | Name |
|---|---|
| Folder 1 | `sales` |
| Folder 2 | `hr` |

2. Click **Create folder** for each one

---

### Step 3 — Enable Access Points to Be Delegated (Bucket Policy)

Before creating access points, the **bucket policy** must delegate access control to the access points. This is done using the `s3:DataAccessPointAccount` condition.

1. Open the bucket → **Permissions** tab → **Bucket policy** → **Edit**
2. Use the **AWS Policy Generator** (https://awspolicygen.s3.amazonaws.com/policygen.html) to build the policy:

| Field | Value |
|---|---|
| Type of Policy | S3 Bucket Policy |
| Effect | Allow |
| Principal | `*` |
| Actions | (select all relevant actions, e.g., `*` for all S3 actions, or scope to `GetObject`, `PutObject`, `ListBucket`) |
| ARN | `arn:aws:s3:::company-data-bucket/*` |

3. Click **Add Condition**:

| Field | Value |
|---|---|
| Condition | `StringEquals` |
| Key | `s3:DataAccessPointAccount` |
| Value | Your **AWS Account ID** |

4. Click **Add Condition** → **Add Statement** → **Generate Policy**
5. Copy the generated JSON and paste it into the bucket policy editor

Example resulting policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "*",
      "Resource": "arn:aws:s3:::company-data-bucket/*",
      "Condition": {
        "StringEquals": {
          "s3:DataAccessPointAccount": "<your-account-id>"
        }
      }
    }
  ]
}
```

> 🔍 **What this does:** It tells the bucket "delegate access decisions to any access point belonging to this AWS account" — the actual fine-grained permissions will be defined on each **access point's own policy**.

6. Click **Save changes**

---

### Step 4 — Create IAM Policies (One per User)

Open a **new browser tab** → go to **IAM → Policies → Create policy**

#### Policy for `user-sales`

1. Select service: **S3**
2. Under **Access level**, expand **List** → check **List Access Point** (and other actions as needed: `ListBucket`, `ListMultiRegionAccessPoint`, `GetAccessPoint`)
3. Click **Next**, name the policy: `sales-access-point-policy`
4. Click **Create policy**

> 💡 This base IAM policy allows the user to *discover/use* access points. The actual data permissions (read/write to `sales/`) will be enforced by the **access point policy** in Step 6.

#### Policy for `user-hr`

Repeat the same steps to create: `hr-access-point-policy`

---

### Step 5 — Create IAM Users

Go to **IAM → Users → Create user**

#### User 1 — `user-sales`

| Field | Value |
|---|---|
| User name | `user-sales` |
| Provide user access to AWS Management Console | **Uncheck** (not needed — this user will use programmatic/CLI access via console sign-in link with access keys, or console with a password if preferred) |

> If console access is needed, check the box, set a custom password, and **uncheck "User must create a new password at next sign-in"** for simplicity in this practical.

1. Click **Next**
2. Permissions options: **Attach policies directly**
3. Select `sales-access-point-policy`
4. Click **Next** → **Create user**

#### User 2 — `user-hr`

Repeat the same steps:
- User name: `user-hr`
- Attach policy: `hr-access-point-policy`

---

### Step 6 — Create Access Points

Go to **S3 → Access Points → Create access point**

#### Access Point 1 — `sales-AP`

| Field | Value |
|---|---|
| Access point name | `sales-AP` |
| Bucket | Browse → select `company-data-bucket` |
| Network origin | **Internet** |

##### Access Point Policy

Scroll to **Access point policy** → click **Policy examples** to see sample templates, then write a policy granting `user-sales` access **only to the `sales/` prefix**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<account-id>:user/user-sales"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:us-west-2:<account-id>:accesspoint/sales-AP/object/sales/*"
      ]
    }
  ]
}
```

> 🔍 **ARN format for access point objects:**
> `arn:aws:s3:<region>:<account-id>:accesspoint/<access-point-name>/object/<prefix>/*`

Click **Create access point**

#### Access Point 2 — `hr-AP`

Repeat with:

| Field | Value |
|---|---|
| Access point name | `hr-AP` |
| Bucket | `company-data-bucket` |
| Network origin | Internet |

Access point policy — grant `user-hr` access only to `hr/*`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<account-id>:user/user-hr"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:us-west-2:<account-id>:accesspoint/hr-AP/object/hr/*"
      ]
    }
  ]
}
```

Click **Create access point**

---

## ✅ Step 7 — Test Access

### Test as `user-sales`

1. Go to **IAM → Users → `user-sales`**
2. Get the **console sign-in link**, or generate **security credentials** (Access Key) as needed
3. Open the sign-in link in a **new private/incognito browser window**
4. Log in as `user-sales`

#### Test 1 — Direct Bucket Access (should fail)

1. Try to navigate to **S3 → `company-data-bucket`** → **sales** folder → **Upload**

> ❌ **Result:** Access Denied — `user-sales` does not have direct bucket-level permissions; access must go **through the access point**.

#### Test 2 — Access Point Access (should succeed for sales/ only)

1. Go to **S3 → Access Points → `sales-AP`**
2. Click **Browse** (or open the access point)
3. Navigate to the **`sales`** folder
4. Click **Upload** → select a file → **Upload**

> ✅ **Result:** Upload succeeds — `user-sales` can access data **only through `sales-AP`**, and only within the `sales/` prefix.

5. Try navigating to the `hr/` folder via `sales-AP`

> ❌ **Result:** Access Denied — the access point policy restricts `user-sales` to the `sales/*` prefix only.

### Test as `user-hr`

Repeat the same tests using `user-hr` and `hr-AP` — should be able to upload to `hr/` only, and denied for `sales/` and direct bucket access.

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **Access Point** | A named, dedicated endpoint for accessing a bucket (or a portion of it) with its own policy |
| **Bucket Policy delegation** | The bucket policy must allow access points (via `s3:DataAccessPointAccount`) to manage access |
| **Access Point Policy** | Defines fine-grained permissions (per user/prefix) for that specific access point |
| **Prefix-scoped access** | Using `accesspoint/<name>/object/<prefix>/*` in the Resource ARN restricts access to a "folder" |
| **Network origin** | Whether the access point is reachable from the Internet or only within a VPC |
| **Direct bucket access vs Access Point access** | Direct access uses the bucket policy/IAM permissions; access point access is an additional, separate permission layer |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Forgetting to add the **`s3:DataAccessPointAccount`** condition to the bucket policy — access points won't function without this delegation
- ❌ Using the wrong **ARN format** for access point resources — must include `/object/<prefix>/*`, not just the bucket ARN
- ❌ Forgetting that IAM user policies (Step 4) only grant **discovery/list permissions** — actual data access is controlled by the **access point policy**
- ❌ Testing while logged in as the **admin/root** account — always test with the actual IAM user to validate restrictions
- ❌ Mixing up region in the ARN (`us-west-2` etc.) — must match your bucket's actual region

---

## 🧹 Cleanup (To Avoid Charges)

1. **Delete Access Points**: S3 → Access Points → select `sales-AP` / `hr-AP` → Delete
2. **Delete IAM Users**: IAM → Users → select `user-sales` / `user-hr` → Delete
3. **Delete IAM Policies**: IAM → Policies → select custom policies → Delete
4. **Empty and delete the bucket**: `company-data-bucket`

---

*Practical by: AWS Cloud Lab*
