# Lab: SSH Tunnel Drill

## Objective

Practice every SSH tunnel type in isolation so you can set each one up from memory. This lab has six exercises — one for each tunnel technique. Complete each one, then do them all timed.

## Prerequisites

Your VMs need to be running with CentOS and Ubuntu on the 172.16.0.0/24 internal network.

### Quick setup for services to tunnel to

**On CentOS (192.168.244.131 / 172.16.0.1):**

```bash
# Create a user
sudo useradd -m -s /bin/bash tunneluser
echo "tunneluser:TunnelP@ss!" | sudo chpasswd

# Start a web server on port 8080
sudo dnf install python3 -y
mkdir -p /tmp/webapp
echo "<h1>CentOS Internal App</h1><p>You reached this through a tunnel!</p>" > /tmp/webapp/index.html
cd /tmp/webapp && nohup python3 -m http.server 8080 &>/dev/null &

# Allow SSH forwarding
sudo sed -i 's/#AllowTcpForwarding yes/AllowTcpForwarding yes/' /etc/ssh/sshd_config
sudo sed -i 's/#GatewayPorts no/GatewayPorts yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

**On Ubuntu (172.16.0.2):**

```bash
# Create a user
sudo useradd -m -s /bin/bash tunneluser
echo "tunneluser:TunnelP@ss!" | sudo chpasswd

# Start a web server on port 80
sudo apt install apache2 -y
echo "<h1>Ubuntu Internal Server</h1><p>FLAG{tunnel_drill_complete}</p>" | sudo tee /var/www/html/index.html
sudo systemctl start apache2

# Start a service on port 3000
echo '{"secret":"FLAG{api_through_tunnel}"}' > /tmp/api_response.txt
cd /tmp && nohup python3 -c "
from http.server import HTTPServer, SimpleHTTPRequestHandler
class H(SimpleHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(open('/tmp/api_response.txt','rb').read())
HTTPServer(('0.0.0.0',3000),H).serve_forever()
" &>/dev/null &

# Allow SSH forwarding
sudo sed -i 's/#AllowTcpForwarding yes/AllowTcpForwarding yes/' /etc/ssh/sshd_config
sudo sed -i 's/#GatewayPorts no/GatewayPorts yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh

# Firewall — only allow from 172.16.0.0/24
sudo iptables -F
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -s 172.16.0.0/24 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT ACCEPT
```

### Verify setup

```bash
# From CentOS — can reach Ubuntu
curl http://172.16.0.2
curl http://172.16.0.2:3000

# From Kali — can't reach Ubuntu
curl http://172.16.0.2      # fails (no route or blocked)
```

---

## Exercise 1: Local Port Forward — One Service

**Goal:** Access CentOS's web app (port 8080) from Kali.

```bash
# From Kali
ssh -L 8080:localhost:8080 tunneluser@192.168.244.131
# Password: TunnelP@ss!

# In another Kali terminal
curl http://localhost:8080
# You should see: "CentOS Internal App"
```

**What happened:** Kali port 8080 → SSH tunnel → CentOS localhost:8080. The keyword `localhost` in the command refers to CentOS's localhost (the far end of the tunnel), not Kali's.

**Exit the SSH session when done.**

---

## Exercise 2: Local Port Forward — Through a Pivot

**Goal:** Access Ubuntu's web server (172.16.0.2:80) from Kali, using CentOS as a pivot.

```bash
# From Kali
ssh -L 9090:172.16.0.2:80 tunneluser@192.168.244.131
# Password: TunnelP@ss!

# In another terminal
curl http://localhost:9090
# You should see: "Ubuntu Internal Server" with the flag
```

**What happened:** Kali port 9090 → SSH tunnel → CentOS → forwards to 172.16.0.2:80. CentOS acts as the relay — it's on both networks.

---

## Exercise 3: Multi-Port Forward

**Goal:** Access Ubuntu's web server AND API from Kali in a single SSH connection.

```bash
# From Kali
ssh -L 9090:172.16.0.2:80 -L 3000:172.16.0.2:3000 -L 2222:172.16.0.2:22 tunneluser@192.168.244.131

# Test each one
curl http://localhost:9090       # Ubuntu web → flag
curl http://localhost:3000       # Ubuntu API → secret
ssh tunneluser@localhost -p 2222 # SSH to Ubuntu through the tunnel
```

**Three services through one SSH connection.** Each `-L` adds another forwarded port.

---

## Exercise 4: Dynamic Port Forward (SOCKS Proxy)

**Goal:** Scan and interact with the entire 172.16.0.0/24 network from Kali.

```bash
# From Kali
ssh -D 9050 -fN tunneluser@192.168.244.131
# Password: TunnelP@ss!
# -f = background, -N = no shell

# Configure proxychains
sudo nano /etc/proxychains4.conf
# Last line: socks5 127.0.0.1 9050

# Scan through the tunnel
proxychains nmap -sT -Pn 172.16.0.2 -p 22,80,3000

# Access services through the tunnel
proxychains curl http://172.16.0.2
proxychains curl http://172.16.0.2:3000

# SSH through the tunnel
proxychains ssh tunneluser@172.16.0.2
```

**The difference from local forward:** Dynamic forward creates a SOCKS proxy — ANY destination and ANY port works through it. Local forward is one specific destination and port.

**Kill the background tunnel when done:**
```bash
ps aux | grep "ssh -D"
kill PID
```

---

## Exercise 5: SSH Jump (-J)

**Goal:** SSH directly to Ubuntu through CentOS in one command.

```bash
# From Kali
ssh -J tunneluser@192.168.244.131 tunneluser@172.16.0.2
# CentOS password: TunnelP@ss!
# Ubuntu password: TunnelP@ss!

# You're now on Ubuntu
whoami
hostname
curl http://localhost:3000
```

**What `-J` does:** SSH connects to CentOS first, then from CentOS connects to Ubuntu. One command, two hops.

**Combine jump with SOCKS:**
```bash
ssh -D 9050 -J tunneluser@192.168.244.131 tunneluser@172.16.0.2
# SOCKS proxy exits from UBUNTU, not CentOS
# Useful if Ubuntu can reach networks that CentOS can't
```

---

## Exercise 6: Reverse Tunnel

**Goal:** Make Ubuntu's web server accessible on Kali, initiated FROM Ubuntu.

```bash
# Step 1: Make sure SSH is running on Kali
sudo systemctl start ssh

# Step 2: From CentOS, SSH to Ubuntu
ssh tunneluser@172.16.0.2

# Step 3: From Ubuntu, create a reverse tunnel back to Kali
ssh -R 7777:localhost:80 YOUR_KALI_USER@192.168.244.129
# This goes: Ubuntu → CentOS → Kali (Ubuntu can reach CentOS, CentOS can reach Kali)

# Step 4: On Kali, test
curl http://localhost:7777
# You should see Ubuntu's web page
```

**What happened:** Ubuntu connected OUT to Kali and said "open port 7777 on Kali, forward anything that hits it back to my port 80." The tunnel is initiated by the INTERNAL machine, bypassing any firewall that blocks inbound connections.

**Reverse SOCKS proxy:**
```bash
# From Ubuntu
ssh -R 1080 YOUR_KALI_USER@192.168.244.129
# SOCKS proxy on Kali port 1080, exits from Ubuntu
```

---

## Timed Challenge

Reset everything (exit all SSH sessions, kill background tunnels). Then:

**Complete all 6 exercises in under 15 minutes.**

```
□ Exercise 1: Local forward to CentOS:8080          (target: 1 min)
□ Exercise 2: Local forward through CentOS to Ubuntu (target: 1 min)
□ Exercise 3: Multi-port forward (3 ports)           (target: 2 min)
□ Exercise 4: Dynamic SOCKS + proxychains scan       (target: 3 min)
□ Exercise 5: SSH jump to Ubuntu                     (target: 1 min)
□ Exercise 6: Reverse tunnel from Ubuntu             (target: 3 min)
```

If you can do all six in under 15 minutes, you'll handle any pivoting scenario on the exam.

---

## Cleanup

```bash
# On CentOS
sudo userdel -r tunneluser
sudo kill $(pgrep -f "http.server 8080") 2>/dev/null

# On Ubuntu
sudo userdel -r tunneluser
sudo kill $(pgrep -f "3000") 2>/dev/null
sudo iptables -F && sudo iptables -P INPUT ACCEPT
```
