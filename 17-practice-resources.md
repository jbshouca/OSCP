# 17 — Practice Resources and Study Plan

The OSCP is a practical exam. Reading about techniques isn't enough — you need to practice them until they're muscle memory. This module tells you exactly where to practice and in what order.

---

## Practice Platforms (ranked by OSCP relevance)

### Tier 1: Essential (do ALL of these)

**OffSec PEN-200 Labs (included with course purchase)**
The actual course labs. Do EVERY module lab and ALL challenge labs — especially the three exam-like challenge labs. These are the closest thing to the real exam.

**Proving Grounds Practice ($19/month)**
Made by OffSec. Machines are rated by difficulty and tagged with OSCP relevance. The "OSCP-like" machines mirror exam difficulty exactly. Do at least 30 machines here.

**HackTheBox (free tier + $14/month for retired machines)**
Massive library. Retired machines have walkthroughs available — use them to learn, but try the machine yourself FIRST before reading the walkthrough. Active machines don't have walkthroughs, which forces you to think independently.

**TryHackMe (free tier + $14/month)**
More guided than HTB — good for building fundamentals before moving to harder platforms. Learning paths walk you through concepts step by step.

### Tier 2: Supplemental

**VulnHub (free)**
Downloadable VMs you run in your own lab. Good for offline practice and custom lab setups.

**OverTheWire — Bandit (free)**
Linux command-line fundamentals. If you're not fully comfortable with Linux CLI, start here before anything else.

**PentesterLab (free + paid tiers)**
Web vulnerability focus. Great for deepening SQLi, XSS, and authentication bypass skills.

---

## Recommended HackTheBox Machines

### Linux — Start with these (easiest to hardest)

| Machine | Skills | Notes |
|---|---|---|
| Lame | SMB exploitation | Easy first box. Great for learning the process. |
| Bashed | Web app, sudo privesc | Very beginner-friendly. phpbash web shell. |
| Nibbles | Web app, file upload, sudo | WordPress-like CMS, cron-based privesc. |
| Shocker | Shellshock, sudo | Classic CGI vulnerability. |
| Beep | Heavy enumeration, LFI | Lots of ports, lots of paths — teaches enumeration. |
| Cronos | DNS, SQLi, cron privesc | Zone transfer → SQLi → writable cron job. |
| Popcorn | File upload bypass, kernel | Upload filter bypass, then kernel exploit. |
| Irked | IRC exploitation, SUID | UnrealIRCd backdoor, SUID binary. |
| FriendZone | DNS, SMB, LFI, cron | Multi-step: zone transfer → SMB → LFI → cron. |
| Networked | File upload, command injection | Upload bypass + command injection privesc. |
| Traverxec | Web exploit, GTFOBins | Nostromo RCE, journalctl sudo. |
| OpenAdmin | Web app, sudo | OpenNetAdmin RCE, nano sudo. |
| Magic | SQLi bypass, file upload | Auth bypass + upload bypass combo. |
| Admirer | Web enum, credential hunting | Heavy enumeration, Adminer database tool. |
| SneakyMailer | Phishing, pip privesc | Email phishing → pip install privesc. |

### Windows — Start with these

| Machine | Skills | Notes |
|---|---|---|
| Legacy | MS08-067 | Ancient but teaches Metasploit workflow. |
| Blue | MS17-010 (EternalBlue) | Classic SMB exploit. |
| Devel | FTP + web shell, token privesc | FTP write to web root, SeImpersonate. |
| Optimum | Web exploit, Windows privesc | HFS RCE, MS16-098 kernel exploit. |
| Bastard | Drupal, Potato privesc | Drupalgeddon + JuicyPotato. |
| Bounty | File upload, Potato | IIS file upload bypass + JuicyPotato. |
| Jerry | Tomcat, default creds | Tomcat manager + WAR upload. Super easy. |
| Access | FTP + credential reuse | FTP files → password in .mdb → cached creds. |
| Buff | Web exploit, chisel | Gym management RCE + chisel tunnel. |
| ServMon | Web exploit, credential hunting | NVMS-1000 LFI → credential hunting. |
| Chatterbox | AChat exploit, password reuse | Buffer overflow exploit + password reuse. |
| Jeeves | Jenkins, KeePass | Jenkins script console + KeePass database. |

### Active Directory — Critical for the exam

| Machine | Skills | Priority |
|---|---|---|
| Active | GPP passwords, Kerberoasting | Do first — teaches core AD concepts |
| Forest | AS-REP Roasting, BloodHound, DCSync | Essential — full AD attack chain |
| Sauna | AS-REP Roasting, WinRM, DCSync | Essential — similar to exam AD set |
| Cascade | LDAP enumeration, credential hunting | Good for LDAP skills |
| Resolute | Password spraying, DnsAdmin privesc | Good for initial access techniques |
| Monteverde | Azure AD, credential reuse | Modern AD misconfigurations |
| Blackfield | AS-REP, BloodHound, SeBackup | Harder — good for final prep |
| Intelligence | LDAP, gMSA, constrained delegation | Advanced AD techniques |

**Do Active, Forest, and Sauna at minimum.** These three cover the core AD attack chain that appears on the exam.

---

## Recommended TryHackMe Rooms

### Foundations (do these first if you're new)

```
Linux Fundamentals Part 1, 2, 3
Windows Fundamentals Part 1, 2, 3
Networking Fundamentals
Web Fundamentals
```

### OSCP-specific rooms

```
# Reconnaissance
Nmap Live Host Discovery
Nmap Basic Port Scans
Nmap Advanced Port Scans

# Web Attacks
OWASP Top 10
SQL Injection
Command Injection
File Inclusion
Upload Vulnerabilities

# Exploitation
Metasploit Introduction
Exploit Vulnerabilities

# Privilege Escalation
Linux Privilege Escalation
Windows Privilege Escalation
Linux PrivEsc (Tib3rius)
Windows PrivEsc (Tib3rius)

# Active Directory
Active Directory Basics
Attacktive Directory
Exploiting Active Directory
AD Certificate Templates

# Pivoting
Wreath (multi-machine network with pivoting)

# Shells
What the Shell?
```

---

## Study Plan — Week by Week

### Weeks 1-2: Reconnaissance and Enumeration

```
Study:
  - Module 01 (Methodology)
  - Module 02 (Reconnaissance)
  - Module 03 (Web Enumeration)
  - Module 04 (Service Enumeration)

Practice:
  - Scan your home lab VMs thoroughly
  - TryHackMe: Nmap rooms, Web Fundamentals
  - HTB: Lame, Bashed (easy boxes to learn the workflow)
  
Goal: Can you scan a network, identify all services, and enumerate
each one systematically? Can you find hidden web directories, read
SMB shares, and enumerate FTP?
```

### Weeks 3-4: Web Attacks and Exploitation

```
Study:
  - Module 05 (Web Attacks)
  - Module 06 (Exploitation)
  - Module 07 (Password Attacks)

Practice:
  - Build vulnerable web apps on your Debian VM (from module 05 labs)
  - TryHackMe: OWASP Top 10, SQL Injection, Command Injection
  - HTB: Nibbles, Shocker, Beep
  - Practice SQLi and command injection until they're second nature

Goal: Can you identify and exploit SQLi, command injection, LFI,
and file upload on a web application? Can you find a public exploit
for a known CVE and modify it to work?
```

### Weeks 5-6: Shells and File Transfers

```
Study:
  - Module 08 (Shells)
  - Module 11 (File Transfers)

Practice:
  - Practice getting reverse shells in every language (bash, python, php, powershell)
  - Practice shell stabilization until it's muscle memory
  - Practice file transfers both directions (wget, certutil, scp, SMB)
  - HTB: Cronos, Irked, FriendZone

Goal: Can you go from command execution to a stable, interactive
reverse shell in under 2 minutes? Can you transfer files to and
from any target?
```

### Weeks 7-8: Linux Privilege Escalation

```
Study:
  - Module 09 (Linux Privesc)

Practice:
  - Set up privesc scenarios on your Debian/Ubuntu/CentOS VMs
  - TryHackMe: Linux PrivEsc, Linux Privilege Escalation
  - HTB: 15-20 Linux machines (focus on privesc)
  - Practice using LinPEAS and interpreting its output
  - Memorize GTFOBins for common SUID/sudo binaries

Goal: Given a low-privilege Linux shell, can you systematically
enumerate and exploit sudo misconfigs, SUID binaries, cron jobs,
writable files, and capabilities? Can you do it in under 30 minutes?
```

### Weeks 9-10: Windows Privilege Escalation

```
Study:
  - Module 10 (Windows Privesc)

Practice:
  - TryHackMe: Windows PrivEsc, Windows Privilege Escalation
  - HTB: 15-20 Windows machines (Devel, Optimum, Bastard, Bounty, etc.)
  - Practice token impersonation (PrintSpoofer, GodPotato)
  - Practice service exploitation (unquoted paths, weak permissions)
  - Practice using WinPEAS and PowerUp

Goal: Given a low-privilege Windows shell, can you identify and
exploit SeImpersonate, unquoted service paths, AlwaysInstallElevated,
stored credentials, and service misconfigurations?
```

### Weeks 11-12: Active Directory

```
Study:
  - Module 13 (Active Directory)

Practice:
  - TryHackMe: Attacktive Directory, Exploiting AD
  - HTB: Active, Forest, Sauna (minimum three AD machines)
  - Practice: AS-REP Roasting, Kerberoasting, password spraying
  - Practice: BloodHound data collection and analysis
  - Practice: PtH with impacket-psexec and evil-winrm
  - Practice: Mimikatz credential dumping
  - Build your own AD lab if possible (Windows Server eval + Windows 11)

Goal: Can you enumerate AD, find attack paths with BloodHound,
Kerberoast service accounts, move laterally with PtH, and
compromise a Domain Controller? Can you do the full chain?
```

### Weeks 13-14: Pivoting and Metasploit

```
Study:
  - Module 12 (Pivoting)
  - Module 14 (Metasploit)

Practice:
  - Your home lab pivoting labs (from pentest-labs repository)
  - TryHackMe: Wreath room (multi-machine with pivoting)
  - Practice SSH tunnels (local, dynamic, reverse, jump)
  - Practice Chisel (SOCKS proxy, port forward)
  - Practice Metasploit through tunnels

Goal: Can you pivot through a compromised host to reach an internal
network? Can you scan, exploit, and escalate through a tunnel?
```

### Weeks 15-16: Practice Exams and Report Writing

```
Study:
  - Module 15 (Report Writing)
  - Module 16 (Exam Strategy)

Practice:
  - Complete 3-5 full "mock exams" (4-5 machines in one sitting, timed)
  - Proving Grounds Practice: do OSCP-like machines under time pressure
  - Write a full report for at least one mock exam
  - Practice the proof screenshot format (flag + whoami + ip a)
  - Review your cheatsheet — make sure everything is on it

Goal: Can you complete 4-5 machines in 12-16 hours? Can you write
a professional report in under 8 hours? Is your cheatsheet complete?
```

---

## Daily Practice Routine

```
Time:     1 hour reading/study + 2 hours hands-on + 30 min notes
Machines: 1-2 per day during active study
By exam:  50+ machines completed (mix of Linux, Windows, AD)

After each machine:
  1. Write up what you learned (commands that worked, things you missed)
  2. Add new commands to your cheatsheet
  3. If you got stuck, note WHY and how you eventually solved it
  4. Rate your performance (what was fast? what was slow?)
```

---

## Key Resources Bookmarked

```
GTFOBins              gtfobins.github.io              Linux SUID/sudo lookup
LOLBAS                lolbas-project.github.io         Windows LOL binaries
HackTricks            book.hacktricks.wiki             Pentesting encyclopedia
PayloadsAllTheThings  github.com/swisskyrepo           Payloads for every vuln type
RevShells             revshells.com                    Reverse shell generator
CyberChef             gchq.github.io/CyberChef         Encode/decode anything
Ippsec                youtube.com/ippsec                HTB video walkthroughs
ExploitDB             exploit-db.com                   Public exploits
Hashcat wiki          hashcat.net/wiki/doku.php?id=hashcat  Hash mode reference
```

---

## The Machine Count That Passes

Based on OSCP pass reports and community data:

```
30 machines   → Probably not enough (unless you're very experienced)
50 machines   → Good foundation — most people who practice this many pass
75+ machines  → Excellent preparation — high confidence going in
100+ machines → Overkill but guarantees you've seen every technique
```

**Quality matters more than quantity.** 50 machines where you understood every step beats 100 machines where you followed walkthroughs without thinking.
