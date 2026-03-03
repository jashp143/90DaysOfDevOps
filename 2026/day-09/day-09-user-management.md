# Day 09 – Linux User & Group Management Challenge

## Users & Groups Created

### Users
| Username   | Home Directory      | Shell      | UID  |
|------------|---------------------|------------|------|
| tokyo      | /home/tokyo         | /bin/bash  | 1001 |
| berlin     | /home/berlin        | /bin/bash  | 1002 |
| professor  | /home/professor     | /bin/bash  | 1003 |
| nairobi    | /home/nairobi       | /bin/bash  | 1004 |

### Groups
| Group Name   | GID  |
|--------------|------|
| developers   | 1004 |
| admins       | 1005 |
| project-team | 1007 |

---

## Group Assignments

| User      | Groups                        |
|-----------|-------------------------------|
| tokyo     | tokyo, developers, project-team |
| berlin    | berlin, developers, admins    |
| professor | professor, admins             |
| nairobi   | nairobi, project-team         |

**Verification output (`groups` command):**
```
tokyo     : tokyo developers project-team
berlin    : berlin developers admins
professor : professor admins
nairobi   : nairobi project-team
```

---

## Directories Created

| Directory           | Owner | Group        | Permissions | Octal |
|---------------------|-------|--------------|-------------|-------|
| /opt/dev-project    | root  | developers   | rwxrwxr-x   | 775   |
| /opt/team-workspace | root  | project-team | rwxrwxr-x   | 775   |

**Verification output (`ls -ld`):**
```
drwxrwxr-x 2 root developers   4096 Mar  3 /opt/dev-project
drwxrwxr-x 2 root project-team 4096 Mar  3 /opt/team-workspace
```

---

## Task-by-Task Breakdown

### Task 1: Create Users

```bash
# Create users with home directories (-m) and bash shell (-s)
useradd -m -s /bin/bash tokyo
useradd -m -s /bin/bash berlin
useradd -m -s /bin/bash professor

# Set passwords using chpasswd (reads user:password from stdin)
echo "tokyo:Tokyo@123"         | chpasswd
echo "berlin:Berlin@123"       | chpasswd
echo "professor:Professor@123" | chpasswd
```

**Verification:**
```bash
grep -E "^tokyo:|^berlin:|^professor:" /etc/passwd
ls -la /home/
```

**Output:**
```
tokyo:x:1001:1001::/home/tokyo:/bin/bash
berlin:x:1002:1002::/home/berlin:/bin/bash
professor:x:1003:1003::/home/professor:/bin/bash
```

---

### Task 2: Create Groups

```bash
groupadd developers
groupadd admins
```

**Verification:**
```bash
grep -E "^developers:|^admins:" /etc/group
```

**Output:**
```
developers:x:1004:
admins:x:1005:
```

---

### Task 3: Assign Users to Groups

```bash
# -aG = append to supplementary groups (do NOT omit -a or primary group is replaced)
usermod -aG developers       tokyo
usermod -aG developers,admins berlin
usermod -aG admins           professor
```

**Verification:**
```bash
groups tokyo
groups berlin
groups professor
grep -E "^developers:|^admins:" /etc/group
```

**Output:**
```
developers:x:1004:tokyo,berlin
admins:x:1005:berlin,professor
```

---

### Task 4: Shared Directory – /opt/dev-project

```bash
mkdir -p /opt/dev-project
chgrp developers /opt/dev-project
chmod 775 /opt/dev-project

# Test: create files as each developer user
su -s /bin/bash tokyo   -c "echo 'Hello from tokyo'  > /opt/dev-project/tokyo-file.txt"
su -s /bin/bash berlin  -c "echo 'Hello from berlin' > /opt/dev-project/berlin-file.txt"
```

**Verification:**
```bash
ls -ld /opt/dev-project
ls -la /opt/dev-project/
```

**Output:**
```
drwxrwxr-x 2 root developers 4096 Mar 3 /opt/dev-project
-rw-rw-r-- 1 berlin berlin   18   Mar 3 berlin-file.txt
-rw-rw-r-- 1 tokyo  tokyo    17   Mar 3 tokyo-file.txt
```

✅ Both `tokyo` and `berlin` (members of `developers`) could create files successfully.

---

### Task 5: Team Workspace – /opt/team-workspace

```bash
# Create new user
useradd -m -s /bin/bash nairobi
echo "nairobi:Nairobi@123" | chpasswd

# Create group and assign members
groupadd project-team
usermod -aG project-team nairobi
usermod -aG project-team tokyo

# Set up shared workspace
mkdir -p /opt/team-workspace
chgrp project-team /opt/team-workspace
chmod 775 /opt/team-workspace

# Test: create file as nairobi
su -s /bin/bash nairobi -c "echo 'Hello from nairobi' > /opt/team-workspace/nairobi-file.txt"
```

**Verification:**
```bash
ls -ld /opt/team-workspace
ls -la /opt/team-workspace/
groups nairobi
groups tokyo
```

**Output:**
```
drwxrwxr-x 2 root project-team 4096 Mar 3 /opt/team-workspace
-rw-rw-r-- 1 nairobi nairobi   19   Mar 3 nairobi-file.txt
```

✅ `nairobi` successfully created a file in the shared workspace.

---

## Complete Commands Reference

```bash
# ── USER MANAGEMENT ───────────────────────────────────────────
useradd -m -s /bin/bash <username>      # Create user with home dir & bash shell
echo "<user>:<pass>" | chpasswd         # Set password non-interactively
passwd <username>                       # Set password interactively
usermod -aG <group1>,<group2> <user>    # Add user to supplementary groups
userdel -r <username>                   # Delete user and home directory

# ── GROUP MANAGEMENT ──────────────────────────────────────────
groupadd <groupname>                    # Create a new group
groupdel <groupname>                    # Delete a group
groups <username>                       # Show all groups a user belongs to
id <username>                           # Show UID, GID, and all groups

# ── DIRECTORY & PERMISSIONS ───────────────────────────────────
mkdir -p /opt/<dirname>                 # Create directory (with parents)
chgrp <group> /path/to/dir             # Change group owner
chmod 775 /path/to/dir                 # Set rwxrwxr-x permissions
ls -ld /path/to/dir                    # Verify directory permissions

# ── VERIFICATION ──────────────────────────────────────────────
grep "^<user>:" /etc/passwd            # Confirm user created
grep "^<group>:" /etc/group            # Confirm group and members
ls -la /home/                          # Confirm home directories
su -s /bin/bash <user> -c "<command>"  # Run command as another user
```

---

## Permission Bits Reference

```
775 = rwx rwx r-x
      │   │   └── others: read + execute
      │   └─────── group:  read + write + execute
      └─────────── owner:  read + write + execute

Symbolic: drwxrwxr-x
```

---

## What I Learned

1. **`-aG` flag is critical** — When using `usermod -aG`, the `-a` (append) flag is essential. Without it, the user is *removed* from all current supplementary groups and only added to the specified one. This is a common and dangerous mistake in production systems.

2. **Group permissions enable team collaboration** — Setting a shared directory's group owner (`chgrp`) and using permission `775` allows all group members to read/write while keeping others read-only. This is the standard pattern for shared project directories in real DevOps environments (e.g., `/var/www`, `/opt/app`).

3. **`/etc/passwd` and `/etc/group` are the source of truth** — These plain text files store user and group metadata. Knowing how to read them directly (`grep`, `cat`) is essential for auditing, debugging, and understanding system state — skills that directly apply to server hardening, CI/CD pipelines, and containerized environments.

---

## Real-World DevOps Connections

- **CI/CD agents** run as dedicated service users (e.g., `jenkins`, `github-actions`) added to specific groups for controlled access to secrets and build directories.
- **Web servers** like Nginx/Apache use `www-data` user/group for file permission isolation.
- **Docker** uses group membership (`docker` group) to allow non-root users to run containers.
- **Setgid bit** (`chmod g+s`) on shared dirs ensures new files inherit the group — useful addition on top of today's `775` setup.

---

*Day 09 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
