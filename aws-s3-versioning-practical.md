# 🗂️ AWS S3 Versioning — Practical Guide

> **Goal:** Understand S3 Versioning, see how S3 behaves *without* versioning vs *with* versioning enabled, and explore previous versions of an object.

---

## 📋 Prerequisites

- An active AWS account
- A text editor to create a sample file

---

## ❓ Conceptual Questions & Answers

### Q1. What is Versioning?

**S3 Versioning** is a bucket-level feature that allows you to **keep multiple variants (versions) of an object within the same bucket**. Every time you upload a file with the same name (key), S3 keeps the old version instead of overwriting it, and assigns the new upload a unique **version ID**.

---

### Q2. How is it related to Git?

S3 Versioning is conceptually similar to **Git's version history**:

| S3 Versioning | Git |
|---|---|
| Keeps multiple versions of an object | Keeps multiple commits/versions of a file |
| Each version has a unique **Version ID** | Each commit has a unique **commit hash** |
| You can restore/access older versions | You can checkout older commits |
| Deleting adds a "delete marker" (doesn't erase history) | Reverting doesn't erase commit history |

> ⚠️ **Key difference:** Git tracks changes at the **content/diff level** with branching, merging, and commit messages. S3 Versioning simply stores **full copies** of each object version — there's no branching or merge concept. It's a simpler, storage-based "undo history" rather than a full version control system.

---

### Q3. How does Versioning work?

1. By default, versioning is **disabled** — uploading a file with the same name **overwrites** the existing object completely (old data is gone)
2. Once **enabled**, every upload (with the same key/filename) creates a **new version** with a unique **Version ID**, and the previous version is preserved (not deleted)
3. When you **download/view** the object normally, S3 serves the **latest (current) version**
4. Older versions remain accessible via the **"Show versions"** toggle in the console, or by specifying a Version ID via CLI/API
5. **Deleting** an object (with versioning enabled) doesn't actually delete it — S3 adds a **delete marker** as the "current version". The object can be restored by deleting the delete marker.

---

### Q4. Will it cost more storage?

**Yes.** Since S3 keeps a **full copy of every version** of an object, storage costs increase proportionally to the number of versions retained.

- Example: A 10 MB file uploaded 5 times (5 versions) = **50 MB** of total storage billed, not 10 MB
- 💡 **Cost management tip:** Use **Lifecycle rules** to automatically transition old versions to cheaper storage classes (e.g., Glacier) or permanently delete them after a set period

---

### Q5. Does it allow you to see "public data" / previous versions?

If the phrase means **"can you view/access previous versions of an object"** — **yes**, via:
- The S3 console's **"Show versions"** toggle (lists all versions with their Version IDs)
- AWS CLI: `aws s3api list-object-versions --bucket <bucket-name>`
- Each version can be individually downloaded, restored, or permanently deleted

> 🔒 Note: Versioning itself does **not** make objects public. Public access is controlled separately via Bucket Policies and Block Public Access settings (see the S3 practical).

---

## 🔧 Step-by-Step Practical

### Part 1 — Behavior WITHOUT Versioning

#### Step 1 — Create a Sample File

On your local machine, create a text file:
```
file.txt
```
Add some content, e.g.:
```
Version 1 - Original content
```

#### Step 2 — Create an S3 Bucket

1. Search **S3** → **Create bucket**

| Field | Value |
|---|---|
| Bucket name | A globally unique name (e.g., `versioning-demo-2026`) |
| Bucket Versioning | **Disable** (default) |

2. Click **Create bucket**

#### Step 3 — Upload `file.txt`

1. Open the bucket → click **Upload**
2. Click **Add files** → select `file.txt`
3. Click **Upload** → **Close**

#### Step 4 — View the File

1. Click on **`file.txt`** in the bucket
2. Click **Open** (opens the file content in a new tab)

> You should see: `Version 1 - Original content`

#### Step 5 — Update the File

1. On your local machine, **edit `file.txt`**:
   ```
   Version 2 - Updated content
   ```
2. Save the file

#### Step 6 — Re-upload the Updated File

1. Go back to the bucket → click **Upload**
2. Add the updated `file.txt` → **Upload** → **Close**

#### Step 7 — Check the Result

1. Click on `file.txt` → **Open**

> ❌ **Result:** You will see **only "Version 2 - Updated content"**. The original version has been **completely overwritten and lost** — without versioning, there's no way to recover "Version 1".

---

### Part 2 — Behavior WITH Versioning Enabled

#### Step 8 — Enable Versioning

1. Open your bucket → go to the **Properties** tab
2. Scroll to **Bucket Versioning**
3. Click **Edit**
4. Select **Enable**
5. Click **Save changes**

#### Step 9 — Upload the File Again (Create New Versions)

1. Edit `file.txt` locally:
   ```
   Version 3 - First version after enabling versioning
   ```
2. Go to the bucket → **Upload** → select `file.txt` → **Upload** → **Close**

3. Repeat the process — edit the file again:
   ```
   Version 4 - Another update
   ```
4. **Upload** again (same filename, overwrites the "current" pointer but keeps history now)

> 🔁 Repeat this upload process a few times with different content to generate multiple versions.

#### Step 10 — View the Current (Latest) Version

1. Click on `file.txt` → **Open**

> ✅ **Result:** Shows only the **most recent** content (e.g., "Version 4 - Another update") — by default, the latest version is what's served.

#### Step 11 — View All Versions

1. In the bucket, click the **"Show versions"** toggle (near the top of the object list)

> 🔍 **Result:** Now you'll see **multiple entries for `file.txt`**, each with:
> - A unique **Version ID**
> - **Last modified** timestamp
> - The **"latest"** version marked accordingly

2. Click on any **older version** → **Open** to view its content

> ✅ **Result:** Each version shows its **own distinct content** — older versions are preserved and individually accessible, unlike Part 1 where the old content was permanently lost.

---

## ✅ Optional — Restore an Older Version

To make an older version the "current" version:

1. With **"Show versions"** enabled, select the **older version** you want to restore
2. Download it, then **re-upload** it (this creates a new version with that content as "latest")

*(Alternatively, delete the newer versions — but this is destructive and not usually recommended.)*

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **Versioning** | Bucket-level setting to retain multiple versions of an object |
| **Version ID** | Unique identifier assigned to each version of an object |
| **Current/Latest version** | The version served by default when accessing the object |
| **Show versions** | Console toggle to view all stored versions of objects |
| **Delete marker** | Placeholder created when "deleting" an object with versioning enabled — doesn't remove actual data |
| **Storage cost** | Increases with each version stored — each version is a full copy |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Forgetting versioning is **disabled by default** — overwrites are permanent without it
- ❌ Assuming versioning **reduces** storage — it actually **increases** it (full copies, not diffs like Git)
- ❌ Not using **"Show versions"** — without it, only the latest version is visible, making it look like older versions don't exist
- ❌ Once enabled, versioning **cannot be fully disabled** — it can only be **suspended** (new uploads won't create versions, but existing versions remain)
- ❌ Forgetting that "deleting" a versioned object just adds a delete marker — the object isn't truly gone unless the delete marker AND all versions are removed

---

## 🧹 Cleanup (To Avoid Charges)

1. Enable **"Show versions"** in the bucket
2. Select **all versions and delete markers** → **Delete**
3. Confirm deletion (type `permanently delete`)
4. **Empty the bucket** (if anything remains) → **Delete the bucket**

---

*Practical by: AWS Cloud Lab*
