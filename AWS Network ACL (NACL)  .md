# 🛡️ AWS Network ACL (NACL) 

> **Goal:** Create a custom Network ACL, associate it with a subnet, and configure inbound and outbound rules to control traffic at the subnet level.

---

## 📋 Prerequisites

- A VPC with at least one subnet already created (see VPC practical)
- Basic understanding of inbound/outbound traffic rules

---

## 🗂️ What is a NACL?

A **Network ACL (NACL)** is a stateless firewall that operates at the **subnet level**. Unlike Security Groups (stateful, instance-level), NACLs:
- Evaluate rules in order of **rule number** (lowest number first)
- Require **both inbound AND outbound rules** explicitly (stateless — return traffic isn't automatically allowed)
- Can have **Allow** or **Deny** rules

---

## 🔧 Step-by-Step Instructions

### Step 1 — Navigate to Network ACLs

1. Go to **VPC Dashboard**
2. In the left menu, under **Security**, click **Network ACLs**

---

### Step 2 — Create a Network ACL

1. Click **Create network ACL**

| Field | Value |
|---|---|
| Name | `my-nacl` |
| VPC | Select your VPC (e.g., `myvpc`) |

2. Click **Create network ACL**

---

### Step 3 — Associate Subnet

1. Select the newly created `my-nacl`
2. Go to the **Subnet associations** tab
3. Click **Edit subnet associations**
4. Select the subnet you want to associate (e.g., `public`)
5. Click **Save changes**

> ⚠️ A subnet can only be associated with **one NACL at a time**. Associating it here will remove it from the default NACL.

---

### Step 4 — Configure Inbound Rules

1. Select `my-nacl` → go to the **Inbound rules** tab
2. Click **Edit inbound rules**
3. Click **Add new rule**

| Rule # | Type | Protocol | Port range | Source | Allow/Deny |
|---|---|---|---|---|---|
| `10` | HTTP | TCP (auto) | `80` (auto) | `0.0.0.0/0` | **Allow** |

4. Click **Save changes**

---

### Step 5 — Configure Outbound Rules

1. Go to the **Outbound rules** tab
2. Click **Edit outbound rules**
3. Click **Add new rule**

| Rule # | Type | Protocol | Port range | Destination | Allow/Deny |
|---|---|---|---|---|---|
| `11` | All traffic | All | All | `0.0.0.0/0` | **Allow** |

4. Click **Save changes**

---

## ✅ Step 6 — Verify

1. Launch/use an EC2 instance in the associated subnet
2. Test access:
   - With **inbound rule 10 (HTTP, Allow)**, web traffic on port 80 should reach the instance
   - With **outbound rule 11 (All traffic, Allow)**, the instance can send responses/traffic out

> 🔎 If access fails, also check: Security Group rules, default NACL deny rule (`*`, rule number `*`, always present at the bottom, denies everything not explicitly allowed), and instance-level firewall (Windows Firewall).

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **NACL** | Stateless, subnet-level firewall |
| **Rule evaluation order** | Rules are processed in ascending order by rule number; first match wins |
| **Rule `*` (asterisk)** | Default catch-all rule — denies all traffic not matched by other rules |
| **Stateless** | Inbound and outbound rules must both be configured — return traffic is NOT automatically allowed |
| **Allow vs Deny** | NACLs can explicitly allow OR deny traffic (unlike Security Groups, which only allow) |
| **One NACL per subnet** | Each subnet is associated with exactly one NACL at a time |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Forgetting to configure **outbound rules** — since NACLs are stateless, inbound HTTP traffic needs a matching outbound rule (or "All traffic" outbound) for responses to be sent back
- ❌ Using rule numbers that conflict — lower numbers are evaluated first; leave gaps (e.g., 10, 20, 30) for future rules
- ❌ Forgetting that the **default NACL** allows all traffic — switching to a custom NACL with limited rules can suddenly block traffic
- ❌ Associating the wrong subnet — double-check subnet associations before testing

---

## 🆚 NACL vs Security Group (Quick Reference)

| Feature | Network ACL | Security Group |
|---|---|---|
| Level | Subnet | Instance (ENI) |
| State | Stateless | Stateful |
| Rule types | Allow & Deny | Allow only |
| Evaluation | In order by rule number | All rules evaluated |
| Default | Allows all traffic | Denies all inbound, allows all outbound |

---

## 🧹 Cleanup (Optional)

1. **Edit subnet associations** → revert the subnet back to the default NACL (or delete association)
2. **Delete the custom NACL** (`my-nacl`)

---

