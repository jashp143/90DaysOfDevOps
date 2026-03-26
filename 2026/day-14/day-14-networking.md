# Day 14 – Networking Fundamentals & Hands-on Checks

> **Environment:** Ubuntu 24.04 container, IP `21.0.0.154/31`, gateway `21.0.0.155`.
> External ICMP and DNS (UDP 53) are blocked by the container's network policy — this itself
> is a useful observation (firewalls exist!). HTTP/S to allowed domains works fine.
> Where a command couldn't reach the internet, the realistic output from a standard Linux
> host is shown alongside the actual container output.

---

## Concept Map: OSI vs TCP/IP

```
OSI Model (7 layers)          TCP/IP Stack (4 layers)     Protocols at each layer
─────────────────────────────────────────────────────────────────────────────────
L7  Application               Application                  HTTP, HTTPS, DNS, SSH, FTP
L6  Presentation                  │                        TLS/SSL (lives here logically)
L5  Session                       │                        (absorbed into Application)
─────────────────────────────────────────────────────────────────────────────────
L4  Transport                 Transport                    TCP (reliable), UDP (fast)
─────────────────────────────────────────────────────────────────────────────────
L3  Network                   Internet                     IP, ICMP, ARP
─────────────────────────────────────────────────────────────────────────────────
L2  Data Link                 Link (Network Access)        Ethernet, Wi-Fi (MAC frames)
L1  Physical                      │                        Cables, radio signals, NIC
─────────────────────────────────────────────────────────────────────────────────
```

**Key points:**
- OSI is the reference model (7 layers, great for diagnosing *where* a problem lives).
  TCP/IP is what the internet actually implements (4 layers, pragmatic).
- The TCP/IP Application layer absorbs OSI L5–L7; the Link layer absorbs OSI L1–L2.

**Where protocols sit:**

| Protocol | OSI Layer | TCP/IP Layer | Notes |
|---|---|---|---|
| HTTP / HTTPS | L7 | Application | HTTPS = HTTP inside TLS |
| DNS | L7 | Application | UDP 53 (queries), TCP 53 (zone transfers) |
| TLS / SSL | L6 | Application | Encryption before HTTP data |
| TCP | L4 | Transport | Connection-oriented, guaranteed delivery |
| UDP | L4 | Transport | Connectionless, low latency, no guarantee |
| IP | L3 | Internet | Addressing and routing |
| ICMP | L3 | Internet | `ping` and `traceroute` use this |
| Ethernet | L2 | Link | MAC-level framing on LAN |

**Real example — `curl https://example.com`:**
```
L7 (Application) → HTTP GET request assembled
L6 (Presentation) → TLS handshake encrypts the payload
L4 (Transport)   → TCP segment with SYN/ACK handshake to port 443
L3 (Internet)    → IP packet routed toward example.com's IP
L2 (Link)        → Ethernet frame sent to the default gateway's MAC
L1 (Physical)    → Bits leave the NIC
```
One `curl` call traverses all 7 OSI layers on the way out, and all 7 in reverse on the way back.

---

## Hands-on Checks

### Identity — `hostname -I` / `ip addr show`

```bash
$ hostname -I
21.0.0.154

$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65522
    inet 127.0.0.1/8 scope global dynamic
2: 7b482401cb-v: <UP,LOWER_UP> mtu 1500
    inet 21.0.0.154/31 scope global dynamic
```

**Observation:** `/31` subnet means only two usable addresses: `.154` (this host) and `.155`
(the gateway). This is a common point-to-point subnet used in container/cloud networking.
Two interfaces: `lo` (loopback, always `127.0.0.1`) and the container veth (`21.0.0.154`).

---

### Reachability — `ping`

```bash
# Container result (ICMP outbound blocked by network policy)
$ ping -c 4 google.com
ping: google.com: Temporary failure in name resolution

# What you'd see on a real EC2 / VM:
$ ping -c 4 google.com
PING google.com (142.250.80.46) 56(84) bytes of data.
64 bytes from 142.250.80.46: icmp_seq=1 ttl=116 time=1.23 ms
64 bytes from 142.250.80.46: icmp_seq=2 ttl=116 time=1.19 ms
64 bytes from 142.250.80.46: icmp_seq=3 ttl=116 time=1.22 ms
64 bytes from 142.250.80.46: icmp_seq=4 ttl=116 time=1.25 ms
--- google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 1.19/1.22/1.25/0.02 ms
```

**Observation:** `ping` uses ICMP (L3). The container blocks ICMP outbound — a reminder that
"no ping response" ≠ "host is down". Many production servers and cloud firewalls drop ICMP
by default. If ping fails, always try `curl` or `nc` on a known TCP port before concluding
the host is unreachable.

---

### Path — `traceroute`

```bash
# Container result (ICMP probes blocked):
$ traceroute google.com
google.com: Temporary failure in name resolution

# What you'd see on a real host:
$ traceroute google.com
traceroute to google.com (142.250.80.46), 30 hops max
 1  10.0.0.1        (gateway)     0.4 ms   0.3 ms   0.4 ms
 2  100.64.0.1      (ISP edge)    2.1 ms   2.0 ms   2.2 ms
 3  * * *           (hop filtered ICMP)
 4  72.14.215.165   (Google backbone) 3.5 ms  3.4 ms  3.5 ms
 5  142.250.80.46   (google.com)  4.2 ms   4.0 ms   4.1 ms
```

**Observation:** `traceroute` sends UDP/ICMP probes with incrementing TTL values.
Each router decrements TTL and sends back an ICMP Time Exceeded when TTL hits 0 — that's
how you map the route. `* * *` hops mean that router silently drops ICMP (normal in production).
A suddenly high-latency hop (e.g., 200ms jump) usually points to a long-haul fibre link or
congested peering point.

---

### Ports — `ss -tulpn`

```bash
$ ss -tulpn
Netid  State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
tcp    LISTEN  0       0       0.0.0.0:2024         0.0.0.0:*          users:(("process_api",pid=1,fd=9))
```

**Observation:** One listening service: `process_api` on TCP port **2024**, bound to all
interfaces (`0.0.0.0`). This is the container's internal API daemon. `ss` flags explained:

| Flag | Meaning |
|---|---|
| `-t` | TCP sockets |
| `-u` | UDP sockets |
| `-l` | Listening sockets only |
| `-p` | Show process name/PID |
| `-n` | Show numeric ports (don't resolve names) |

---

### Name Resolution — `dig` / `nslookup`

```bash
# Container has no DNS resolver running (DNS queries blocked)
# Resolution here is via /etc/hosts static entries:
$ cat /etc/hosts | grep anthropic
160.79.104.10 api.anthropic.com

# What dig looks like on a real host with 8.8.8.8 resolver:
$ dig google.com
; <<>> DiG 9.18.x <<>> google.com
;; QUESTION SECTION:
;google.com.            IN  A

;; ANSWER SECTION:
google.com.     299 IN  A   142.250.80.46

;; Query time: 3 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; MSG SIZE  rcvd: 55
```

**Observation:** `dig` outputs three key sections — QUESTION (what was asked), ANSWER (the
resolved IPs), and stats (query time, which resolver was used). The `299` is the TTL in
seconds — after that, the cache expires and the resolver re-queries authoritative DNS.
When DNS fails: check `/etc/resolv.conf` for nameserver config, and test with
`dig @8.8.8.8 <domain>` to bypass local resolver and query Google's DNS directly.

---

### HTTP Check — `curl -I`

```bash
$ curl -I https://api.anthropic.com
HTTP/1.1 200 OK
date: Thu, 26 Mar 2026 08:58:29 GMT
server: envoy

HTTP/2 404
date: Thu, 26 Mar 2026 08:58:29 GMT
server: cloudflare
cf-cache-status: DYNAMIC
```

**Observation:** Two HTTP responses! The first `200 OK` is the internal Envoy proxy
(L7 load balancer) accepting the connection. The second `HTTP/2 404` is Cloudflare's
edge returning "not found" for the bare domain `/` — expected, since the API requires
a specific endpoint path. Key header to note: `server: cloudflare` tells you there's
a CDN/proxy in front — the response headers aren't from the origin server directly.

---

### Connections Snapshot — `netstat -an`

```bash
$ netstat -an
Proto  Recv-Q  Send-Q  Local Address       Foreign Address     State
tcp    0       0       21.0.0.154:2024     10.4.85.194:43602   ESTABLISHED
tcp    0       0       0.0.0.0:2024        0.0.0.0:*           LISTEN

$ netstat -an | grep -c ESTABLISHED
1
$ netstat -an | grep -c LISTEN
1
```

**Observation:** 1 ESTABLISHED connection (the live session from the orchestrator to
this container on port 2024) and 1 LISTEN entry. On a busy web server you'd see hundreds
of ESTABLISHED connections to port 80/443. If you see many connections in `TIME_WAIT`
state, the server recently closed a lot of connections — normal after traffic bursts.

---

## Mini Task: Port Probe & Interpret

**Target:** `localhost:2024` (the `process_api` service found with `ss -tulpn`)

```bash
$ nc -zv localhost 2024
Connection to localhost (127.0.0.1) 2024 port [tcp/*] succeeded!
```

**Result: Reachable.** The TCP handshake completed — the port is open and the process is
accepting connections. If this had failed, next checks would be:

1. `ps aux | grep process_api` — is the process actually running?
2. `systemctl status process_api` — did the service crash? Check logs.
3. `iptables -L -n` — is a local firewall rule blocking the port?
4. `ss -tulpn | grep 2024` — did the port binding change?

---

## Reflection

**Which command gives the fastest signal when something is broken?**

`ping` for L3 reachability and `curl -I` for L7 application health. Ping tells you in <1s
whether the IP is routable. Curl tells you whether the full stack — TCP, TLS, HTTP — is
working end to end. Between the two you can instantly narrow the problem to "routing/firewall"
vs "application/config".

**What layer would you inspect next if…**

| Failure | Layer to Inspect | Immediate Commands |
|---|---|---|
| DNS fails | L7 (Application) → L3 (Internet) | `dig @8.8.8.8 domain` (bypass local resolver); check `/etc/resolv.conf`; verify port 53 not firewalled |
| HTTP 500 shows up | L7 (Application) | Check app logs (`journalctl -u appname`); look at backend error rate; check DB connectivity |
| Can ping but can't curl | L4 (Transport) | `nc -zv host 80`; check firewall rules; verify service is listening |
| Can't ping at all | L3 (Network) | `ip route show`; check security groups; `traceroute` to find where packets stop |

**Two follow-up checks in a real incident:**

1. **`curl -v` (verbose)** — shows the full TLS handshake, DNS resolution, TCP connect time,
   and exact HTTP headers. When `curl -I` gives a bad status, `-v` shows exactly *where*
   the connection broke (DNS? TCP? TLS? HTTP?) with timestamps.

2. **`ss -s` (socket statistics summary)** — gives a quick count of TCP states system-wide
   (established, time_wait, close_wait, etc.). A spike in `close_wait` means the application
   isn't closing connections — likely a connection leak. A spike in `time_wait` is usually
   harmless (normal TCP teardown), but can indicate a traffic surge.

---

## Commands Reference

```bash
# Identity
hostname -I                        # quick IP list
ip addr show                       # full interface details with subnet masks
ip route show                      # routing table, find default gateway

# Reachability
ping -c 4 <host>                   # L3 check, 4 packets
ping -c 4 -I eth0 <host>           # force specific interface

# Path tracing
traceroute <host>                  # UDP probes (default)
traceroute -T -p 80 <host>         # TCP SYN probes (bypasses ICMP blocks)
tracepath <host>                   # no root required, simpler output

# Port inspection
ss -tulpn                          # listening sockets with process names
ss -s                              # socket summary (count per state)
netstat -an | grep ESTABLISHED     # all active connections

# DNS
dig <domain>                       # full DNS query output
dig @8.8.8.8 <domain>             # query Google DNS directly
dig <domain> +short                # just the IP
nslookup <domain>                  # simpler interactive lookup

# HTTP
curl -I <url>                      # headers only (HEAD request)
curl -v <url>                      # verbose: shows DNS + TCP + TLS + HTTP
curl -o /dev/null -w "%{http_code} %{time_total}s" <url>  # status + timing

# Port probe
nc -zv <host> <port>               # TCP connect test
nc -zvu <host> <port>              # UDP connect test
```

---

*Day 14 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
