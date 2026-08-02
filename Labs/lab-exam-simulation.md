# Lab: OSCP Exam Simulation

## Overview

This lab simulates the OSCP exam format using ALL your VMs. Set it up, then give yourself a strict time limit to complete the entire chain. Practice under pressure — the exam is 23 hours 45 minutes, but you should aim to finish this lab in under 8 hours.

**Scoring:**
```
Debian (standalone 1):    10 pts local.txt + 10 pts proof.txt = 20 pts
Windows 11 (standalone 2): 10 pts local.txt + 10 pts proof.txt = 20 pts
AD Set (CentOS + Ubuntu):  40 pts (all or nothing — must compromise both)
                                                        Total: 80 pts
                                                    Pass: 56 pts (70%)
```

---

## Setup — Build the Exam Environment

### Important: Set up ALL machines BEFORE starting the timer

Have someone else set this up if possible so you don't know the answers. If you're setting it up yourself, wait at least 24 hours before attempting it so you forget the specifics.

### Standalone 1: Debian (192.168.244.132)

```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php php-mysql mariadb-server openssh-server vsftpd -y
sudo systemctl enable --now apache2 ssh vsftpd mariadb

# Create local.txt and proof.txt (like the real exam)
echo "LOCAL{$(openssl rand -hex 16)}" | sudo tee /home/local.txt
sudo chmod 644 /home/local.txt
echo "PROOF{$(openssl rand -hex 16)}" | sudo tee /root/proof.txt
sudo chmod 600 /root/proof.txt

# Create a low-privilege user for initial access
sudo useradd -m -s /bin/bash webuser
echo "webuser:Sunshine2026!" | sudo chpasswd

# FTP with anonymous access and a hint
sudo sed -i 's/anonymous_enable=NO/anonymous_enable=YES/' /etc/vsftpd.conf 2>/dev/null
echo "anonymous_enable=YES" | sudo tee -a /etc/vsftpd.conf
echo "IT Department Notice:" | sudo tee /srv/ftp/notice.txt
echo "New web portal deployed at /portal" | sudo tee -a /srv/ftp/notice.txt
echo "Admin credentials are in the database" | sudo tee -a /srv/ftp/notice.txt
sudo systemctl restart vsftpd

# Web portal with SQLi
sudo mysql << 'SQL'
CREATE DATABASE portal;
CREATE TABLE portal.admins (id INT AUTO_INCREMENT PRIMARY KEY, username VARCHAR(50), password VARCHAR(100), role VARCHAR(20));
INSERT INTO portal.admins VALUES (1, 'siteadmin', 'W3bAdmin!2026', 'admin');
INSERT INTO portal.admins VALUES (2, 'editor', 'Edit0rPass', 'editor');
INSERT INTO portal.admins VALUES (3, 'webuser', 'Sunshine2026!', 'user');
GRANT ALL ON portal.* TO 'portaldb'@'localhost' IDENTIFIED BY 'dbpass123';
FLUSH PRIVILEGES;
SQL

sudo mkdir -p /var/www/html/portal
cat << 'PHPEOF' | sudo tee /var/www/html/portal/index.php
<?php
$conn = new mysqli("localhost","portaldb","dbpass123","portal");
$msg="";
if(isset($_POST['u'])&&isset($_POST['p'])){
    $r=$conn->query("SELECT * FROM admins WHERE username='".$_POST['u']."' AND password='".$_POST['p']."'");
    if($r&&$row=$r->fetch_assoc()){
        $msg="<h2>Welcome {$row['username']}!</h2><p>Role: {$row['role']}</p>";
        if($row['role']=='admin') $msg.="<p><a href='admin.php'>Admin Panel</a></p>";
    } else $msg="<p>Invalid login.</p>";
}
?>
<html><body><h1>Company Portal</h1>
<form method="POST">User:<input name="u"><br>Pass:<input type="password" name="p"><br>
<button>Login</button></form><?php echo $msg; ?></body></html>
PHPEOF

# Admin panel with file upload
cat << 'PHPEOF' | sudo tee /var/www/html/portal/admin.php
<?php
$msg="";
if(isset($_FILES['doc'])){
    $t="/var/www/html/portal/uploads/".basename($_FILES['doc']['name']);
    move_uploaded_file($_FILES['doc']['tmp_name'],$t);
    $msg="<p>Uploaded: <a href='uploads/".basename($_FILES['doc']['name'])."'>View</a></p>";
}
?>
<html><body><h1>Admin Panel - Document Upload</h1>
<form method="POST" enctype="multipart/form-data">
<input type="file" name="doc"><button>Upload</button></form>
<?php echo $msg; ?></body></html>
PHPEOF

sudo mkdir -p /var/www/html/portal/uploads
sudo chmod 777 /var/www/html/portal/uploads

echo "<h1>Company Website</h1><!-- Portal at /portal -->" | sudo tee /var/www/html/index.html

# Privilege escalation: sudo misconfiguration
echo "webuser ALL=(root) NOPASSWD: /usr/bin/less" | sudo tee /etc/sudoers.d/webuser

sudo iptables -F && sudo iptables -P INPUT ACCEPT
```

### Standalone 2: Windows 11 (192.168.244.xxx)

Run as Administrator in PowerShell:

```powershell
# Create flags
"LOCAL{" + -join ((48..57)+(65..90)+(97..122) | Get-Random -Count 32 | % {[char]$_}) + "}" | Out-File C:\Users\Public\local.txt
"PROOF{" + -join ((48..57)+(65..90)+(97..122) | Get-Random -Count 32 | % {[char]$_}) + "}" | Out-File C:\Users\Administrator\Desktop\proof.txt

# Create a low-privilege user
net user helpdesk H3lpd3sk! /add

# Install a vulnerable web server (Python-based for simplicity)
mkdir C:\WebApp
@"
from http.server import HTTPServer, CGIHTTPRequestHandler
import os
os.chdir('C:\\WebApp')
handler = CGIHTTPRequestHandler
handler.cgi_directories = ['/cgi-bin']
HTTPServer(('0.0.0.0', 8080), handler).serve_forever()
"@ | Out-File C:\WebApp\server.py

mkdir C:\WebApp\cgi-bin

# Create a CGI script with command injection
@"
#!python
import cgi, subprocess, os
print("Content-type: text/html\n")
form = cgi.FieldStorage()
if 'host' in form:
    host = form['host'].value
    result = subprocess.run(f'ping -n 2 {host}', shell=True, capture_output=True, text=True)
    print(f"<pre>{result.stdout}</pre>")
print('<form>Host: <input name="host"><button>Ping</button></form>')
"@ | Out-File C:\WebApp\cgi-bin\check.py

@"
<h1>IT Helpdesk Portal</h1>
<p><a href='/cgi-bin/check.py'>Network Diagnostic Tool</a></p>
<!-- Note: helpdesk credentials are helpdesk / H3lpd3sk! -->
"@ | Out-File C:\WebApp\index.html

# Start the web server (run in background)
Start-Process python -ArgumentList "C:\WebApp\server.py" -WindowStyle Hidden

# Privilege escalation: AlwaysInstallElevated
reg add HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated /t REG_DWORD /d 1 /f
reg add HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated /t REG_DWORD /d 1 /f

# Also store credentials for privesc path 2
cmdkey /add:fileserver /user:Administrator /pass:Admin2026!

# Disable firewall for lab
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

### AD Set: CentOS (192.168.244.131 / 172.16.0.1) as simulated "AD Machine 1"

```bash
sudo systemctl stop firewalld && sudo systemctl disable firewalld
sudo dnf install iptables-services httpd php -y
sudo systemctl enable --now sshd httpd

sudo ip addr add 172.16.0.1/24 dev ens224 2>/dev/null
sudo ip link set ens224 up

# Create flags
echo "PROOF_AD1{$(openssl rand -hex 16)}" | sudo tee /root/proof.txt
sudo chmod 600 /root/proof.txt

# Web server with credentials leak
echo "<h1>Internal Dashboard</h1>" | sudo tee /var/www/html/index.html
sudo mkdir -p /var/www/html/.backup
echo "SSH credentials for Ubuntu server:" | sudo tee /var/www/html/.backup/notes.txt
echo "user: srvadmin / pass: Srv@dmin2026!" | sudo tee -a /var/www/html/.backup/notes.txt
echo "IP: 172.16.0.2" | sudo tee -a /var/www/html/.backup/notes.txt

# Create initial access user (weak SSH creds found via brute force)
sudo useradd -m -s /bin/bash operator
echo "operator:operator" | sudo chpasswd

# Privesc: SUID nmap (gives root through interactive mode emulation)
sudo cp /usr/bin/bash /usr/local/bin/monitor_tool
sudo chmod u+s /usr/local/bin/monitor_tool

# Enable IP forwarding for pivot
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf

sudo sed -i 's/#AllowTcpForwarding yes/AllowTcpForwarding yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# iptables
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p icmp -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT ACCEPT
sudo iptables -A FORWARD -s 192.168.244.0/24 -d 172.16.0.0/24 -j ACCEPT
sudo iptables -A FORWARD -s 172.16.0.0/24 -d 192.168.244.0/24 -j ACCEPT
sudo iptables -P FORWARD DROP
sudo iptables -t nat -A POSTROUTING -s 192.168.244.0/24 -o ens224 -j MASQUERADE
sudo service iptables save
```

### AD Set: Ubuntu (172.16.0.2) as simulated "AD Machine 2 / DC"

```bash
sudo ip addr add 172.16.0.2/24 dev ens38 2>/dev/null
sudo ip link set ens38 up
sudo apt update && sudo apt install openssh-server -y
sudo systemctl enable --now ssh

echo "PROOF_AD2{$(openssl rand -hex 16)}" | sudo tee /root/proof.txt
sudo chmod 600 /root/proof.txt

sudo useradd -m -s /bin/bash srvadmin
echo "srvadmin:Srv@dmin2026!" | sudo chpasswd

# Privesc: writable cron + SUID python
sudo mkdir -p /opt/scripts
echo '#!/bin/bash' | sudo tee /opt/scripts/health_check.sh
echo 'echo "check ran" >> /tmp/health.log' | sudo tee -a /opt/scripts/health_check.sh
sudo chmod 777 /opt/scripts/health_check.sh
echo "*/2 * * * * root /opt/scripts/health_check.sh" | sudo tee /etc/cron.d/health

sudo cp /usr/bin/python3 /usr/local/bin/python3_monitor
sudo chmod u+s /usr/local/bin/python3_monitor

# Firewall: only internal
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -s 172.16.0.0/24 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT ACCEPT
```

---

## START THE TIMER

**Time limit: 8 hours.** Write down your start time.

You know:
- Your Kali IP: 192.168.244.129
- Network range: 192.168.244.0/24
- There's also an internal network (you don't know the range yet)

**Begin with: `nmap -sn 192.168.244.0/24`**

---

## Scoring Guide

### After you finish (or time runs out), check your score:

```
STANDALONE 1 — Debian (192.168.244.132):
  □ local.txt captured (cat /home/local.txt)                     10 pts
  □ proof.txt captured (cat /root/proof.txt as root)             10 pts
  □ Proof screenshot: flag + whoami + ip a                       required

STANDALONE 2 — Windows (192.168.244.xxx):
  □ local.txt captured (type C:\Users\Public\local.txt)          10 pts
  □ proof.txt captured (type ...\proof.txt as admin)             10 pts
  □ Proof screenshot: flag + whoami + ipconfig                   required

AD SET — CentOS → Ubuntu:
  □ CentOS proof.txt captured                                    }
  □ Ubuntu proof.txt captured                                    } 40 pts
  □ Both required for ANY points                                 } (all or nothing)
  □ Proof screenshots from both machines                         required

TOTAL: _____ / 80
PASS:  56 / 80 (70%)
```

### Grading your performance

```
80/80 in < 4 hours:  Excellent — you're ready for the exam
80/80 in < 8 hours:  Good — practice speed
60-70/80:            Decent — review what you missed
40-60/80:            Needs work — review the modules for your weak areas
< 40/80:             Go back to individual labs and practice fundamentals
```

---

## Hints (only read if stuck for 30+ minutes)

<details>
<summary>Standalone 1 Hint 1</summary>
Check FTP for anonymous access. There's a file with information about the web application.
</details>

<details>
<summary>Standalone 1 Hint 2</summary>
The web portal login is vulnerable to SQL injection. Try authentication bypass to access the admin panel, then use the file upload feature.
</details>

<details>
<summary>Standalone 1 Hint 3</summary>
For privilege escalation, check sudo -l. The user can run a specific command as root. Check GTFOBins for that command.
</details>

<details>
<summary>Standalone 2 Hint 1</summary>
There's a web server running on a non-standard port. Run a full port scan to find it.
</details>

<details>
<summary>Standalone 2 Hint 2</summary>
The web application has a diagnostic tool with command injection. Check the HTML source for credential hints.
</details>

<details>
<summary>Standalone 2 Hint 3</summary>
For privilege escalation, check AlwaysInstallElevated registry keys and stored credentials (cmdkey /list).
</details>

<details>
<summary>AD Set Hint 1</summary>
CentOS has a web server. Run directory brute force with dotfile detection. Try gobuster with a wordlist that includes entries starting with dots.
</details>

<details>
<summary>AD Set Hint 2</summary>
CentOS has a user with a very weak password. Think about common username=password combinations.
</details>

<details>
<summary>AD Set Hint 3</summary>
To reach Ubuntu, you need an SSH tunnel through CentOS. The credentials for Ubuntu are in the hidden web directory on CentOS.
</details>

---

## After the Simulation

### Write a report

Practice the full report-writing workflow:

```
1. Organize your notes and screenshots (30 minutes)
2. Write the report using the OffSec template (2-3 hours)
3. For each machine: enumeration → exploitation → escalation
4. Include every command and screenshot
5. Export as PDF
```

### Debrief yourself

```
□ What machines did I complete? What did I miss?
□ Where did I spend the most time? Was that the right allocation?
□ What enumeration did I skip that would have helped?
□ Did I document as I went, or did I scramble to remember later?
□ Did I take breaks? Did I get frustrated and lose focus?
□ What would I do differently next time?
```

---

## Cleanup

### Debian
```bash
sudo mysql -e "DROP DATABASE portal; DROP USER 'portaldb'@'localhost';"
sudo userdel -r webuser && sudo rm -rf /var/www/html/portal /home/local.txt /root/proof.txt
sudo rm /etc/sudoers.d/webuser
```

### Windows
```powershell
net user helpdesk /delete
reg delete HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated /f
reg delete HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated /f
cmdkey /delete:fileserver
Stop-Process -Name python -Force 2>$null
Remove-Item -Recurse C:\WebApp
Remove-Item C:\Users\Public\local.txt, C:\Users\Administrator\Desktop\proof.txt
```

### CentOS
```bash
sudo userdel -r operator && sudo rm -f /usr/local/bin/monitor_tool /root/proof.txt
sudo rm -rf /var/www/html/.backup
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -P INPUT ACCEPT && sudo iptables -P FORWARD ACCEPT
```

### Ubuntu
```bash
sudo userdel -r srvadmin && sudo rm -f /usr/local/bin/python3_monitor /root/proof.txt
sudo rm -f /opt/scripts/health_check.sh /etc/cron.d/health
sudo iptables -F && sudo iptables -P INPUT ACCEPT
```
