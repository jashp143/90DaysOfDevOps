# Day 08 – Cloud Server Setup: Docker, Nginx & Web Deployment

**Name:** [Your Name]  
**Date:** [Date]  
**Cloud Provider:** [AWS EC2 / Utho]  
**Instance IP:** [Your Instance IP]

---

## Commands Used

### Part 1: Launch & SSH Access

```bash
# Connect to your EC2 instance (AWS)
ssh -i your-key.pem ubuntu@<your-instance-ip>

# Connect to your instance (Utho)
ssh root@<your-instance-ip>
```

### Part 2: System Update & Nginx Installation

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install Nginx
sudo apt install nginx -y

# Start and enable Nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Verify Nginx is running
sudo systemctl status nginx
```

### Part 3: Security Group / Firewall Configuration

```bash
# Allow HTTP traffic (port 80) on Ubuntu firewall
sudo ufw allow 'Nginx HTTP'
sudo ufw status

# (On AWS: Configure Security Group via Console to allow port 80)
```

### Part 4: Extract Nginx Logs

```bash
# View Nginx access logs
sudo cat /var/log/nginx/access.log

# View Nginx error logs
sudo cat /var/log/nginx/error.log

# Save logs to a file
sudo cat /var/log/nginx/access.log > ~/nginx-logs.txt
sudo cat /var/log/nginx/error.log >> ~/nginx-logs.txt

# Download log file to your local machine (run from local terminal)
# For AWS:
scp -i your-key.pem ubuntu@<your-instance-ip>:~/nginx-logs.txt .

# For Utho:
scp root@<your-instance-ip>:~/nginx-logs.txt .
```

---

## Screenshots

| Screenshot | Description |
|---|---|
| `ssh-connection.png` | Terminal showing successful SSH connection to the cloud instance |
| `nginx-webpage.png` | Browser showing the Nginx welcome page at `http://<instance-ip>` |
| `docker-nginx.png` | Nginx service status output (`systemctl status nginx`) |

---

## Challenges Faced

### Challenge 1: [Describe issue here]
**Problem:** [What went wrong]  
**Solution:** [How you fixed it]

### Challenge 2: [Describe issue here]
**Problem:** [What went wrong]  
**Solution:** [How you fixed it]

---

## What I Learned

- **Cloud Instance Provisioning** – How to launch and configure a virtual machine on AWS/Utho, including choosing the right AMI, instance type, and key pair for SSH access.
- **SSH Remote Access** – How to securely connect to a remote server using SSH keys and manage files/processes from the command line.
- **Nginx Web Server** – How to install, start, enable on boot, and verify a web server; how Nginx serves a default page out of the box.
- **Security Groups & Firewalls** – How cloud security groups and OS-level firewalls (UFW) control inbound/outbound traffic, and why opening port 80 is necessary for public web access.
- **Log Management** – Where Nginx stores access and error logs, how to view and extract them, and how to transfer files from a remote server to a local machine using `scp`.

---

## Architecture Overview

```
Internet
    │
    ▼
[Security Group: Port 80 Open]
    │
    ▼
[Cloud Instance (EC2/Utho)]
    │
    ▼
[Nginx Web Server :80]
    │
    ▼
[Default Welcome Page]
```

---

## Why This Matters for DevOps

This exercise introduced foundational skills used every day in production DevOps:

- **Cloud infrastructure provisioning** – Spinning up and configuring servers on demand
- **Remote server management** – SSH, key-based auth, and secure access control
- **Service deployment** – Installing and running web-facing applications
- **Log management** – Accessing, reading, and archiving application logs
- **Security** – Configuring firewalls and cloud security groups to control exposure

---

## Resources

- [Nginx Documentation](https://nginx.org/en/docs/)
- [AWS EC2 Getting Started](https://docs.aws.amazon.com/ec2/)
- [UFW Firewall Guide](https://help.ubuntu.com/community/UFW)
- [SCP Command Reference](https://man7.org/linux/man-pages/man1/scp.1.html)

---

*Submitted as part of #90DaysOfDevOps – Day 08*  
*#DevOpsKaJosh #TrainWithShubham*
