# 📢 AWS SNS (Simple Notification Service) — Topic & Subscription Practical

> **Goal:** Create an SNS topic, subscribe an email address to it, confirm the subscription, and publish a message to all subscribers.

---

## 📋 Prerequisites

- An active AWS account
- Access to an email inbox to confirm the subscription

---

## 🗂️ Concept: SNS Topics & Subscriptions

**Amazon SNS (Simple Notification Service)** is a **pub/sub (publish-subscribe)** messaging service.

| Term | Description |
|---|---|
| **Topic** | A communication channel — messages are published to a topic |
| **Subscription** | An endpoint (email, SMS, SQS, Lambda, etc.) that receives messages published to a topic |
| **Publisher** | Sends ("publishes") messages to the topic |
| **Subscriber** | Receives messages via their chosen protocol (email, SMS, etc.) |

> 💡 One message published to a topic is delivered to **all** its subscribers — this is the "fan-out" pattern.

---

## 🔧 Step-by-Step Instructions

### Step 1 — Create an SNS Topic

1. In the search bar, type **SNS** (Simple Notification Service) and open it
2. Click **Topics → Create topic**

| Field | Value |
|---|---|
| Type | **Standard** *(supports all protocols; "FIFO" is for strict ordering and limited protocols)* |
| Name | e.g., `my-topic` |
| Display name | *(optional)* — shown as the sender name for SMS notifications |

3. Leave other settings as default
4. Click **Create topic**

---

### Step 2 — Create a Subscription

1. Open your topic (`my-topic`)
2. Go to the **Subscriptions** tab → click **Create subscription**

| Field | Value |
|---|---|
| Topic ARN | Auto-filled (your topic's ARN) |
| Protocol | **Email** *(free of cost — no charges for email notifications)* |
| Endpoint | Your email address (e.g., `you@example.com`) |

3. Click **Create subscription**

---

### Step 3 — Confirm the Subscription

> ⚠️ The subscription status will initially show **"Pending confirmation"** — emails (and most other protocols except SQS/Lambda) require manual confirmation.

1. Check your email inbox for a message from **AWS Notifications**
2. Open the email and click **"Confirm subscription"**
3. Go back to the SNS console → refresh the **Subscriptions** tab

> ✅ The status should now show **"Confirmed"**.

---

### Step 4 — Publish a Message

1. Go back to the topic → click **Publish message**

| Field | Value |
|---|---|
| Subject | e.g., `Test Notification` |
| Message body | e.g., `Hello! This is a test message from SNS.` |
| Time to Live (TTL) | *(optional)* — maximum time (in seconds) SNS will attempt to deliver the message before giving up; mainly relevant for mobile push notifications |

2. Click **Publish message**

---

### Step 5 — Verify

1. Check your email inbox
2. You should receive an email with:
   - **Subject**: the subject you entered
   - **Body**: the message content
   - Footer text indicating it was sent via Amazon SNS, with an **unsubscribe link**

> ✅ **Result:** The message was published to the topic and delivered to all confirmed subscribers (in this case, your email).

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **Topic** | The channel messages are published to |
| **Standard vs FIFO topic** | Standard = high throughput, best-effort order; FIFO = strict order, exactly-once delivery |
| **Subscription** | An endpoint registered to receive messages from a topic |
| **Protocol** | The delivery mechanism (Email, SMS, SQS, Lambda, HTTP/S, etc.) |
| **Pending confirmation** | Subscription state before the endpoint owner confirms (required for email/SMS/HTTP) |
| **TTL (Time to Live)** | Max duration SNS attempts message delivery before discarding |
| **Publish message** | Sends the message to all confirmed subscribers of the topic |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Forgetting to **confirm the subscription** via the email link — unconfirmed subscriptions won't receive messages
- ❌ Checking the **spam/junk folder** if the confirmation email doesn't appear in the inbox
- ❌ Assuming the message is delivered **instantly without confirmation** — pending subscriptions are skipped
- ❌ Using a topic name with special characters not allowed by SNS (only alphanumeric, hyphens, underscores)
- ❌ Forgetting that **email protocol is free**, but other protocols (SMS, mobile push) may incur charges

---

## 🧹 Cleanup (Optional)

1. **Delete subscriptions**: SNS → Topic → Subscriptions tab → select → Delete
2. **Delete the topic**: SNS → Topics → select `my-topic` → Delete

---

*Practical by: AWS Cloud Lab*
