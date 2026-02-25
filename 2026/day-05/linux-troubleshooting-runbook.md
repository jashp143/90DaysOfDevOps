# Linux Troubleshooting Runbook – Day 05
**Date:** 2026-02-25  
**Author:** Jash Prajapati  
**Target Service / Process:** `process_api` (PID 1 – the container's init/API process)  
**Environment:** Ubuntu 24.04.4 LTS (Noble Numbat) · Kernel 4.4.0 · x86_64

---

## 1. Environment Basics

### Commands run

```bash
uname -a
# Linux runsc 4.4.0 #1 SMP Sun Jan 10 15:06:54 PST 2016 x86_64 GNU/Linux

cat /etc/os-release
# PRETTY_NAME="Ubuntu 24.04.4 LTS"
# VERSION_ID="24.04"
```

**Observation:** Kernel 4.4 is older than the Ubuntu 24.04 userland would normally carry; this is a sandboxed container runtime (gVisor/runsc). Knowing the container runtime is critical – some tools like `journalctl` and `ss` may behave differently or be unavailable.

---

## 2. Filesystem Sanity

### Commands run

```bash
mkdir /tmp/runbook-demo
cp /etc/hosts /tmp/runbook-demo/hosts-copy && ls -l /tmp/runbook-demo

# Output:
# total 1
# -rwxr-xr-x 1 root root 98 Feb 25 05:50 hosts-copy
```

**Observation:** Write access to `/tmp` is confirmed. The `/etc/hosts` file is only 98 bytes – minimal container hosts file, expected. Filesystem responds normally.

---

## 3. CPU & Memory

### Commands run

```bash
ps aux --sort=-%cpu | head -10
# PID  %CPU  %MEM  COMMAND
#  25  100   0.0   ps aux          ← the ps command itself at 100% (transient)
#  24   66   0.0   /bin/sh -c ...
#   1    3   0.1   process_api

ps -o pid,pcpu,pmem,comm -p 1
# PID  %CPU  %MEM  COMMAND
#   1   3.0   0.1  process_api

free -h
# Mem:   9.0Gi   11Mi   9.0Gi   0B   8.3Mi   9.0Gi
# Swap:    0B     0B     0B
```

**Observation:** Memory pressure is negligible – only 11 MiB of 9 GiB used. `process_api` idles at ~3% CPU; the spike to 100% was the `ps` command itself (transient, expected). No swap is configured, which would be a concern under heavy load.

---

## 4. Disk & IO

### Commands run

```bash
df -h
# Filesystem   Size  Used  Avail  Use%  Mounted on
# none         9.9G  2.2M  9.9G    1%  /
# none         315G  0     315G    0%  /dev/shm
# none         1.0P  0     1.0P    0%  /mnt/user-data/outputs  (among others)

du -sh /var/log
# 973K  /var/log
```

**Observation:** Root filesystem is at 1% capacity – no disk pressure. `/var/log` is under 1 MB, indicating a fresh or minimal container. All mounted `/mnt` paths are petabyte-scale shared volumes; effectively unlimited for this workload.

---

## 5. Network

### Commands run

```bash
cat /proc/net/tcp | head -5
# sl  local_address   rem_address  st
#  3: 00000000:07E8   00000000:0000  0A   ← 0.0.0.0:2024 LISTEN
# 40: C8000015:07E8   4418040A:D332  01   ← established connection

curl -sI http://localhost:2024
# (No curl binary; used /proc/net/tcp instead)
```

**Observation:** Port `0x07E8 = 2024` is listening on all interfaces (state `0A` = LISTEN). One active TCP connection exists (state `01` = ESTABLISHED), confirming the API process is serving live traffic. `ss` and `netstat` were unavailable in this sandbox; `/proc/net/tcp` was used as the fallback.

---

## 6. Logs Reviewed

### Commands run

```bash
journalctl -n 20
# -- No entries --   (systemd journal empty in this container)

tail -n 30 /var/log/dpkg.log
# Last entries: 2026-02-18 20:00:22 status installed ca-certificates-java
# (package install log from container image build; no runtime errors)

tail -n 20 /var/log/bootstrap.log
# Shows initial package setup at image build time; clean, no errors.
```

**Observation:** No runtime error logs found. The journal is empty (expected in a minimal sandbox container). `dpkg.log` and `bootstrap.log` only reflect build-time activity from 2026-02-18 – nothing suspicious. Log verbosity is minimal; would need app-level logging to debug application issues.

---

## 7. Quick Findings

| Area | Status | Note |
|------|--------|------|
| CPU | ✅ Normal | `process_api` at ~3%, no sustained spikes |
| Memory | ✅ Healthy | 11 MiB / 9 GiB used; no swap configured |
| Disk | ✅ Healthy | Root FS at 1%; logs < 1 MB |
| Network | ✅ Active | Port 2024 listening, 1 live connection |
| Logs | ⚠️ Minimal | No errors, but log coverage is very thin |

**Overall:** System is healthy. The only risk factor is the absence of swap and thin logging coverage, which would make memory pressure or application crashes harder to diagnose.

---

## 8. If This Worsens – Next Steps

### Step 1 – Investigate a CPU/Memory spike
```bash
# Watch process resource use in real time
watch -n 2 'ps -o pid,pcpu,pmem,vsz,rss,comm -p 1'

# Check for memory growth over time (run twice, 30s apart)
cat /proc/1/status | grep -E 'VmRSS|VmSize|Threads'
```
If RSS grows unbounded → suspect a memory leak; consider restarting the process with `kill -HUP 1` (graceful reload) before a hard restart.

### Step 2 – Increase log verbosity and collect traces
```bash
# Redirect process_api stderr to a file if no log flags exist
/process_api ... 2>/var/log/process_api.log &

# Attach strace to the running process for 30 seconds
strace -p 1 -e trace=network,read,write -o /tmp/strace_api.txt -T &
sleep 30 && kill %1
grep -E 'EAGAIN|ECONNREFUSED|ETIMEDOUT' /tmp/strace_api.txt
```
Look for repeated failed syscalls or blocked I/O that points to the bottleneck.

### Step 3 – Controlled restart with evidence collection
```bash
# 1. Snapshot state before restart
ps aux > /tmp/before_restart.txt
cat /proc/net/tcp > /tmp/tcp_before.txt

# 2. Graceful restart attempt
kill -TERM 1        # ask for clean shutdown
sleep 5
kill -KILL 1        # force kill only if TERM didn't work

# 3. Compare state after restart
diff /tmp/before_restart.txt <(ps aux)
```
If the process does not recover cleanly, escalate by collecting a core dump (`ulimit -c unlimited`) before the kill and filing an incident with the dump + strace output attached.

---

*Runbook version 1.0 – revisit after first real incident to update with actual observed thresholds.*
