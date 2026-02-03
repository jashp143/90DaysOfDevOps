# Linux Commands Cheat Sheet for DevOps

A quick reference guide for essential Linux commands used in production environments.

---

## 📊 Process Management

| Command | Usage |
|---------|-------|
| `ps aux` | Display all running processes with detailed info |
| `ps aux \| grep <name>` | Find specific process by name |
| `top` | Real-time view of system processes and resource usage |
| `htop` | Interactive process viewer (enhanced top) |
| `kill <PID>` | Terminate process by Process ID |
| `kill -9 <PID>` | Force kill a process (SIGKILL) |
| `killall <name>` | Kill all processes by name |
| `systemctl status <service>` | Check status of a systemd service |
| `systemctl start/stop/restart <service>` | Manage systemd services |
| `journalctl -u <service>` | View logs for a specific service |
| `nice -n <priority> <command>` | Run command with modified priority (-20 to 19) |
| `bg` | Resume suspended job in background |
| `fg` | Bring background job to foreground |

---

## 📁 File System Operations

| Command | Usage |
|---------|-------|
| `df -h` | Display disk space usage in human-readable format |
| `du -sh <directory>` | Show size of specific directory |
| `du -h --max-depth=1` | Show sizes of subdirectories one level deep |
| `find / -name <filename>` | Search for file by name from root |
| `find . -type f -mtime -7` | Find files modified in last 7 days |
| `find . -size +100M` | Find files larger than 100MB |
| `tar -czvf archive.tar.gz <dir>` | Create compressed tar archive |
| `tar -xzvf archive.tar.gz` | Extract compressed tar archive |
| `ln -s <target> <link>` | Create symbolic link |
| `chmod 755 <file>` | Change file permissions (rwxr-xr-x) |
| `chown user:group <file>` | Change file owner and group |
| `lsof -i :<port>` | List processes using specific port |
| `lsof -p <PID>` | List files opened by specific process |

---

## 🌐 Networking & Troubleshooting

| Command | Usage |
|---------|-------|
| `ping <host>` | Test network connectivity to host |
| `ping -c 4 <host>` | Ping with specific count (4 packets) |
| `ip addr show` | Display network interfaces and IP addresses |
| `ip route show` | Display routing table |
| `curl <URL>` | Transfer data from/to server |
| `curl -I <URL>` | Fetch HTTP headers only |
| `wget <URL>` | Download files from web |
| `dig <domain>` | DNS lookup and query information |
| `nslookup <domain>` | Query DNS name servers |
| `netstat -tulpn` | Show listening ports and services |
| `ss -tulpn` | Modern replacement for netstat |
| `traceroute <host>` | Trace network path to destination |
| `tcpdump -i eth0` | Capture network packets on interface |
| `nc -zv <host> <port>` | Test if port is open (netcat) |
| `ifconfig` | Legacy command to configure network interfaces |

---

## 📝 Log Analysis & Monitoring

| Command | Usage |
|---------|-------|
| `tail -f /var/log/syslog` | Follow log file in real-time |
| `tail -n 100 <logfile>` | Display last 100 lines of file |
| `grep -i "error" <logfile>` | Search for "error" (case-insensitive) |
| `grep -r "pattern" /var/log/` | Recursively search pattern in directory |
| `less <logfile>` | View large files page by page |
| `cat <file> \| grep <pattern>` | Filter file content by pattern |
| `wc -l <file>` | Count number of lines in file |

---

## ⚡ System Information

| Command | Usage |
|---------|-------|
| `uname -a` | Display system information |
| `uptime` | Show system uptime and load average |
| `free -h` | Display memory usage in human-readable format |
| `vmstat 1 5` | Report virtual memory statistics (1 sec, 5 times) |
| `lscpu` | Display CPU architecture information |
| `hostnamectl` | Query and change system hostname |

---

## 🔐 User & Permission Management

| Command | Usage |
|---------|-------|
| `whoami` | Display current username |
| `id` | Display user and group IDs |
| `sudo -i` | Switch to root user with root environment |
| `su - <username>` | Switch to another user |
| `passwd <username>` | Change user password |

---

## 💡 Quick Tips for DevOps

### Finding Troublesome Processes
```bash
# Find processes consuming most CPU
ps aux --sort=-%cpu | head -10

# Find processes consuming most memory
ps aux --sort=-%mem | head -10
```

### Disk Space Emergency
```bash
# Find largest directories in current location
du -h --max-depth=1 | sort -hr | head -10

# Find large files (>100MB) in system
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null
```

### Network Diagnostics
```bash
# Check if service is responding
curl -I -s http://localhost:8080 | head -1

# Test DNS resolution
dig +short google.com

# Find process using specific port
lsof -i :8080
```

---

## 🎯 Common Production Scenarios

### Service Down Investigation
1. `systemctl status <service>` - Check service status
2. `journalctl -u <service> -n 50` - Check recent logs
3. `ps aux | grep <service>` - Verify process running
4. `netstat -tulpn | grep <port>` - Check if port is listening

### High CPU Usage
1. `top` - Identify process consuming CPU
2. `ps aux --sort=-%cpu | head -10` - Top CPU consumers
3. `kill -15 <PID>` - Gracefully stop process
4. `systemctl restart <service>` - Restart if needed

### Disk Full
1. `df -h` - Check disk usage by partition
2. `du -sh /var/log/*` - Check log directory sizes
3. `find /var/log -type f -name "*.log" -mtime +30 -delete` - Clean old logs
4. `journalctl --vacuum-time=7d` - Clean systemd logs

---

## 📚 Man Pages
Remember: When in doubt, use `man <command>` to read the manual!

**Example:** `man grep`, `man ssh`, `man systemctl`

---

**Created for:** #90DaysOfDevOps Challenge  
**Day:** 03 - Linux Commands Practice  
**Focus:** DevOps essentials for production troubleshooting

---

*Keep this cheat sheet handy! These commands will save you during critical production incidents.* 🚀
