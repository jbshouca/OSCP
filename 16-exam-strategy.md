# 16 — Exam Strategy

The OSCP exam is as much about strategy and mental endurance as it is about technical skill. 24 hours is a marathon. This module covers how to plan, execute, and survive it.

---

## The Week Before

### Technical preparation

```
□ Kali VM is fully updated (sudo apt update && sudo apt upgrade)
□ VPN connection to OffSec labs works
□ All tools installed and tested (nmap, gobuster, chisel, linpeas, winpeas, etc.)
□ Tools directory prepared (~/oscp-tools/ with everything you'll need)
□ Report template downloaded from OffSec support page
□ Cheatsheet printed and ready
□ Practiced with 2-3 Proving Grounds or HTB machines as a mock exam
□ Reviewed weak areas (AD? Windows privesc? Web attacks?)
```

### Logistical preparation

```
□ Exam scheduled at a time when you'll be fresh
□ Workspace set up (quiet room, dual monitors if possible)
□ Webcam and screen sharing tested (OffSec proctors watch you)
□ Food and drinks prepped (meals you can eat at your desk)
□ Caffeine strategy planned (don't overdo it — crash later is worse)
□ Phone on silent, notifications off
□ Told family/roommates not to disturb you
□ Backup internet connection ready (phone hotspot)
```

### Mental preparation

```
□ Got good sleep the night before (minimum 7 hours)
□ Accepted that you WILL get stuck — that's normal
□ Planned break schedule (every 2-3 hours, take 10-15 minutes)
□ Reminded yourself: "the answer is in the enumeration"
□ Set a positive mindset: 70 points is the target, not 100
```

---

## Exam Day — Hour by Hour

### Hour 0:00 — The first 15 minutes

**DO NOT start hacking immediately.** Read the exam panel carefully.

```
1. Read every instruction
2. Note all target IPs
3. Identify which machines are the AD set vs standalones
4. Note the network topology (subnets, any VPN details)
5. Set up your notes structure:
   mkdir -p ~/oscp/exam/{ad_machine1,ad_machine2,ad_dc,standalone1,standalone2,standalone3}
```

### Hour 0:15 — Launch all scans simultaneously

```bash
# Quick scans on every target (run all at once)
nmap -sV -sC -oA ~/oscp/exam/standalone1/quick STANDALONE1_IP &
nmap -sV -sC -oA ~/oscp/exam/standalone2/quick STANDALONE2_IP &
nmap -sV -sC -oA ~/oscp/exam/standalone3/quick STANDALONE3_IP &
nmap -sV -sC -oA ~/oscp/exam/ad_machine1/quick AD_IP1 &
nmap -sV -sC -oA ~/oscp/exam/ad_machine2/quick AD_IP2 &
nmap -sV -sC -oA ~/oscp/exam/ad_dc/quick AD_DC_IP &

# Full port scans in background (catch high ports)
nmap -p- --min-rate=1000 -oA ~/oscp/exam/standalone1/full STANDALONE1_IP &
nmap -p- --min-rate=1000 -oA ~/oscp/exam/standalone2/full STANDALONE2_IP &
nmap -p- --min-rate=1000 -oA ~/oscp/exam/standalone3/full STANDALONE3_IP &

# UDP on all targets
sudo nmap -sU --top-ports 20 -oA ~/oscp/exam/standalone1/udp STANDALONE1_IP &
sudo nmap -sU --top-ports 20 -oA ~/oscp/exam/standalone2/udp STANDALONE2_IP &
sudo nmap -sU --top-ports 20 -oA ~/oscp/exam/standalone3/udp STANDALONE3_IP &
```

While scans run, start looking at the first quick scan results that come back.

### Hours 0:30 — 5:00 — AD Set (attempt for 4-5 hours)

**Why first:** It's worth 40 points, more than any two standalones combined. Do it while you're fresh and focused.

```
AD Attack Flow:
1. Identify DC (ports 88, 389, 636)
2. Enumerate first machine (web app? service exploit?)
3. Get initial foothold on Machine 1
4. Enumerate AD (users, groups, shares, SPNs)
5. Kerberoast / AS-REP Roast / password spray
6. Find credentials for Machine 2
7. Lateral movement (PtH, evil-winrm, psexec)
8. Local privesc on Machine 2 → dump creds
9. Find path to DC
10. Compromise DC → proof.txt from all three
```

**If you get the AD set in 4 hours:** You have 40 points and 19 hours left for 30 more points. You're in an excellent position.

**If you're stuck after 5 hours:** Move to standalones. Come back to AD later with fresh eyes. Don't burn all your time here.

### Hours 5:00 — 8:00 — Standalone 1

```
Pick the machine with the most enumeration results.
Follow the methodology:
1. Review nmap output (quick + full port scan)
2. Enumerate every port
3. Find the vulnerability
4. Get initial access → local.txt (10 points)
5. Privesc → proof.txt (10 points)
```

### Hours 8:00 — 11:00 — Standalone 2

Same approach. If you're going well, you might have 60-80 points by now.

### Hours 11:00 — 14:00 — Standalone 3

Same approach. Even if you only get initial access (10 points), every point counts.

### Hours 14:00 — 20:00 — Return to unsolved machines

```
At this point, reassess:
- How many points do I have?
- What's the easiest remaining path to 70?
- Is the AD set partially done? Can I finish it?
- Did I miss something on a standalone? Re-enumerate.
- Should I use my Metasploit shot now?
```

This is where the **Metasploit decision** happens. If you're stuck on a standalone and you haven't used Metasploit yet:

```bash
# This is the time to use it
msfconsole
search [vulnerability you identified]
use [exploit]
set PAYLOAD windows/x64/meterpreter/reverse_tcp
exploit
# If it works → getsystem → hashdump → root/SYSTEM → done
```

### Hours 20:00 — 23:45 — Final cleanup

```
□ Verify ALL flags are captured (cat local.txt, cat proof.txt)
□ Every proof screenshot has: flag + whoami + ip a/ipconfig
□ Notes are complete for every compromised machine
□ Screenshots are saved and named clearly
□ You know exactly what you'll write in the report
```

---

## The "I'm Stuck" Decision Tree

```
Stuck on a machine for more than 1.5 hours?
│
├── Have you done a FULL port scan (-p-)?
│   └── NO → run it now, wait for results
│
├── Have you checked UDP?
│   └── NO → nmap -sU --top-ports 20
│
├── Have you tried a bigger gobuster wordlist?
│   └── NO → directory-list-2.3-medium.txt with extensions
│
├── Have you read EVERY web page's source code?
│   └── NO → do it now, look for comments and hidden paths
│
├── Have you tried found credentials on EVERY service?
│   └── NO → try them on SSH, RDP, WinRM, SMB, web logins
│
├── Have you checked searchsploit for EVERY version?
│   └── NO → search every version string from nmap
│
├── None of the above helped?
│   ├── Take a 15-minute break (walk, eat, drink water)
│   ├── Come back and re-read your nmap output from scratch
│   ├── Consider: is there a service you dismissed too quickly?
│   └── If still stuck after 2.5 hours total → move to another target
│       (come back later with fresh perspective)
```

---

## Points Strategy

### The math

```
Need 70 points to pass.

IDEAL:   AD(40) + SA1(20) + SA2(20) = 80 ✓  (buffer for mistakes)
GOOD:    AD(40) + SA1(20) + SA2_local(10) = 70 ✓
TIGHT:   AD(40) + SA1(20) + SA2_local(10) = 70 ✓
RISKY:   SA1(20) + SA2(20) + SA3(20) = 60  (need bonus — dangerous)
```

**Always prioritize the AD set.** It's 40% of the exam and the single biggest block of points.

### Partial credit matters

Even if you can't fully compromise a standalone machine, getting initial access (local.txt) is worth 10 points. On a tight exam, that 10 points might be the difference between 65 and 75.

```
Don't give up on a machine even if you can't do privesc.
10 points for initial access is better than 0 points.
```

---

## Breaks and Self-Care

This sounds soft but it's the most practical advice. People fail the OSCP because they burn out at hour 16 and can't think clearly for the remaining 8 hours.

```
Every 2-3 hours:
  - Stand up
  - Walk around for 5-10 minutes
  - Drink water
  - Eat something (not just caffeine)

At the 12-hour mark:
  - Take a real 30-minute break
  - Eat a full meal
  - Step completely away from the computer
  - Reassess your progress and adjust your plan

Caffeine strategy:
  - Don't front-load caffeine (you'll crash at hour 14)
  - Steady intake — one coffee/energy drink every 3-4 hours
  - Stop caffeine at hour 18 if you plan to sleep
  
Sleep:
  - If you have 70+ points → consider sleeping for 3-4 hours
  - If you're under 70 → push through but take micro-breaks
  - A tired brain makes stupid mistakes — sometimes 2 hours of sleep
    makes you more productive than 6 hours of exhausted grinding
```

---

## The Mindset That Passes

```
"The answer is in the enumeration."
  → When stuck, enumerate more, don't guess.

"Try harder doesn't mean try the same thing harder."
  → Change your approach. Different wordlist. Different attack vector.

"70 is the goal, not 100."
  → Don't chase perfection. Chase passing.

"Every machine is designed to be solvable."
  → OffSec designed these machines to be compromised. The path exists.

"Taking a break is not giving up."
  → Fresh eyes find things tired eyes miss.
```
