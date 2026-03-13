# 📁 Day 10 – File Permissions & File Operations

> **Program:** 90 Days of DevOps | #DevOpsKaJosh | TrainWithShubham
> **Date:** March 13, 2026

---

## ✅ Task 1: Create Files

```bash
touch devops.txt                                          # Create empty file
echo "These are my DevOps learning notes for Day 10 - File Permissions!" > notes.txt
echo 'echo "Hello DevOps"' > script.sh                   # Create shell script
```

### 🔍 Verify with `ls -l`

```
-rw-r--r-- 1 root root  0 Mar 13 09:16 devops.txt
-rw-r--r-- 1 root root 66 Mar 13 09:16 notes.txt
-rw-r--r-- 1 root root 20 Mar 13 09:16 script.sh
```

---

## ✅ Task 2: Read Files

```bash
cat notes.txt
# Output: These are my DevOps learning notes for Day 10 - File Permissions!

vim -R script.sh             # View in read-only mode

head -5 /etc/passwd
# root:x:0:0:root:/root:/bin/bash
# daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
# bin:x:2:2:bin:/bin:/usr/sbin/nologin
# sys:x:3:3:sys:/dev:/usr/sbin/nologin
# sync:x:4:65534:sync:/bin:/bin/sync

tail -5 /etc/passwd
# nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
# ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
# systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
# messagebus:x:100:101::/nonexistent:/usr/sbin/nologin
# polkitd:x:997:997:User for polkitd:/:/usr/sbin/nologin
```

---

## ✅ Task 3: Understand Permissions

Format: `[type][owner rwx][group rwx][others rwx]`

| Symbol | Octal | Meaning     |
|--------|-------|-------------|
| `r`    | 4     | Read        |
| `w`    | 2     | Write       |
| `x`    | 1     | Execute     |
| `-`    | 0     | No permission |

### 📋 Initial Permissions

| File         | Permissions  | Owner  | Group  | Others |
|--------------|-------------|--------|--------|--------|
| `devops.txt` | `-rw-r--r--` | rw-    | r--    | r--    |
| `notes.txt`  | `-rw-r--r--` | rw-    | r--    | r--    |
| `script.sh`  | `-rw-r--r--` | rw-    | r--    | r--    |

**Answer:** By default, `touch` and `echo >` create files with `644`. The owner can read/write; group and others can only read. Nobody can execute yet.

---

## ✅ Task 4: Modify Permissions

### 1️⃣ Make `script.sh` executable → run it

```bash
chmod +x script.sh
./script.sh
# Output: Hello DevOps
```
**After:** `-rwxr-xr-x` (755)

### 2️⃣ Set `devops.txt` to read-only

```bash
chmod a-w devops.txt
```
**After:** `-r--r--r--` (444) — no one can write

### 3️⃣ Set `notes.txt` to 640

```bash
chmod 640 notes.txt
```
**After:** `-rw-r-----` — owner: rw, group: r, others: none

### 4️⃣ Create `project/` directory with 755

```bash
mkdir project/
chmod 755 project/
```
**After:** `drwxr-xr-x` — owner: rwx, group: rx, others: rx

---

## ✅ Task 5: Test Permissions

### 🚫 Writing to a read-only file

```bash
echo "test" >> devops.txt
# bash: devops.txt: Permission denied
```

### 🚫 Executing without execute permission

```bash
./notes.txt
# bash: ./notes.txt: Permission denied
```

> Linux enforces permissions strictly. Even the file owner cannot bypass them without explicitly changing them first using `chmod`.

---

## 📊 Before & After Summary

| File         | Before       | Command          | After        |
|--------------|-------------|-----------------|-------------|
| `script.sh`  | `-rw-r--r--` | `chmod +x`       | `-rwxr-xr-x` |
| `devops.txt` | `-rw-r--r--` | `chmod a-w`      | `-r--r--r--` |
| `notes.txt`  | `-rw-r--r--` | `chmod 640`      | `-rw-r-----` |
| `project/`   | `drwxr-xr-x` | `chmod 755`      | `drwxr-xr-x` |

---

## 🛠️ All Commands Used

```bash
# File Creation
touch devops.txt
echo "content" > notes.txt
echo 'echo "Hello DevOps"' > script.sh
vim script.sh

# File Reading
cat notes.txt
vim -R script.sh
head -5 /etc/passwd
tail -5 /etc/passwd

# Viewing Permissions
ls -l
ls -ld project/

# Modifying Permissions
chmod +x script.sh
chmod a-w devops.txt
chmod 640 notes.txt
mkdir project/
chmod 755 project/

# Running & Testing
./script.sh
echo "test" >> devops.txt   # tests read-only
./notes.txt                 # tests no-execute
```

---

## 💡 What I Learned

1. **Permissions are 3-part:** Owner, Group, Others — each having Read (4), Write (2), Execute (1). The numeric mode is just the sum of these values per group.

2. **`chmod` has two syntaxes:** Symbolic (`chmod +x`, `chmod a-w`) for quick changes and octal (`chmod 755`, `chmod 640`) for setting exact permissions in one shot — both are powerful in different scenarios.

3. **Linux permission errors are immediate:** The OS enforces access at the kernel level. Getting `Permission denied` is not a bug — it's Linux working exactly as designed. In real DevOps, wrong permissions on scripts or config files are a common root cause of deployment failures.

---

#90DaysOfDevOps #DevOpsKaJosh #TrainWithShubham
