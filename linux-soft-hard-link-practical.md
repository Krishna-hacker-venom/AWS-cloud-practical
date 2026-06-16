# 🔗 Linux Soft Link & Hard Link — Practical Guide

> **Goal:** Understand the difference between soft (symbolic) links and hard links in Linux, and practice creating both using the `ln` command.

---

## 📋 Prerequisites

- A Linux system (Ubuntu/Amazon Linux/CentOS, etc.) with terminal access
- Basic familiarity with Linux file system navigation (`cd`, `ls`, `pwd`)
- Sudo/root privileges (for working in `/var/www/html`)

---

## 🗂️ Concepts

### Soft Link (Symbolic Link)

A **soft link** (symlink) is like a **shortcut** — it's a separate file that simply **points to the path** of another file.

| Property | Description |
|---|---|
| Points to | The **file path/name** (not the data itself) |
| File size | Very small (just stores the path string) |
| Can link across | Different file systems / partitions |
| Can link to | Files **or directories** |
| If original is deleted | The symlink becomes **broken** (dangling link) — points to nothing |
| Inode | Has its **own, different** inode number |

### Hard Link

A **hard link** is like a **backup/mirror** — it's another name pointing **directly to the same data on disk (same inode)**.

| Property | Description |
|---|---|
| Points to | The same **inode** (actual data blocks on disk) |
| File size | Same as original (it's essentially the same file) |
| Can link across | **Only within the same file system/partition** |
| Can link to | **Files only** (not directories, in most cases) |
| If original is deleted | The hard link **still works** — data remains accessible (since both names point to the same inode, and data is only removed when the link count reaches 0) |
| Inode | Shares the **same** inode number as the original |

---

## 📦 Dependencies / Packages Required

- The `ln` command is part of **GNU coreutils**, which is **pre-installed on all standard Linux distributions** — no additional package installation is required
- If using Apache to test via browser (as `/var/www/html` suggests), install Apache:
  ```bash
  sudo apt update
  sudo apt install apache2 -y      # Debian/Ubuntu
  # OR
  sudo yum install httpd -y        # Amazon Linux/CentOS
  sudo systemctl start httpd
  sudo systemctl enable httpd
  ```

---

## 🔧 Step-by-Step Practical

### Step 1 — Create the Original File

1. Navigate to the Apache web root directory:
   ```bash
   cd /var/www/html
   ```
2. Create `index.html`:
   ```bash
   sudo nano index.html
   ```
3. Add some content, e.g.:
   ```html
   <h1>Hello from index.html</h1>
   ```
4. Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` in nano)

---

### Step 2 — Create a Soft Link

Use the `-s` flag with `ln` to create a **symbolic link**:

```bash
sudo ln -s /var/www/html/index.html /var/www/html/index_soft.html
```

> **Syntax:** `ln -s <target_file> <link_name>`

#### Verify the Soft Link

```bash
ls -l
```

Output will look like:
```
lrwxrwxrwx 1 root root   23 Jun 15 10:00 index_soft.html -> /var/www/html/index.html
-rw-r--r-- 1 root root  150 Jun 15 09:55 index.html
```

> 🔍 Notice:
> - The `l` at the start of the permissions (`lrwxrwxrwx`) indicates it's a **link**
> - The arrow `->` shows what file it **points to**
> - File size of the symlink (`23`) is just the length of the target path string — much smaller than the actual file

---

### Step 3 — How Soft Link Works (Test)

1. Open the soft link file — it shows the **same content** as `index.html`:
   ```bash
   cat index_soft.html
   ```
   Output: `<h1>Hello from index.html</h1>`

2. **Edit the original file** (`index.html`) and add more content. Re-run `cat index_soft.html` — the soft link **reflects the change immediately** (since it just points to the same path).

---

### Step 4 — What Happens If You Delete the Soft Link Target?

1. Delete the **original file**:
   ```bash
   sudo rm index.html
   ```
2. Try to access the soft link:
   ```bash
   cat index_soft.html
   ```

> ❌ **Result:** `cat: index_soft.html: No such file or directory`
>
> The soft link becomes a **broken/dangling link** — it still exists as a file (`ls -l` will show it in red, often with `?` for permissions), but points to nothing since the target path no longer exists.

3. Recreate `index.html` to continue the practical:
   ```bash
   echo "<h1>Hello from index.html (recreated)</h1>" | sudo tee index.html
   ```

> ✅ Once `index.html` exists again at the same path, `index_soft.html` works again automatically — since the symlink only stores the **path**, not the data.

---

### Step 5 — Create a Hard Link

Use `ln` **without** the `-s` flag to create a **hard link**:

```bash
sudo ln /var/www/html/index.html /var/www/html/index_hard.html
```

> **Syntax:** `ln <target_file> <link_name>`

#### Verify the Hard Link

```bash
ls -li
```

Output will look like:
```
123456 -rw-r--r-- 2 root root  45 Jun 15 10:05 index.html
123456 -rw-r--r-- 2 root root  45 Jun 15 10:05 index_hard.html
```

> 🔍 Notice:
> - The **first column (inode number)** is **identical** for both files (`123456`)
> - The **link count** (3rd column) shows `2` — meaning 2 filenames point to this same inode
> - File sizes are **identical** — they're the same data

---

### Step 6 — How Hard Link Works (Test)

1. View content of the hard link:
   ```bash
   cat index_hard.html
   ```
   Output matches `index.html`

2. **Edit `index.html`** and add new content. Run `cat index_hard.html` again — it shows the **updated content too** (both filenames point to the same data blocks).

---

### Step 7 — What Happens If You Delete the Original File (Hard Link)?

1. Delete the **original file**:
   ```bash
   sudo rm index.html
   ```
2. Check the hard link:
   ```bash
   cat index_hard.html
   ```

> ✅ **Result:** The content is **still accessible**! Unlike a soft link, deleting the original file does **not** break the hard link — because the data isn't actually removed from disk until **all** hard links (filenames) pointing to that inode are deleted (link count reaches 0).

3. Verify link count decreased:
   ```bash
   ls -li index_hard.html
   ```
   Link count is now `1`.

---

## ✅ Quick Reference — Commands

| Command | Description |
|---|---|
| `ln -s <target> <link_name>` | Create a **soft (symbolic) link** |
| `ln <target> <link_name>` | Create a **hard link** |
| `ls -l` | View file details (look for `l` prefix and `->` for symlinks) |
| `ls -li` | View **inode numbers** to compare hard links |
| `readlink <symlink>` | Show the target path of a symlink |
| `rm <link_name>` | Remove a link (does not affect target's data unless it's the last hard link) |

---

## 📌 Key Concepts Summary

| Aspect | Soft Link | Hard Link |
|---|---|---|
| Analogy | Shortcut | Backup/mirror (same file, different name) |
| Points to | File path | Inode (actual data) |
| Inode number | Different | Same as original |
| Cross-filesystem | ✅ Yes | ❌ No |
| Link to directories | ✅ Yes | ❌ No (generally) |
| If original deleted | ❌ Breaks (dangling) | ✅ Still works |
| File size | Small (path length) | Same as original |

---

## ⚠️ Common Mistakes to Avoid

- ❌ Forgetting the `-s` flag — running `ln` without `-s` creates a **hard link**, not a soft link
- ❌ Using **relative paths** for symlinks when the working directory might change — prefer **absolute paths** (e.g., `/var/www/html/index.html`) to avoid broken links
- ❌ Expecting a hard link to work **across different partitions/file systems** — it will fail with `Invalid cross-device link`
- ❌ Confusing "deleting the link" with "deleting the data" — for hard links, data is only removed when **all** links to the inode are deleted
- ❌ Trying to hard-link a **directory** — not permitted in standard usage

---

## 🧹 Cleanup

```bash
sudo rm -f /var/www/html/index_soft.html
sudo rm -f /var/www/html/index_hard.html
sudo rm -f /var/www/html/index.html
```

---

*Practical by: Linux Lab*
