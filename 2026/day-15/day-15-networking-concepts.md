# Day 15 – Networking Concepts: DNS, IP, Subnets & Ports

---

## Task 1: DNS – How Names Become IPs

### What happens when you type `google.com` in a browser?

1. **Browser cache** → Your browser first checks if it already knows the IP from a recent lookup.
2. **OS resolver / `/etc/hosts`** → If not cached, the OS checks its local hosts file, then asks the configured DNS resolver (e.g., `8.8.8.8` or your router).
3. **Recursive resolution** → The resolver walks the DNS tree: it asks a root nameserver (`"who handles .com?"`), then a `.com` TLD nameserver (`"who handles google.com?"`), then Google's authoritative nameserver (`"what's the A record for google.com?"`).
4. **Answer returned** → The IP (e.g., `142.250.80.46`) is returned to the browser, cached for the TTL duration, and the TCP connection to that IP begins.

The whole process typically completes in under 50ms. Every domain query you've ever made followed this exact path.

```
Browser → OS cache → /etc/resolv.conf resolver
                         │
                    Root DNS (.)
                         │
                    TLD DNS (.com)
                         │
                    Authoritative DNS (google.com NS)
                         │
                    A record: 142.250.80.46  ← answer
```

---

### DNS Record Types

| Record | Purpose | Example |
|---|---|---|
| `A` | Maps a domain name to an **IPv4 address** | `google.com → 142.250.80.46` |
| `AAAA` | Maps a domain name to an **IPv6 address** | `google.com → 2607:f8b0:4004::200e` |
| `CNAME` | **Alias** — points one name to another name (not an IP) | `www.example.com → example.com` |
| `MX` | **Mail exchange** — directs email for a domain to a mail server | `example.com → mail.example.com` |
| `NS` | **Nameserver** — identifies which DNS servers are authoritative for the domain | `google.com → ns1.google.com` |

**CNAME gotcha:** You cannot set a CNAME on a root domain (`example.com`). That's why services like Cloudflare offer "CNAME flattening" or you use an `A` record + their proxy for the apex.

---

### `dig google.com` — Reading the Output

```bash
$ dig google.com

; <<>> DiG 9.18.39 <<>> google.com
;; global options: +cmd

;; QUESTION SECTION:
;google.com.                    IN  A           ← asking for IPv4 address

;; ANSWER SECTION:
google.com.         299 IN  A   142.250.80.46   ← A record: TTL=299s, IP=142.250.80.46
google.com.         299 IN  A   142.250.80.78   ← multiple A records = load balancing
google.com.         299 IN  A   142.250.80.100

;; Query time: 4 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)               ← which resolver answered
;; WHEN: Thu Mar 26 08:58:00 UTC 2026
;; MSG SIZE rcvd: 71
```

> **Container note:** This container has no DNS resolver (`/etc/resolv.conf` points to
> `127.0.0.1:53` which isn't running). Resolution works via `/etc/hosts` static entries
> (e.g., `api.anthropic.com → 160.79.104.10`). The dig output above is representative
> of what you'd see on a real Linux host.

**Key fields to read in `dig` output:**

| Field | What it means |
|---|---|
| `299 IN A` | TTL=299 seconds, Internet class, A record type |
| `142.250.80.46` | The resolved IPv4 address |
| `SERVER: 8.8.8.8` | The resolver that answered the query |
| `Query time: 4 msec` | Total round-trip for this DNS lookup |

**Quick `dig` one-liners:**
```bash
dig google.com +short          # just the IP(s), no noise
dig google.com MX              # query MX records specifically
dig @1.1.1.1 google.com        # force Cloudflare's resolver
dig -x 142.250.80.46           # reverse DNS: IP → hostname
```

---

## Task 2: IP Addressing

### IPv4 Structure

An IPv4 address is a **32-bit number** written as four decimal octets (0–255) separated by dots:

```
192 . 168 .  1  . 10
 │      │    │    └── Host part (which device on the network)
 │      │    └─────── Network/subnet part
 └──────┴──────────── Network prefix
```

Each octet = 8 bits → 4 × 8 = **32 bits total** → 2³² = ~4.3 billion possible addresses.

We ran out of IPv4 addresses globally in 2011. That's why NAT (Network Address Translation)
and IPv6 exist.

---

### Public vs Private IPs

| Type | Purpose | Example | Routable on internet? |
|---|---|---|---|
| **Public** | Globally unique, assigned by ISPs | `142.250.80.46` (Google) | ✅ Yes |
| **Private** | Internal networks only, reused everywhere | `192.168.1.10` (your laptop) | ❌ No |

**Private IP ranges (RFC 1918):**

| Range | CIDR | Common use |
|---|---|---|
| `10.0.0.0 – 10.255.255.255` | `10.0.0.0/8` | Large enterprise, cloud VPCs (AWS, GCP default) |
| `172.16.0.0 – 172.31.255.255` | `172.16.0.0/12` | Docker default bridge network (`172.17.0.0/16`) |
| `192.168.0.0 – 192.168.255.255` | `192.168.0.0/16` | Home routers, small office networks |

Private IPs communicate with the internet through **NAT** — the router swaps the private
source IP with its own public IP before the packet leaves your network.

---

### `ip addr show` — Identifying Private IPs

```bash
$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP>
    inet 127.0.0.1/8        ← loopback (special, not RFC 1918 but also not public)

2: 7b482401cb-v: <UP,LOWER_UP>
    inet 21.0.0.154/31      ← NOT a standard private range
    inet6 fe80::cc68:5bff:fec9:679e/64  ← link-local IPv6 (fe80::/10 range)
```

**Observation:** `21.0.0.154` is not in a standard RFC 1918 private range — it's a cloud
provider internal IP in a container-to-host /31 point-to-point link. Not routable on the
public internet. `fe80::` is an IPv6 link-local address, automatically assigned and only
valid on the local network segment.

---

## Task 3: CIDR & Subnetting

### What does `/24` mean?

CIDR (Classless Inter-Domain Routing) notation `192.168.1.0/24` means:
- The **first 24 bits** of the address identify the **network**
- The remaining **8 bits** identify individual **hosts**

```
192.168.1.0/24 in binary:
11000000.10101000.00000001. 00000000
├──────────────────────────┤└───────┘
        24 bits (network)    8 bits (hosts)
```

The `/` number = how many bits are locked to the network prefix. Higher number = smaller network.

### Why do we subnet?

1. **Security isolation** — place the database on `10.0.2.0/24` and the web servers on
   `10.0.1.0/24`. A firewall rule can block all traffic between subnets except what's needed.
2. **Efficient IP allocation** — don't give a 2-server team a `/16` (65k IPs). Give them a `/28` (14 IPs).
3. **Broadcast domain control** — ARP broadcasts stay within a subnet. Huge flat networks
   generate excessive broadcast traffic that degrades performance.
4. **Cloud networking** — AWS VPCs and GCP VPCs require you to define subnets to place
   EC2 instances, RDS, and load balancers in specific availability zones.

---

### CIDR Table

| CIDR | Subnet Mask | Total IPs | Usable Hosts | Notes |
|---|---|---|---|---|
| `/24` | `255.255.255.0` | 256 | **254** | Standard small network; -2 for network + broadcast |
| `/16` | `255.255.0.0` | 65,536 | **65,534** | AWS default VPC size |
| `/28` | `255.255.255.240` | 16 | **14** | Smallest practical subnet for a service tier |

**The -2 rule:** Every subnet loses 2 addresses — the **network address** (all host bits = 0,
e.g., `192.168.1.0`) and the **broadcast address** (all host bits = 1, e.g., `192.168.1.255`).
Neither can be assigned to a host.

**Quick mental math:** Usable hosts = `2^(32 - prefix) - 2`
- `/24` → `2^8 - 2` = 254
- `/16` → `2^16 - 2` = 65,534
- `/28` → `2^4 - 2` = 14

---

### Subnetting a Network — Worked Example

Splitting `10.0.0.0/24` into four `/26` subnets:

```
10.0.0.0/26   → 10.0.0.0   – 10.0.0.63   (62 hosts)  ← Web tier
10.0.0.64/26  → 10.0.0.64  – 10.0.0.127  (62 hosts)  ← App tier
10.0.0.128/26 → 10.0.0.128 – 10.0.0.191  (62 hosts)  ← DB tier
10.0.0.192/26 → 10.0.0.192 – 10.0.0.255  (62 hosts)  ← Reserved/mgmt
```

Each `/26` has 64 IPs (62 usable). Four of them exactly fill the original `/24`.

---

## Task 4: Ports – The Doors to Services

### What is a port?

An IP address identifies a **machine**. A port identifies a specific **service/process** on
that machine. Without ports, a server couldn't distinguish between incoming SSH, HTTP, and
database connections — they all arrive at the same IP.

Ports are **16-bit numbers** (0–65535):
- **0–1023** — Well-known/system ports (need root to bind)
- **1024–49151** — Registered ports (common app ports like 8080, 3306)
- **49152–65535** — Ephemeral/dynamic ports (OS assigns these to outgoing connections)

When your browser connects to `google.com:443`, the destination port is 443 (HTTPS server),
and the OS assigns your browser an ephemeral source port like `51234`.

---

### Common Ports

| Port | Protocol | Service | Notes |
|---|---|---|---|
| `22` | TCP | **SSH** | Secure Shell — remote terminal access |
| `80` | TCP | **HTTP** | Unencrypted web traffic |
| `443` | TCP | **HTTPS** | Encrypted web (HTTP over TLS) |
| `53` | UDP/TCP | **DNS** | Name resolution; UDP for queries, TCP for zone transfers |
| `3306` | TCP | **MySQL** | MySQL/MariaDB database |
| `6379` | TCP | **Redis** | In-memory cache / message broker |
| `27017` | TCP | **MongoDB** | MongoDB document database |

**Other ports worth knowing as a DevOps engineer:**

| Port | Service |
|---|---|
| `5432` | PostgreSQL |
| `8080` | Common HTTP alternative / Java app servers (Tomcat) |
| `9200` | Elasticsearch |
| `2181` | Zookeeper |
| `9092` | Kafka |
| `2379/2380` | etcd (Kubernetes control plane) |
| `10250` | Kubernetes kubelet API |

---

### `ss -tulpn` — Matching Ports to Services

```bash
$ ss -tulpn
Netid  State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
tcp    LISTEN  0       0       0.0.0.0:2024         0.0.0.0:*          process_api (pid=1)
```

**Port 2024** → `process_api` — the container's internal orchestration daemon.

This is not in the well-known ports table (22, 80, 443…), which is normal for
internal services. DevOps lesson: always check `ss -tulpn` on a new server before
assuming what's running. Never assume port → service without verifying.

**On a typical Ubuntu server you'd expect to see:**
```
tcp  LISTEN  0.0.0.0:22     →  sshd
tcp  LISTEN  0.0.0.0:80     →  nginx
tcp  LISTEN  0.0.0.0:443    →  nginx
tcp  LISTEN  127.0.0.1:3306 →  mysqld    (note: 127.0.0.1 only = only local connections)
```

---

## Task 5: Putting It Together

### `curl http://myapp.com:8080` — what's happening?

1. **DNS (L7)** → `myapp.com` is resolved to an IP via the OS resolver chain
2. **IP addressing (L3)** → The OS checks if the target IP is local or needs routing to the gateway
3. **TCP (L4)** → A three-way handshake is initiated to port `8080` on the target IP
4. **Port (L4)** → Port 8080 tells the server "this connection is for the web app", not SSH or something else
5. **HTTP (L7)** → The `GET /` request is sent over the established TCP connection

Every layer from the model is exercised in a single curl command.

---

### App can't reach database at `10.0.1.50:3306` — what to check?

Work through the layers, bottom-up:

```
Step 1 — L3 reachability:    ping 10.0.1.50
                               └─ fails? check routing (ip route show) and security groups/firewall
Step 2 — L4 port reachability: nc -zv 10.0.1.50 3306
                               └─ fails? MySQL might not be running, or port 3306 is firewalled
Step 3 — L7 service health:   mysql -h 10.0.1.50 -u user -p
                               └─ fails? check MySQL user grants (GRANT ... TO 'user'@'10.0.1.%')
Step 4 — App config:          check DB_HOST env var / connection string in app config
```

Common culprits in order of frequency:
1. Security group / firewall rule missing (port 3306 not open between app and DB subnets)
2. MySQL bound to `127.0.0.1` only (`bind-address = 127.0.0.1` in `my.cnf`)
3. DB user doesn't have remote access grants
4. Wrong hostname/IP in app config

---

## What I Learned

1. **DNS is a distributed tree, not a single lookup.** Every query you make potentially
   touches root servers → TLD servers → authoritative nameservers in sequence. TTL is your
   cache duration — a low TTL (60s) means faster propagation of changes but more DNS traffic.
   This matters for deployments: if you're rotating IPs, set TTL low *before* the change.

2. **CIDR prefix length directly controls blast radius.** A `/28` limits mistakes to 14
   hosts. A `/16` means 65k hosts share a broadcast domain. In cloud architectures
   (AWS VPC), get the subnet sizing right at design time — you can't resize a subnet
   after creation without destroying and recreating it.

3. **Ports are the first thing to check in any connectivity failure.** `nc -zv host port`
   is the fastest way to isolate whether a problem is "L3 routing" (ping fails) vs
   "L4 firewall blocking the port" (ping works, nc fails) vs "L7 application not
   responding" (nc works, app returns 500). Always work bottom-up through the layers.

---

## Commands Reference

```bash
# DNS
dig <domain>                     # full query with TTL, record types, server used
dig <domain> +short              # just the IP
dig <domain> MX                  # specific record type
dig @8.8.8.8 <domain>           # use a specific resolver
dig -x <ip>                      # reverse DNS lookup
nslookup <domain>                # simpler alternative

# IP / Interface
ip addr show                     # all interfaces, IPs, and subnet masks
ip route show                    # routing table (find default gateway)
hostname -I                      # quick list of all assigned IPs

# Subnetting (Python one-liner)
python3 -c "import ipaddress; n=ipaddress.IPv4Network('192.168.1.0/24'); print(n.netmask, n.num_addresses)"

# Ports
ss -tulpn                        # all listening TCP/UDP ports with process names
ss -tnp                          # established TCP connections with process names
nc -zv <host> <port>             # test if TCP port is reachable
nc -zvu <host> <port>            # test UDP port
lsof -i :<port>                  # which process owns a port (alternative to ss)
```

---

*Day 15 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
