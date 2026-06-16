# 🖥️ Task: VPC with Public Web Server & Private Database Server

> **Goal:** Create a VPC with public and private subnets, launch an Amazon Linux EC2 instance in each (web server in public, database server in private), and verify each by writing a custom file to demonstrate they're distinct instances.

---

## 📋 Prerequisites

- An active AWS account
- A single key pair for both instances (e.g., `ec2-key`)

---

## 🗂️ Architecture Overview

```
VPC: 10.0.0.0/16
├── Public Subnet:  10.0.0.0/24
│   └── EC2 (Amazon Linux) — "webserver"
│       └── file: hello.txt → "Hello text"
└── Private Subnet: 10.0.1.0/24
    └── EC2 (Amazon Linux) — "dbserver"
        └── file: goodbye.txt → "Good Bye text"

Key pair (ec2-key) used for BOTH instances
```

> 📝 **Note:** The original task notes specified `10.0.10.0/24` for the private subnet. Since the VPC CIDR is `10.0.0.0/16` and the public subnet already uses `10.0.0.0/24`, the private subnet should use a **non-overlapping** range such as **`10.0.1.0/24`** (used below).

---

## 🔧 Step-by-Step Instructions

### Step 1 — Create the VPC

1. Search **VPC → Create VPC**
2. Select **VPC only**

| Field | Value |
|---|---|
| Name tag | `task-vpc` |
| IPv4 CIDR | `10.0.0.0/16` |

3. Click **Create VPC**

---

### Step 2 — Create Two Subnets

Go to **Subnets → Create subnet**

#### Public Subnet

| Field | Value |
|---|---|
| VPC ID | `task-vpc` |
| Subnet name | `B2G-Public` |
| Availability Zone | e.g., `ap-south-1a` |
| IPv4 CIDR | `10.0.0.0/24` |

#### Private Subnet

Click **Add new subnet**:

| Field | Value |
|---|---|
| Subnet name | `B2G-Private` |
| Availability Zone | e.g., `ap-south-1a` (or a different AZ for HA) |
| IPv4 CIDR | `10.0.1.0/24` |

3. Click **Create subnet**

---

### Step 3 — Create & Attach Internet Gateway (for Public Subnet)

1. Go to **Internet Gateways → Create internet gateway**

| Field | Value |
|---|---|
| Name | `task-igw` |

2. Click **Create internet gateway** → **Actions → Attach to VPC** → select `task-vpc`

---

### Step 4 — Create Route Table for Public Subnet

1. Go to **Route Tables → Create route table**

| Field | Value |
|---|---|
| Name | `public-rt` |
| VPC | `task-vpc` |

2. Click **Create route table**
3. **Subnet Associations** tab → **Edit subnet associations** → select `B2G-Public` → **Save**
4. **Routes** tab → **Edit routes** → **Add route**:

| Destination | Target |
|---|---|
| `0.0.0.0/0` | `task-igw` |

5. **Save changes**

> 💡 The private subnet (`B2G-Private`) uses the **default route table** (local only) — no internet route needed for this task, since the database server doesn't require outbound internet access.

---

### Step 5 — Launch EC2 Instance in Public Subnet (Web Server)

1. Go to **EC2 → Launch Instance**

| Field | Value |
|---|---|
| Name | `webserver` |
| AMI | **Amazon Linux** (latest) |
| Instance type | `t2.micro` |
| Key pair | `ec2-key` (create if it doesn't exist) |
| Network settings | Edit |
| VPC | `task-vpc` |
| Subnet | `B2G-Public` |
| Auto-assign Public IP | **Enable** |
| Security group | Allow SSH (port 22) from your IP |

2. Click **Launch instance**

---

### Step 6 — Launch EC2 Instance in Private Subnet (Database Server)

1. Go to **EC2 → Launch Instance**

| Field | Value |
|---|---|
| Name | `dbserver` |
| AMI | **Amazon Linux** (latest) |
| Instance type | `t2.micro` |
| Key pair | `ec2-key` *(same key pair as webserver)* |
| Network settings | Edit |
| VPC | `task-vpc` |
| Subnet | `B2G-Private` |
| Auto-assign Public IP | **Disable** |
| Security group | Allow SSH (port 22) **only from the public subnet's CIDR** (`10.0.0.0/24`) |

2. Click **Launch instance**

> 💡 Since `dbserver` has no public IP, you'll connect to it **through `webserver`** (bastion-style access) using the same key pair.

---

### Step 7 — Create the File on the Web Server

1. SSH into `webserver` using its **public IP**:
   ```bash
   ssh -i ec2-key.pem ec2-user@<webserver-public-ip>
   ```
2. Create the file:
   ```bash
   echo "Hello text" > hello.txt
   cat hello.txt
   ```

---

### Step 8 — Create the File on the Database Server

1. From your local machine, copy the key pair to `webserver` (or use SSH agent forwarding) so you can hop to `dbserver`:
   ```bash
   scp -i ec2-key.pem ec2-key.pem ec2-user@<webserver-public-ip>:~/
   ```
2. From within `webserver`, SSH into `dbserver` using its **private IP**:
   ```bash
   chmod 400 ec2-key.pem
   ssh -i ec2-key.pem ec2-user@<dbserver-private-ip>
   ```
3. Create the file:
   ```bash
   echo "Good Bye text" > goodbye.txt
   cat goodbye.txt
   ```

---

## ✅ Step 9 — Verify

1. On `webserver`: `cat hello.txt` → outputs `Hello text`
2. On `dbserver`: `cat goodbye.txt` → outputs `Good Bye text`
3. Confirm `dbserver` has **no public IP** (check in EC2 console) and **cannot be reached directly from the internet** — only via `webserver`

> ✅ **Result:** Two distinct EC2 instances in separate subnets (public/private), accessible with the same key pair, each holding their own file — demonstrating subnet isolation and bastion-host access patterns.

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **Public subnet** | Has a route to an Internet Gateway — instances can have public IPs |
| **Private subnet** | No direct route to the internet — instances are isolated from inbound internet traffic |
| **Bastion/jump host pattern** | Using a public instance as an intermediary to reach private instances |
| **Shared key pair** | The same `.pem` file can be used to SSH into multiple instances |
| **Non-overlapping CIDRs** | Subnets within a VPC must have distinct, non-overlapping IP ranges |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Using overlapping CIDR blocks for public and private subnets (e.g., both `10.0.0.0/24`) — causes subnet creation errors
- ❌ Forgetting that the **private instance has no internet route** — `yum update` or package installs will fail unless a NAT Gateway is added
- ❌ Trying to SSH into the private instance **directly from your laptop** — it has no public IP; must go through the public instance
- ❌ Forgetting to **copy/secure the key pair** (`chmod 400`) on the bastion host before using it to hop to the private instance
- ❌ Security group on `dbserver` not allowing SSH from the **public subnet's CIDR** — connection will be refused

---

## 🧹 Cleanup (To Avoid Charges)

1. **Terminate both EC2 instances** (`webserver`, `dbserver`)
2. **Delete route tables, internet gateway** (detach first), and **subnets**
3. **Delete the VPC** (`task-vpc`)
4. **Delete the key pair** if no longer needed

---

*Practical by: AWS Cloud Lab*
