# Linux Architecture, Processes, and systemd

## Core Components of Linux

### 1. **The Kernel**
- The core of the operating system that manages hardware resources
- Handles memory management, process scheduling, device drivers, and system calls
- Acts as a bridge between applications and hardware
- Operates in privileged mode with direct hardware access

### 2. **User Space**
- Where all user applications and processes run
- Includes shells, utilities, and application programs
- Operates with restricted privileges for security
- Communicates with kernel through system calls

### 3. **Init System (systemd)**
- First process started by kernel (PID 1)
- Responsible for initializing the system and managing services
- Parent of all other processes
- Modern Linux systems use **systemd** as the init system

---

## Process Management

### What is a Process?
A running instance of a program with its own memory space, resources, and execution context.

### Process States

| State | Description |
|-------|-------------|
| **Running (R)** | Currently executing on CPU or ready to run |
| **Sleeping (S)** | Waiting for an event or resource (interruptible) |
| **Uninterruptible Sleep (D)** | Waiting for I/O, cannot be interrupted |
| **Stopped (T)** | Paused by signal (Ctrl+Z) |
| **Zombie (Z)** | Terminated but parent hasn't read exit status |

### Process Creation
- Processes are created using `fork()` system call (creates copy of parent)
- `exec()` replaces the copied process with a new program
- Every process has a **PID** (Process ID) and **PPID** (Parent Process ID)

---

## What is systemd?

**systemd** is the modern init system and service manager for Linux.

### Key Responsibilities:
- **Boots the system** - starts services in parallel for faster boot
- **Manages services** - start, stop, restart, enable, disable services
- **Handles dependencies** - ensures services start in correct order
- **Logs management** - collects logs via journald
- **Resource control** - uses cgroups to limit CPU, memory per service

### Why systemd Matters:
- Replaced older SysVinit system for better performance
- Provides standardized way to manage services across distributions
- Essential for DevOps: most production services run under systemd
- Enables service auto-restart on failure
- Centralized logging makes debugging easier

---

## 5 Essential Daily Commands

### 1. `ps aux`
```bash
ps aux
```
View all running processes with CPU and memory usage

### 2. `top` / `htop`
```bash
top
```
Real-time monitoring of system processes and resource usage

### 3. `systemctl status <service>`
```bash
systemctl status nginx
```
Check status, logs, and health of any systemd service

### 4. `journalctl`
```bash
journalctl -u nginx -f
```
View systemd service logs (follow mode for real-time)

### 5. `kill` / `pkill`
```bash
kill -9 <PID>
pkill -f process_name
```
Terminate processes by PID or name

---

## Quick systemd Commands Reference

```bash
# Start a service
systemctl start nginx

# Stop a service
systemctl stop nginx

# Restart a service
systemctl restart nginx

# Enable service at boot
systemctl enable nginx

# Disable service at boot
systemctl disable nginx

# Check service status
systemctl status nginx

# View all active services
systemctl list-units --type=service

# View failed services
systemctl --failed
```

---

## DevOps Takeaways

Understanding Linux internals helps you:

✅ **Debug faster** - Know where to look when services crash  
✅ **Optimize resources** - Identify CPU/memory hungry processes  
✅ **Manage services** - Control application lifecycle confidently  
✅ **Read logs effectively** - Use journalctl to trace issues  
✅ **Automate deployments** - Create custom systemd services  

**Remember**: Almost every production server runs Linux. Mastering these basics saves hours during incidents and makes you a more effective DevOps engineer.

