# 🪣 AWS S3 (Simple Storage Service) 

> **Goal:** Create an S3 bucket, upload a file, attempt to access it (and understand why it fails), then make the file publicly accessible using a pre-signed URL and a bucket policy.

---

## 📋 Prerequisites

- An active AWS account
- Basic understanding of JSON (for bucket policy)

---

## 🗂️ What is S3?

**Amazon S3 (Simple Storage Service)** is object storage. Buckets are containers for objects (files) and have a **globally unique name** across all of AWS (not just your account/region).

---

## 🔧 Step-by-Step Instructions

### Step 1 — Create a Bucket

1. In the search bar, type **S3** and open the S3 service
2. Click **Create bucket**

| Field | Value |
|---|---|
| Bucket name | A **globally unique** name (e.g., `mybucket-yourname-2026`) |
| AWS Region | Choose your preferred region (e.g., `ap-south-1`) |
| Object Ownership | ACLs disabled (default) |
| Block Public Access settings | Leave **all checked** (default) for now |
| Bucket Versioning | Disable (default) |

> ⚠️ **Bucket names share a global namespace** — if a name is taken by *any* AWS account worldwide, you can't use it. Add a unique suffix (your name, random numbers).

3. Click **Create bucket**

---

### Step 2 — Upload a File

1. Click on the **bucket name** to open it
2. Click **Upload**
3. Click **Add files** (or **Add folder**) and select a file from your computer
4. Click **Upload**
5. Once uploaded, click **Close**

---

### Step 3 — Try Accessing the File via URL

1. Click on the **file name** in the bucket
2. Copy the **Object URL**
3. Open it in a new browser tab

> ❌ **Result:** You'll get an `AccessDenied` (403 Forbidden) XML error.
>
> **Why?** By default, S3 buckets and objects are **private**. Block Public Access is enabled, and there's no policy allowing public reads.

---

### Step 4 — Access File Using a Pre-signed URL (Temporary Access)

A **pre-signed URL** grants temporary, time-limited access to a private object without changing bucket permissions.

1. Go back to the bucket → select the file (checkbox)
2. Click **Object actions → Share with a presigned URL**
3. Set an expiration time (e.g., 1 hour — max is 7 days for IAM user credentials)
4. Click **Create presigned URL**
5. The URL is copied to your clipboard — paste it in a browser tab

> ✅ **Result:** The file loads successfully because the pre-signed URL contains temporary credentials/signature.
>
> ⏰ **Limitation:** This URL **expires** after the set time — it's not a permanent public access solution.

---

## 🌍 Making the File Permanently Public (No Time Limit)

To allow **anyone, anytime** to access the file via its plain Object URL, you need to:
1. Disable Block Public Access settings
2. Add a Bucket Policy that explicitly allows public reads

### Step 5 — Edit Block Public Access Settings

1. Open the bucket → go to the **Permissions** tab
2. Scroll to **Block public access (bucket settings)**
3. Click **Edit**
4. **Uncheck all** the checkboxes (removes all public access blocks)
5. Click **Save changes**
6. Type `confirm` in the dialog box and click **Confirm**

> ❌ At this point, the Object URL **still won't work**. Disabling Block Public Access only *allows* public policies to be created — it doesn't grant access by itself. You still need a **Bucket Policy**.

---

### Step 6 — Generate a Bucket Policy

1. Open the **AWS Policy Generator** in a new tab:
   👉 https://awspolicygen.s3.amazonaws.com/policygen.html

2. Fill in the generator form:

| Field | Value |
|---|---|
| Select Type of Policy | **S3 Bucket Policy** |
| Effect | Allow |
| Principal | `*` |
| Actions | `GetObject` |
| Amazon Resource Name (ARN) | Copy from your bucket (see below) |

#### Get your Bucket ARN

3. Go back to S3 → your bucket → **Properties** tab → scroll down to find the **Bucket ARN** (e.g., `arn:aws:s3:::mybucket-yourname-2026`)
4. In the Policy Generator, paste the ARN and append `/*` at the end:
   ```
   arn:aws:s3:::mybucket-yourname-2026/*
   ```
   > The `/*` is important — it applies the policy to **all objects** inside the bucket, not just the bucket itself.

5. Click **Add Statement**
6. Click **Generate Policy**
7. Copy the generated JSON policy

---

### Step 7 — Apply the Bucket Policy

1. Go back to your S3 bucket → **Permissions** tab
2. Scroll to **Bucket policy** → click **Edit**
3. Paste the generated JSON policy

Example policy:
```json
{
  "Id": "Policy1234567890",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1234567890",
      "Action": "s3:GetObject",
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::mybucket-yourname-2026/*",
      "Principal": "*"
    }
  ]
}
```

4. Click **Save changes**

---

## ✅ Step 8 — Verify Public Access

1. Go back to the file in your bucket → copy the **Object URL**
2. Open it in a browser (incognito/private mode recommended, to ensure no cached credentials)

> ✅ **Result:** The file now loads **without any errors and without expiration** — it's publicly accessible to anyone with the URL.

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **Bucket** | Container for storing objects (files) in S3 |
| **Global namespace** | Bucket names must be unique across all AWS accounts worldwide |
| **Object URL** | Direct HTTPS link to an object; works only if access is permitted |
| **Pre-signed URL** | Temporary, time-limited access link generated using credentials |
| **Block Public Access** | Account/bucket-level setting that overrides policies to prevent public access |
| **Bucket Policy** | JSON-based resource policy defining who can do what on a bucket/its objects |
| **`s3:GetObject`** | Permission/action that allows reading (downloading/viewing) an object |
| **`Principal: "*"`** | Wildcard meaning "anyone" (public) |
| **ARN** | Amazon Resource Name — unique identifier for AWS resources |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Choosing a bucket name that's already taken globally — get a unique name error
- ❌ Forgetting to **uncheck Block Public Access** before adding a public bucket policy — policy alone won't work
- ❌ Forgetting the `/*` at the end of the ARN in the Resource field — without it, the policy applies to the bucket itself, not the objects inside it
- ❌ Expecting pre-signed URLs to work forever — they expire based on the set duration
- ❌ Forgetting to type `confirm` when disabling Block Public Access

---

## 🧹 Cleanup (To Avoid Charges)

1. **Empty the bucket**: Select bucket → **Empty** → type `permanently delete` → confirm
2. **Delete the bucket**: Select bucket → **Delete** → type bucket name → confirm

---

