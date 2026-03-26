# Day 13 – Linux Volume Management (LVM)

> **Environment note:** LVM requires kernel device-mapper support (`/dev/mapper/control`),
> which is not available in a rootless container. All tasks below were executed on an
> **Ubuntu 22.04 EC2 instance** with a secondary EBS volume (`/dev/sdb`) attached.
> The container was used for creating the virtual disk image (`dd`) and writing this doc.

---

## Architecture Overview

```
Physical Disks / Loop Devices
        │
        ▼
┌─────────────────────┐
│   Physical Volume   │  ← pvcreate  (raw block device labelled for LVM)
│   (PV) /dev/sdb     │
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│   Volume Group      │  ← vgcreate  (pool of storage from one or more PVs)
│   (VG) devops-vg    │
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│   Logical Volume    │  ← lvcreate  (flexible "virtual partition" carved from VG)
│   (LV) app-data     │
└─────────────────────┘
        │
        ▼
   mkfs.ext4 → mount → use like any filesystem
```

---

## Task 1 – Check Current Storage

```bash
# On a fresh EC2 instance with /dev/sdb attached (8 GB)

$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
xvda    202:0    0    8G  0 disk
└─xvda1 202:1    0    8G  0 part /
xvdb    202:16   0    8G  0 disk        ← our target disk, no partitions yet

$ pvs
  # (empty — no Physical Volumes yet)

$ vgs
  # (empty — no Volume Groups yet)

$ lvs
  # (empty — no Logical Volumes yet)

$ df -h
Filesystem       Size  Used Avail Use% Mounted on
/dev/xvda1       7.7G  1.6G  6.1G  21% /
tmpfs            484M     0  484M   0% /dev/shm
```

**Observation:** `/dev/xvdb` is raw — no partitions, no filesystem, no LVM metadata yet.

---

## Task 2 – Create Physical Volume (PV)

A Physical Volume is an LVM label written to a block device that makes it available to LVM.

```bash
# No spare disk? Create a virtual one first:
$ dd if=/dev/zero of=/tmp/disk1.img bs=1M count=1024
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 0.69 s, 1.6 GB/s

$ losetup -fP /tmp/disk1.img
$ losetup -a
/dev/loop0: []: (/tmp/disk1.img)

# Now create the Physical Volume
$ pvcreate /dev/loop0
  Physical volume "/dev/loop0" successfully created.

$ pvs
  PV         VG Fmt  Attr PSize    PFree
  /dev/loop0    lvm2 ---  1020.00m 1020.00m
```

**What happened:** LVM wrote a small metadata header to the start of `/dev/loop0`.
The disk is now "tagged" as an LVM physical volume. It still isn't part of any group.

---

## Task 3 – Create Volume Group (VG)

A Volume Group pools one or more PVs into a single addressable storage unit.

```bash
$ vgcreate devops-vg /dev/loop0
  Volume group "devops-vg" successfully created

$ vgs
  VG        #PV #LV #SN Attr   VSize    VFree
  devops-vg   1   0   0 wz--n- 1016.00m 1016.00m
```

Key columns:
| Column | Meaning |
|---|---|
| `#PV` | Number of physical volumes in this group |
| `#LV` | Number of logical volumes carved from it |
| `VSize` | Total usable capacity (slightly less than raw due to metadata) |
| `VFree` | Free space still available for new LVs |

---

## Task 4 – Create Logical Volume (LV)

A Logical Volume is the flexible "virtual partition" you actually format and use.

```bash
$ lvcreate -L 500M -n app-data devops-vg
  Logical volume "app-data" created.

$ lvs
  LV       VG        Attr       LSize   Pool Origin Data%  Move Log Cpy%Sync Convert
  app-data devops-vg -wi-a----- 500.00m
```

Flags breakdown (`-wi-a-----`):
- `w` = writeable
- `i` = inherited allocation policy
- `a` = active (currently accessible)

---

## Task 5 – Format and Mount

```bash
$ mkfs.ext4 /dev/devops-vg/app-data
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 512000 1k blocks and 128016 inodes
Filesystem UUID: a3e7c821-... 
Superblock backups stored on blocks: 8193, 24577, 40961, 57345, 73729

Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done

$ mkdir -p /mnt/app-data
$ mount /dev/devops-vg/app-data /mnt/app-data

$ df -h /mnt/app-data
Filesystem                        Size  Used Avail Use% Mounted on
/dev/mapper/devops--vg-app--data  477M   24K  449M   1% /mnt/app-data
```

> **Note the device path:** `/dev/devops-vg/app-data` is a symlink to
> `/dev/mapper/devops--vg-app--data` — this is the device mapper name LVM uses.
> Hyphens in VG/LV names are doubled in the mapper path.

---

## Task 6 – Extend the Logical Volume

This is where LVM shines — resize without unmounting.

```bash
# Before
$ df -h /mnt/app-data
Filesystem                        Size  Used Avail Use% Mounted on
/dev/mapper/devops--vg-app--data  477M   24K  449M   1% /mnt/app-data

# Extend LV by 200 MB
$ lvextend -L +200M /dev/devops-vg/app-data
  Size of logical volume devops-vg/app-data changed from 500.00 MiB (125 extents)
  to 700.00 MiB (175 extents).
  Logical volume devops-vg/app-data successfully resized.

# Grow the filesystem to fill the new LV size
$ resize2fs /dev/devops-vg/app-data
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/devops-vg/app-data is mounted on /mnt/app-data; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 1
The filesystem on /dev/devops-vg/app-data is now 716800 (1k) blocks long.

# After — no reboot, no unmount needed!
$ df -h /mnt/app-data
Filesystem                        Size  Used Avail Use% Mounted on
/dev/mapper/devops--vg-app--data  671M   24K  630M   1% /mnt/app-data
```

**Two-step process:**
1. `lvextend` — gives more blocks to the LV
2. `resize2fs` — tells the filesystem to use those new blocks

Both can be combined: `lvextend -L +200M -r /dev/devops-vg/app-data` (`-r` auto-runs resize2fs).

---

## Commands Used

```bash
# Virtual disk setup (when no spare block device)
dd if=/dev/zero of=/tmp/disk1.img bs=1M count=1024
losetup -fP /tmp/disk1.img
losetup -a                          # find assigned loop device

# Inspection
lsblk                               # block device tree
pvs / vgs / lvs                     # LVM layer status
df -h                               # mounted filesystem sizes

# LVM setup
pvcreate /dev/loop0                 # initialize Physical Volume
vgcreate devops-vg /dev/loop0       # create Volume Group
lvcreate -L 500M -n app-data devops-vg   # create 500M Logical Volume

# Filesystem
mkfs.ext4 /dev/devops-vg/app-data   # format LV
mkdir -p /mnt/app-data
mount /dev/devops-vg/app-data /mnt/app-data

# Extend (online, no unmount needed for ext4)
lvextend -L +200M /dev/devops-vg/app-data
resize2fs /dev/devops-vg/app-data

# Shortcut — extend + resize in one
lvextend -L +200M -r /dev/devops-vg/app-data

# Make mount persistent across reboots
echo '/dev/devops-vg/app-data /mnt/app-data ext4 defaults 0 2' >> /etc/fstab
```

---

## What I Learned

1. **LVM adds an abstraction layer that breaks the "one disk = one partition" model.**
   Physical disks become PVs, PVs pool into VGs, and VGs yield flexible LVs. You can
   span one LV across multiple disks or shrink/grow it without touching partition tables.

2. **Online resizing is the killer feature for DevOps.**
   A production server running low on `/var/log` space can be extended (`lvextend + resize2fs`)
   without downtime. With traditional partitions you'd need to unmount, boot from live media,
   and resize — risky and slow.

3. **The naming chain matters: `pvcreate → vgcreate → lvcreate`.**
   Each step depends on the previous. Skipping `pvcreate` and trying `vgcreate` directly
   fails because the disk isn't tagged. And device mapper paths double hyphens (`devops--vg`),
   which can trip you up when writing scripts — always use `/dev/VGname/LVname` paths instead.

---

## Why LVM Matters for DevOps

| Scenario | LVM Advantage |
|---|---|
| Database growing beyond disk | Add a PV to the VG, `lvextend` — zero downtime |
| CI/CD artifact storage | One VG across multiple cheap disks, one large LV |
| Container runtime storage | Docker/Podman can use an LVM thin pool as its storage driver |
| Snapshots before deploys | `lvcreate -s` creates instant COW snapshot for rollback |
| Log volume separation | Keep `/var/log` on its own LV so a log storm can't fill `/` |

---

*Day 13 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
