# 01 — Penetration Testing Methodology

The methodology is the most important thing you'll learn. Tools change, vulnerabilities change, but the process stays the same. Every machine you touch on the OSCP — and every machine in your career — follows this pattern. If you internalize this, you'll never sit at a terminal wondering "what do I do next?"

---

## The Core Process

```
SCAN → ENUMERATE → EXPLOIT → ESCALATE → LOOT → PIVOT → REPEAT
```

That's it. Seven words. But understanding what each one REALLY means and knowing when to cycle back is what separates someone who passes the OSCP from someone who fails.

---

## Phase 1: Scanning — "What's out there?"

Scanning answers two questions: **what hosts exist** and **what ports are open on each host**.

### What you do

```bash
# Host discovery — find all live machines
nmap -sn 192.168.244.0/24

# Port scan each host — find open services
nmap -sV -sC -oN target1.txt TARGET_IP

# Full port scan in background (catches ports above 1000)
nmap -p- --min-rate=1000 -oN target1_full.txt TARGET_IP &

# UDP scan (slow but important — SNMP, DNS, TFTP hide here)
sudo nmap -sU --top-ports 20 -oN target1_udp.txt TARGET_IP
```

### What you're looking for

Open ports = attack surface. Each open port is a door. Your job is to check every door.

```
Port 21 (FTP)    → Can I log in anonymously? Are there files to download?
Port 22 (SSH)    → Do I have credentials? Is the version vulnerable?
Port 80 (HTTP)   → Is there a web app? What's in the source? Hidden directories?
Port 445 (SMB)   → Can I list shares? Can I connect without credentials?
Port 3306 (MySQL) → Can I connect? Default credentials?
```

### How to think about it

Scanning is NOT just running nmap and moving on. It's building a mental map:

```
Target: 192.168.244.132
├── Port 21 (FTP) — vsftpd 3.0.3
│   └── Questions: Anonymous login? Writable? Known CVEs?
├── Port 22 (SSH) — OpenSSH 9.2
│   └── Questions: Need credentials. Try found creds here later.
├── Port 80 (HTTP) — Apache 2.4.57
│   └── Questions: What's the web app? Hidden dirs? Login page? Injection?
└── Port 8080 (HTTP) — Tomcat 9.0
    └── Questions: Manager console? Default creds? WAR deployment?
```

Write this down. Physically. For every target. This is your attack plan.

### Common mistake

Running nmap once with default settings and moving on. Default nmap only scans the **top 1000 ports** out of 65,535. The foothold might be on port 8888 or 13337 or 50000. Always run a full port scan (`-p-`) even if it takes longer.

---

## Phase 2: Enumeration — "What can I learn about each service?"

This is where 80% of your time should go. Enumeration means digging into every open port to extract as much information as possible. The goal is to find one of three things:

1. **A vulnerability** you can exploit for a shell
2. **Credentials** you can use to log in somewhere
3. **Information** that leads you to either of the above

### What you do

For EVERY open port:

```
1. Banner grab (nc -vn TARGET PORT)
2. Check the exact version against searchsploit
3. Run service-specific enumeration tools
4. Look for default credentials
5. Look for misconfigurations
6. Document everything
```

### Enumeration by service

Each service has its own enumeration approach. This is covered in depth in modules 03 (web) and 04 (services), but here's the thought process:

**Web server (80/443):**
```
What is it?      → whatweb, HTTP headers (Apache? Nginx? IIS?)
What's on it?    → Browse it, read source, check robots.txt
What's hidden?   → gobuster/dirb with extensions (-x php,txt,bak)
What CMS?        → WordPress (wpscan), Drupal, Joomla
Login pages?     → Try default creds, test for SQLi
Input fields?    → Test injection (SQL, command, XSS)
File uploads?    → Can I upload a shell?
```

**SMB (139/445):**
```
Can I connect without creds?  → smbclient -L //TARGET -N
Can I list users?              → enum4linux -a TARGET
Can I access shares?           → smbclient //TARGET/share -N
What OS version?               → nmap --script smb-os-discovery
Any known vulns?               → nmap --script smb-vuln*
```

**FTP (21):**
```
Anonymous login?  → ftp TARGET (user: anonymous, pass: blank)
Readable files?   → ls, get everything
Writable?         → try: put test.txt
Version exploit?  → searchsploit vsftpd
```

### The golden rule of enumeration

**When you're stuck, enumerate more.** Don't jump to running exploits randomly. Don't try kernel exploits because you're frustrated. Go back and enumerate something you missed.

Ask yourself:
- Did I read EVERY file on the FTP server?
- Did I check EVERY directory the web brute force found?
- Did I read the HTML source of EVERY page?
- Did I try the credentials I found on EVERY service?
- Did I check EVERY port, including the ones I thought weren't important?
- Did I run a full port scan? What about UDP?

Nine times out of ten, the answer is "no, I skipped something." That something is where the foothold is.

---

## Phase 3: Exploitation — "How do I get in?"

Exploitation is using what you found during enumeration to get a shell on the target. This should feel natural if you enumerated thoroughly — the path should be obvious.

### Common initial access paths (in order of how often they appear on OSCP)

```
1. Web application vulnerability (SQLi, command injection, LFI, file upload)
   → Most common. At least 1-2 exam machines use this path.

2. Known exploit for an outdated service version
   → searchsploit finds a CVE with working exploit code.
   → Modify the exploit (change IPs/ports), run it, get a shell.

3. Default or weak credentials
   → admin:admin on a web panel, anonymous FTP, no-password database.
   → Sometimes the foothold is just logging in because the password is terrible.

4. Credential reuse
   → Password found on Machine A works on Machine B's SSH.
   → This is especially common in the AD set.

5. Misconfiguration
   → Writable SMB share in the web root (upload a shell).
   → FTP root is the same as the web root (upload a shell).
   → Exposed .git directory (clone the repo, find creds in commit history).
```

### What NOT to do

```
DON'T: Try random exploits against a service without confirming the version matches
DON'T: Run Metasploit's autopwn or automated exploitation
DON'T: Brute force SSH for 3 hours (if it's not cracking in 15 minutes, that's not the path)
DON'T: Try kernel exploits before you've exhausted easier options
DON'T: Skip reading the exploit code before running it
```

### Reading and modifying exploits

When you find an exploit on ExploitDB:

```bash
# Search
searchsploit apache 2.4.49

# Copy it to your working directory
searchsploit -m 50383

# READ IT before running
cat 50383.py
```

Look for:
- What language is it? (Python, Ruby, C, Bash)
- Does it need dependencies? (`import requests` → `pip install requests`)
- What variables need changing? (target IP, port, callback IP)
- What does it actually do? (Does it give a shell? Read a file? Create a user?)
- Is it destructive? (Does it crash the service?)

Modify the variables:
```python
# In the exploit code, look for lines like:
target = "10.10.10.10"        # Change to YOUR target
port = 80
lhost = "10.10.14.5"          # Change to YOUR Kali IP
lport = 4444                   # Your listener port
```

Set up your listener, then run the exploit:
```bash
# Terminal 1: listener
nc -lvnp 4444

# Terminal 2: exploit
python3 50383.py
```

---

## Phase 4: Privilege Escalation — "How do I become root/SYSTEM?"

You have a shell. You're a low-privilege user (www-data, apache, a regular user). You need root (Linux) or SYSTEM/Administrator (Windows).

**This is worth 50% of every standalone machine's points.** Initial access gets you 10 points (local.txt). Privilege escalation gets you the other 10 (proof.txt). You MUST be good at privesc.

### The process is the same as the overall methodology — enumerate, then exploit

```
1. Check: who am I? (whoami, id)
2. Check: what can I do? (sudo -l)
3. Check: what's misconfigured? (SUID binaries, writable files, cron jobs)
4. Check: what's vulnerable? (kernel version, installed software)
5. Exploit the weakest finding
```

Detailed techniques are in modules 09 (Linux) and 10 (Windows).

### The most important privesc check

On Linux, `sudo -l` is the FIRST thing you run. Always. If this shows you can run something as root, that's probably your privesc path. Check GTFOBins for how to exploit it.

On Windows, `whoami /priv` is the equivalent. If you see `SeImpersonatePrivilege`, that's a Potato attack. If you see `SeBackupPrivilege`, you can read any file.

---

## Phase 5: Looting — "What's on this machine that helps me go further?"

After escalating (or even before), search the machine for:

```bash
# Credentials for other machines
grep -r "password" /home/ /opt/ /var/www/ /etc/ 2>/dev/null
cat /home/*/.bash_history
cat /home/*/.ssh/id_rsa
cat /var/www/html/config.php

# Network information
ip a              # What interfaces does this machine have? (hidden subnets?)
ip route          # What networks can it reach?
arp -a            # What other machines has it talked to?
ss -tlnp          # What services are listening locally?
cat /etc/hosts    # Any internal hostnames mapped?
cat ~/.ssh/known_hosts  # What machines has this user SSH'd to?

# Flags
find / -name "local.txt" -o -name "proof.txt" 2>/dev/null
```

### Why looting matters on the exam

On standalone machines, you loot for your documentation and to collect flags. But on the AD set, **looting Machine 1 gives you the credentials or information to access Machine 2.** The chain won't work without thorough post-exploitation enumeration on each machine.

---

## Phase 6: Pivoting — "This machine can see networks I can't"

If the compromised machine has multiple network interfaces (like your CentOS VM with 192.168.244.x AND 172.16.0.x), it can reach networks your Kali can't. You need to route your tools through it.

```bash
# Check for multiple interfaces
ip a
# If you see two IPs on different subnets → pivoting opportunity

# Set up a SOCKS proxy
ssh -D 9050 user@COMPROMISED_HOST
# Now: proxychains nmap -sT -Pn INTERNAL_TARGET
```

Detailed pivoting techniques are in module 12.

---

## Phase 7: Repeat — "Do it all again on the next target"

Start the whole process over on the next machine. Scan → enumerate → exploit → escalate → loot → pivot → repeat.

The key insight: each machine you compromise gives you more information and access for the next one. Credentials reuse across machines. Network access expands through pivoting. The attack chain builds on itself.

---

## The OSCP Exam — Applying the Methodology

### Time allocation

You have 23 hours 45 minutes. Here's how to use them:

```
00:00 - 00:15   Start full nmap scans on ALL targets simultaneously
                Quick scans first (-sV -sC), full port scans in background (-p-)

00:15 - 00:30   While scans run, review the exam panel
                Note all IPs, identify which is the AD set vs standalones
                Set up your notes template

00:30 - 06:00   AD SET (attempt for 4-5 hours max)
                This is worth 40 points — do it while you're fresh
                If you get stuck after 5 hours, switch to standalones

06:00 - 10:00   Standalone Machine 1 (target the one with most enumeration results)
                2-3 hours should be enough for a full compromise

10:00 - 14:00   Standalone Machine 2
                Same approach — enumerate hard, find the path

14:00 - 18:00   Standalone Machine 3
                Or return to AD if you didn't finish it

18:00 - 22:00   Return to anything unsolved
                Fresh eyes often see what tired eyes missed

22:00 - 23:45   Cleanup, verify all flags, take final screenshots
                Make sure every proof screenshot has:
                flag content + whoami + ip a/ipconfig
```

### When you're stuck — the checklist

Print this and keep it next to you during the exam:

```
□ Did I run a full port scan (-p-)? Check for high ports.
□ Did I check UDP? (nmap -sU --top-ports 20)
□ Did I read the page source of every web page?
□ Did I run gobuster with a big wordlist AND file extensions?
□ Did I try default credentials on every login page?
□ Did I search every service version in searchsploit?
□ Did I check every file on FTP/SMB shares?
□ Did I try found credentials on EVERY service and EVERY machine?
□ Did I check for LFI/SQLi/command injection on EVERY input field?
□ Did I check robots.txt, sitemap.xml, .htaccess?
□ Did I look at ALL the nmap script output (not just the ports)?
□ Did I take a 10-minute break? (seriously — fresh eyes find things)
```

### Documentation as you go

Don't wait until after the exam to document. Document AS you work.

For every machine, keep a running log:

```
## Target: 192.168.X.X (Standalone 1)

### Nmap
[paste full nmap output]

### Enumeration
- Port 80: Apache 2.4.57
  - gobuster found: /admin, /uploads, /config.bak
  - /admin has login page
  - /config.bak contains DB credentials
  - Tried admin:admin — didn't work
  - Tried SQLi on login — ' OR 1=1-- works!

### Initial Access
- SQLi authentication bypass on /admin login
- Found file upload in admin panel
- Uploaded PHP shell
- Got reverse shell as www-data
- SCREENSHOT: [shell + whoami + id]
- local.txt: [FLAG_VALUE]
- SCREENSHOT: [flag + whoami + ip a]

### Privilege Escalation
- sudo -l shows: (root) NOPASSWD: /usr/bin/vim
- GTFOBins: sudo vim → :!bash → root shell
- SCREENSHOT: [root shell + whoami + id]
- proof.txt: [FLAG_VALUE]
- SCREENSHOT: [flag + whoami + ip a]
```

This format translates directly into your report. You're not doing extra work — you're writing the report in real time.

---

## Lab: Practice the Full Methodology

Use any of the four labs from the pentest-labs repository. Before you start each one, remind yourself:

```
1. I will scan ALL ports
2. I will enumerate EVERY service
3. I will check EVERY credential against EVERY login
4. I will READ the source of EVERY web page
5. I will document EVERY step
6. When stuck, I will enumerate more, not guess
```

Time yourself. Can you complete the full attack chain in under 4 hours? Under 3? Under 2? The exam gives you roughly 4 hours per machine (24 hours ÷ 6 machines). Getting fast at the methodology is what passing looks like.

---

## The Mindset

The OSCP is as much a mental challenge as a technical one. 24 hours is exhausting. You will get stuck. You will feel like you've tried everything. When that happens:

1. **Take a break.** Walk away for 15 minutes. Eat something. Drink water.
2. **Re-read your notes.** Something you scanned past might jump out now.
3. **Go back to basics.** Re-run nmap. Re-run gobuster with different wordlists. Re-check everything.
4. **Try the thing you think is "too simple."** Sometimes the answer is a default password.
5. **Move to a different target.** Come back later with fresh eyes.

The difference between passing and failing is almost never "they knew more tools." It's "they didn't give up, and they enumerated harder."
