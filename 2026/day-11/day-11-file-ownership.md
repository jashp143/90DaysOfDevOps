# Day 11 – File Ownership Challenge (chown & chgrp)

## Files & Directories Created

| File/Directory | Purpose |
|---|---|
| `devops-file.txt` | chown owner-change demo |
| `team-notes.txt` | chgrp group-change demo |
| `project-config.yaml` | combined owner:group demo |
| `app-logs/` | directory ownership demo |
| `heist-project/vault/gold.txt` | recursive ownership demo |
| `heist-project/plans/strategy.conf` | recursive ownership demo |
| `bank-heist/access-codes.txt` | practice challenge |
| `bank-heist/blueprints.pdf` | practice challenge |
| `bank-heist/escape-plan.txt` | practice challenge |

---

## Ownership Changes

### Task 1 – Understanding Ownership

Running `ls -l` shows file metadata in this format:

```
-rw-r--r-- 1 root root 0 Mar 26 08:40 devops-file.txt
            ^ ^^^^  ^^^^
            │  │     └── group owner
            │  └── user owner
            └── link count
```

**Owner vs Group:**
- **Owner** – the individual user account that created or was assigned the file. Has the most control (read/write/execute based on the first permission triplet).
- **Group** – a collection of users. All members of this group share the group-permission triplet on the file.

---

### Task 2 – chown (owner only)

```
devops-file.txt: root:root → tokyo:root → berlin:root
```

```bash
$ ls -l devops-file.txt
-rw-r--r-- 1 root root 0 Mar 26 08:40 devops-file.txt

$ sudo chown tokyo devops-file.txt
-rw-r--r-- 1 tokyo root 0 Mar 26 08:40 devops-file.txt

$ sudo chown berlin devops-file.txt
-rw-r--r-- 1 berlin root 0 Mar 26 08:40 devops-file.txt
```

---

### Task 3 – chgrp (group only)

```
team-notes.txt: root:root → root:heist-team
```

```bash
$ sudo groupadd heist-team
$ sudo chgrp heist-team team-notes.txt
-rw-r--r-- 1 root heist-team 0 Mar 26 08:40 team-notes.txt
```

---

### Task 4 – Combined owner:group

```
project-config.yaml: root:root → professor:heist-team
app-logs/:           root:root → berlin:heist-team
```

```bash
$ sudo chown professor:heist-team project-config.yaml
-rw-r--r-- 1 professor heist-team 0 Mar 26 08:40 project-config.yaml

$ mkdir app-logs
$ sudo chown berlin:heist-team app-logs/
drwxr-xr-x 2 berlin heist-team 4096 Mar 26 08:41 app-logs
```

---

### Task 5 – Recursive Ownership

```
heist-project/ (all contents): root:root → professor:planners
```

```bash
$ sudo groupadd planners
$ sudo chown -R professor:planners heist-project/

$ ls -lR heist-project/
heist-project/:
drwxr-xr-x 2 professor planners 4096 Mar 26 08:41 plans
drwxr-xr-x 2 professor planners 4096 Mar 26 08:41 vault

heist-project/plans:
-rw-r--r-- 1 professor planners 0 Mar 26 08:41 strategy.conf

heist-project/vault:
-rw-r--r-- 1 professor planners 0 Mar 26 08:41 gold.txt
```

The `-R` flag propagated the ownership change to every file and subdirectory inside `heist-project/` in a single command.

---

### Task 6 – Practice Challenge

```
bank-heist/access-codes.txt: root:root → tokyo:vault-team
bank-heist/blueprints.pdf:   root:root → berlin:tech-team
bank-heist/escape-plan.txt:  root:root → nairobi:vault-team
```

```bash
$ sudo useradd -M -s /bin/bash tokyo
$ sudo useradd -M -s /bin/bash berlin
$ sudo useradd -M -s /bin/bash nairobi
$ sudo groupadd vault-team
$ sudo groupadd tech-team

$ sudo chown tokyo:vault-team   bank-heist/access-codes.txt
$ sudo chown berlin:tech-team   bank-heist/blueprints.pdf
$ sudo chown nairobi:vault-team bank-heist/escape-plan.txt

$ ls -l bank-heist/
-rw-r--r-- 1 tokyo   vault-team 0 Mar 26 08:41 access-codes.txt
-rw-r--r-- 1 berlin  tech-team  0 Mar 26 08:41 blueprints.pdf
-rw-r--r-- 1 nairobi vault-team 0 Mar 26 08:41 escape-plan.txt
```

---

## Commands Used

```bash
# View ownership
ls -l filename
ls -ld directory/
ls -lR directory/       # recursive listing

# Create users and groups
sudo useradd -M -s /bin/bash username
sudo groupadd groupname

# Change owner only
sudo chown newowner filename

# Change group only
sudo chgrp newgroup filename

# Change both owner and group
sudo chown owner:group filename

# Change only group (via chown)
sudo chown :groupname filename

# Recursive change on directories
sudo chown -R owner:group directory/
```

---

## What I Learned

1. **Ownership has two layers** – every file/directory has an individual *user owner* and a *group owner*. Linux checks these independently when evaluating permissions (owner bits → group bits → other bits).

2. **`chown` is a superset of `chgrp`** – `chown owner:group` replaces the need for a separate `chgrp` call. Using `chown :group` changes only the group, making `chgrp` largely optional in practice.

3. **`-R` is powerful but must be used carefully** – recursive ownership changes affect every file in the tree instantly. In production, always `ls -lR` the target directory first so you know exactly what you're changing before running `chown -R`.

---

## Why This Matters for DevOps

| Scenario | Ownership Relevance |
|---|---|
| App deployments | Web server user (e.g., `www-data`) must own static assets |
| Shared CI/CD artifacts | Build runner user needs write access; no one else should |
| Container volumes | Host UID/GID must match container process UID/GID |
| Log management | Log aggregation daemons need read access; app process needs write |
| Database data dirs | DB process (e.g., `postgres`) must exclusively own data directory |

---

*Day 11 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
