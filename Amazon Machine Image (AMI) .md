# AWS: Amazon Machine Image (AMI)

> **Topic:** AMI — Types, Configuration, and Creating a Custom AMI  
> **Service Used:** Amazon EC2 (AMI)

---

## What is an AMI?

An **Amazon Machine Image (AMI)** is a pre-configured template used to launch EC2 instances. It contains:

1. **Configuration** — architecture (32/64-bit), virtualization type
2. **Pre-installed OS** — Windows, Amazon Linux, Ubuntu, etc.
3. **Application files** — any software/packages already installed on the instance

> Think of an AMI as a **snapshot/blueprint** of an EC2 instance that can be reused to launch identical instances instantly.

---

## AMI Types

| # | Type | Description | Trusted? |
|---|---|---|---|
| 1 | **AWS Managed AMI** | Provided and maintained by AWS (e.g., Amazon Linux 2, Windows Server) | Yes |
| 2 | **Custom AMI** | Created by you from your own configured EC2 instance | Yes (your own) |
| 3 | **AWS Marketplace AMI** | Third-party AMIs sold/listed on AWS Marketplace | Trusted (verified vendors) |
| 4 | **Community AMI** | Publicly shared AMIs from other AWS users | Not trusted by default |

> **Rule of thumb:** Use AWS Managed or Marketplace AMIs for production. Be cautious with Community AMIs — they are unverified.

---

## What an AMI Contains

```
AMI
 ├── OS (e.g., Amazon Linux, Windows Server)
 ├── Installed software (e.g., Apache, Node.js)
 ├── EBS Volume snapshot (root + any attached volumes)
 └── Launch permissions (who can use this AMI)
```

---

## How to Create a Custom (Customer-Managed) AMI

> **Use case:** You configured an EC2 instance (e.g., installed Apache, set up your app) and want to save that state to launch more identical instances later — used in Launch Templates and Auto Scaling Groups.

### Steps:

1. Go to **EC2 → Instances**
2. Select the instance you want to capture
3. Click **Actions → Image and Templates → Create Image**
4. Fill in the details:

| Field | Value |
|---|---|
| Image name | e.g., `my-web-server-ami` |
| Description | e.g., Apache + app pre-installed |
| Reboot instance | Enable (recommended for data consistency) |
| Delete on termination | Enable for root volume (default) |

5. Click **Create Image**

---

## Verify the AMI

1. Go to **EC2 → AMIs** (left sidebar)
2. Check the **Status** column:

| Status | Meaning |
|---|---|
| `pending` | AMI is being created |
| `available` | AMI is ready to use |

---

## AMI Status Checks (What AWS Verifies During Creation)

When an AMI is created, AWS runs the following checks:

| Check | What it verifies |
|---|---|
| **Infrastructure check** | Underlying hardware and hypervisor are healthy |
| **OS check** | Operating system is reachable and running correctly |
| **EBS Volume check** | Root and attached EBS volumes are snapshotted successfully |

---

## AMI Components Captured

| Component | Included in AMI? |
|---|---|
| OS (Windows / Linux) | Yes |
| Installed software & files | Yes |
| EBS root volume snapshot | Yes |
| Hardware configuration | Metadata only (instance type is set at launch) |
| Instance store (ephemeral) | No — not persisted |
| RAM / running processes | No — AMI is a disk snapshot, not a live capture |

---

## Key Concepts Summary

| Concept | Description |
|---|---|
| **AMI** | Template to launch EC2 instances with pre-configured OS + software |
| **Custom AMI** | AMI you create from your own running EC2 instance |
| **EBS Snapshot** | Point-in-time copy of an EBS volume — stored inside an AMI |
| **Launch Template** | Uses an AMI to define what new instances should look like |
| **Community AMI** | Publicly shared, unverified — use with caution |
| **Reboot on Create** | Ensures file system consistency when taking the AMI snapshot |

---

## Cleanup (To Avoid Charges)

1. Go to **EC2 → AMIs** → Select AMI → **Deregister AMI**
2. Go to **EC2 → Snapshots** → Delete the associated EBS snapshot
