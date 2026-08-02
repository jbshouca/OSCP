# Supplementary: Core Cybersecurity Concepts

This covers foundational security knowledge every cybersecurity professional needs — regardless of whether you're doing offensive security, defensive security, GRC, or engineering. These topics come up in interviews, on the job, and in advanced certifications.

---

## Security Frameworks and Compliance

### NIST Cybersecurity Framework (CSF)

The most widely adopted framework in the US. Organizes security into five functions:

```
IDENTIFY     — know what you have (asset inventory, risk assessment)
PROTECT      — safeguards to limit impact (access control, encryption, training)
DETECT       — find events when they happen (monitoring, logging, IDS)
RESPOND      — take action on detected events (IR plan, communication, mitigation)
RECOVER      — restore services after an incident (backup, DR plan, lessons learned)
```

**Why you should know it:** Almost every organization maps their security program to NIST CSF. When someone asks "what's your security posture?" they're usually asking about these five functions.

### CIS Controls (Center for Internet Security)

18 prioritized security controls ranked by impact. The first six give you the most bang for your buck:

```
1.  Inventory and Control of Enterprise Assets
2.  Inventory and Control of Software Assets
3.  Data Protection
4.  Secure Configuration of Enterprise Assets and Software
5.  Account Management
6.  Access Control Management
7.  Continuous Vulnerability Management
8.  Audit Log Management
9.  Email and Web Browser Protections
10. Malware Defenses
11. Data Recovery
12. Network Infrastructure Management
13. Network Monitoring and Defense
14. Security Awareness and Skills Training
15. Service Provider Management
16. Application Software Security
17. Incident Response Management
18. Penetration Testing
```

**Control 18 is your job as a pentester.** But understanding all 18 helps you write better reports — you can map your findings to the control that failed.

### ISO 27001

International standard for Information Security Management Systems (ISMS). Organizations get **certified** against ISO 27001. Key concepts:

```
- Risk assessment and treatment
- Statement of Applicability (which controls apply)
- Continuous improvement cycle (Plan-Do-Check-Act)
- Annex A controls (114 controls across 14 domains)
- Regular audits (internal + external)
```

### SOC 2

Service Organization Controls — proves a company handles customer data securely. Based on five Trust Service Criteria:

```
Security         — protection against unauthorized access
Availability     — system is available for operation and use
Processing Integrity — processing is complete, valid, accurate, timely
Confidentiality  — information designated as confidential is protected
Privacy          — personal information is collected, used, retained, and disclosed properly
```

**Type I:** Controls are designed properly (point-in-time assessment)
**Type II:** Controls are operating effectively (over a period, usually 6-12 months)

### PCI DSS (Payment Card Industry Data Security Standard)

Required if you handle credit card data. 12 requirements:

```
1.  Install and maintain network security controls
2.  Apply secure configurations to all system components
3.  Protect stored account data
4.  Protect cardholder data with strong cryptography during transmission
5.  Protect all systems and networks from malicious software
6.  Develop and maintain secure systems and software
7.  Restrict access to system components and cardholder data by business need to know
8.  Identify users and authenticate access to system components
9.  Restrict physical access to cardholder data
10. Log and monitor all access to system components and cardholder data
11. Test security of systems and networks regularly
12. Support information security with organizational policies and programs
```

---

## Vulnerability Management

### The vulnerability management lifecycle

```
1. DISCOVER     — scan for vulnerabilities (Nessus, Qualys, OpenVAS)
2. PRIORITIZE   — rank by severity, exploitability, asset value
3. REMEDIATE    — patch, mitigate, or accept the risk
4. VERIFY       — rescan to confirm the fix worked
5. REPORT       — document what was found and what was done
6. REPEAT       — continuous cycle, not a one-time event
```

### CVSS (Common Vulnerability Scoring System)

Every CVE gets a CVSS score from 0.0 to 10.0:

```
0.0         None
0.1 - 3.9   Low
4.0 - 6.9   Medium
7.0 - 8.9   High
9.0 - 10.0  Critical
```

**CVSS metrics that matter:**

```
Attack Vector (AV):
  Network (N)      — exploitable remotely over the network
  Adjacent (A)     — requires same network segment
  Local (L)        — requires local access
  Physical (P)     — requires physical access

Attack Complexity (AC):
  Low (L)          — no special conditions needed
  High (H)         — requires specific conditions (race condition, MITM position)

Privileges Required (PR):
  None (N)          — no authentication needed (worst case)
  Low (L)           — requires regular user credentials
  High (H)          — requires admin credentials

User Interaction (UI):
  None (N)          — no user action needed
  Required (R)      — victim must click a link, open a file, etc.
```

A vulnerability with AV:N/AC:L/PR:N/UI:N is the worst case — remotely exploitable, easy, no auth, no interaction. That's a CVSS 9+.

### CVE (Common Vulnerabilities and Exposures)

```
Format: CVE-YEAR-NUMBER
Example: CVE-2021-44228 (Log4Shell)

Where to look them up:
  - NVD (National Vulnerability Database): nvd.nist.gov
  - MITRE CVE: cve.mitre.org
  - ExploitDB: exploit-db.com
  - GitHub Advisory Database: github.com/advisories
```

---

## Incident Response

### The incident response lifecycle (NIST SP 800-61)

```
1. PREPARATION
   - IR plan documented and tested
   - IR team identified with roles and responsibilities
   - Communication plan (who to notify, when, how)
   - Tools and runbooks ready
   - Backups verified

2. DETECTION AND ANALYSIS
   - Monitor alerts (SIEM, IDS/IPS, EDR)
   - Triage: is this a real incident or a false positive?
   - Determine scope: how many systems are affected?
   - Classify severity: P1 (critical) through P4 (low)
   - Document everything — timeline, artifacts, decisions

3. CONTAINMENT, ERADICATION, REMEDIATION
   - Short-term containment: isolate affected systems
   - Evidence preservation: forensic images before changes
   - Eradication: remove malware, close attack vector
   - Remediation: patch the vulnerability, reset credentials
   - Long-term containment: implement additional controls

4. POST-INCIDENT ACTIVITY
   - Lessons learned meeting (within 1-2 weeks)
   - Update IR plan based on findings
   - Improve detection rules
   - Document the incident fully
   - Share indicators of compromise (IOCs) with the community
```

### Common incident types

```
Malware/Ransomware:
  - Isolate the machine immediately (pull network cable, not power)
  - Don't pay the ransom (no guarantee of data return)
  - Check backup integrity
  - Preserve evidence for forensics

Phishing:
  - Identify all recipients
  - Check who clicked / entered credentials
  - Reset affected credentials
  - Block the phishing domain/IP
  - Quarantine the email from all mailboxes

Data Breach:
  - Determine what data was exposed
  - Legal/compliance notification requirements (GDPR: 72 hours)
  - Notify affected individuals
  - Engage legal counsel
  - Preserve evidence

Insider Threat:
  - Disable the account immediately
  - Preserve audit logs
  - Engage HR and legal
  - Review all access the individual had
  - Check for data exfiltration
```

---

## SIEM and Log Analysis

### What a SIEM does

Security Information and Event Management — collects logs from everything (servers, firewalls, endpoints, applications), correlates them, and generates alerts.

```
Sources → Collection → Normalization → Correlation → Alerting → Investigation

Sources: Windows Event Logs, syslog, firewall logs, proxy logs, 
         cloud audit logs, EDR alerts, DNS logs, email logs
```

### Key log sources and what to look for

**Windows Event Logs:**
```
Security Log:
  4624  — Successful logon
  4625  — Failed logon (brute force detection)
  4634  — Logoff
  4648  — Logon using explicit credentials (PtH, runas)
  4672  — Special privileges assigned (admin logon)
  4688  — New process created (with command line if auditing enabled)
  4720  — User account created
  4732  — Member added to security group
  7045  — New service installed (persistence, PsExec)

System Log:
  7034  — Service crashed unexpectedly
  7036  — Service started or stopped
  1074  — System shutdown/restart
```

**Linux syslog:**
```
/var/log/auth.log    — authentication events (SSH logins, sudo usage)
/var/log/syslog      — general system messages
/var/log/kern.log    — kernel messages
/var/log/apache2/    — web server access and error logs
/var/log/cron        — cron job execution
```

**Firewall logs:**
```
Blocked connections → potential scanning or exploitation attempts
Allowed connections to unusual ports → potential C2 traffic
Large outbound transfers → potential data exfiltration
Connections to known bad IPs → compromise indicators
```

### Common detection rules (what SOC analysts watch for)

```
Brute force:       >5 failed logins from same source in 5 minutes (Event 4625)
Password spray:    1 failed login against >10 users in 1 minute
Lateral movement:  Successful SMB auth to >3 machines in 5 minutes
Privilege escalation: 4672 from a non-admin account
Persistence:       7045 (new service) outside change windows
Data exfiltration: >100MB outbound to a single external IP
C2 beaconing:      Regular interval connections to the same external IP
Credential dump:   lsass.exe accessed by unusual process
```

---

## Network Security

### Defense in depth

```
PERIMETER:
  - Firewall (block unauthorized inbound/outbound traffic)
  - IDS/IPS (detect/prevent known attack patterns)
  - WAF (protect web applications)
  - DDoS protection (absorb volumetric attacks)

NETWORK:
  - Segmentation (separate critical systems from general network)
  - VLANs (logical network separation)
  - ACLs (access control lists on routers/switches)
  - 802.1X (network access control — authenticate before network access)
  - Network monitoring (NetFlow, packet capture)

ENDPOINT:
  - EDR (Endpoint Detection and Response)
  - Antivirus/anti-malware
  - Host-based firewall
  - Application whitelisting
  - Disk encryption (BitLocker, FileVault, LUKS)
  - Patch management

IDENTITY:
  - Multi-factor authentication (MFA)
  - Privileged Access Management (PAM)
  - Identity governance
  - Single Sign-On (SSO)
  - Conditional access policies

DATA:
  - Encryption at rest and in transit
  - Data Loss Prevention (DLP)
  - Classification and labeling
  - Backup and recovery
  - Access controls (least privilege)

APPLICATION:
  - Secure development lifecycle (SDLC)
  - Code review and SAST/DAST
  - Dependency scanning
  - API security
  - Input validation
```

### IDS vs IPS

```
IDS (Intrusion Detection System):
  - MONITORS traffic and generates alerts
  - Does NOT block — just watches and reports
  - Positioned on a span/mirror port (copies of traffic)
  - Example: Snort in IDS mode, Suricata, OSSEC

IPS (Intrusion Prevention System):
  - BLOCKS traffic that matches attack signatures
  - Sits INLINE (traffic passes through it)
  - Can drop packets, reset connections, block IPs
  - Example: Snort in IPS mode, Suricata, Palo Alto

Detection methods:
  Signature-based  — matches known attack patterns (fast, misses new attacks)
  Anomaly-based    — detects deviation from normal (catches new attacks, more false positives)
  Behavioral       — monitors for suspicious behavior patterns
```

### VPN types

```
Site-to-site VPN:
  - Connects two networks (office A ↔ office B)
  - Always on
  - Implemented on firewalls/routers

Remote access VPN:
  - Individual users connect to the corporate network
  - On-demand
  - SSL/TLS VPN (OpenConnect, Cisco AnyConnect) or IPsec (IKEv2)

Protocols:
  IPsec     — network layer, strong encryption, complex setup
  OpenVPN   — application layer, TLS-based, flexible
  WireGuard — modern, fast, simple, smaller codebase
  SSL/TLS   — browser-based or thick client
```

---

## Threat Intelligence

### Threat frameworks

**MITRE ATT&CK:**
The most important framework for understanding how attacks work. Maps real-world attacker techniques into a matrix:

```
Tactic (WHY)          → Technique (HOW)           → Procedure (SPECIFIC EXAMPLE)

Initial Access        → Phishing                  → Spear phishing with macro-enabled document
Execution             → Command Line Interface     → PowerShell execution of encoded payload
Persistence           → Registry Run Keys          → HKLM\...\Run with malware path
Privilege Escalation  → Token Impersonation        → PrintSpoofer for SeImpersonate
Defense Evasion       → Obfuscated Files           → Base64-encoded PowerShell
Credential Access     → OS Credential Dumping      → Mimikatz sekurlsa::logonpasswords
Discovery             → Network Service Scanning   → nmap scan of internal network
Lateral Movement      → Pass the Hash              → impacket-psexec with NTLM hash
Collection            → Data from Local System     → Copying sensitive documents
Exfiltration          → Exfiltration Over C2       → Data sent through Cobalt Strike beacon
Impact                → Data Encrypted for Impact  → Ransomware encryption
```

**Use ATT&CK in your pentest reports:** Map each finding to an ATT&CK technique. "The attacker used T1053.005 (Scheduled Task/Job) for persistence." This gives defenders a shared vocabulary.

**Cyber Kill Chain (Lockheed Martin):**

```
1. Reconnaissance    — research the target
2. Weaponization     — create the exploit/payload
3. Delivery          — send the payload (email, web, USB)
4. Exploitation      — trigger the vulnerability
5. Installation      — install persistence mechanism
6. C2                — establish command and control channel
7. Actions on Obj.   — achieve the goal (steal data, deploy ransomware)
```

### Indicators of Compromise (IOCs)

```
Types:
  IP addresses       — C2 server IPs, known malicious hosts
  Domain names       — Phishing domains, C2 domains
  File hashes        — MD5/SHA256 of malware samples
  URLs               — Malicious download links
  Email addresses    — Phishing sender addresses
  Registry keys      — Persistence mechanisms
  File paths         — Known malware locations
  Mutex names        — Malware identifier strings
  YARA rules         — Pattern-matching rules for malware detection

Sources:
  - VirusTotal (virustotal.com) — check file hashes and URLs
  - AlienVault OTX (otx.alienvault.com) — threat intelligence platform
  - MISP (open-source threat intelligence platform)
  - AbuseIPDB — check if an IP is reported as malicious
  - Shodan — check if an IP has exposed services
  - GreyNoise — check if an IP is a mass scanner
```

---

## Digital Forensics Basics

### Order of volatility (what to collect first)

```
Most volatile (collect first):
  1. CPU registers, cache
  2. RAM (memory dump)
  3. Network connections
  4. Running processes
  5. Disk cache
  6. Disk (filesystem, swap)
  7. Remote logging
  8. Physical media (USB, backup tapes)
Least volatile (collect last)
```

Volatile data disappears when you power off the machine. Memory contains passwords, encryption keys, running processes, and network connections that are gone after a reboot.

### Chain of custody

```
- Document who collected what, when, and how
- Use write-blockers when imaging disks
- Hash all evidence (MD5 + SHA256) before and after analysis
- Store evidence securely with access controls
- Any break in chain of custody can make evidence inadmissible
```

### Common forensics tools

```
Disk imaging:    dd, dc3dd, FTK Imager, Guymager
Memory analysis: Volatility (Python), Rekall
Disk analysis:   Autopsy, Sleuth Kit, FTK
Network:         Wireshark, tcpdump, NetworkMiner
Timeline:        log2timeline/plaso
Registry:        Registry Explorer, RegRipper
Email:           EML/MSG viewers, MailXaminer
Mobile:          Cellebrite, UFED, Oxygen Forensics
```

---

## Secure Software Development

### OWASP Top 10 (2021 — know all ten)

```
A01: Broken Access Control        — users access unauthorized functions/data
A02: Cryptographic Failures       — weak encryption, exposed sensitive data
A03: Injection                    — SQLi, command injection, LDAP injection
A04: Insecure Design              — missing security controls in architecture
A05: Security Misconfiguration    — default configs, unnecessary features enabled
A06: Vulnerable Components        — outdated libraries with known CVEs
A07: Authentication Failures      — weak passwords, no MFA, session flaws
A08: Data Integrity Failures      — insecure deserialization, untrusted CI/CD
A09: Logging & Monitoring Failures — no audit trail, no alerting
A10: SSRF                         — server makes requests to attacker-controlled URLs
```

### Secure development practices

```
SAST (Static Application Security Testing):
  - Scan source code for vulnerabilities without running it
  - Tools: SonarQube, Checkmarx, Semgrep
  - Find: SQLi, XSS, hardcoded credentials, buffer overflows

DAST (Dynamic Application Security Testing):
  - Test the running application from the outside
  - Tools: OWASP ZAP, Burp Suite, Nikto
  - Find: injection, authentication flaws, misconfigurations

SCA (Software Composition Analysis):
  - Scan dependencies for known vulnerabilities
  - Tools: Snyk, Dependabot, npm audit
  - Find: vulnerable library versions (Log4j, etc.)

Secrets scanning:
  - Detect credentials committed to source code
  - Tools: git-secrets, TruffleHog, GitLeaks
  - Find: API keys, passwords, tokens in git history
```

---

## Physical Security

### Why pentesters should understand it

Physical access bypasses almost every digital control. If an attacker can walk into your server room, game over. Real-world penetration tests often include physical security assessments.

```
Attack techniques:
  Tailgating        — follow an employee through a badge-controlled door
  Social engineering — pretend to be a delivery driver, IT support, new employee
  Lock picking       — bypass physical locks
  Badge cloning      — copy RFID badge with a Flipper Zero or Proxmark
  Dumpster diving    — find documents, hardware, credentials in the trash
  USB drop           — leave infected USB drives in the parking lot
  Shoulder surfing   — watch someone type their password

Defense:
  Mantrap/vestibule  — two doors, second won't open until first closes
  Security guards    — human verification
  Cameras (CCTV)     — monitoring and deterrence
  Badge readers      — authenticate before entry
  Visitor logs       — track who enters and when
  Shredding          — destroy sensitive documents
  USB lockdown       — disable USB ports or whitelist devices
  Clean desk policy  — no sensitive info left visible
```

---

## Risk Management

### Risk formula

```
Risk = Threat × Vulnerability × Impact

Threat:         who/what could exploit the vulnerability (attacker, natural disaster)
Vulnerability:  the weakness that could be exploited
Impact:         the damage if exploited (financial, reputational, operational)
```

### Risk response strategies

```
MITIGATE   — reduce the risk (patch, add controls, encrypt data)
TRANSFER   — shift the risk to someone else (insurance, outsourcing)
ACCEPT     — acknowledge the risk and do nothing (documented decision)
AVOID      — eliminate the activity that creates the risk entirely
```

### Risk assessment process

```
1. Identify assets (what do we need to protect?)
2. Identify threats (what could go wrong?)
3. Identify vulnerabilities (what weaknesses exist?)
4. Assess likelihood (how probable is this?)
5. Assess impact (how bad would it be?)
6. Calculate risk (likelihood × impact)
7. Prioritize (address highest risks first)
8. Treat (mitigate, transfer, accept, or avoid)
9. Monitor (ongoing — risks change)
```

---

## Security Architecture Principles

```
Defense in depth         — multiple layers of security, no single point of failure
Least privilege          — minimum access needed to do the job
Separation of duties     — no single person controls an entire critical process
Need to know             — access only to information required for your role
Zero Trust               — never trust, always verify, regardless of network location
Fail secure              — system defaults to a secure state when it fails
Security by design       — build security in from the start, not bolt it on later
Assume breach            — design systems as if the attacker is already inside
```

---

## Career-Relevant Security Topics

### Security Operations Center (SOC) roles

```
SOC Analyst Tier 1 (Triage):
  - Monitor alerts from SIEM
  - Initial triage: true positive or false positive?
  - Escalate confirmed incidents to Tier 2
  - Create tickets and document findings

SOC Analyst Tier 2 (Investigation):
  - Deep investigation of escalated incidents
  - Correlate events across multiple data sources
  - Determine scope and impact
  - Develop detection rules and playbooks

SOC Analyst Tier 3 (Hunt):
  - Proactive threat hunting (look for threats that alerts missed)
  - Malware analysis
  - Develop advanced detection methods
  - Red team/purple team exercises

SOC Manager:
  - Team leadership and operations
  - Process improvement
  - Metrics and reporting to leadership
  - Tool selection and architecture
```

### Security engineering

```
Infrastructure security:
  - Firewall rule management
  - Network segmentation design
  - VPN and remote access architecture
  - Cloud security configuration (AWS/Azure/GCP)
  - Container security (Docker, Kubernetes)
  - CI/CD pipeline security

Identity and access management (IAM):
  - Directory services (Active Directory, Azure AD)
  - SSO implementation (SAML, OAuth, OIDC)
  - MFA deployment
  - Privileged access management (PAM)
  - Identity governance and lifecycle

Application security:
  - Secure code review
  - SAST/DAST integration into CI/CD
  - API security
  - Web application firewall (WAF) management
  - Bug bounty program management
```

### GRC (Governance, Risk, and Compliance)

```
Security policy development
Risk assessment and management
Compliance auditing (SOC 2, ISO 27001, PCI DSS, HIPAA)
Vendor risk management (third-party security assessment)
Business continuity and disaster recovery planning
Security awareness training
Data privacy (GDPR, CCPA, HIPAA)
Regulatory reporting
```

---

## Industry-Standard Tools Reference

### Offensive tools (you use these)

```
Scanning:       nmap, Masscan, Rustscan
Web:            Burp Suite, OWASP ZAP, Nikto, Gobuster, Feroxbuster
Exploitation:   Metasploit, Cobalt Strike, Impacket, CrackMapExec
AD:             BloodHound, Rubeus, Mimikatz, Certipy, PowerView
Passwords:      Hashcat, John the Ripper, Hydra
Wireless:       Aircrack-ng, Kismet, Bettercap
Social Eng:     Gophish, SET (Social Engineering Toolkit)
C2 frameworks:  Cobalt Strike, Sliver, Mythic, Havoc
```

### Defensive tools (you should understand these)

```
SIEM:           Splunk, Microsoft Sentinel, Elastic SIEM, QRadar
EDR:            CrowdStrike Falcon, Microsoft Defender for Endpoint, SentinelOne, Carbon Black
Firewall:       Palo Alto, Fortinet, Cisco ASA, pfSense
IDS/IPS:        Snort, Suricata, Zeek (Bro)
Vuln scanning:  Nessus, Qualys, OpenVAS, Rapid7 InsightVM
WAF:            Cloudflare WAF, AWS WAF, ModSecurity
Email security: Proofpoint, Mimecast, Microsoft Defender for O365
PAM:            CyberArk, BeyondTrust, Delinea
SOAR:           Splunk SOAR, Cortex XSOAR, Tines
Forensics:      Autopsy, Volatility, FTK, EnCase
```
