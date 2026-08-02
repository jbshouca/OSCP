# Supplementary: Linux Administration for Security Professionals

Whether you're attacking or defending Linux systems, you need solid administration skills. This covers the essential sysadmin knowledge that every cybersecurity professional uses daily.

---

## User and Permission Management

### Understanding /etc/passwd

```bash
cat /etc/passwd | head -5
# root:x:0:0:root:/root:/bin/bash
```

```
root  : x    : 0   : 0   : root    : /root     : /bin/bash
│       │      │     │      │         │            │
│       │      │     │      │         │            └── login shell
│       │      │     │      │         └── home directory
│       │      │     │      └── comment/description
│       │      │     └── primary group ID (GID)
│       │      └── user ID (UID) — 0 = root
│       └── password placeholder (actual hash in /etc/shadow)
└── username
```

**Security-relevant entries:**
```bash
# Users with login shells (potential targets)
grep -E "/bin/(bash|sh|zsh|fish)" /etc/passwd

# Users with UID 0 (root-equivalent — should only be root)
awk -F: '$3 == 0 {print $1}' /etc/passwd

# Service accounts (nologin shell — can't log in interactively)
grep "nologin\|false" /etc/passwd
```

### Understanding /etc/shadow

```bash
sudo cat /etc/shadow | head -3
# root:$6$salt$hash...:19500:0:99999:7:::
```

```
root : $6$salt$hash : 19500 : 0 : 99999 : 7 :  :  :
│      │               │       │    │       │
│      │               │       │    │       └── warning days before expiry
│      │               │       │    └── max days between password changes
│      │               │       └── min days between changes
│      │               └── last password change (days since epoch)
│      └── password hash ($6$ = SHA-512, $5$ = SHA-256, $1$ = MD5)
└── username
```

```
Hash prefixes:
$1$   MD5 (old, weak)
$5$   SHA-256
$6$   SHA-512 (current standard)
$2y$  bcrypt (some systems)
!     Account locked (can't log in with password)
*     Account disabled
```

### Managing users

```bash
# Create a user
sudo useradd -m -s /bin/bash newuser       # -m = create home dir, -s = set shell
echo "newuser:password" | sudo chpasswd     # set password non-interactively

# Modify a user
sudo usermod -aG sudo newuser               # add to sudo group
sudo usermod -s /usr/sbin/nologin newuser   # disable login
sudo usermod -L newuser                      # lock account

# Delete a user
sudo userdel -r newuser                      # -r = remove home directory

# Check user info
id newuser
groups newuser
```

### File permissions deep dive

```bash
ls -la /etc/passwd
# -rw-r--r-- 1 root root 1234 Jul 31 10:00 /etc/passwd
```

```
-rw-r--r--
│├──┤├──┤├──┤
│ │   │   │
│ │   │   └── Other:  r-- (read only)
│ │   └────── Group:  r-- (read only)
│ └────────── Owner:  rw- (read + write)
└──────────── Type:   - (regular file), d (directory), l (symlink)
```

**Numeric permissions:**
```
r = 4    read
w = 2    write
x = 1    execute

chmod 755 = rwxr-xr-x  (owner: full, group: read+exec, other: read+exec)
chmod 644 = rw-r--r--  (owner: read+write, group: read, other: read)
chmod 600 = rw-------  (owner only)
chmod 777 = rwxrwxrwx  (EVERYTHING — dangerous, never in production)
chmod 4755 = rwsr-xr-x (SUID set — runs as owner regardless of who executes)
```

**Special permissions for pentesting:**
```
SUID (4xxx): File executes as the FILE OWNER, not the person running it
  chmod u+s file  or  chmod 4755 file
  Shows as: -rwsr-xr-x
  Find them: find / -perm -4000 -type f 2>/dev/null
  
SGID (2xxx): File executes with the FILE GROUP's permissions
  chmod g+s file  or  chmod 2755 file
  Shows as: -rwxr-sr-x
  
Sticky bit (1xxx): Only the file owner can delete files in the directory
  chmod +t dir  or  chmod 1777 dir
  Shows as: drwxrwxrwt
  Example: /tmp (anyone can create files, only owner can delete their own)
```

---

## Process Management

```bash
# View running processes
ps aux                                     # all processes with details
ps aux | grep apache                       # filter for a specific process
ps -ef --forest                            # show process tree (parent/child)

# Top (real-time monitoring)
top                                        # real-time process viewer
htop                                       # better version (install if not present)

# Find what's listening on ports
ss -tlnp                                   # TCP listening ports with process info
ss -ulnp                                   # UDP listening ports
netstat -tlnp                              # older equivalent

# Find a process by port
ss -tlnp | grep :80                        # what's on port 80?
lsof -i :80                                # alternative

# Kill processes
kill PID                                    # graceful termination (SIGTERM)
kill -9 PID                                 # force kill (SIGKILL)
killall process_name                        # kill all instances by name
pkill -f "pattern"                          # kill by command pattern

# Background and foreground
command &                                   # run in background
Ctrl+Z                                     # suspend current process
bg                                          # resume in background
fg                                          # bring to foreground
jobs                                        # list background jobs
```

**For pentesting:** Check `ps aux` on every compromised machine. Hidden services, scheduled tasks, and custom applications reveal what the machine does and what else it connects to.

---

## systemd and Services

```bash
# Service management
sudo systemctl start service_name
sudo systemctl stop service_name
sudo systemctl restart service_name
sudo systemctl status service_name
sudo systemctl enable service_name          # start on boot
sudo systemctl disable service_name         # don't start on boot

# List all services
systemctl list-units --type=service
systemctl list-units --type=service --state=running

# View service configuration
systemctl cat service_name

# Check service logs
journalctl -u service_name                  # all logs for this service
journalctl -u service_name -f               # follow (live tail)
journalctl -u service_name --since "1 hour ago"
journalctl -u service_name --since today

# Check what ports a service listens on
systemctl status service_name | grep -i "pid"
ss -tlnp | grep PID_NUMBER
```

### Creating a custom service

```bash
# Create the service file
sudo nano /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=www-data
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
```

**For pentesting:** Custom services often run vulnerable applications. Check `/etc/systemd/system/` and `/lib/systemd/system/` for non-standard services. The `ExecStart` line shows what binary runs and as which user.

---

## Disk and Filesystem Management

```bash
# View disk usage
df -h                                       # filesystem usage (human readable)
du -sh /var/log/                            # size of a directory
du -sh /*  2>/dev/null | sort -h            # largest top-level directories

# View partitions and disks
lsblk                                       # block devices tree
fdisk -l                                     # partition table

# Mount filesystems
sudo mount /dev/sdb1 /mnt                   # mount a partition
sudo mount -t cifs //SERVER/share /mnt/smb -o username=user  # mount SMB
sudo mount -t nfs SERVER:/export /mnt/nfs   # mount NFS
sudo mount -o loop image.iso /mnt/iso       # mount an ISO

# Unmount
sudo umount /mnt

# View fstab (auto-mount on boot)
cat /etc/fstab

# Check mounted filesystems
mount | grep -v "tmpfs\|proc\|sys"
```

---

## Log Files — Where Everything Happens

### Important log locations

```
/var/log/syslog           General system log (Debian/Ubuntu)
/var/log/messages         General system log (CentOS/RHEL)
/var/log/auth.log         Authentication events (Debian/Ubuntu)
/var/log/secure           Authentication events (CentOS/RHEL)
/var/log/kern.log         Kernel messages
/var/log/dmesg            Boot messages
/var/log/apache2/         Apache web server logs
/var/log/nginx/           Nginx web server logs
/var/log/mysql/           MySQL logs
/var/log/cron             Cron job execution log
/var/log/mail.log         Mail server logs
/var/log/faillog          Failed login attempts
/var/log/lastlog          Last login for each user
```

### Reading and searching logs

```bash
# Tail (last N lines — most recent events)
tail -20 /var/log/auth.log
tail -f /var/log/auth.log                   # follow in real time

# Search logs
grep "Failed" /var/log/auth.log
grep -i "error" /var/log/syslog | tail -20

# Time-based search with journalctl
journalctl --since "2026-07-31 10:00:00" --until "2026-07-31 11:00:00"
journalctl --since "1 hour ago"
journalctl --since today

# Check who logged in
last                                         # recent logins
lastb                                        # failed login attempts
who                                          # currently logged in users
w                                            # who's logged in and what they're doing
```

### Log rotation

```bash
# Logs rotate to prevent disk filling up
ls /var/log/auth.log*
# auth.log       (current)
# auth.log.1     (previous)
# auth.log.2.gz  (older, compressed)

# Search across rotated logs
zgrep "Failed" /var/log/auth.log*
# zgrep reads compressed files
```

**For pentesting:** After getting root, check logs to see if your activity was logged. Understand what defenders can see. Also check logs for OTHER users' activities — they might reveal passwords, paths, or patterns.

---

## SSH Administration

```bash
# SSH config file
/etc/ssh/sshd_config                        # server config (the SSH daemon)
~/.ssh/config                               # client config (your SSH connections)

# Key settings in sshd_config
PermitRootLogin no                          # disable root SSH login
PasswordAuthentication yes                   # allow password login
PubkeyAuthentication yes                    # allow key-based login
AllowUsers user1 user2                      # whitelist specific users
AllowTcpForwarding yes                      # enable port forwarding
GatewayPorts yes                            # allow remote port binds
MaxAuthTries 3                              # limit login attempts

# Generate SSH keys
ssh-keygen -t ed25519 -C "comment"          # modern, secure
ssh-keygen -t rsa -b 4096                   # RSA, widely compatible

# Copy your public key to a server (passwordless login)
ssh-copy-id user@server

# SSH client config (saves typing)
cat ~/.ssh/config
# Host webserver
#     HostName 192.168.244.132
#     User admin
#     Port 22
#     IdentityFile ~/.ssh/id_ed25519
#
# Now just: ssh webserver
```

---

## Networking Commands

```bash
# IP address and interfaces
ip a                                         # all interfaces and IPs
ip a show eth0                               # specific interface
ip link set eth0 up/down                     # enable/disable interface

# Routing
ip route                                     # routing table
ip route add 172.16.0.0/24 via 192.168.244.131   # add a route
ip route del 172.16.0.0/24                   # remove a route

# DNS
cat /etc/resolv.conf                         # DNS servers
nslookup domain.com                          # look up a domain
dig domain.com                               # detailed DNS query

# Network connections
ss -tlnp                                     # listening TCP ports
ss -tanp                                     # all TCP connections
ss -ulnp                                     # listening UDP ports

# Firewall (iptables)
sudo iptables -L -n -v                       # list rules
sudo iptables -S                             # list rules in command format

# Testing connectivity
ping -c 4 target                             # ICMP echo
traceroute target                            # trace the path
mtr target                                   # continuous traceroute
nc -zvn target port                          # test TCP port
curl -I http://target                        # test HTTP
```

---

## Package Management

### Debian/Ubuntu (apt)

```bash
sudo apt update                              # refresh package lists
sudo apt upgrade                             # upgrade installed packages
sudo apt install package_name                # install a package
sudo apt remove package_name                 # remove (keep config)
sudo apt purge package_name                  # remove + delete config
apt search keyword                           # search for packages
apt list --installed                         # list installed packages
dpkg -l | grep keyword                       # search installed packages
dpkg -L package_name                         # list files in a package
```

### CentOS/RHEL (dnf/yum)

```bash
sudo dnf update                              # upgrade all packages
sudo dnf install package_name
sudo dnf remove package_name
dnf search keyword
dnf list installed
rpm -qa | grep keyword
rpm -ql package_name                         # list files in a package
```

---

## Cron Jobs (Scheduled Tasks)

```bash
# View current user's cron
crontab -l

# Edit current user's cron
crontab -e

# System-wide cron
cat /etc/crontab
ls /etc/cron.d/
ls /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.weekly/ /etc/cron.monthly/

# Cron format
# MIN HOUR DAY MONTH DOW  command
  */5  *    *   *     *    /opt/scripts/backup.sh    # every 5 minutes
  0    2    *   *     *    /usr/bin/apt update        # daily at 2:00 AM
  0    0    1   *     *    /opt/monthly_report.sh     # 1st of every month
  30   8    *   *     1-5  /opt/workday_start.sh      # weekdays at 8:30 AM
```

---

## Quick Reference: Commands Every Security Professional Uses

```bash
# System info
uname -a                    # kernel version
cat /etc/os-release         # OS details
hostnamectl                 # hostname and OS info

# User info
whoami && id                # who am I, what groups
sudo -l                     # what can I sudo
cat /etc/passwd             # all users
cat /etc/group              # all groups

# Network
ip a                        # interfaces and IPs
ss -tlnp                    # listening ports
ip route                    # routing table

# Files
find / -name "filename" 2>/dev/null          # find a file
find / -perm -4000 -type f 2>/dev/null       # find SUID files
find / -writable -type f 2>/dev/null         # find writable files
grep -r "pattern" /path/                      # search in files

# Processes
ps aux                      # all processes
ss -tlnp                    # what's listening
systemctl list-units --type=service --state=running

# Logs
tail -f /var/log/auth.log   # watch auth events live
journalctl -f               # watch all logs live
last                        # recent logins
```
