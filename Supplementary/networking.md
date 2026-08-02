# Supplementary: Networking Fundamentals for Cybersecurity

You can't hack what you don't understand. Networking knowledge is the foundation of everything in cybersecurity — scanning, exploitation, pivoting, firewall evasion, and traffic analysis all require understanding how networks work at a fundamental level.

---

## The OSI Model — How Data Moves

Every piece of network communication travels through seven layers. Understanding these helps you know WHERE an attack happens and WHAT tools to use.

```
Layer 7: Application     HTTP, FTP, SSH, DNS, SMTP, SMB
                          ↑ This is where web attacks happen (SQLi, XSS)
                          
Layer 6: Presentation    SSL/TLS encryption, data formatting
                          ↑ This is where encryption/decryption happens
                          
Layer 5: Session         Session management, authentication
                          ↑ This is where sessions and cookies live
                          
Layer 4: Transport       TCP, UDP — ports, connections, reliability
                          ↑ This is where port scanning works
                          ↑ Firewalls often filter here (port-based rules)
                          
Layer 3: Network         IP addresses, routing (IPv4, IPv6, ICMP)
                          ↑ This is where IP routing, VPNs, and IP spoofing happen
                          ↑ Routers and iptables work here
                          
Layer 2: Data Link       MAC addresses, Ethernet, WiFi, ARP
                          ↑ This is where ARP spoofing and MAC flooding happen
                          ↑ Switches work here
                          
Layer 1: Physical        Cables, radio waves, fiber optics
                          ↑ Physical access attacks
```

### What matters for pentesting

```
Layer 7 (Application):  SQLi, XSS, command injection, LFI — most OSCP attacks
Layer 4 (Transport):    Port scanning, firewall evasion, service identification
Layer 3 (Network):      IP routing, pivoting, tunneling, iptables
Layer 2 (Data Link):    ARP spoofing, VLAN hopping (more advanced, less on OSCP)
```

---

## TCP vs UDP — When It Matters

### TCP (Transmission Control Protocol)

```
Reliable:     guarantees delivery (retransmits lost packets)
Ordered:      packets arrive in sequence
Connection:   three-way handshake before data flows
Overhead:     more headers, slower, but dependable

Common TCP services:
  22  SSH       80  HTTP      443 HTTPS     445 SMB
  21  FTP       25  SMTP      110 POP3      3389 RDP
  23  Telnet    143 IMAP      3306 MySQL    5985 WinRM
```

### UDP (User Datagram Protocol)

```
Unreliable:   no guarantee of delivery (fire and forget)
Unordered:    packets may arrive out of sequence
Connectionless: no handshake — just send
Fast:          minimal overhead

Common UDP services:
  53  DNS       67/68 DHCP    69  TFTP
  123 NTP       161  SNMP     500 IKE (VPN)
  1194 OpenVPN  5060 SIP
```

### Why this matters for scanning

```bash
# TCP scan — reliable, fast (SYN scan waits for SYN-ACK)
nmap -sS TARGET        # knows immediately: open, closed, or filtered

# UDP scan — slow, unreliable (no response could mean open OR filtered)
nmap -sU TARGET         # can't distinguish "open and silent" from "filtered"
                        # that's why it takes so much longer
```

---

## Subnetting — Understanding Network Boundaries

### CIDR notation

```
192.168.244.0/24

192.168.244.0  = the network address
/24            = the subnet mask (24 bits for network, 8 bits for hosts)
               = 255.255.255.0

Hosts available: 2^8 - 2 = 254 hosts (192.168.244.1 through 192.168.244.254)
.0 = network address (not usable)
.255 = broadcast address (not usable)
```

### Common subnet sizes

| CIDR | Subnet mask | Hosts | Use case |
|---|---|---|---|
| /8 | 255.0.0.0 | 16,777,214 | Class A (10.0.0.0/8) |
| /16 | 255.255.0.0 | 65,534 | Class B (172.16.0.0/16) |
| /24 | 255.255.255.0 | 254 | Class C — most common in labs and small networks |
| /25 | 255.255.255.128 | 126 | Half a /24 |
| /26 | 255.255.255.192 | 62 | Quarter of a /24 |
| /27 | 255.255.255.224 | 30 | Common for small server subnets |
| /28 | 255.255.255.240 | 14 | Small subnet |
| /30 | 255.255.255.252 | 2 | Point-to-point links (router ↔ router) |
| /32 | 255.255.255.255 | 1 | Single host (used in routing) |

### Private IP ranges (RFC 1918)

```
10.0.0.0/8       10.0.0.0    — 10.255.255.255      (Class A private)
172.16.0.0/12    172.16.0.0  — 172.31.255.255       (Class B private)
192.168.0.0/16   192.168.0.0 — 192.168.255.255      (Class C private)
```

These ranges are never routed on the internet. If you see them, you're on a private/internal network.

### Quick subnetting for pentesting

```
"What machines are on the same subnet as me?"

My IP:      192.168.244.129/24
Subnet:     192.168.244.0/24
Range:      192.168.244.1 — 192.168.244.254
Scan:       nmap -sn 192.168.244.0/24

My IP:      172.16.0.1/24
Subnet:     172.16.0.0/24
Range:      172.16.0.1 — 172.16.0.254
Scan:       nmap -sn 172.16.0.0/24

My IP:      10.10.14.5/23
Subnet:     10.10.14.0/23
Range:      10.10.14.1 — 10.10.15.254  (a /23 spans two class C ranges)
Scan:       nmap -sn 10.10.14.0/23
```

---

## Routing — How Packets Find Their Way

### How routing works

Every machine has a **routing table** — a list of rules that determines where to send each packet:

```bash
ip route

# Example output:
default via 192.168.244.2 dev ens33           # default gateway — send everything here
192.168.244.0/24 dev ens33 scope link          # local subnet — send directly (no gateway)
172.16.0.0/24 dev ens224 scope link            # second subnet — send directly via second NIC
```

**Reading the routing table:**

```
"default via 192.168.244.2"
  → If the destination doesn't match any other rule, send to 192.168.244.2
  → This is the default gateway (usually the router)

"192.168.244.0/24 dev ens33"
  → If the destination is in 192.168.244.0/24, send directly out ens33
  → No gateway needed — the destination is on the same network segment

"172.16.0.0/24 dev ens224"
  → If the destination is in 172.16.0.0/24, send directly out ens224
```

### Adding routes (for pivoting)

```bash
# "To reach 172.16.0.0/24, send traffic to CentOS (192.168.244.131)"
sudo ip route add 172.16.0.0/24 via 192.168.244.131

# Verify
ip route get 172.16.0.2
# Output: 172.16.0.2 via 192.168.244.131 dev ens33

# Remove a route
sudo ip route del 172.16.0.0/24 via 192.168.244.131
```

### Why routes matter for pentesting

```
Scenario: You compromised CentOS. It has two interfaces:
  ens33:  192.168.244.131 (your network)
  ens224: 172.16.0.1 (internal network)

Your Kali (192.168.244.129) can't reach 172.16.0.0/24 directly.
You need to either:
  1. Add a route: ip route add 172.16.0.0/24 via 192.168.244.131
     (CentOS needs IP forwarding + NAT for this to work)
  2. Use an SSH tunnel through CentOS (more reliable, encrypted)
  3. Use Chisel through CentOS (when SSH is blocked)
```

---

## DNS — How Names Become IPs

### How DNS resolution works

```
You type: google.com
1. Your computer checks its local cache
2. If not cached → asks the configured DNS server (e.g., 8.8.8.8)
3. DNS server checks: does it know google.com? 
   - If yes → returns the IP
   - If no → asks the root DNS servers → .com DNS → google.com DNS → gets the IP
4. DNS server returns the IP to you
5. Your computer connects to that IP
```

### DNS for pentesting

```bash
# Check DNS configuration
cat /etc/resolv.conf
# nameserver 192.168.244.2

# Look up an IP
nslookup domain.com
dig domain.com
host domain.com

# Reverse lookup (IP → hostname)
nslookup 192.168.244.140
dig -x 192.168.244.140

# Zone transfer (dump ALL records — if allowed)
dig axfr corp.local @192.168.244.140
# If this works, you get every hostname and IP in the domain
# Huge information disclosure — always try it

# DNS brute force subdomains
gobuster dns -d corp.local -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -r 192.168.244.140:53
```

### /etc/hosts file (local DNS override)

```bash
# Sometimes web applications respond differently based on the hostname
# Add the hostname to /etc/hosts to test
echo "192.168.244.132 corporate.local" | sudo tee -a /etc/hosts

# Now you can browse to: http://corporate.local
# And the request goes to 192.168.244.132 with the Host header set correctly
```

**When you need this on the OSCP:** If nmap shows a web server with a specific hostname (in the SSL cert or redirect), add it to `/etc/hosts`. Some web apps serve different content based on the hostname.

---

## NAT — Network Address Translation

### How NAT works

NAT translates private IP addresses to public ones (and back). Your home router does this — all your devices share one public IP.

```
Your home network:
  Laptop: 192.168.1.10 → Router → Internet (as 203.0.113.5)
  Phone:  192.168.1.11 → Router → Internet (as 203.0.113.5)
  
  Both appear as 203.0.113.5 to the outside world.
  The router tracks which internal device made each request.
```

### NAT types for pentesting

```
SNAT (Source NAT):
  Changes the SOURCE IP of outgoing packets
  Used for: letting private networks access the internet
  iptables: -j MASQUERADE or -j SNAT

DNAT (Destination NAT):
  Changes the DESTINATION IP of incoming packets
  Used for: port forwarding (redirect incoming traffic to an internal server)
  iptables: -j DNAT --to-destination INTERNAL_IP:PORT

PAT (Port Address Translation):
  Multiple internal IPs share one external IP, differentiated by port numbers
  This is what MASQUERADE does — it's the most common type of NAT
```

### NAT in pivoting

```bash
# On CentOS (the pivot) — masquerade traffic going to the internal network
sudo iptables -t nat -A POSTROUTING -s 192.168.244.0/24 -o ens224 -j MASQUERADE

# What this does:
# Traffic from 192.168.244.x going out ens224 (to 172.16.0.x)
# Gets its source IP rewritten to CentOS's ens224 IP (172.16.0.1)
# Ubuntu sees the traffic from 172.16.0.1 (allowed) not 192.168.244.x (blocked)
```

---

## ARP — How Devices Find Each Other on a LAN

### How ARP works

When your machine wants to send a packet to 192.168.244.132, it needs the **MAC address** (hardware address) of that machine. ARP (Address Resolution Protocol) resolves IPs to MACs.

```
1. Your machine broadcasts: "Who has 192.168.244.132? Tell 192.168.244.129"
2. 192.168.244.132 responds: "I have 192.168.244.132, my MAC is 00:0c:29:xx:xx:xx"
3. Your machine caches this mapping (ARP cache)
4. Now it can send packets directly to that MAC address
```

```bash
# View your ARP cache
arp -a
ip neigh

# Example output:
# 192.168.244.132 dev ens33 lladdr 00:0c:29:ab:cd:ef STALE
# 192.168.244.131 dev ens33 lladdr 00:0c:29:12:34:56 REACHABLE
```

### ARP for pentesting

```bash
# ARP scan (fast host discovery on local subnet)
arp-scan 192.168.244.0/24

# nmap ARP scan (most reliable on local networks)
nmap -sn -PR 192.168.244.0/24

# ARP is only for local subnets — it doesn't cross routers
# That's why pivoting is needed to reach other subnets
```

---

## Common Ports Quick Reference

```
Port    Service         What to do with it
20/21   FTP             Check anonymous login, upload if writable
22      SSH             Try credentials, check version for CVEs
23      Telnet          Cleartext — capture credentials, try default creds
25      SMTP            User enumeration (VRFY), check for open relay
53      DNS             Zone transfer (dig axfr), subdomain brute force
80      HTTP            Web enumeration, directory brute force, injection testing
88      Kerberos        Identifies a Domain Controller — AD attacks
110     POP3            Email access, credential brute force
111     RPCBind         Enumerate NFS and other RPC services
135     MSRPC           Windows RPC — rpcclient, rpcinfo
139     NetBIOS/SMB     File sharing, enumeration (enum4linux)
143     IMAP            Email access
161     SNMP (UDP)      Community string brute force → massive info dump
389     LDAP            AD enumeration (ldapsearch)
443     HTTPS           Same as 80 but encrypted — check SSL cert for hostnames
445     SMB             File shares, user enumeration, EternalBlue
636     LDAPS           Encrypted LDAP
993     IMAPS           Encrypted IMAP
995     POP3S           Encrypted POP3
1433    MSSQL           xp_cmdshell for RCE, credential brute force
1521    Oracle DB       TNS listener attacks
2049    NFS             Mount exports, check no_root_squash
3306    MySQL           Credential attacks, UDF exploitation, file read/write
3389    RDP             Brute force, BlueKeep (CVE-2019-0708)
5432    PostgreSQL      Credential attacks, COPY TO PROGRAM for RCE
5985    WinRM (HTTP)    evil-winrm with credentials or hash
5986    WinRM (HTTPS)   Same but encrypted
6379    Redis           Unauthenticated access, write SSH keys, webshell
8080    HTTP alt        Alternative web server, Tomcat, Jenkins
8443    HTTPS alt       Alternative HTTPS
8888    Various         Web admin panels, Jupyter notebooks
9090    Various         Web admin panels, Prometheus
27017   MongoDB         NoSQL injection, unauthenticated access
61616   ActiveMQ        CVE-2023-46604 deserialization RCE
```

---

## Networking Troubleshooting for Pentesting

When things don't connect, debug systematically:

```bash
# Can I reach the host at all?
ping TARGET

# Is the port open?
nc -zvn TARGET PORT
nmap -Pn -p PORT TARGET

# Is my routing correct?
ip route get TARGET_IP
traceroute TARGET

# Is a firewall blocking me?
# Check iptables on the target (if you have a shell):
sudo iptables -L -n -v

# Check firewall logs:
sudo grep "DROPPED\|BLOCKED" /var/log/syslog       # Debian/Ubuntu
sudo grep "DROPPED\|BLOCKED" /var/log/messages      # CentOS

# Is DNS resolving?
nslookup TARGET_HOSTNAME
dig TARGET_HOSTNAME

# Am I sending traffic? (watch with tcpdump)
sudo tcpdump -i eth0 host TARGET_IP

# Is the service actually listening? (from the target)
ss -tlnp | grep PORT
netstat -tlnp | grep PORT
```
