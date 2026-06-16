# 🗃️ AWS Directory Service & FSx — Practical Guide

> **Goal:** Set up an AWS Managed Microsoft AD directory, join an EC2 Windows instance to the domain, and create an Amazon FSx for Windows File Server share accessible to domain users.

---

## 📋 Prerequisites

- An active AWS account
- A VPC with at least 2 subnets in **different Availability Zones** (Directory Service requires this for redundancy)

---

## 🗂️ Concept: AWS Directory Service

**AWS Directory Service** provides multiple ways to use **Microsoft Active Directory (AD)** with other AWS services. Directories store information about **users, groups, and computers**, and administrators use this information to manage access to resources.

| Concept | Description |
|---|---|
| **Directory** | A database of users, groups, computers, and permissions |
| **AWS Managed Microsoft AD** | A fully managed AD running on AWS, compatible with standard AD features |
| **Domain join** | Connecting an EC2 instance to the directory so it can authenticate domain users |

---

## 🔧 Part 1 — Set Up AWS Managed Microsoft AD

### Step 1 — Create the Directory

1. In the search bar, type **Directory Service** and open it
2. Click **Set up directory**
3. Select **AWS Managed Microsoft AD**
4. Click **Next**

| Field | Value |
|---|---|
| Edition | **Standard** |
| Directory DNS name | `magicbus.com` |
| Admin password | Set a strong password (e.g., `Venom@123`) — **save this, you'll need it to log in** |

5. Click **Next**

### Step 2 — Networking

| Field | Value |
|---|---|
| VPC | Default VPC (or your custom VPC) |
| Subnets | Select **two subnets in different Availability Zones** |

> ⚠️ AWS Managed Microsoft AD requires **two subnets in two different AZs** for high availability — this is mandatory, not optional.

6. Review and click **Create directory**

> ⏳ Directory creation takes **20-45 minutes**. Wait until the status shows **Active** before proceeding.

---

## 🔧 Part 2 — Create an EC2 Instance and Join it to the Domain

### Step 3 — Create an IAM Role for Domain Join

1. Search **IAM → Roles → Create role**
2. Trusted entity type: **AWS service**
3. Use case: **EC2**
4. Attach the following policies:

| Policy | Purpose |
|---|---|
| `AmazonSSMManagedInstanceCore` | Allows Systems Manager (SSM) to manage the instance |
| `AmazonSSMDirectoryServiceAccess` | Allows the instance to join the AWS Managed Microsoft AD domain via SSM |

5. Name the role: e.g., `EC2-DomainJoin-Role`
6. Click **Create role**

---

### Step 4 — Launch EC2 Instance with Domain Join Configured

1. Go to **EC2 → Launch Instance**

| Field | Value |
|---|---|
| AMI | Windows Server (latest) |
| Instance type | `t2.micro` (or as needed) |
| Key pair | Create or select existing |
| Network | Same VPC as the directory; choose one of the directory's subnets |
| IAM instance profile | Select `EC2-DomainJoin-Role` (under **Advanced details**) |

2. Scroll to **Advanced details**
3. Find **Domain join directory** → select your directory (`magicbus.com`)

> ✅ With the IAM role and domain join directory both set, the instance will **automatically join the domain** on first boot via SSM.

4. Click **Launch instance**

---

### Step 5 — Connect to the Domain-Joined Instance

1. Once the instance is running, select it → **Connect → RDP client**
2. Download the remote desktop file, or use `mstsc` with the **public IP**
3. Click **"More choices" → "Use a different account"**
4. Log in with **domain credentials**:

| Field | Value |
|---|---|
| Username | `magicbus.com\admin` *(or `admin@magicbus.com`)* |
| Password | The admin password set in Step 1 (e.g., `Venom@123`) |

> ✅ **Result:** Successful login confirms the instance is **joined to the `magicbus.com` domain**, and you're authenticated as a domain admin (not a local user).

---

## 🔧 Part 3 — Create Amazon FSx for Windows File Server

### Step 6 — Create the File System

1. Search **FSx → Create file system**
2. Select **Amazon FSx for Windows File Server**
3. Choose **Standard create** (for full configuration control)
4. Click **Next**

#### File System Details

| Field | Value |
|---|---|
| File system name | e.g., `magicbus-fileshare` |
| Storage capacity | Minimum **20 GB** |
| Throughput capacity | Default (or as needed) |

#### Network & Security

| Field | Value |
|---|---|
| VPC | Same VPC as the directory/EC2 instance |
| Subnet | Choose a **different subnet** than the EC2 instance (for AZ distribution, if Multi-AZ) |
| VPC Security Groups | **Create new security group** |

##### Security Group Configuration

1. Click **Create security group** (opens in new context)

| Field | Value |
|---|---|
| Name | e.g., `fsx-sg` |
| Inbound rule | Type: **All traffic**, Source: **Anywhere (0.0.0.0/0)** *(for lab/testing — restrict in production)* |

2. Return to FSx creation and select this security group

#### Windows Authentication

| Field | Value |
|---|---|
| Authentication | Select your **AWS Managed Microsoft AD** directory (`magicbus.com`) |
| Deployment type | **Multi-AZ** |

#### Backup

| Field | Value |
|---|---|
| Automatic backups | **Disable** (for lab simplicity — enable in production) |

5. Review and click **Create file system**

> ⏳ Wait for the file system status to become **Available**.

---

### Step 7 — Mount the File Share on the EC2 Instance

1. Go to **FSx → your file system → Attach**
2. Copy the provided commands — there will be options for **PowerShell** and **Windows File Explorer mapping**

#### Option A — Map Network Drive via File Explorer

1. RDP into the domain-joined EC2 instance
2. Open **File Explorer** → right-click **This PC** → **Map network drive**
3. Enter the FSx file share path (e.g., `\\amznfsxXXXXXXXX\share`)
4. Choose a drive letter → **Finish**

#### Option B — Mount via PowerShell/CMD

1. Open **PowerShell** (or CMD) on the instance
2. Run the mount command provided by FSx (from the **Attach** dialog), e.g.:
   ```powershell
   net use Z: \\amznfsxXXXXXXXX\share /user:magicbus\admin
   ```
3. Verify access:
   ```powershell
   dir Z:\
   ```

> ✅ **Result:** The FSx file share is now mounted and accessible — domain users with appropriate permissions can read/write files to this shared storage.

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **AWS Managed Microsoft AD** | Fully managed Active Directory hosted by AWS |
| **Domain join** | Process of connecting a computer/instance to a directory so it can use domain credentials |
| **SSM (Systems Manager)** | Used to automate domain join without manual configuration on the instance |
| **Amazon FSx for Windows File Server** | Fully managed Windows-native file storage, integrates with AD for permissions |
| **Multi-AZ deployment** | FSx file system replicated across two AZs for high availability |
| **Mapped network drive** | Windows feature to access a network file share as a local drive letter |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Selecting subnets in the **same AZ** for the directory — AWS Managed Microsoft AD requires two different AZs
- ❌ Forgetting to attach the **IAM role with SSM + Directory Service policies** before launching the EC2 instance — domain join won't happen automatically
- ❌ Logging in with a **local account** instead of domain credentials (`magicbus.com\admin`) — won't prove domain join succeeded
- ❌ Using overly permissive security group rules (`All traffic`/`Anywhere`) in production — fine for lab, but restrict in real environments
- ❌ Not waiting for the directory/FSx file system to reach **Active/Available** status before proceeding

---

## 🧹 Cleanup (To Avoid Charges)

> ⚠️ AWS Managed Microsoft AD and FSx are relatively **expensive** services — clean up promptly after the lab.

1. **Delete the FSx file system**: FSx → select file system → Delete
2. **Terminate the EC2 instance**
3. **Delete the AWS Managed Microsoft AD directory**: Directory Service → select directory → Delete
4. **Delete the security group** created for FSx
5. **Delete the IAM role** (`EC2-DomainJoin-Role`)

---

*Practical by: AWS Cloud Lab*
