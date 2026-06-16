# 📩 AWS S3 Event Notification (with SQS) — Practical Guide

> **Goal:** Configure an S3 bucket to send **event notifications** to an **SQS queue** whenever an object is created, and verify by sending/receiving messages from the queue.

---

## 📋 Prerequisites

- An active AWS account
- An existing S3 bucket (see S3 practical)

---

## 🗂️ Concept: S3 Event Notifications

**S3 Event Notifications** allow your bucket to automatically send a notification when certain **events** happen on objects in the bucket (e.g., object created, deleted, restored).

These notifications can be sent to one of three destinations:

| Destination | Description |
|---|---|
| **Amazon SNS** (Simple Notification Service) | Pub/sub messaging — fan-out to multiple subscribers (email, SMS, etc.) |
| **Amazon SQS** (Simple Queue Service) | Message queue — events are queued for processing by consumers |
| **AWS Lambda** | Directly triggers a function to run in response to the event |

In this practical, we'll use **SQS** as the destination.

---

## 🔧 Step-by-Step Instructions

### Step 1 — Create an SQS Queue

1. In the search bar, type **SQS** (Simple Queue Service) and open it
2. Click **Create queue**

| Field | Value |
|---|---|
| Type | **Standard** *(default — supports high throughput; "FIFO" is the alternative for strict ordering)* |
| Name | e.g., `s3-event-queue` |
| Other settings | Leave as default |

3. Click **Create queue**
4. Note the **Queue ARN** — you'll need it shortly (visible on the queue's **Details** tab)

---

### Step 2 — Configure S3 Event Notification

1. Go to **S3** → open your bucket
2. Go to the **Properties** tab
3. Scroll down to **Event notifications**
4. Click **Create event notification**

| Field | Value |
|---|---|
| Event name | e.g., `s3-to-sqs-notification` |
| Prefix | (optional — leave blank for all objects) |
| Suffix | (optional — leave blank for all objects) |
| Event types | Check **All object create events** (`s3:ObjectCreated:*`) |
| Destination | **SQS Queue** |
| Specify SQS queue | Choose `s3-event-queue` from the dropdown |

5. Click **Save changes**

---

### Step 3 — Fix the Permissions Error

> ❌ When saving, you'll likely get an error like:
> *"Unable to validate the following destination configurations... Permissions on the destination queue do not allow S3 to publish notifications"*

This happens because, by default, **only the queue owner (root)** can send messages to the SQS queue. S3 needs explicit permission to publish to it.

#### Fix: Add an SQS Access Policy

1. Go to **SQS** → select `s3-event-queue`
2. Go to the **Access policy** tab → click **Edit**
3. Add/modify the policy to allow S3 to send messages. Example policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Allow-S3-SendMessage",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:<region>:<account-id>:s3-event-queue",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::<your-bucket-name>"
        }
      }
    }
  ]
}
```

> Replace `<region>`, `<account-id>`, and `<your-bucket-name>` with your actual values (find these on the queue's Details tab and bucket's Properties tab).

4. Click **Save**

---

### Step 4 — Re-save the S3 Event Notification

1. Go back to **S3** → your bucket → **Properties → Event notifications**
2. Open the event notification you created (or create it again if it didn't save)
3. Click **Save changes**

> ✅ It should now save successfully without the permissions error.

---

### Step 5 — Test — Upload a File to S3

1. Go to your bucket → **Upload**
2. Add any file (e.g., `test.txt`) → **Upload** → **Close**

---

### Step 6 — Verify — Poll for Messages in SQS

1. Go to **SQS** → select `s3-event-queue`
2. Click **Send and receive messages**
3. Under the **Receive messages** section, click **Poll for messages**

> ✅ **Result:** A message should appear containing **JSON event data** about the upload — including the bucket name, object key, event type (`ObjectCreated:Put`), timestamp, and more.

4. Click on the message to **view its full body/details**

---

## ✅ Quick Verification Checklist

- [ ] SQS queue created (Standard type)
- [ ] S3 event notification configured for "All object create events"
- [ ] SQS access policy updated to allow `s3.amazonaws.com` to `SendMessage`
- [ ] Event notification saved successfully (no permission error)
- [ ] Uploading a file to S3 generates a message in the SQS queue
- [ ] Message polled and viewed successfully

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **S3 Event Notification** | Automatically triggers an action when an event occurs on bucket objects |
| **Event types** | e.g., `s3:ObjectCreated:*`, `s3:ObjectRemoved:*`, `s3:ObjectRestore:*` |
| **SQS (Simple Queue Service)** | A managed message queue for decoupled, asynchronous processing |
| **Standard Queue** | Default queue type — high throughput, at-least-once delivery, best-effort ordering |
| **FIFO Queue** | Strict ordering and exactly-once processing, lower throughput |
| **Access Policy** | Resource-based policy on the SQS queue defining who can send/receive messages |
| **Poll for messages** | Manually checking the queue for new messages (console feature) |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Forgetting to update the **SQS access policy** — S3 cannot publish to the queue without explicit permission (this is the "root-only" error mentioned)
- ❌ Using the wrong **Queue ARN** or **Bucket ARN** in the access policy — causes a mismatch and the policy won't take effect
- ❌ Forgetting to **re-save** the S3 event notification after fixing the SQS policy
- ❌ Expecting **real-time push** notifications in the console — SQS requires **polling** to view messages
- ❌ Not selecting the correct **event type** (e.g., selecting "Object removed" when testing uploads)

---

## 🧹 Cleanup (To Avoid Charges)

1. **Delete the S3 event notification**: S3 → bucket → Properties → Event notifications → select → Delete
2. **Delete the SQS queue**: SQS → select `s3-event-queue` → Delete
3. **Empty and delete the S3 bucket** (if no longer needed)

---

*Practical by: AWS Cloud Lab*
