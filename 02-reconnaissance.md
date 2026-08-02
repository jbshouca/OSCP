# 02 — Reconnaissance and Scanning

Reconnaissance is the foundation of everything that follows. If you scan poorly, you miss ports. If you miss ports, you miss services. If you miss services, you miss the foothold. Every failed OSCP attempt has the same root cause: "I didn't enumerate enough." This module makes sure that doesn't happen to you.

---

## Understanding What Scanning Actually Does

When you run nmap against a target, you're sending specifically crafted network packets to each port and analyzing what comes back. Understanding this helps you choose the right scan type and interpret results correctly.

### TCP three-way handshake (review)

Every TCP connection starts with three packets:

```
Your machine → SYN        → Target      "I want to connect"
Your machine ← SYN-ACK    ← Target      "OK, I accept"
Your machine → ACK        → Target      "Great, we're connected"
```

nmap's default SYN scan (`-sS`) sends ONLY the first SYN packet and watches for the response:

```
SYN → port is OPEN       Target responds with SYN-ACK (something is listening)
SYN → port is CLOSED     Target responds with RST (port accessible, nothing listening)
SYN → port is FILTERED   No response at all (firewall silently dropped the packet)
```

The SYN scan never completes the handshake — it sends a RST after receiving the SYN-ACK. This is why it's called a "stealth" scan (though modern IDS/IPS detect it easily).

### TCP connect scan

The `-sT` scan completes the full three-way handshake. It's slower and noisier, but:
- Doesn't require root (SYN scan does)
- Works through SOCKS proxies (proxychains — this is important for pivoting)
- More reliable in some environments

```bash
# This is what you MUST use through proxychains
proxychains nmap -sT -Pn TARGET
```

### UDP scanning

UDP has no handshake — you send a packet and hope for a response:

```
Packet → port is OPEN        Service responds with data
Packet → port is CLOSED      Target responds with ICMP "port unreachable"
Packet → port is OPEN|FILTERED  No response (could be open but silent, or filtered)
```

UDP scanning is slow because there's no reliable way to tell "open and silent" from "filtered." But it's important — SNMP (161), DNS (53), TFTP (69), and NFS (111) use UDP and are common attack vectors.

---

## The OSCP Scanning Approach

### Step 1: Quick scan — get fast results

```bash
nmap -sV -sC -oN quick_scan.txt TARGET_IP
```

**What each flag does:**

| Flag | Full name | What it does | Why you need it |
|---|---|---|---|
| `-sV` | Service Version detection | Probes open ports to determine the exact software and version | Knowing "OpenSSH 7.2" vs just "SSH" tells you which exploits to look for |
| `-sC` | Script scanning (default scripts) | Runs a curated set of NSE scripts for common checks | Automatically checks for anonymous FTP, grabs HTTP titles, shows SSH hostkeys, enumerates SMB, etc. |
| `-oN` | Output Normal | Saves results to a text file | You WILL forget what you found. Always save scan results. |

This scan covers the top 1000 ports and finishes in 1-5 minutes.

### Step 2: Full port scan — catch everything

```bash
nmap -p- --min-rate=1000 -oN full_scan.txt TARGET_IP
```

| Flag | What it does | Why |
|---|---|---|
| `-p-` | Scan ALL 65,535 ports | The foothold might be on port 8443 or 50000. Default nmap only checks 1000 ports. |
| `--min-rate=1000` | Send at least 1000 packets per second | Speeds up the scan. Without this, a full scan can take 20+ minutes. |

Run this in the background while you start enumerating results from the quick scan.

### Step 3: Targeted deep scan on discovered ports

```bash
# From the full scan, you found ports 22, 80, 443, 8080, 9090
nmap -sV -sC -p 22,80,443,8080,9090 -oN targeted_scan.txt TARGET_IP
```

This gives you version/script results for ports the full scan found but the quick scan missed.

### Step 4: UDP scan

```bash
sudo nmap -sU --top-ports 20 -oN udp_scan.txt TARGET_IP
```

Only scan the top 20 UDP ports to keep it fast. The important ones:

```
53    DNS      — zone transfer might dump all domain records
67/68 DHCP     — rare, but information disclosure
69    TFTP     — unauthenticated file access
111   RPCBind  — enumerate NFS, other RPC services  
123   NTP      — rarely exploitable but can leak info
161   SNMP     — THIS IS THE BIG ONE — leaks everything if community string is found
500   IKE      — VPN enumeration
```

### Step 5: Vulnerability scan (if needed)

```bash
nmap --script vuln -p FOUND_PORTS -oN vuln_scan.txt TARGET_IP
```

Runs NSE vulnerability detection scripts. Can be noisy and slow but sometimes finds things you'd miss manually, especially for older services.

---

## Reading nmap Output — Understanding Every Line

Here's real nmap output and what each part means:

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-07-20 14:30 EDT
Nmap scan report for 192.168.244.132
Host is up (0.00045s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT     STATE    SERVICE  VERSION
21/tcp   open     ftp      vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r-- 1 0 0 187 Jul 20 10:00 readme.txt
22/tcp   open     ssh      OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey:
|   256 ab:cd:ef:12:34:56 (ECDSA)
|_  256 ab:cd:ef:12:34:57 (ED25519)
80/tcp   open     http     Apache httpd 2.4.57 ((Debian))
|_http-title: Company Web Portal
|_http-server-header: Apache/2.4.57 (Debian)
| http-robots-txt: 1 disallowed entry
|_/admin
3306/tcp filtered mysql
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

### Line-by-line breakdown

```
Host is up (0.00045s latency).
```
Target is alive. 0.00045 seconds = less than 1ms round trip. This is a local VM, which is why latency is so low. On an internet target, you'd see 20-200ms.

```
Not shown: 997 closed tcp ports (conn-refused)
```
nmap scanned 1000 ports (the default). 997 of them responded with RST (closed — accessible but nothing listening). 3 ports are open, shown below. 1 is filtered.

```
21/tcp   open     ftp      vsftpd 3.0.3
```
Port 21 is open, running FTP. The exact software is vsftpd version 3.0.3. **Immediately check:** `searchsploit vsftpd 3.0.3`.

```
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r-- 1 0 0 187 Jul 20 10:00 readme.txt
```
The `-sC` default scripts automatically checked for anonymous FTP login and it's **allowed**. There's a file called `readme.txt` available. **This is free information — download it immediately.**

```
80/tcp   open     http     Apache httpd 2.4.57 ((Debian))
|_http-title: Company Web Portal
| http-robots-txt: 1 disallowed entry
|_/admin
```
Port 80 is running Apache 2.4.57 on Debian. The page title is "Company Web Portal." The robots.txt file has a disallowed entry: `/admin`. **The developers don't want search engines to index /admin — that means something interesting is there. Check it immediately.**

```
3306/tcp filtered mysql
```
Port 3306 (MySQL) is **filtered** — a firewall is silently dropping packets to this port. You can't connect directly, but if you get a shell on this machine, you might be able to connect to MySQL locally (127.0.0.1:3306).

```
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
nmap thinks this is a Linux system based on service fingerprints. This tells you to think Linux when you get a shell — Linux privesc techniques, Linux file paths, Linux commands.

### What you should extract from this scan

```
IMMEDIATE ACTION ITEMS:
1. Download readme.txt from FTP (anonymous login works!)
2. Browse http://TARGET/admin (hidden from robots.txt)
3. Search: searchsploit vsftpd 3.0.3
4. Search: searchsploit apache 2.4.57
5. Search: searchsploit openssh 9.2
6. Full directory brute force on port 80
7. Note: MySQL is filtered — check locally if you get a shell
```

Every nmap scan should produce a list like this. If you scan and just think "okay, ports are open," you're not extracting enough value.

---

## Banner Grabbing with netcat

nmap's `-sV` does version detection automatically, but sometimes you want to interact with a service directly. netcat lets you connect to a port and see the raw banner — the text a service sends when you first connect.

```bash
# SSH banner
nc -vn 192.168.244.132 22
```

Output:
```
Connection to 192.168.244.132 22 port [tcp/*] succeeded!
SSH-2.0-OpenSSH_9.2p1 Debian-2+deb12u1
```

That string tells you: SSH protocol version 2.0, OpenSSH version 9.2p1, running on Debian. Search: `searchsploit openssh 9.2p1`.

```bash
# HTTP banner
nc -vn 192.168.244.132 80
```

Then type:
```
HEAD / HTTP/1.0
```
Press Enter twice. You'll get:

```
HTTP/1.1 200 OK
Date: Mon, 20 Jul 2026 18:30:01 GMT
Server: Apache/2.4.57 (Debian)
Content-Type: text/html
```

The `Server` header reveals Apache 2.4.57. Some servers hide this, some don't.

```bash
# FTP banner
nc -vn 192.168.244.132 21
```

Output:
```
220 (vsFTPd 3.0.3)
```

Direct version disclosure.

```bash
# SMTP banner
nc -vn 192.168.244.132 25
```

Output:
```
220 mail.company.local ESMTP Postfix
```

Reveals the mail server software AND the internal domain name (`company.local`).

### When to use nc vs nmap -sV

| Situation | Use |
|---|---|
| First scan of a target | nmap -sV (scans all ports at once) |
| nmap didn't identify the version | nc to manually grab the banner |
| You want to interact with the service | nc (type commands, see responses) |
| Through a proxychains tunnel | nc (proxychains nc -vn TARGET PORT) |
| Quick check of one specific port | nc -zvn TARGET PORT (faster than nmap for one port) |

---

## NSE Scripts — nmap's Built-in Enumeration

NSE (Nmap Scripting Engine) extends nmap from a port scanner to a full enumeration tool. There are over 600 scripts covering everything from vulnerability detection to brute forcing.

### Finding scripts

```bash
# List all scripts
ls /usr/share/nmap/scripts/

# Find scripts for a specific service
ls /usr/share/nmap/scripts/ | grep smb
ls /usr/share/nmap/scripts/ | grep http
ls /usr/share/nmap/scripts/ | grep ftp
ls /usr/share/nmap/scripts/ | grep ssh
ls /usr/share/nmap/scripts/ | grep vuln

# Read what a script does
nmap --script-help smb-enum-shares
```

### Running scripts

```bash
# Default scripts (safe, informative — already included in -sC)
nmap -sC TARGET

# All vulnerability scripts
nmap --script vuln TARGET

# Specific script
nmap --script smb-enum-shares -p 445 TARGET

# Multiple specific scripts
nmap --script smb-enum-shares,smb-enum-users,smb-os-discovery -p 445 TARGET

# All scripts for a service
nmap --script "smb-*" -p 445 TARGET

# Scripts with arguments
nmap --script smb-enum-shares --script-args smbusername=admin,smbpassword=pass -p 445 TARGET
```

### Most useful scripts for OSCP

**SMB:**
```bash
nmap --script smb-enum-shares -p 445 TARGET      # List shares
nmap --script smb-enum-users -p 445 TARGET       # List users
nmap --script smb-os-discovery -p 445 TARGET     # OS version
nmap --script smb-vuln* -p 445 TARGET            # Known vulnerabilities (EternalBlue, etc.)
nmap --script smb-brute -p 445 TARGET            # Brute force credentials
```

**HTTP:**
```bash
nmap --script http-enum -p 80 TARGET             # Enumerate common directories
nmap --script http-title -p 80 TARGET            # Page title
nmap --script http-headers -p 80 TARGET          # Response headers
nmap --script http-robots-txt -p 80 TARGET       # robots.txt contents
nmap --script http-shellshock -p 80 TARGET       # Test for Shellshock
```

**FTP:**
```bash
nmap --script ftp-anon -p 21 TARGET              # Anonymous login check
nmap --script ftp-brute -p 21 TARGET             # Brute force
nmap --script ftp-vuln* -p 21 TARGET             # Known vulnerabilities
```

**DNS:**
```bash
nmap --script dns-zone-transfer -p 53 TARGET     # Attempt zone transfer
nmap --script dns-brute TARGET                    # Brute force subdomains
```

**SNMP:**
```bash
nmap -sU --script snmp-brute -p 161 TARGET       # Brute force community strings
nmap -sU --script snmp-info -p 161 TARGET        # System information
nmap -sU --script snmp-processes -p 161 TARGET   # Running processes
```

---

## Port States — Understanding What nmap Tells You

| State | What happened | What it means | What to do |
|---|---|---|---|
| `open` | Received SYN-ACK | A service is listening and accepting connections | Enumerate this service immediately |
| `closed` | Received RST | Port is accessible but nothing is listening | Usually not useful, but confirms the host is up |
| `filtered` | No response | A firewall is silently dropping packets | Can't connect from here. Might be accessible from another machine (pivoting) or locally (after getting a shell). |
| `unfiltered` | Received ACK (ACK scan only) | Port is accessible but can't determine if open or closed | Rare — try a different scan type |
| `open\|filtered` | No response (UDP) | Either open and silent, or filtered | Try service-specific probes or move on |

### Important: filtered doesn't mean the port is closed

When you see `filtered`, a firewall is blocking YOUR access. But:
- The service might be accessible from another machine on the internal network
- The service might be accessible from localhost once you have a shell
- The firewall might only block certain source IPs

This is exactly the scenario in the labs — Ubuntu's SSH shows as `filtered` from Kali, but it's `open` from CentOS because CentOS is on the allowed subnet.

---

## Saving and Organizing Scan Results

### During the exam

Create a structured directory:

```bash
mkdir -p ~/oscp/exam/{target1,target2,target3,ad_set}
cd ~/oscp/exam

# Scan each target into its directory
nmap -sV -sC -oA target1/quick 192.168.X.1
nmap -p- --min-rate=1000 -oA target1/full 192.168.X.1
nmap -sV -sC -oA target2/quick 192.168.X.2
# etc.
```

The `-oA` flag creates three output files at once:
```
target1/quick.nmap     Normal text output (human readable)
target1/quick.xml      XML output (import into other tools)
target1/quick.gnmap    Greppable output (parse with grep/awk)
```

### Grep through greppable output

```bash
# Find all open ports across all targets
grep "open" ~/oscp/exam/*/quick.gnmap

# Find all HTTP services
grep "80/open" ~/oscp/exam/*/quick.gnmap

# Find all SSH services
grep "22/open" ~/oscp/exam/*/quick.gnmap

# Extract just IPs with a specific port open
grep "445/open" ~/oscp/exam/*/quick.gnmap | awk '{print $2}'
```

---

## Scan Speed and Timing

```bash
# Timing templates (T0=paranoid → T5=insane)
nmap -T4 TARGET                # Aggressive — good for local VMs and exams
nmap -T3 TARGET                # Normal (default)
nmap -T1 TARGET                # Sneaky — avoids IDS detection

# For the OSCP exam, use T4 — speed matters and stealth doesn't
```

| Template | Speed | Use case |
|---|---|---|
| `-T0` / `-T1` | Very slow | Real-world engagements where you need to avoid detection |
| `-T2` / `-T3` | Normal | General scanning |
| `-T4` | Fast | Lab environment, OSCP exam |
| `-T5` | Aggressive | Can miss results on slow/unstable networks |

For the exam, `-T4` is ideal. You're not trying to be stealthy — you're trying to be fast.

---

## Lab: Complete Scanning Practice

### Set up different services on your VMs

```bash
# On Debian — install multiple services
sudo apt install apache2 vsftpd openssh-server samba -y
sudo systemctl enable --now apache2 vsftpd ssh smbd

# On CentOS — install different services
sudo dnf install httpd openssh-server -y
sudo systemctl enable --now httpd sshd

# On Ubuntu — install more services
sudo apt install openssh-server mysql-server -y
sudo systemctl enable --now ssh mysql
```

### Practice the full scanning workflow from Kali

```bash
# Create your working directory
mkdir -p ~/oscp/lab_practice
cd ~/oscp/lab_practice

# Step 1: Host discovery
nmap -sn 192.168.244.0/24 | tee host_discovery.txt

# Step 2: Quick scan all targets
nmap -sV -sC -oA debian_quick 192.168.244.132
nmap -sV -sC -oA centos_quick 192.168.244.131
nmap -sV -sC -oA ubuntu_quick 192.168.244.130

# Step 3: Full port scan (run in background)
nmap -p- --min-rate=1000 -oA debian_full 192.168.244.132 &
nmap -p- --min-rate=1000 -oA centos_full 192.168.244.131 &
nmap -p- --min-rate=1000 -oA ubuntu_full 192.168.244.130 &

# Step 4: UDP scan
sudo nmap -sU --top-ports 20 -oA debian_udp 192.168.244.132
sudo nmap -sU --top-ports 20 -oA centos_udp 192.168.244.131

# Step 5: Banner grab interesting services
nc -vn 192.168.244.132 22
nc -vn 192.168.244.132 21
nc -vn 192.168.244.131 22

# Step 6: Search for exploits for every version you found
searchsploit vsftpd
searchsploit apache 2.4
searchsploit openssh 9.2

# Step 7: Run vulnerability scripts
nmap --script vuln 192.168.244.132
nmap --script smb-vuln* -p 445 192.168.244.132
```

### Practice exercise

After scanning, write up your findings as if this were an exam:

```
## Target: 192.168.244.132

### Open Ports
- 21/tcp — FTP (vsftpd 3.0.3) — anonymous login ALLOWED
- 22/tcp — SSH (OpenSSH 9.2p1)
- 80/tcp — HTTP (Apache 2.4.57) — robots.txt has /admin
- 139/tcp — SMB
- 445/tcp — SMB

### Immediate Actions
1. Download files from FTP
2. Check /admin on web server
3. Enumerate SMB shares
4. searchsploit every version

### Searchsploit Results
- vsftpd 3.0.3: No known exploits
- Apache 2.4.57: No critical exploits
- OpenSSH 9.2: No critical exploits
- SMB: Need version — check enum4linux
```

This is exactly what your exam notes should look like after the scanning phase. Clear, organized, and actionable.

---

## Exam Tips for Scanning

1. **Start ALL scans immediately.** Don't scan one machine at a time. Start quick scans on every target in the first 5 minutes. Start full scans in the background. Maximize your time.

2. **Always do full port scans.** The number one mistake on the exam is missing a service running on a high port that the default scan doesn't cover.

3. **Save everything.** Use `-oA` to create all output formats. You'll need the normal output for your report and the greppable output for quick searches.

4. **Read the script output.** When `-sC` runs, it produces detailed output below each port. Don't just look at the port list — read the script results. Anonymous FTP, robots.txt entries, HTTP titles — all found automatically.

5. **UDP matters.** SNMP on UDP 161 can dump entire system configurations, running processes, and installed software. 15 minutes of UDP scanning can save you hours of manual enumeration.

6. **Come back to scanning.** When you're stuck on a machine, come back and re-scan. Try different script categories, different wordlists, or scan ports you initially ignored.
