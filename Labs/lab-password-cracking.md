# Lab: Password Cracking Hands-On

## Objective

Practice identifying, extracting, and cracking password hashes using hashcat and john. Generate real hashes, crack them, and understand the tools deeply.

---

## Setup: Generate Hashes to Crack

### Create a hash file with multiple types

On Kali:

```bash
mkdir ~/cracking-lab && cd ~/cracking-lab

# Generate MD5 hashes
echo -n "password123" | md5sum | awk '{print $1}' > md5_hashes.txt
echo -n "welcome1" | md5sum | awk '{print $1}' >> md5_hashes.txt
echo -n "Summer2026!" | md5sum | awk '{print $1}' >> md5_hashes.txt

# Generate SHA1 hashes
echo -n "letmein" | sha1sum | awk '{print $1}' > sha1_hashes.txt
echo -n "trustno1" | sha1sum | awk '{print $1}' >> sha1_hashes.txt

# Generate SHA256 hashes
echo -n "dragon" | sha256sum | awk '{print $1}' > sha256_hashes.txt
echo -n "monkey" | sha256sum | awk '{print $1}' >> sha256_hashes.txt

# Generate NTLM hashes (Windows passwords)
python3 -c "
import hashlib
passwords = ['Password1!', 'Welcome123', 'Admin2026', 'P@ssw0rd']
for p in passwords:
    h = hashlib.new('md4', p.encode('utf-16le')).hexdigest()
    print(h)
" > ntlm_hashes.txt

# Generate Linux shadow hashes ($6$ SHA-512)
python3 -c "
import crypt
passwords = ['sunshine', 'football', 'charlie']
for p in passwords:
    h = crypt.crypt(p, '\$6\$randomsalt\$')
    print(h)
" > shadow_hashes.txt

# Generate bcrypt hashes
python3 -c "
import bcrypt
passwords = [b'iloveyou', b'batman', b'access']
for p in passwords:
    h = bcrypt.hashpw(p, bcrypt.gensalt()).decode()
    print(h)
" > bcrypt_hashes.txt 2>/dev/null || echo "Install bcrypt: pip3 install bcrypt --break-system-packages"

echo "Hash files created:"
ls -la *_hashes.txt
```

---

## Exercise 1: Identify Unknown Hashes

Before cracking, you need to know what type of hash you're dealing with.

```bash
# Use hashid
hashid '5f4dcc3b5aa765d61d8327deb882cf99'
# Output: MD5, MD4, ... (MD5 is the most likely for a 32-char hex string)

hashid '5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8'
# Output: SHA-1 (40 hex chars)

hashid '$6$randomsalt$longhashhere'
# Output: SHA-512 Crypt (Linux shadow)

hashid 'a4f49c406510bdcab6824ee7c30fd852'
# Output: MD5 or NTLM (both are 32 hex chars — context tells you which)
```

### How to tell MD5 from NTLM

Both are 32 hex characters. The difference is context:

```
Found in a web application database     → probably MD5 (mode 0)
Found in Windows SAM dump               → definitely NTLM (mode 1000)
Found from secretsdump/hashdump         → NTLM (mode 1000)
Starts with "aad3b435b51404ee"          → the LM part is empty, the other half is NTLM
```

---

## Exercise 2: Crack MD5 Hashes

```bash
cd ~/cracking-lab

# With hashcat
hashcat -m 0 md5_hashes.txt /usr/share/wordlists/rockyou.txt
# -m 0 = MD5 mode

# Check results
hashcat -m 0 md5_hashes.txt --show
# Output:
# 482c811da5d5b4bc6d497ffa98491e38:password123
# fd37ca5e76ea4f3e882b28d5b0a3e104:welcome1
# ...

# With john
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt md5_hashes.txt
john --format=raw-md5 --show md5_hashes.txt
```

**What happened:** Hashcat computed the MD5 hash of every word in rockyou.txt and compared it to your hashes. When a computed hash matched one of yours, it found the plaintext.

---

## Exercise 3: Crack NTLM Hashes (Windows)

```bash
# With hashcat
hashcat -m 1000 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt
hashcat -m 1000 ntlm_hashes.txt --show

# With john
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt ntlm_hashes.txt
john --format=nt --show ntlm_hashes.txt
```

### Using rules (crack harder passwords)

```bash
# Rules mutate each word: password → Password, PASSWORD, p@ssword, password1, etc.
hashcat -m 1000 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# What rules do:
# password → Password (capitalize first letter)
# password → password1 (append number)
# password → password! (append special char)
# password → p@ssword (leet speak substitutions)
# password → drowssap (reverse)
# ... hundreds of mutations per word
```

---

## Exercise 4: Crack Linux Shadow Hashes

```bash
# These are the $6$ hashes from /etc/shadow
hashcat -m 1800 shadow_hashes.txt /usr/share/wordlists/rockyou.txt
# Mode 1800 = sha512crypt

# This is MUCH slower than MD5/NTLM (by design — shadow hashes use key stretching)
# May take minutes instead of seconds

# With john
john --wordlist=/usr/share/wordlists/rockyou.txt shadow_hashes.txt
# John auto-detects the format from the $6$ prefix
```

**Why shadow hashes are slow:** They use thousands of rounds of hashing (key stretching) specifically to make brute forcing slow. An MD5 hash cracks at millions per second. A shadow hash cracks at thousands per second.

---

## Exercise 5: Crack from a Real Shadow File

Simulate extracting hashes from a compromised machine:

```bash
# Create a fake /etc/passwd and /etc/shadow
cat << 'EOF' > fake_passwd
root:x:0:0:root:/root:/bin/bash
admin:x:1000:1000::/home/admin:/bin/bash
user1:x:1001:1001::/home/user1:/bin/bash
svc_backup:x:1002:1002::/home/svc_backup:/bin/bash
EOF

cat << 'EOF' > fake_shadow
root:$6$rounds=5000$saltsalt$WqGLjLSFpXOHZbiHCmPmBDCy/yh0MjmFAPWB7iBqjwJSjGlKkLMN7mFCLsGYo6A0eQeVMn8.S2pNbmQxkZ1G0:19500:0:99999:7:::
admin:$6$rounds=5000$peppers1$HVOYmXcjhKg0mIKxh3c6Xtuz6mF6La/bqajSw5IYPFcZnVDx98wLjpRO6zJ1R2F7.Xy5RaCFbQYm0o/hq5o3F1:19500:0:99999:7:::
user1:$6$rounds=5000$nacl1234$8M1A5ICp1G9lp0zPnZZ7x8Y3GyLPknZmqMiFC6DmCk3tH2UYK6KXRlcGQ8vOaF5qM7bFv8nP3n5xK3zr3gM81:19500:0:99999:7:::
svc_backup:$6$rounds=5000$bkupsalt$zQxTlMh3KJPSiTe6HxKBK0NsQFq6xVn9i1YA6M5VaCdhBJq5xU8E4wLZP9UqFG7K4wC1L3lRPBU9gT2Kj7YG1:19500:0:99999:7:::
EOF

# Use unshadow to combine them (what you'd do on a real engagement)
unshadow fake_passwd fake_shadow > combined.txt

# Crack
john --wordlist=/usr/share/wordlists/rockyou.txt combined.txt
john --show combined.txt
```

---

## Exercise 6: Crack SSH Key Passphrases

```bash
# Generate a password-protected SSH key
ssh-keygen -t rsa -b 2048 -f test_key -N "letmein123"

# Extract the hash for cracking
ssh2john test_key > ssh_hash.txt

# Crack it
john --wordlist=/usr/share/wordlists/rockyou.txt ssh_hash.txt
john --show ssh_hash.txt

# The passphrase is revealed — now you can use the key
ssh -i test_key user@target
```

---

## Exercise 7: Crack ZIP File Passwords

```bash
# Create a password-protected ZIP
echo "SECRET DATA: FLAG{zip_cracked}" > secret.txt
zip -P "monkey123" protected.zip secret.txt
rm secret.txt

# Extract the hash
zip2john protected.zip > zip_hash.txt

# Crack it
john --wordlist=/usr/share/wordlists/rockyou.txt zip_hash.txt
john --show zip_hash.txt

# Extract with the found password
unzip -P "monkey123" protected.zip
cat secret.txt
```

---

## Exercise 8: Kerberos Hash Cracking (OSCP AD practice)

```bash
# Simulate a Kerberoast hash (these are what GetUserSPNs returns)
# In a real engagement, you'd get this from:
# impacket-GetUserSPNs domain/user:pass -dc-ip DC -request

# Create a simulated Kerberos TGS hash file
# (In practice, just pipe the GetUserSPNs output to a file)

# Crack with hashcat
# hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt

# Simulate an AS-REP hash
# hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt

echo "For real Kerberos hash cracking:"
echo "  1. Run: impacket-GetUserSPNs domain/user:pass -dc-ip DC -request -outputfile tgs.txt"
echo "  2. Crack: hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt"
echo "  3. Or for AS-REP: hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt"
```

---

## Hashcat Mode Reference (the ones you'll use)

| Mode | Hash type | Where you find it | Speed |
|---|---|---|---|
| 0 | MD5 | Web databases, older apps | Very fast |
| 100 | SHA1 | Web databases | Fast |
| 1000 | NTLM | Windows SAM, secretsdump, mimikatz | Very fast |
| 1400 | SHA256 | Modern web apps | Moderate |
| 1800 | sha512crypt ($6$) | Linux /etc/shadow | Slow |
| 3200 | bcrypt ($2y$) | WordPress, modern apps | Very slow |
| 5600 | NTLMv2 | Network capture (Responder) | Moderate |
| 13100 | Kerberos TGS | Kerberoasting | Moderate |
| 18200 | Kerberos AS-REP | AS-REP Roasting | Moderate |
| 22000 | WPA-PBKDF2-PMKID | WiFi handshake/PMKID | Slow |

---

## Cleanup

```bash
rm -rf ~/cracking-lab
```
