# 🌐 AWS VPC (Virtual Private Cloud) — Practical Guide

> **Goal:** Set up a VPC with public and private subnets, launch two EC2 instances (one public, one private), and configure NAT Gateway so the private instance can access the internet.

---

## 📋 Prerequisites

- An active AWS account
- Basic understanding of networking (IP ranges, subnets)
- RDP client available on your local machine (built-in on Windows via `mstsc`)

---

## 🗂️ Architecture Overview

```
VPC: 10.0.0.0/16
├── Public Subnet:  10.0.0.0/24  (AZ: ap-south-1a)
│   ├── Internet Gateway (igw)
│   ├── Route Table 1 → 0.0.0.0/0 → IGW
│   ├── NAT Gateway (nat)
│   └── EC2 Windows (Public) — Auto-assign Public IP: Enabled
└── Private Subnet: 10.0.1.0/24  (AZ: ap-south-1a)
    ├── Route Table 2 → 0.0.0.0/0 → NAT Gateway
    └── EC2 Windows (Private) — Auto-assign Public IP: Disabled
```

---

## 🔧 Step-by-Step Instructions

### Step 1 — Select Region

1. Log in to the **AWS Management Console**
2. In the top-right corner, select your region: **Asia Pacific (Mumbai) — `ap-south-1`**

---

### Step 2 — Create a VPC

1. In the search bar, type **VPC** and open the VPC service
2. Click **Create VPC**
3. Select **VPC only**
4. Fill in the details:

| Field | Value |
|---|---|
| Name tag | `myvpc` |
| IPv4 CIDR block | Manual input |
| IPv4 CIDR | `10.0.0.0/16` |
| IPv6 CIDR block | No IPv6 CIDR block |
| Tenancy | Default |

5. Click **Create VPC**

---

### Step 3 — Create Subnets

Navigate to **VPC Dashboard → Subnets → Create Subnet**

#### 3a — Public Subnet

| Field | Value |
|---|---|
| VPC ID | `myvpc` |
| Subnet name | `public` |
| Availability Zone | `ap-south-1a` |
| IPv4 VPC CIDR | `10.0.0.0/16` |
| IPv4 Subnet CIDR | `10.0.0.0/24` |

#### 3b — Private Subnet

Click **Add new subnet** and fill in:

| Field | Value |
|---|---|
| Subnet name | `private` |
| Availability Zone | `ap-south-1a` |
| IPv4 Subnet CIDR | `10.0.1.0/24` |

> ⚠️ **Important:** The private subnet CIDR must be different from the public one. Use `10.0.1.0/24` (not `10.0.0.0/24`).

6. Click **Create Subnet**

---

### Step 4 — Create & Attach an Internet Gateway

1. In the VPC Dashboard, go to **Internet Gateways**
2. Click **Create internet gateway**
3. Name it: `igw`
4. Click **Create internet gateway**
5. After creation, click **Actions → Attach to VPC**
6. Select `myvpc` and click **Attach internet gateway**

---

### Step 5 — Create Route Table for Public Subnet

1. Go to **VPC Dashboard → Route Tables**
2. Click **Create route table**

| Field | Value |
|---|---|
| Name | `route-table-1` |
| VPC | `myvpc` |

3. Click **Create route table**

#### Associate with Public Subnet

4. Select `route-table-1` → **Subnet Associations** tab
5. Click **Edit subnet associations**
6. Select **public** subnet only
7. Click **Save associations**

#### Add Internet Route

8. Go to the **Routes** tab → Click **Edit routes**
9. Click **Add route**

| Field | Value |
|---|---|
| Destination | `0.0.0.0/0` |
| Target | Internet Gateway → `igw` |

10. Click **Save changes**

---

### Step 6 — Launch EC2 Instances

Navigate to **EC2 → Instances → Launch Instances**

#### 6a — Public Instance (Windows)

| Field | Value |
|---|---|
| Name | `windows-public` |
| AMI | Windows Server (latest) |
| Instance type | `t2.micro` (or as needed) |
| Key pair | Create or select existing key pair |
| VPC | `myvpc` |
| Subnet | `public` |
| Auto-assign Public IP | **Enable** |

Click **Launch Instance**

#### 6b — Private Instance (Windows)

| Field | Value |
|---|---|
| Name | `windows-private` |
| AMI | Windows Server (same as above) |
| Instance type | `t2.micro` |
| Key pair | **Same key pair** as the public instance |
| VPC | `myvpc` |
| Subnet | `private` |
| Auto-assign Public IP | **Disable** |

Click **Launch Instance**

---

### Step 7 — Connect to EC2 Instances via RDP

#### Connect to the Public Instance

1. Press **Windows + R**, type `mstsc`, and press Enter
2. In the **Computer** field, enter the **public IP address** of `windows-public`
3. Click **Connect**
4. Username: `Administrator`
5. To get the password:
   - Go to EC2 Console → Select the instance
   - Click **Actions → Security → Get Windows Password**
   - Upload your `.pem` key pair file
   - Click **Decrypt Password**
6. Paste the decrypted password and click **OK**

#### Connect to the Private Instance (via Public Instance)

1. Once inside the **public instance**, open **Run (Win + R)** → type `mstsc`
2. Enter the **private IP address** of `windows-private`
3. Use the same `Administrator` credentials
4. Open a browser and navigate to `google.com` to test connectivity

> ❌ At this point, the private instance **will not** have internet access. NAT Gateway is needed (next step).

---

### Step 8 — Create a NAT Gateway

1. In the AWS Console, search for **NAT Gateways**
2. Click **Create NAT Gateway**

| Field | Value |
|---|---|
| Name | `nat` |
| Subnet | `public` *(NAT must be in the public subnet)* |
| Connectivity type | Public |
| Elastic IP | Click **Allocate Elastic IP** |

3. Click **Create NAT Gateway**

> ⏳ Wait a few minutes for the NAT Gateway status to become **Available**.

---

### Step 9 — Create Route Table for Private Subnet

1. Go to **VPC Dashboard → Route Tables**
2. Click **Create route table**

| Field | Value |
|---|---|
| Name | `route-table-2` |
| VPC | `myvpc` |

3. Click **Create route table**

#### Associate with Private Subnet

4. Select `route-table-2` → **Subnet Associations** tab
5. Click **Edit subnet associations**
6. Select the **private** subnet
7. Click **Save associations**

#### Add NAT Gateway Route

8. Go to the **Routes** tab → Click **Edit routes**
9. Click **Add route**

| Field | Value |
|---|---|
| Destination | `0.0.0.0/0` |
| Target | NAT Gateway → `nat` |

10. Click **Save changes**

---

### Step 10 — Test Internet Access from Private Instance

1. RDP into the **public** instance (Step 7)
2. From the public instance, RDP into the **private** instance using its **private IP**
3. Open a browser and go to `https://www.google.com`
4. ✅ If the page loads, NAT Gateway is working correctly!

---

## 🧹 Cleanup (To Avoid Charges)

> AWS charges for NAT Gateways and Elastic IPs even when idle. Clean up after the practical.

1. **Delete NAT Gateway** → NAT Gateways → Select `nat` → Delete
2. **Release Elastic IP** → Elastic IPs → Release address
3. **Terminate EC2 Instances** → EC2 → Instances → Terminate
4. **Delete Route Tables** (custom ones only)
5. **Detach & Delete Internet Gateway**
6. **Delete Subnets**
7. **Delete VPC**

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **VPC** | Isolated virtual network in AWS |
| **Subnet** | A segment of a VPC's IP range |
| **Internet Gateway (IGW)** | Allows public subnet instances to access the internet |
| **NAT Gateway** | Allows private subnet instances to access the internet (outbound only) |
| **Route Table** | Defines traffic routing rules within a VPC |
| **Elastic IP** | Static public IP address required by NAT Gateway |
| **CIDR `0.0.0.0/0`** | Represents all internet traffic (default route) |

---

*Practical by: AWS Cloud Lab | Region: ap-south-1 (Mumbai)*
