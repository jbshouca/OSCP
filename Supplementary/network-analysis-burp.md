# Supplementary: Network Analysis and Web Proxying

Two tools every cybersecurity professional must know: **Wireshark** for analyzing network traffic and **Burp Suite** for testing web applications. These aren't OSCP-exam-specific, but they're used daily in real engagements and expected knowledge for any security role.

---

## Wireshark and Packet Analysis

### What Wireshark does

Wireshark captures and analyzes network packets — every byte of data flowing across a network interface. It lets you see exactly what's being sent and received, decode protocols, and find security issues like cleartext credentials, suspicious connections, and attack traffic.

### When to use it

```
Pentesting:
  - Capture credentials sent in cleartext (HTTP, FTP, Telnet, SMTP)
  - Analyze how an exploit communicates
  - Debug why your reverse shell isn't connecting
  - Understand a protocol before writing a custom exploit

Incident response:
  - Analyze captured network traffic for indicators of compromise
  - Identify C2 (command and control) communications
  - Find data exfiltration
  - Reconstruct attacker activity

General:
  - Debug network connectivity issues
  - Understand how protocols work at the packet level
  - Verify encryption is working (TLS traffic should be unreadable)
```

### Essential Wireshark filters

```
# Filter by IP
ip.addr == 192.168.244.132              Show all traffic to/from this IP
ip.src == 192.168.244.129               Show traffic FROM this IP
ip.dst == 192.168.244.132               Show traffic TO this IP

# Filter by port
tcp.port == 80                          HTTP traffic
tcp.port == 443                         HTTPS traffic
tcp.port == 22                          SSH traffic
tcp.port == 445                         SMB traffic
udp.port == 53                          DNS traffic

# Filter by protocol
http                                    All HTTP traffic
dns                                     All DNS traffic
smb                                     All SMB traffic
ftp                                     All FTP traffic
tcp                                     All TCP traffic
icmp                                    Ping/ICMP traffic

# Useful combinations
http.request.method == "POST"           Only HTTP POST requests (login forms!)
http.request.uri contains "login"       Requests to login pages
ftp.request.command == "PASS"           FTP password transmissions
dns.qry.name contains "evil"           DNS queries for suspicious domains
tcp.flags.syn == 1 && tcp.flags.ack == 0  SYN packets only (port scans)

# Find cleartext credentials
http.request.method == "POST" && http contains "password"
ftp.request.command == "USER" || ftp.request.command == "PASS"

# Follow a TCP stream (see the full conversation)
# Right-click any packet → Follow → TCP Stream
```

### tcpdump (command-line alternative)

When you don't have a GUI (SSH'd into a server), use tcpdump:

```bash
# Capture all traffic
sudo tcpdump -i eth0

# Capture and save to a file (open in Wireshark later)
sudo tcpdump -i eth0 -w capture.pcap

# Filter by host
sudo tcpdump -i eth0 host 192.168.244.132

# Filter by port
sudo tcpdump -i eth0 port 80
sudo tcpdump -i eth0 port 61616

# Filter by protocol
sudo tcpdump -i eth0 tcp
sudo tcpdump -i eth0 icmp

# Show packet contents in ASCII
sudo tcpdump -i eth0 -A port 80

# Show packet contents in hex and ASCII
sudo tcpdump -i eth0 -X port 80

# Capture only N packets
sudo tcpdump -i eth0 -c 100 -w capture.pcap

# Capture with a specific filter
sudo tcpdump -i eth0 'src host 192.168.244.132 and dst port 4444'

# Read a saved capture
tcpdump -r capture.pcap
tcpdump -r capture.pcap -A | grep -i "password"
```

### Practical: Finding credentials in a capture

```bash
# Capture FTP traffic (FTP sends credentials in cleartext)
sudo tcpdump -i eth0 -A port 21 | grep -i "USER\|PASS"
# Output:
# USER admin
# PASS SecretPassword123

# Capture HTTP POST data (login forms)
sudo tcpdump -i eth0 -A port 80 | grep -i "user\|pass\|login"
# Output:
# username=admin&password=Welcome123

# Capture SMTP credentials
sudo tcpdump -i eth0 -A port 25 | grep -i "AUTH\|LOGIN"
```

---

## Burp Suite

### What Burp Suite does

Burp Suite is a web proxy — it sits between your browser and the target web server, intercepting every request and response. You can view, modify, and replay HTTP requests. It's the primary tool for manual web application testing.

### Setup

```
1. Open Burp Suite (comes with Kali)
   Applications → Web Application Analysis → Burp Suite

2. Configure your browser to use Burp as a proxy:
   Proxy settings: 127.0.0.1:8080

3. For HTTPS: install Burp's CA certificate in your browser
   Browse to http://burp → download the CA certificate → install it
```

### Core tools in Burp Suite

**Proxy (the foundation):**
```
Intercept requests between your browser and the target.
- View the full HTTP request (method, headers, body, cookies)
- Modify any part before it reaches the server
- Forward the (modified) request
- Drop requests you don't want to send

Use case: Change a hidden form field, modify a cookie value,
add an injection payload to a POST parameter
```

**Repeater (manual testing):**
```
Send a request, get a response, modify, send again.
- Perfect for testing injection payloads iteratively
- No need to re-click through the browser each time
- Compare responses side by side

Use case: Testing SQLi payloads one at a time,
finding the right command injection syntax
```

**Intruder (automated attacks):**
```
Automate sending many variations of a request.
- Brute force login forms
- Fuzz parameters with wordlists
- Test many injection payloads automatically
- Identify different responses (length, status code)

Use case: Password brute forcing, directory enumeration through parameters
Note: Community Edition is rate-limited (very slow)
```

**Decoder:**
```
Encode/decode data in various formats.
- Base64, URL encoding, HTML encoding, hex
- Useful for decoding cookies, tokens, and encoded parameters

Use case: Decode a base64 cookie to see if it contains user info
```

### Burp Suite workflow for web testing

```
1. Browse the target normally with Burp proxying
   - Burp records every request in the HTTP history
   - This builds your "site map" automatically

2. Review the site map
   - Look at every endpoint Burp discovered
   - Check for interesting parameters, cookies, hidden fields

3. Send interesting requests to Repeater
   - Right-click → Send to Repeater
   - Test injection payloads manually

4. For each parameter, test:
   - Single quote (') → SQLi?
   - <script>alert(1)</script> → XSS?
   - ../../etc/passwd → LFI?
   - ;whoami → command injection?

5. If you find a vulnerability, use Repeater to refine the exploit
   - Adjust the payload
   - Handle encoding issues
   - Extract data step by step
```

### Intercepting and modifying requests

```
Scenario: A web form has a hidden field "role=user"
You want to change it to "role=admin"

1. Turn on Intercept in Burp Proxy
2. Submit the form in your browser
3. Burp catches the request and shows it to you:
   POST /register HTTP/1.1
   Host: target.com
   
   username=me&password=pass&role=user

4. Change "role=user" to "role=admin"
5. Click Forward
6. The server receives your modified request
7. You might now have an admin account
```

### Using Burp for SQL injection

```
1. Browse to the login page, submit a test login
2. Burp catches the POST request
3. Send it to Repeater (Ctrl+R)
4. In Repeater, modify the username parameter:
   
   Original: username=admin&password=test
   Modified: username=admin' OR 1=1--&password=test

5. Click Send
6. Check the response — did you bypass authentication?
7. Try different payloads:
   username=' OR 1=1--
   username=admin'--
   username=' UNION SELECT 1,2,3--
```

---

## Practical: Debug a reverse shell with tcpdump

When your reverse shell isn't connecting, use tcpdump to see what's happening:

```bash
# Terminal 1: Start your listener
nc -lvnp 4444

# Terminal 2: Watch for the connection
sudo tcpdump -i eth0 port 4444

# Terminal 3: Trigger the reverse shell on the target
```

**If you see SYN packets arriving but no shell:**
```
The target is trying to connect but your listener isn't accepting.
Check: is nc listening? (ss -tlnp | grep 4444)
Check: is your firewall blocking? (sudo iptables -L -n)
```

**If you see NO packets:**
```
The target can't reach you.
Check: is the target's firewall blocking outbound to port 4444?
Try: use port 443 or 80 instead (commonly allowed outbound)
Check: did you use the right Kali IP in the payload?
```

**If you see RST packets:**
```
Something is actively rejecting the connection.
Check: is another service using port 4444?
Check: is your firewall sending RST?
```
