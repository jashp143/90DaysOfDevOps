# Day 04 – Linux Practice: Processes and Services

**Date:** February 04, 2026  
**Focus:** Hands-on Linux fundamentals - Processes, Services, and Logs

---

## 🎯 Overview

This document captures real command outputs from practicing Linux process management, service inspection, and log analysis. All commands were executed on an Ubuntu 24 system.

---

## 📊 Process Checks

### 1. Viewing Running Processes with `ps`

**Command:**
```bash
ps aux | head -20
```

**Purpose:** Display all running processes with detailed information

**Output:**
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  1.3  0.1 352336 18484 ?        Ssl  14:29   0:00 /process_api --addr 0.0.0.0:2024
root        11 50.0  0.0  10848  3960 ?        S    14:29   0:00 /bin/sh -c ps aux | head -20
root        12  100  0.0  15996  8376 ?        R    14:29   0:00 ps aux
root        13  100  0.0  10756  4208 ?        S    14:29   0:00 head -20
```

**Key Observations:**
- PID 1 is the main process (process_api)
- CPU and memory usage are minimal on this system
- Process states: R (Running), S (Sleeping), Ssl (Session leader sleeping)

---

### 2. Monitoring System Resources with `top`

**Command:**
```bash
top -bn1 | head -20
```

**Purpose:** Get a snapshot of system resource usage and top processes

**Output:**
```
top - 14:29:54 up 0 min,  0 user,  load average: 0.00, 0.00, 0.00
Tasks:   4 total,   1 running,   3 sleeping,   0 stopped,   0 zombie
%Cpu(s):   0.0 us,   0.0 sy,   0.0 ni, 100.0 id,   0.0 wa,   0.0 hi,   0.0 si 
MiB Mem :   9216.0 total,   9200.4 free,     15.6 used,      8.6 buff/cache     
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   9200.4 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    1 root      20   0  352644  19156      0 S   0.0   0.2   0:00.21 process_api
   42 root      20   0   10848   3612      0 S   0.0   0.0   0:00.01 sh
   43 root      20   0   16972   8984      0 R   0.0   0.1   0:00.02 top
```

**Key Observations:**
- System has 9216 MB total memory with 9200 MB free
- Load average is 0.00 (system is idle)
- Only 4 processes running total
- No swap being used

---

### 3. Sorting Processes by Memory Usage

**Command:**
```bash
ps aux --sort=-%mem | head -10
```

**Purpose:** Identify memory-intensive processes

**Output:**
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  1.7  0.2 352644 19228 ?        Ssl  14:29   0:00 /process_api
root        59  100  0.0  15996  9216 ?        R    14:30   0:00 ps aux --sort=-%mem
root        58  0.0  0.0  10848  3708 ?        S    14:30   0:00 /bin/sh -c ps aux
root        60  100  0.0  10756  3092 ?        S    14:30   0:00 head -10
```

**Key Observations:**
- process_api (PID 1) uses the most memory at 0.2%
- RSS (Resident Set Size) shows actual physical memory used
- The system is very light on memory usage

---

## 🔧 Service Checks

### 4. Listing Available Services

**Command:**
```bash
service --status-all 2>&1 | head -20
```

**Purpose:** Check all available services on the system

**Output:**
```
 [ - ]  dbus
 [ - ]  procps
 [ - ]  x11-common
```

**Key Observations:**
- Three services are available on this system
- `[ - ]` indicates services are stopped/inactive
- Limited services due to containerized environment

---

### 5. Inspecting a Specific Service: dbus

**Command:**
```bash
service dbus status
```

**Purpose:** Check the status of the D-Bus message bus service

**Output:**
```
 * dbus is not running
```

**What is dbus?**
- D-Bus is an inter-process communication (IPC) system
- Allows applications to communicate with each other
- Essential for desktop environments and many system services
- Not running in this environment (likely not needed)

**Troubleshooting Notes:**
- Service is installed but not active
- This is expected in containerized environments where dbus may not be required
- In a full desktop/server environment, dbus typically runs automatically

---

### 6. Attempting systemctl Commands

**Command:**
```bash
systemctl list-units --type=service --state=running
```

**Output:**
```
System has not been booted with systemd as init system (PID 1). Can't operate.
Failed to connect to bus: Host is down
```

**Key Observations:**
- This system doesn't use systemd as its init system
- Traditional `service` command is available instead
- Different init systems: systemd, SysVinit, upstart, etc.
- Important to adapt commands based on the init system in use

---

## 📜 Log Checks

### 7. Exploring Log Directory

**Command:**
```bash
ls -la /var/log/ | head -15
```

**Purpose:** See what log files are available

**Output:**
```
total 663
drwxr-xr-x  5 root root               280 Nov 21 01:56 .
drwxr-xr-x 11 root root               260 Oct 13 14:09 ..
-rw-r--r--  1 root root             20360 Nov 21 01:59 alternatives.log
drwxr-xr-x  2 root root               100 Nov 21 01:59 apt
-rw-r--r--  1 root root             61229 Oct 13 14:03 bootstrap.log
-rw-rw----  1 root utmp                 0 Oct 13 14:02 btmp
-rw-r--r--  1 root root            587913 Nov 21 02:00 dpkg.log
-rw-r--r--  1 root root                 0 Oct 13 14:03 faillog
drwxr-sr-x  2 root systemd-journal     40 Nov 21 01:55 journal
-rw-rw-r--  1 root utmp                 0 Oct 13 14:02 lastlog
-rw-rw-r--  1 root utmp                 0 Oct 13 14:02 wtmp
```

**Important Log Files:**
- `dpkg.log` - Package management activities
- `apt/` - APT package manager logs
- `btmp` - Failed login attempts
- `lastlog` - Last login information
- `faillog` - Failed authentication attempts

---

### 8. Viewing Recent Package Events

**Command:**
```bash
tail -n 30 /var/log/dpkg.log
```

**Purpose:** Check recent package installation/configuration activity

**Output (snippet):**
```
2025-11-21 01:59:15 status installed libnice10:amd64 0.1.21-2build3
2025-11-21 01:59:15 configure libgstreamer-plugins-bad1.0-0:amd64 1.24.2-1ubuntu4
2025-11-21 01:59:15 status installed gstreamer1.0-plugins-bad:amd64 1.24.2-1ubuntu4
2025-11-21 01:59:16 trigproc libc-bin:amd64 2.39-0ubuntu8.6
2025-11-21 02:00:08 status installed ca-certificates-java:all 20240118
```

**Key Observations:**
- Shows package installations and configurations
- Trigger processing (trigproc) for dependency updates
- Timestamps help track when changes occurred
- Useful for troubleshooting package-related issues

---

### 9. Searching for Errors in Logs

**Command:**
```bash
grep -i error /var/log/dpkg.log | tail -10
```

**Purpose:** Find error-related entries in package logs

**Output:**
```
2025-11-21 01:55:40 install liberror-perl:all <none> 0.17029-2
2025-11-21 01:55:40 status half-installed liberror-perl:all 0.17029-2
2025-11-21 01:55:40 status unpacked liberror-perl:all 0.17029-2
2025-11-21 01:56:37 configure liberror-perl:all 0.17029-2
2025-11-21 01:56:37 status installed liberror-perl:all 0.17029-2
```

**Key Observations:**
- The grep found "error" in package name (liberror-perl)
- These are normal installations, not actual errors
- Shows importance of understanding context in log analysis
- Real errors would show different status messages

---

### 10. Checking Kernel Messages

**Command:**
```bash
dmesg | tail -20
```

**Purpose:** View recent kernel ring buffer messages

**Output:**
```
[    0.000000] Starting gVisor...
[    0.341771] Generating random numbers by fair dice roll...
[    0.486080] Moving files to filing cabinet...
[    0.721498] Segmenting fault lines...
[    1.163252] Daemonizing children...
[    1.212899] Deleting VFS and rebuilding it from scratch...
[    1.570931] Reading process obituaries...
[    2.465548] Ready!
```

**Key Observations:**
- Shows system boot sequence
- Kernel messages help diagnose hardware/driver issues
- Timestamps indicate boot progression
- In production, you'd see hardware detection, module loading, etc.

---

## 🔍 Mini Troubleshooting Flow: Service Not Running

### Scenario: dbus service is not running

**Step 1: Identify the Problem**
```bash
service dbus status
# Output: * dbus is not running
```

**Step 2: Check if Service Exists**
```bash
service --status-all | grep dbus
# Output: [ - ]  dbus
```
✅ Service exists but is stopped

**Step 3: Check System Logs**
```bash
grep -i dbus /var/log/dpkg.log | tail -5
```
Would show if dbus package was recently installed/modified

**Step 4: Attempt to Start (if needed)**
```bash
service dbus start
```
(Not executed as dbus isn't needed in this environment)

**Step 5: Verify and Monitor**
```bash
service dbus status
# Check if service started successfully
```

### Troubleshooting Takeaways:
- Always verify service status first
- Check logs for clues about failures
- Understand if the service is actually needed
- In containerized environments, many traditional services aren't required

---

## 📚 Key Commands Summary

### Process Management
| Command | Purpose |
|---------|---------|
| `ps aux` | Show all running processes |
| `ps -ef` | Show processes in hierarchical format |
| `top` | Real-time process monitoring |
| `pgrep <name>` | Find process IDs by name |
| `ps aux --sort=-%mem` | Sort processes by memory usage |

### Service Management
| Command | Purpose |
|---------|---------|
| `service --status-all` | List all services |
| `service <name> status` | Check specific service status |
| `systemctl list-units --type=service` | List systemd services |
| `systemctl status <name>` | Check systemd service status |

### Log Analysis
| Command | Purpose |
|---------|---------|
| `tail -n 50 /var/log/file` | View last 50 lines of log |
| `grep -i keyword /var/log/file` | Search logs for keywords |
| `dmesg` | View kernel messages |
| `journalctl -u <service>` | View systemd service logs |
| `ls -la /var/log/` | List available log files |

---

## 💡 Lessons Learned

1. **Process Monitoring:**
   - Different commands provide different views (ps vs top)
   - Understanding PID, CPU%, and MEM% is crucial
   - Sorting helps identify resource-intensive processes

2. **Service Management:**
   - Not all systems use systemd - adapt to the environment
   - Service status tells you if something is running
   - Services can be installed but not running

3. **Log Analysis:**
   - Logs are essential for troubleshooting
   - Context matters when searching for errors
   - Timestamps help correlate events
   - Different logs serve different purposes

4. **DevOps Relevance:**
   - Quick diagnosis saves time in production incidents
   - Knowing where to look first is critical
   - Building muscle memory with basic commands pays off
   - Understanding the system's init type affects your approach

---

## 🎓 Why This Matters for DevOps

In production environments:
- **Processes** - You need to quickly identify runaway processes consuming resources
- **Services** - Critical services going down requires immediate investigation
- **Logs** - First place to look when something breaks
- **Speed** - Knowing these commands by heart means faster resolution times

When an alert fires at 3 AM, you won't have time to Google basic commands. This practice builds the muscle memory needed for effective incident response.

---

## 🔗 Resources Referenced

- Linux man pages (`man ps`, `man top`, `man systemctl`)
- Day 02 & Day 03 DevOps notes
- System log file documentation
- Ubuntu system administration guides

---

## ✅ Practice Checklist

- [x] Ran 6+ commands
- [x] Included 3 process commands (ps, top, ps with sort)
- [x] Included 2 service commands (service --status-all, service dbus status)
- [x] Included 3 log commands (ls /var/log, tail dpkg.log, grep, dmesg)
- [x] Inspected one service in detail (dbus)
- [x] Created mini troubleshooting flow
- [x] Documented observations and learnings

---

**Practice Date:** February 04, 2026  
**System:** Ubuntu 24.04  
**Total Commands Executed:** 10+  
**Time Spent:** ~45 minutes

#90DaysOfDevOps #DevOpsKaJosh #TrainWithShubham
