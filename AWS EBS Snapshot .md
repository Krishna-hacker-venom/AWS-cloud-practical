# AWS EBS Snapshot 

> **Goal:** Create a snapshot (backup) of an EBS volume, verify it in the Snapshots menu, and create a new volume from that snapshot.

---

## 📋 Prerequisites

- An EC2 instance with an attached EBS volume (see EBS Volume practical)
- Some data already written to the volume (e.g., `test.txt` on D: drive)

---

## 🗂️ What is a Snapshot?

An **EBS Snapshot** is a point-in-time backup of an EBS volume, stored in S3 (managed by AWS, not directly accessible). Snapshots are **incremental** — only the changed blocks since the last snapshot are saved, saving storage costs.

---

## 🔧 Step-by-Step Instructions

### Step 1 — Create a Snapshot

1. Go to **EC2 Dashboard → Elastic Block Store → Volumes**
2. Select the volume you want to back up (checkbox)
3. Click **Actions → Create snapshot**

| Field | Value |
|---|---|
| Description | e.g., `Backup of MyVolume - 14 June 2026` |
| Tags | Optional (e.g., Name = `myvolume-snapshot`) |

4. Click **Create snapshot**

---

### Step 2 — Verify the Snapshot

1. In the left menu, go to **Elastic Block Store → Snapshots**
2. You should see your snapshot listed with:
   - **Status**: starts as `pending`, changes to `completed`
   - **Description**: the text you entered
   - **Volume size**: matches the source volume

> ⏳ Wait until the status shows **completed** before proceeding.

---

### Step 3 — Create a New Volume from the Snapshot

1. Go to **Snapshots** → select your snapshot (checkbox)
2. Click **Actions → Create volume from snapshot**

| Field | Value |
|---|---|
| Volume type | General Purpose SSD (gp3) — default |
| Size | Same as snapshot (or larger) |
| Availability Zone | Choose the AZ where you want to attach this volume |

3. Click **Create volume**

---

### Step 4 — Attach and Verify the New Volume

1. Go to **Volumes** → select the newly created volume
2. Click **Actions → Attach volume**
3. Select a target EC2 instance (must be in the **same AZ**) → click **Attach volume**
4. RDP into that instance → open **Disk Management** (`diskmgmt.msc`)
5. Bring the disk **Online** (right-click → Online)
6. Assign a drive letter if needed (right-click volume → **Change Drive Letter and Paths → Add**)
7. Open **File Explorer** → check the drive

> ✅ **Result:** The data from the original volume (e.g., `test.txt`) is present — confirming the snapshot successfully preserved the data.

---

## 📌 Key Concepts Summary

| Concept | Description |
|---|---|
| **Snapshot** | Point-in-time backup of an EBS volume, stored in S3 |
| **Incremental backup** | Only changed blocks are stored after the first snapshot, reducing cost |
| **Create volume from snapshot** | Generates a new, independent EBS volume with the snapshot's data |
| **AZ for new volume** | Can be created in **any AZ** within the same region (unlike the original volume's restriction) |
| **Status: pending → completed** | Snapshot must finish before it can reliably be used to create volumes |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Creating a volume from snapshot **before** the snapshot status shows `completed`
- ❌ Forgetting that the new volume is **independent** — changes to it do not affect the original volume or snapshot
- ❌ Trying to attach the new volume to an instance in a **different AZ** than the volume was created in
- ❌ Forgetting to bring the new disk **online** in Disk Management after attaching

---

## 🧹 Cleanup (To Avoid Charges)

1. **Detach** the new volume from the instance
2. **Delete** the new volume (Volumes → select → Actions → Delete volume)
3. **Delete the snapshot** (Snapshots → select → Actions → Delete snapshot)
4. **Terminate** any EC2 instances used for testing

---

