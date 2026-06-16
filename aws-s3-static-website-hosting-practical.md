# 🌐 AWS S3 Static Website Hosting — Practical Guide

> **Goal:** Host a static website (HTML/CSS) directly from an S3 bucket — no web servers required.

---

## 📋 Prerequisites

- An active AWS account
- An HTML template/file (e.g., `index.html` with CSS)

---

## 🗂️ Concept: Static Web Hosting

**Static website hosting** is a method of serving **pre-built web assets** (HTML, CSS, JavaScript, images) directly to users **without provisioning, managing, or maintaining any underlying web servers**.

> 📝 **Static vs Dynamic:** Static websites deliver content **exactly as stored** — there is **no server-side code execution** (no PHP, no database queries, no backend processing). Every visitor sees the same files. This makes S3 an ideal, low-cost, highly scalable hosting solution for static content like portfolios, documentation sites, and landing pages.

---

## 🔧 Step-by-Step Instructions

### Step 1 — Create a Bucket

1. Search **S3** → **Create bucket**

| Field | Value |
|---|---|
| Bucket name | A globally unique name (e.g., `mywebsite-2026`) |
| AWS Region | Your preferred region |
| Object Ownership | ACLs disabled (default) |
| Block Public Access settings | **Uncheck "Block all public access"** |

2. When unchecking Block Public Access, AWS shows a warning checkbox:
   - ✅ Check **"I acknowledge that the current settings might result in this bucket and the objects within becoming public"**

3. Click **Create bucket**

> ⚠️ Static website hosting **requires public read access** — that's why Block Public Access must be disabled for this bucket.

---

### Step 2 — Download an HTML Template

1. Download a free HTML/CSS template from a site like [W3Schools](https://www.w3schools.com/howto/) or any free template provider, or create your own simple `index.html`:

```html
<!DOCTYPE html>
<html>
<head>
  <title>My Static Website</title>
</head>
<body>
  <h1>Hello from S3 Static Hosting!</h1>
</body>
</html>
```

2. Ensure the main page file is named **`index.html`**

---

### Step 3 — Upload the HTML File(s) to the Bucket

1. Open your bucket → click **Upload**
2. Click **Add files** (and **Add folder** for CSS/images/assets if applicable)
3. Select `index.html` (and any other assets)
4. Click **Upload** → **Close**

---

### Step 4 — Add a Bucket Policy for Public Read Access

Since the bucket needs to serve content to **all visitors**, add a bucket policy granting public `GetObject` permission.

1. Go to the bucket → **Permissions** tab
2. Scroll to **Bucket policy** → click **Edit**
3. Paste the following policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::mywebsite-2026/*"
    }
  ]
}
```

> 🔍 **Policy breakdown:**
> - **Principal: `"*"`** → applies to everyone (public)
> - **Action: `"s3:GetObject"`** → allows reading/downloading objects
> - **Resource: `arn:aws:s3:::mywebsite-2026/*`** → applies to **all objects** in the bucket (note the `/*`)

> 💡 Replace `mywebsite-2026` with your actual bucket name (find the Bucket ARN under the **Properties** tab if needed).

4. Click **Save changes**

---

### Step 5 — Enable Static Website Hosting

1. Go to the bucket → **Properties** tab
2. Scroll all the way to the bottom → **Static website hosting**
3. Click **Edit**

| Field | Value |
|---|---|
| Static website hosting | **Enable** |
| Hosting type | Host a static website |
| Index document | `index.html` |
| Error document | `error.html` *(optional — create one if desired)* |

4. Click **Save changes**

---

### Step 6 — Access the Website

1. After saving, scroll back down to **Static website hosting** in the Properties tab
2. You'll now see a **Bucket website endpoint** URL, e.g.:
   ```
   http://mywebsite-2026.s3-website.ap-south-1.amazonaws.com
   ```
3. Click the URL (or copy-paste into a browser)

> ✅ **Result:** Your `index.html` page loads — the static website is live!

---

## ✅ Quick Verification Checklist

- [ ] Bucket created with Block Public Access **disabled** (acknowledged)
- [ ] `index.html` (and assets) uploaded
- [ ] Bucket policy added allowing public `s3:GetObject`
- [ ] Static website hosting **enabled** with `index.html` set as the index document
- [ ] Website endpoint URL loads the page successfully

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **Static website hosting** | Serving pre-built files directly from S3, no server needed |
| **Index document** | The default page served when visiting the root URL (e.g., `index.html`) |
| **Error document** | Custom page shown for errors (e.g., 404 Not Found) |
| **Bucket website endpoint** | The public URL AWS generates for the hosted site |
| **Block Public Access** | Must be disabled, since static hosting requires public reads |
| **Bucket Policy (`s3:GetObject`, `Principal: "*"`)** | Grants everyone permission to read/view files in the bucket |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Forgetting to **uncheck Block Public Access** and acknowledge the warning — static hosting won't work with it enabled
- ❌ Forgetting the **bucket policy** — even with hosting enabled, the site returns `403 Forbidden` without public read permission
- ❌ Forgetting the `/*` at the end of the **Resource ARN** in the bucket policy — without it, the policy applies to the bucket itself, not the files inside
- ❌ Naming the main file something other than what's set as the **Index document** (must match exactly, e.g., `index.html`)
- ❌ Using the **Object URL** instead of the **Bucket website endpoint** — they behave differently (the website endpoint supports redirects, index/error documents; the object URL does not)

---

## 🆚 Object URL vs Website Endpoint

| Feature | Object URL (`s3.amazonaws.com`) | Website Endpoint (`s3-website...`) |
|---|---|---|
| Serves index document at root | ❌ No | ✅ Yes |
| Supports error documents | ❌ No | ✅ Yes |
| Supports redirects | ❌ No | ✅ Yes |
| HTTPS support | ✅ Yes | ❌ No (HTTP only — use CloudFront for HTTPS) |

---

## 🧹 Cleanup (To Avoid Charges)

1. **Disable static website hosting**: Properties → Static website hosting → Disable
2. **Empty the bucket**: select bucket → Empty → confirm
3. **Delete the bucket**: select bucket → Delete → confirm

---

*Practical by: AWS Cloud Lab*
