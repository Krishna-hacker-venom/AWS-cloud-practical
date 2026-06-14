# AWS EBS Volume — Detach & Attach Between Instances

> **Goal:** Launch two EC2 Windows instances, create and initialize an additional EBS volume on one instance, write data to it, then detach the volume and attach it to the second instance to verify the data persists.

---

## 📋 Prerequisites

- An active AWS account
- Basic familiarity with Windows Disk Management

---

## 🗂️ What is EBS?

**Amazon EBS (Elastic Block Store)** provides persistent block-level storage volumes for EC2 instances. A volume can be **detached from one instance and attached to another** (within the same Availability Zone), and the data on it persists.

> ⚠️ **Important:** An EBS volume can only be attached to instances **in the same Availability Zone** as the volume itself.

---

## 🔧 Step-by-Step Instructions

### Step 1 — Launch EC2 Instances

1. Go to **EC2 → Launch Instance**
2. Configure:

| Field | Value |
|---|---|
| Number of instances | `2` |
| AMI | Windows Server (latest) |
| Instance type | `t2.micro` (default) |
| Key pair | Create or select existing (same for both) |
| Network settings | Default |
| Storage | Default root volume |

> ✅ Make sure both instances launch in the **same Availability Zone** (e.g., `ap-south-1a`) — EBS volumes can only attach to instances in the same AZ.

3. Click **Launch instance**
4. Note the instance names as **`windows-1`** and **`windows-2`** for clarity

---

### Step 2 — Create an Additional EBS Volume

1. In the EC2 console, go to **Elastic Block Store → Volumes**
2. Click **Create volume**

| Field | Value |
|---|---|
| Volume type | General Purpose SSD (gp3) — default |
| Size | e.g., `1 GiB` (or as needed) |
| Availability Zone | **Same AZ as `windows-1`** |

3. Click **Create volume**

---

### Step 3 — Attach Volume to Machine 1

1. Select the newly created volume → **Actions → Attach volume**
2. Select instance: `windows-1`
3. Device name: leave default (e.g., `/dev/sdf` or `xvdf`)
4. Click **Attach volume**

---

### Step 4 — Initialize the Volume on Machine 1

1. **RDP into `windows-1`** (Windows + R → `mstsc`)
2. Open **Server Manager → Tools → Computer Management → Disk Management**
   *(or press Windows + R → type `diskmgmt.msc`)*
3. The new disk will appear as **Offline** and **Not Initialized**
4. **Right-click the disk** (left panel, disk number area) → **Online**
5. **Right-click the disk again** → **Initialize Disk**
6. Choose partition style: **MBR** or **GPT** → Click **OK**
7. Right-click the **unallocated space** → **New Simple Volume**
8. Follow the wizard:
   - Assign drive letter: **D**
   - File system: **NTFS**
   - Volume label: e.g., `MyVolume`
9. Click **Finish**

---

### Step 5 — Create a File in D: Drive

1. Open **File Explorer** → navigate to **D: drive**
2. Create a new file (e.g., right-click → **New → Text Document**)
3. Name it `test.txt`, add some content, and **save**

> 📝 This file will be used to verify data persists after moving the volume to `windows-2`.

---

### Step 6 — Detach the Volume

1. Go back to the **EC2 console** (separate browser tab)

> 💡 **Tip:** Keep one tab open on the **EC2 Instances** page and another on the **Volumes** page for easy switching.

2. Navigate to **Elastic Block Store → Volumes**
3. Select the volume attached to `windows-1`
4. Click **Actions → Detach volume**
5. Confirm by clicking **Detach**
6. Wait for the volume state to change to **Available**

> ⚠️ **Important:** Before detaching, it's best practice to first **take the disk offline** in Disk Management on `windows-1` (right-click disk → **Offline**) to avoid data corruption. AWS will also warn you if the volume is still in use.

---

### Step 7 — Attach the Volume to Machine 2

1. With the volume in **Available** state, select it
2. Click **Actions → Attach volume**
3. Select instance: `windows-2`
4. Device name: leave default
5. Click **Attach volume**

---

### Step 8 — Bring the Volume Online on Machine 2

1. **RDP into `windows-2`**
2. Open **Disk Management** (Windows + R → `diskmgmt.msc`)
3. The attached disk appears as **Offline**
4. **Right-click the disk** → **Online**

> ✅ No need to re-initialize or create a new volume — the disk already has its NTFS partition and data from `windows-1`.

5. The drive should appear with its assigned letter (or you may need to assign a drive letter: right-click the volume → **Change Drive Letter and Paths** → **Add** → assign `D`)

---

### Step 9 — Verify Data Persistence

1. Open **File Explorer** on `windows-2` → navigate to the **D: drive**
2. Confirm `test.txt` is present with the same content you saved earlier

> ✅ **Result:** The file persisted across instances — proving EBS volumes are independent of instance lifecycle and can be moved between instances (within the same AZ).

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **EBS Volume** | Persistent block storage, independent of EC2 instance lifecycle |
| **Availability Zone constraint** | Volumes can only attach to instances in the **same AZ** |
| **Online/Offline (Disk Management)** | A disk must be brought **Online** before it can be used; should be set **Offline** before detaching |
| **Initialize Disk** | Required for new/raw volumes before creating partitions |
| **Detach → Attach** | Allows moving data between instances without re-copying files |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Creating the volume in a **different AZ** than the instances — attachment will fail
- ❌ Detaching a volume while it's still mounted/online and in use — risk of data corruption; take it offline first
- ❌ Trying to **re-initialize** the disk on `windows-2` — this would erase existing data; just bring it **Online**
- ❌ Forgetting to assign a drive letter on `windows-2` after bringing the disk online

---

## 🧹 Cleanup (To Avoid Charges)

1. **Detach the EBS volume** from `windows-2`
2. **Delete the EBS volume** (Volumes → select → Actions → Delete volume)
3. **Terminate both EC2 instances** (`windows-1` and `windows-2`)

---

