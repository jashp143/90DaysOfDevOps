# Day 12 – Revision Notes & Checkpoints (Days 01–11)

Short consolidation pass: retention, not new topics.

---

## Mindset & plan (Day 01)

- **Revisit:** A 90-day plan should stay honest—goals can shift as you learn what actually takes time.
- **Checkpoint:** Three goals still point at *Linux comfort*, *automation/CI thinking*, and *one deployable path* (VM/cloud). Tweaks: add 15 min/week for *pure* revision (like today).
- **Today’s tweak:** Treat “I’ll fix it in the docs later” as debt—one line in the plan beats a vague intention.

---

## Processes & services (Days 04–05)

**Commands to re-run on a Linux host (or WSL):**

| Command | What to notice |
|--------|----------------|
| `ps aux \| head` | PPID, CPU%, CMD—who owns the process |
| `systemctl status ssh` (or `nginx`, `docker`) | `Active:` line, recent log snippet, exit codes |

**Observation (template—fill after you run):**  
- `ps`: top few lines show shell + short-lived vs long-running services.  
- `systemctl status <unit>`: confirms *loaded/enabled* state and ties errors to `journalctl`.

**Checkpoint:** If a service looks “failed,” next step is `journalctl -u <unit> -e --no-pager` (or `-n 50`).

---

## File skills quick practice (Days 06–11)

Example micro-drill (safe in a temp dir):

```bash
mkdir -p /tmp/day12-rev && cd /tmp/day12-rev
echo "revision" >> notes.txt
ls -l notes.txt
chmod 640 notes.txt
ls -l notes.txt
cp notes.txt notes.bak
```

- **Checkpoint:** After `chmod 640`, owner read/write, group read, others nothing—matches mental model from numeric and symbolic modes.

---

## Cheat sheet refresh (Day 03)—5 commands first in an incident

1. **`ps aux`** — what is running, under which user.  
2. **`systemctl status <service>`** — is the unit active/failed, and a log hint.  
3. **`journalctl -u <service> -n 100 --no-pager`** — recent failure lines without paging noise in scripts.  
4. **`ls -la`** — permissions and ownership on config paths.  
5. **`curl -I <url>`** or **`ss -tlnp`** — quick “is something listening / responding?” split.

---

## User / group sanity (Days 09 & 11)

**Small scenario (run as root or with `sudo` where appropriate):**

```bash
sudo useradd -m -s /bin/bash day12demo
sudo id day12demo
sudo chown day12demo:day12demo /tmp/day12-rev/notes.txt   # if that user should own the file
ls -l /tmp/day12-rev/notes.txt
```

- **Checkpoint:** `id` shows uid/gid/groups; `ls -l` confirms user:group on disk match intent.

---

## Mini self-check — answers

### 1) Which 3 commands save you the most time right now, and why?

- **`grep -R` / `rg`** (if installed)—cuts through logs and configs to the needle.  
- **`systemctl status` + `journalctl -u`**—one-two punch for “service is wrong” without guessing.  
- **`ls -la`**—instant read on permissions/ownership before you `chmod`/`chown` blindly.

### 2) How do you check if a service is healthy? List the exact 2–3 commands you’d run first.

1. `systemctl status <service>` — state, loaded unit, last log lines.  
2. `journalctl -u <service> -n 80 --no-pager` — errors and restarts.  
3. (If it’s network-facing) `ss -tlnp | grep <port>` or `curl -fsS http://127.0.0.1:<port>/health` — proves bind + response.

### 3) How do you safely change ownership and permissions without breaking access? Give one example command.

- **Rule:** Change ownership to the *account that must write*, give group read/write only if a shared group is real, avoid `777`. Verify with `ls -l` before and after.  
- **Example:** `sudo chown deploy:webapp /var/www/app/config.yml && sudo chmod 640 /var/www/app/config.yml` — owner read/write, group read, others none; adjust group if `webapp` must read.

### 4) What will you focus on improving in the next 3 days?

- Day 13+: **systemd/journalctl fluency** (filters, since boot).  
- **One scripted backup** of a config before editing (`cp file file.bak.$(date +%Y%m%d)`).  
- **Numeric `chmod` drill** until 640/750/755 are automatic without a table.

---

## Key takeaways (2–3 lines for “learn in public”)

Reinforced that DevOps debugging is **observe → logs → permissions → network**, in that order when something breaks. The command I’ll remember cold: **`journalctl -u <service> -n 50 --no-pager`** for a fast, copy-pasteable failure window.
