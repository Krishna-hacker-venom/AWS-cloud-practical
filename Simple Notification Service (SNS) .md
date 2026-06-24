# Simple Notification Service (SNS)

> **Topic:** SNS — Topics, Subscriptions, and Publishing Messages  
> **Service Used:** Amazon SNS (Simple Notification Service)  
> **Cost:** Free tier eligible (Email protocol is free)

---

## What is SNS?

Amazon SNS is a **push-based messaging service** that sends notifications to subscribers when a message is published to a topic.

```
Publisher
   |
   v
SNS Topic
   |
   +--> Email Subscriber
   +--> SMS Subscriber
   +--> Lambda / SQS / HTTP (other protocols)
```

---

## Part 1 — Create an SNS Topic

1. Go to **AWS Console → SNS → Topics → Create Topic**
2. Configure:

| Setting | Value |
|---|---|
| Type | Standard |
| Name | any (e.g., `my-sns-topic`) |
| Display name (for SMS) | shown only to **yourself** (internal label) |
| Display name (for others) | shown to **subscribers** in notification |

> **Display Name** appears as the sender name in email/SMS notifications received by subscribers.

3. Everything else — default
4. Click **Create Topic**

---

## Part 2 — Create a Subscription

> A subscription defines **who** receives messages and **how** they receive them.

1. Go to your created Topic → **Subscriptions → Create Subscription**
2. Configure:

| Setting | Value |
|---|---|
| Topic ARN | auto-filled (your topic) |
| Protocol | **Email** (free of cost) |
| Endpoint | your email address |

3. Click **Create Subscription**

### Confirm the Subscription

- AWS sends a **confirmation email** to the endpoint
- Open the email and click **"Confirm subscription"**
- Status changes from `PendingConfirmation` → `Confirmed`

> Without confirming, you will **not** receive published messages.

---

## Part 3 — Publish a Message

1. Go to **SNS → Topics → select your topic**
2. Click **Publish Message**
3. Configure:

| Setting | Value / Notes |
|---|---|
| Subject | Short title for the message (shown in email subject line) |
| TTL (Time to Leave) | How long (in seconds) SNS retries delivery before discarding — mainly for mobile push |
| Message body | The actual notification content |

4. Click **Publish Message**

### Result

- Subscribed email address receives the message within seconds
- Email subject = what you set as **Subject**
- Email body = what you typed in **Message body**

---

## Key Concepts Summary

| Concept | Description |
|---|---|
| **Topic** | A named channel that groups messages and subscribers together |
| **Subscription** | Registers an endpoint (email, SMS, etc.) to receive messages from a topic |
| **Protocol** | Delivery method — Email, SMS, HTTP/S, Lambda, SQS |
| **Display Name** | Sender name shown to subscribers in notifications |
| **Publish** | Action of sending a message to all confirmed subscribers of a topic |
| **TTL (Time to Leave)** | Max time SNS will retry delivery before dropping the message |
| **ARN** | Amazon Resource Name — unique identifier for the topic |

---

## Cleanup (To Avoid Charges)

1. Go to **SNS → Subscriptions** → Delete subscription
2. Go to **SNS → Topics** → Delete topic
