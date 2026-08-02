# OSCP (PEN-200) Comprehensive Study Guide

A complete, in-depth study guide for the Offensive Security Certified Professional certification. Every concept is explained thoroughly with examples, hands-on labs, and breakdowns of what's happening under the hood.

## What is the OSCP?

The OSCP is the most respected hands-on penetration testing certification in cybersecurity. Unlike certifications with multiple-choice questions, the OSCP is a **23 hour 45 minute practical exam** where you hack into real machines and then write a professional penetration test report within the following 24 hours.

OffSec's motto is "Try Harder" — the exam tests your ability to think, research, and persist through roadblocks, not just run tools.

## The Exam in Detail

### Structure

The exam gives you access to a network with two components:

**3 standalone machines (60 points total):**
- Each machine is worth 20 points
- 10 points for getting initial access (low-privilege shell) and submitting `local.txt`
- 10 points for escalating privileges to root/SYSTEM and submitting `proof.txt`
- These machines are independent — compromising one doesn't help with another

**1 Active Directory set (40 points total):**
- 3 machines chained together in an Active Directory domain
- You must compromise the ENTIRE chain — there is NO partial credit
- Typically: foothold on Machine 1 → lateral movement to Machine 2 → compromise Domain Controller (Machine 3)
- Submit `proof.txt` from all three machines

### Scoring

```
Standalone Machine 1:  10 (local.txt) + 10 (proof.txt) = 20 points
Standalone Machine 2:  10 (local.txt) + 10 (proof.txt) = 20 points
Standalone Machine 3:  10 (local.txt) + 10 (proof.txt) = 20 points
AD Set (3 machines):   40 points (all or nothing)
                                              Total: 100 points
                                         To pass:  70 points
```

### Winning strategies

The math tells you what to prioritize:

```
SAFE PATH: AD set (40) + 2 full standalones (40) = 80 points ✓
MINIMUM:   AD set (40) + 1 full standalone (20) + 1 local.txt (10) = 70 points ✓
RISKY:     3 full standalones (60) + no AD = need 10 bonus points (very risky)
```

**The AD set is the most important thing on the exam.** It's 40% of your score with no partial credit. If you can't do AD, you need ALL three standalones fully compromised plus bonus points — that's harder than learning AD.

### Tool restrictions

**Metasploit:** You can only use Metasploit (including Meterpreter) on **ONE target** during the entire exam. Once you use it on a machine, that machine is "locked" to Metasploit. Choose wisely — save it for the hardest standalone machine.

**Exception:** The `multi/handler` module is **exempt** from the restriction. You can use it on any target to catch reverse shells. This is important — you can generate payloads with MSFVenom, use `multi/handler` to catch them on every machine, but you can't use Metasploit's exploits or post-exploitation modules on more than one target.

**Banned:**
- Commercial exploitation tools (Cobalt Strike, Canvas, Core Impact)
- Automated exploitation (SQLMap's `--os-shell`, auto-exploitation features)
- AI tools for generating exploits or attack strategies
- Spoofing attacks (Responder, ARP spoofing, LLMNR poisoning)
- Denial of Service attacks
- Any pre-existing knowledge of the exam machines

**Allowed:**
- Nmap, Nikto, GoBuster, Feroxbuster, DirBuster, Burp Suite Community
- All Impacket tools, CrackMapExec, BloodHound, Rubeus, Mimikatz
- Chisel, Ligolo-ng, SSH, socat, netcat
- Hashcat, John the Ripper, Hydra
- Custom scripts and tools you write yourself
- SearchSploit / ExploitDB

### The report

You MUST submit a penetration test report within 24 hours after the exam ends. **No report = automatic fail**, regardless of how many machines you compromised.

The report must include for each machine:
- IP address, hostname, OS
- Full enumeration results
- Step-by-step exploitation process (exact commands)
- Screenshots of every significant step
- Proof screenshots showing: flag content + `whoami` + `ip a`/`ipconfig` in the SAME screenshot

## Study Guide Structure

Each file is a complete deep-dive into its topic with explanations, examples, labs, and exam tips.

### Core Methodology
| File | Topic | Exam Weight |
|---|---|---|
| [01-methodology.md](01-methodology.md) | How to approach every target systematically | Foundation |
| [02-reconnaissance.md](02-reconnaissance.md) | nmap, netcat, host discovery, port scanning | Every machine |

### Enumeration (where 80% of your time goes)
| File | Topic | Exam Weight |
|---|---|---|
| [03-web-enumeration.md](03-web-enumeration.md) | Directory busting, source analysis, CMS identification | High |
| [04-service-enumeration.md](04-service-enumeration.md) | SMB, FTP, SNMP, DNS, SMTP, NFS, databases | High |

### Exploitation
| File | Topic | Exam Weight |
|---|---|---|
| [05-web-attacks.md](05-web-attacks.md) | SQLi, command injection, LFI/RFI, file upload, XSS | High |
| [06-exploitation.md](06-exploitation.md) | Finding and using public exploits, modifying code | High |
| [07-password-attacks.md](07-password-attacks.md) | Brute forcing, hash cracking, password spraying | Medium |
| [08-shells.md](08-shells.md) | Reverse shells, bind shells, web shells, shell upgrades | Critical |

### Post-Exploitation
| File | Topic | Exam Weight |
|---|---|---|
| [09-linux-privesc.md](09-linux-privesc.md) | sudo, SUID, cron, capabilities, kernel exploits | Critical (50% of standalone points) |
| [10-windows-privesc.md](10-windows-privesc.md) | Tokens, services, registry, scheduled tasks, potato attacks | Critical (50% of standalone points) |
| [11-file-transfers.md](11-file-transfers.md) | Moving files between attacker and target | Every machine |

### Pivoting and Lateral Movement
| File | Topic | Exam Weight |
|---|---|---|
| [12-pivoting-tunneling.md](12-pivoting-tunneling.md) | SSH tunnels, Chisel, Ligolo, proxychains, iptables | AD set + some standalones |

### Active Directory (40% of exam)
| File | Topic | Exam Weight |
|---|---|---|
| [13-active-directory.md](13-active-directory.md) | AD enumeration, Kerberoasting, PtH, lateral movement, DCSync | 40 points |

### Exam Preparation
| File | Topic |
|---|---|
| [14-metasploit.md](14-metasploit.md) | Framework usage, when to use your one Metasploit shot |
| [15-report-writing.md](15-report-writing.md) | Report structure, screenshot requirements, templates |
| [16-exam-strategy.md](16-exam-strategy.md) | Time management, order of operations, mental preparation |
| [17-practice-resources.md](17-practice-resources.md) | HTB boxes, TryHackMe rooms, Proving Grounds, study timeline |
| [cheatsheet.md](cheatsheet.md) | Quick-reference commands for exam day |

## Lab Environment

This guide uses a home lab with four VMs for hands-on practice:

| VM | OS | IP | Role |
|---|---|---|---|
| Kali | Kali Linux | 192.168.244.129 | Attacker machine |
| Debian | Debian 12+ | 192.168.244.132 | Target / pivot host |
| CentOS | CentOS Stream 9 | 192.168.244.131 | Target / middleware |
| Ubuntu | Ubuntu 26.04 | 192.168.244.130 | Target / internal server |

CentOS and Ubuntu each have a second NIC on a shared Host-only network (172.16.0.0/24) for practicing pivoting and multi-subnet attacks.

## Study Timeline (12-16 weeks)

```
Weeks 1-2:   Reconnaissance + Enumeration (modules 01-04)
             Goal: Comfortable with nmap, gobuster, enum4linux, manual enumeration
             Practice: Scan everything in your lab, enumerate every service

Weeks 3-4:   Web Attacks + Exploitation (modules 05-06)
             Goal: Find and exploit SQLi, command injection, LFI, file upload
             Practice: Set up vulnerable web apps, exploit them manually

Weeks 5-6:   Password Attacks + Shells (modules 07-08)
             Goal: Crack hashes, brute force services, catch reverse shells reliably
             Practice: Set up auth services, attack them, catch shells from every VM

Weeks 7-8:   Linux Privilege Escalation (module 09)
             Goal: Identify and exploit every common Linux privesc vector
             Practice: 20+ Linux machines on HTB/Proving Grounds

Weeks 9-10:  Windows Privilege Escalation (module 10)
             Goal: Identify and exploit every common Windows privesc vector
             Practice: 20+ Windows machines on HTB/Proving Grounds

Weeks 11-12: Active Directory (module 13)
             Goal: Complete an AD attack chain from enumeration to DC compromise
             Practice: HTB AD machines, TryHackMe AD rooms, build your own AD lab

Weeks 13-14: Pivoting + File Transfers + Metasploit (modules 11-12, 14)
             Goal: Tunnel through multiple networks, move tools to targets
             Practice: Multi-machine labs with pivoting requirements

Weeks 15-16: Practice Exams + Report Writing (modules 15-17)
             Goal: Complete 3+ full practice exams within the time limit
             Practice: PG Practice OSCP-like machines, write full reports

Daily routine:
  1 hour:   Study one topic (read, understand, take notes)
  2 hours:  Practice on machines (HTB, TryHackMe, Proving Grounds, your lab)
  30 min:   Write up notes on what you learned and commands that worked

Target: 50+ machines completed before exam day
```
