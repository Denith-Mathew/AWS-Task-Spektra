# EBS Volume Mounting Guide on Amazon Linux

---

## What is EBS?

**EBS (Elastic Block Store)** is a virtual hard disk provided by AWS that you attach to your EC2 instance — just like plugging in an external USB drive to your laptop.

```
AWS Cloud
│
└── EC2 Instance
      ├── nvme0n1 (Root Volume - 20GB)  ← OS lives here (auto-attached)
      └── nvme1n1 (EBS Volume - 50GB)   ← Extra disk (you attach manually)
```

---

## EBS vs Root Volume

| Feature | Root Volume | EBS Volume |
|---|---|---|
| **Purpose** | OS + system files | Extra storage |
| **Auto Mounted** | ✅ Yes | ❌ No (manual) |
| **Deleted on Stop** | Depends on setting | ❌ No (persists) |
| **Example** | nvme0n1 | nvme1n1 |

---

## Step-by-Step: Mount an EBS Volume

### Step 1 — Check Attached Disks

```bash
lsblk
```

**Output:**
```
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
nvme0n1       259:0    0  20G  0 disk
├─nvme0n1p1   259:1    0  20G  0 part /
├─nvme0n1p127 259:2    0   1M  0 part
└─nvme0n1p128 259:3    0  10M  0 part /boot/efi
nvme1n1       259:4    0  50G  0 disk        ← No mountpoint = not mounted
```

> `nvme1n1` has no MOUNTPOINTS — means it is attached but NOT usable yet.

---

### Step 2 — Switch to Root User

```bash
sudo su
```

---

### Step 3 — Format the Disk

```bash
mkfs -t ext4 /dev/nvme1n1
```

**What this does:**
```
mkfs   = Make Filesystem
-t     = Type
ext4   = Linux filesystem format (most common)

Think of it like: formatting a new USB drive before use
```

**Output you will see:**
```
Creating filesystem with 13107200 4k blocks and 3276800 inodes
Filesystem UUID: 5366acdb-05f0-4660-a0e1-e0ac173b7a2b
Writing superblocks and filesystem accounting information: done
```

> ⚠️ **Only format ONCE** — formatting again will erase all data!

---

### Step 4 — Create a Mount Point (Folder)

```bash
mkdir /data
```

> A mount point is just an **empty folder** where the disk will be attached.

---

### Step 5 — Mount the Volume

```bash
mount /dev/nvme1n1 /data
```

```
mount           = attach the disk
/dev/nvme1n1    = the EBS device
/data           = the folder to attach it to
```

---

### Step 6 — Verify Mounting

```bash
df -h
```

**Output:**
```
Filesystem        Size  Used Avail Use% Mounted on
/dev/nvme0n1p1     20G  1.7G   19G   9% /
/dev/nvme1n1       49G   24K   47G   1% /data    ← ✅ Mounted!
```

---

### Step 7 — Test by Writing a File

```bash
# Correct way to write to file
echo "EBS volume is working" > /data/test.txt

# Read it back
cat /data/test.txt
```

**Output:**
```
EBS volume is working
```

---

## Auto Mount on Reboot (Permanent Mount)

> Without this, the mount is lost after every reboot!

```bash
# Step 1: Get UUID of the volume
blkid /dev/nvme1n1

# Output:
# /dev/nvme1n1: UUID="5366acdb-05f0-4660-a0e1-e0ac173b7a2b" TYPE="ext4"

# Step 2: Edit fstab file
nano /etc/fstab

# Step 3: Add this line at the bottom
UUID=5366acdb-05f0-4660-a0e1-e0ac173b7a2b  /data  ext4  defaults,nofail  0  2

# Step 4: Test fstab is correct
mount -a
```

---

## Common Commands Summary

| Command | Purpose |
|---|---|
| `lsblk` | List all disks and partitions |
| `df -h` | Show disk space usage |
| `mkfs -t ext4 /dev/nvme1n1` | Format disk |
| `mkdir /data` | Create mount point |
| `mount /dev/nvme1n1 /data` | Mount disk |
| `umount /data` | Unmount disk |
| `blkid /dev/nvme1n1` | Get disk UUID |
| `cat /etc/fstab` | View auto-mount config |

---

## echo `>` vs `>>` in Linux

| Command | Meaning |
|---|---|
| `echo "text" > file.txt` | ✅ Write (overwrites existing content) |
| `echo "text" >> file.txt` | ✅ Append (adds to existing content) |
| `echo "text" file.txt` | ❌ Wrong — just prints to screen, does NOT write to file |

---

## Your Actual Terminal Session (What Happened)

```bash
sudo su                          # ✅ Switched to root
lsblk                            # ✅ Listed disks - saw nvme1n1 (50GB, no mount)
mkfs -t ext4 /dev/nvme1n1        # ✅ Formatted the disk
mkdir /data                      # ✅ Created mount folder
mount /dev/nvme1n1 /data         # ✅ Mounted the disk
df -h                            # ✅ Verified - /data shows 49G available
echo "EBS volume" > /data/test.txt # ✅ Correct - writes "EBS volume" into test.txt
echo "how r u" > /data/test1.txt # ✅ Correct - wrote to file
cat /data/test1.txt              # ✅ Read file - output: "how r u"
```

---

## Visual Summary

```
BEFORE MOUNTING                    AFTER MOUNTING
─────────────────                  ────────────────
nvme1n1 (50GB)                     nvme1n1 (50GB)
┌─────────────┐                    ┌─────────────┐
│             │    mount →         │ /data        │
│  UNUSABLE   │                    │ test.txt     │
│             │                    │ test1.txt    │
└─────────────┘                    └─────────────┘
No mountpoint ❌                   Mounted at /data ✅
```

---

> 🔑 **Key Takeaway:** EBS volumes must be **formatted** and **mounted** before use.
> They persist even after EC2 stop/start — your data stays safe!
