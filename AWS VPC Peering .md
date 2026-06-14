# 🌐 AWS VPC Peering 

> **Goal:** Create two VPCs in different regions (Mumbai & Singapore), establish a VPC peering connection between them, and configure routing so resources in both VPCs can communicate using private IPs.

---

## 📋 Prerequisites

- An active AWS account
- Completed the basic VPC practical (recommended)
- Two different non-overlapping CIDR blocks (e.g., `10.0.0.0/16` and `20.0.0.0/16`)

---

## 🗂️ Architecture Overview

```
Region: Mumbai (ap-south-1)              Region: Singapore (ap-southeast-1)
┌────────────────────────┐               ┌────────────────────────┐
│ VPC: mumbai-vpc         │               │ VPC: singapore-vpc     │
│ CIDR: 10.0.0.0/16       │               │ CIDR: 20.0.0.0/16       │
│                         │               │                         │
│ Subnet: mumbai-subnet   │   Peering     │ Subnet: singapore-subnet│
│ + Internet Gateway      │◄─────────────►│ + Internet Gateway      │
│ + Route Table           │  Connection   │ + Route Table           │
│ + EC2 Instance          │               │ + EC2 Instance (Windows)│
└────────────────────────┘               └────────────────────────┘

Route Table (Mumbai)    → 20.0.0.0/16 → Peering Connection
Route Table (Singapore) → 10.0.0.0/16 → Peering Connection
```

---

## 🔧 Step-by-Step Instructions

### Step 0 — Setup Browser Tabs

1. Open **Tab 1** → Set region to **Asia Pacific (Mumbai) — `ap-south-1`**
2. Open **Tab 2** → Set region to **Asia Pacific (Singapore) — `ap-southeast-1`**

> 💡 Keep both tabs open throughout — you'll switch between them frequently.

---

## 🇮🇳 Part 1 — Mumbai Region Setup (Tab 1)

### Step 1 — Create VPC

1. Search **VPC** in the search bar → **Create VPC**
2. Select **VPC only**

| Field | Value |
|---|---|
| Name tag | `mumbai-vpc` |
| IPv4 CIDR block | Manual input |
| IPv4 CIDR | `10.0.0.0/16` |

3. Click **Create VPC**

---

### Step 2 — Create Subnet

1. Go to **Subnets → Create subnet**

| Field | Value |
|---|---|
| VPC ID | `mumbai-vpc` |
| Subnet name | `mumbai-subnet` |
| Availability Zone | `ap-south-1a` |
| IPv4 Subnet CIDR | `10.0.0.0/24` |

2. Click **Create subnet**

---

### Step 3 — Create & Attach Internet Gateway

1. Go to **Internet Gateways → Create internet gateway**

| Field | Value |
|---|---|
| Name tag | `mumbai-igw` |

2. Click **Create internet gateway**
3. Select it → **Actions → Attach to VPC**
4. Choose `mumbai-vpc` → **Attach internet gateway**

---

### Step 4 — Create Route Table

1. Go to **Route Tables → Create route table**

| Field | Value |
|---|---|
| Name | `mumbai-rt` |
| VPC | `mumbai-vpc` |

2. Click **Create route table**

#### Subnet Association

3. Select `mumbai-rt` → **Subnet Associations** tab → **Edit subnet associations**
4. Select `mumbai-subnet` → **Save associations**

#### Add Internet Route (for general internet access)

5. Go to **Routes** tab → **Edit routes** → **Add route**

| Destination | Target |
|---|---|
| `0.0.0.0/0` | `mumbai-igw` |

6. Click **Save changes**

---

## 🇸🇬 Part 2 — Singapore Region Setup (Tab 2)

Repeat the same steps in the **Singapore** tab with the following values:

### Step 5 — Create VPC

| Field | Value |
|---|---|
| Name tag | `singapore-vpc` |
| IPv4 CIDR | `20.0.0.0/16` |

### Step 6 — Create Subnet

| Field | Value |
|---|---|
| VPC ID | `singapore-vpc` |
| Subnet name | `singapore-subnet` |
| Availability Zone | `ap-southeast-1a` |
| IPv4 Subnet CIDR | `20.0.0.0/24` |

### Step 7 — Create & Attach Internet Gateway

| Field | Value |
|---|---|
| Name tag | `singapore-igw` |

Attach it to `singapore-vpc`.

### Step 8 — Create Route Table

| Field | Value |
|---|---|
| Name | `singapore-rt` |
| VPC | `singapore-vpc` |

- Associate with `singapore-subnet`
- Add route: `0.0.0.0/0` → `singapore-igw`

---

## 🖥️ Part 3 — Launch EC2 Instance (Singapore)

1. Go to **EC2 → Launch Instance**

| Field | Value |
|---|---|
| Name | `singapore-instance` |
| AMI | Windows Server (latest) |
| Instance type | `t2.micro` |
| Key pair | Create or select existing |
| Network settings | Edit |
| VPC | `singapore-vpc` |
| Subnet | `singapore-subnet` |
| Auto-assign Public IP | **Enable** |

2. Click **Launch Instance**

---

## 🔗 Part 4 — VPC Peering Connection

### Step 9 — Create Peering Connection (from Mumbai tab)

1. In **Tab 1 (Mumbai)**, go to **VPC Dashboard → Peering Connections**
2. Click **Create peering connection**

| Field | Value |
|---|---|
| Peering connection name | `mumbai-singapore-peering` |
| VPC (Requester) | `mumbai-vpc` |
| Region | **Another region** → select `Asia Pacific (Singapore)` |
| VPC (Accepter) ID | Paste the **VPC ID of `singapore-vpc`** *(copy from Tab 2 → Your VPCs)* |

3. Click **Create peering connection**

---

### Step 10 — Accept Peering Connection (from Singapore tab)

1. Switch to **Tab 2 (Singapore)**
2. Go to **VPC Dashboard → Peering Connections**
3. Select the pending peering request
4. Click **Actions → Accept request**
5. Confirm by clicking **Accept request** again

---

## 🛣️ Part 5 — Update Route Tables for Peering

### Step 11 — Singapore Route Table (Tab 2)

1. Go to **Route Tables** → select `singapore-rt`
2. Go to **Routes** tab → **Edit routes** → **Add route**

| Destination | Target |
|---|---|
| `10.0.0.0/16` | Peering Connection → `mumbai-singapore-peering` |

3. Click **Save changes**

---

### Step 12 — Mumbai Route Table (Tab 1)

1. Go to **Route Tables** → select `mumbai-rt`
2. Go to **Routes** tab → **Edit routes** → **Add route**

| Destination | Target |
|---|---|
| `20.0.0.0/16` | Peering Connection → `mumbai-singapore-peering` |

3. Click **Save changes**

---

## ✅ Step 13 — Verify Peering Connection

1. Launch an EC2 instance in `mumbai-subnet` as well (same steps as Part 3, using `mumbai-vpc`)
2. Note the **private IP addresses** of both instances
3. RDP into one instance and try to **ping** the private IP of the other
4. ✅ If ping succeeds (or RDP connects), the peering connection is working correctly

> ⚠️ Make sure **Security Groups** on both instances allow inbound traffic (ICMP for ping, RDP port 3389) from the peer VPC's CIDR range.

---

## 🧹 Cleanup (To Avoid Charges)

1. **Terminate EC2 instances** in both regions
2. **Delete the peering connection** (from either region)
3. **Remove peering routes** from both route tables
4. **Delete route tables, subnets, and internet gateways** (detach first) in both regions
5. **Delete both VPCs** (`mumbai-vpc` and `singapore-vpc`)

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **VPC Peering** | A networking connection between two VPCs enabling private IP communication |
| **Requester VPC** | The VPC that initiates the peering request |
| **Accepter VPC** | The VPC that must accept the peering request |
| **Cross-region peering** | Peering between VPCs in different AWS regions |
| **CIDR overlap** | Peered VPCs **must not** have overlapping CIDR ranges |
| **Route Table Update** | Each VPC's route table must point to the peer VPC's CIDR via the peering connection |
| **Security Groups** | Must allow traffic from the peer VPC's CIDR for actual communication to work |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Using the **same or overlapping CIDR** for both VPCs (e.g., both `10.0.0.0/16`) — peering will fail
- ❌ Forgetting to **accept** the peering connection from the accepter region
- ❌ Forgetting to **update route tables on both sides** — peering connection alone does not enable traffic
- ❌ Security group rules blocking traffic even after routing is correctly configured

---

*Practical by: AWS Cloud Lab | Regions: ap-south-1 (Mumbai) & ap-southeast-1 (Singapore)*
