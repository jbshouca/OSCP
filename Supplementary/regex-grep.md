# Supplementary: Regex, Grep, and Data Extraction

Every cybersecurity role — pentesting, IR, SOC, engineering — requires pulling specific data out of massive amounts of text. Log files, scan output, config files, packet captures, code. Regex (regular expressions) and grep are how you do it. Master these and you'll work 10x faster.

---

## grep — Your Most Used Tool

### Basic usage

```bash
# Search for a string in a file
grep "password" config.txt

# Search recursively through all files in a directory
grep -r "password" /var/www/

# Case insensitive
grep -i "password" config.txt

# Show line numbers
grep -n "error" /var/log/syslog

# Show surrounding context (3 lines before and after)
grep -C 3 "failed" /var/log/auth.log

# Only show the matching part (not the whole line)
grep -o "192\.168\.[0-9]*\.[0-9]*" access.log

# Count matches
grep -c "404" access.log

# Invert match (show lines that DON'T match)
grep -v "DEBUG" application.log

# Multiple patterns (OR)
grep -E "error|warning|critical" /var/log/syslog
# or
grep "error\|warning\|critical" /var/log/syslog

# Match whole words only (don't match "password" inside "passwords")
grep -w "pass" config.txt

# List only filenames that contain the match
grep -rl "password" /var/www/
# -r = recursive, -l = filenames only

# Quiet mode (just check if it exists — for scripts)
grep -q "root" /etc/passwd && echo "root exists"
```

### grep flags cheat sheet

| Flag | What it does | Example |
|---|---|---|
| `-i` | Case insensitive | `grep -i "error" log` |
| `-r` | Recursive (search directories) | `grep -r "password" /etc/` |
| `-n` | Show line numbers | `grep -n "failed" auth.log` |
| `-l` | Show only filenames | `grep -rl "secret" /var/www/` |
| `-c` | Count matches | `grep -c "404" access.log` |
| `-v` | Invert (lines NOT matching) | `grep -v "DEBUG" app.log` |
| `-o` | Only show the match itself | `grep -o "[0-9]*\.[0-9]*" log` |
| `-E` | Extended regex (egrep) | `grep -E "error|warn" log` |
| `-P` | Perl-compatible regex (most powerful) | `grep -P "\d{1,3}\.\d{1,3}" log` |
| `-A 3` | Show 3 lines After match | `grep -A 3 "error" log` |
| `-B 3` | Show 3 lines Before match | `grep -B 3 "error" log` |
| `-C 3` | Show 3 lines of Context (before + after) | `grep -C 3 "error" log` |
| `-w` | Match whole words only | `grep -w "cat" file` (won't match "category") |

---

## Regex — Pattern Matching Language

### Characters with special meaning

```
.       Any single character           a.c matches "abc", "a1c", "a-c"
*       Zero or more of the previous   ab*c matches "ac", "abc", "abbc", "abbbc"
+       One or more of the previous    ab+c matches "abc", "abbc" (not "ac")
?       Zero or one of the previous    ab?c matches "ac", "abc" (not "abbc")
^       Start of line                  ^root matches lines starting with "root"
$       End of line                    bash$ matches lines ending with "bash"
[]      Character class                [abc] matches "a", "b", or "c"
[^]     Negated character class        [^abc] matches anything except "a", "b", "c"
|       OR                             cat|dog matches "cat" or "dog"
()      Grouping                       (ab)+ matches "ab", "abab", "ababab"
\       Escape special character       \. matches a literal dot
{}      Repetition count               a{3} matches "aaa"
```

### Character classes

```
[a-z]       Any lowercase letter
[A-Z]       Any uppercase letter
[0-9]       Any digit
[a-zA-Z]    Any letter
[a-zA-Z0-9] Any alphanumeric character
[^0-9]      Anything that's NOT a digit

Shorthand (use with grep -P):
\d          Any digit           [0-9]
\D          Not a digit         [^0-9]
\w          Word character      [a-zA-Z0-9_]
\W          Not a word char     [^a-zA-Z0-9_]
\s          Whitespace          [ \t\n\r]
\S          Not whitespace      [^ \t\n\r]
```

### Repetition

```
*       Zero or more        a* matches "", "a", "aa", "aaa"
+       One or more         a+ matches "a", "aa", "aaa" (not "")
?       Zero or one         a? matches "" or "a"
{3}     Exactly 3           a{3} matches "aaa"
{2,5}   Between 2 and 5     a{2,5} matches "aa", "aaa", "aaaa", "aaaaa"
{3,}    3 or more           a{3,} matches "aaa", "aaaa", etc.
```

---

## Security-Specific Regex Patterns

### Extract IP addresses

```bash
# Match IPv4 addresses
grep -oE "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" logfile.txt

# More precise (each octet 0-255)
grep -oP "\b(?:(?:25[0-5]|2[0-4]\d|1?\d{1,2})\.){3}(?:25[0-5]|2[0-4]\d|1?\d{1,2})\b" logfile.txt

# Find unique IPs and count occurrences
grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" access.log | sort | uniq -c | sort -rn | head -20
```

### Extract URLs

```bash
# Find URLs in text
grep -oE "https?://[^ \"'>]+" page.html

# Find domains
grep -oP "https?://\K[^/\s]+" page.html
```

### Extract email addresses

```bash
grep -oE "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" file.txt
```

### Extract MAC addresses

```bash
grep -oiE "([0-9a-f]{2}:){5}[0-9a-f]{2}" file.txt
```

### Find credentials in files

```bash
# Generic password patterns
grep -rniE "password\s*[:=]\s*\S+" /var/www/
grep -rniE "passwd\s*[:=]\s*\S+" /etc/
grep -rniE "secret\s*[:=]\s*\S+" /opt/

# Database connection strings
grep -rniE "(mysql|postgres|mongodb)://[^\"' ]+" /var/www/

# API keys and tokens
grep -rniE "(api[_-]?key|token|secret[_-]?key)\s*[:=]\s*['\"][^'\"]+['\"]" /var/www/

# AWS access keys
grep -rniE "AKIA[0-9A-Z]{16}" /var/www/

# Private keys
grep -rl "BEGIN.*PRIVATE KEY" /home/
grep -rl "BEGIN RSA" /home/
```

### Analyze auth logs for attacks

```bash
# Failed SSH logins
grep "Failed password" /var/log/auth.log

# Count failed logins per IP
grep "Failed password" /var/log/auth.log | grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" | sort | uniq -c | sort -rn

# Successful logins from unusual IPs
grep "Accepted password" /var/log/auth.log

# Brute force detection (IPs with >10 failures)
grep "Failed password" /var/log/auth.log | grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" | sort | uniq -c | sort -rn | awk '$1 > 10'

# Track a specific attacker's activity
grep "192.168.244.129" /var/log/auth.log
```

### Analyze web server logs

```bash
# Find all 404 errors (directory brute force indicator)
grep " 404 " /var/log/apache2/access.log | head -20

# Find SQL injection attempts
grep -iE "union.*select|or.*1=1|'.*--|\bselect\b.*\bfrom\b" /var/log/apache2/access.log

# Find command injection attempts
grep -E "%[0-9a-fA-F]{2}.*(%3B|%7C|%26)" /var/log/apache2/access.log
grep -iE "(/bin/(bash|sh|cmd)|whoami|/etc/passwd)" /var/log/apache2/access.log

# Find LFI attempts
grep -E "\.\./\.\." /var/log/apache2/access.log

# Most requested paths (find what attackers are looking for)
awk '{print $7}' /var/log/apache2/access.log | sort | uniq -c | sort -rn | head -20

# Most active IPs
awk '{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -rn | head -10

# Requests per hour (detect scanning patterns)
awk '{print $4}' /var/log/apache2/access.log | cut -d: -f1-2 | sort | uniq -c | sort -rn | head -10
```

### Extract data from nmap output

```bash
# Get just the open ports from nmap output
grep "open" scan.txt | grep -v "filtered"

# Extract port numbers only
grep "open" scan.txt | grep -oE "^[0-9]+" 

# Find all hosts with a specific port open (from greppable output)
grep "80/open" scan.gnmap | awk '{print $2}'

# Find all hosts with SSH
grep "22/open" scan.gnmap | awk '{print $2}'
```

---

## awk — Column-Based Data Processing

awk processes text column by column. It's perfect for extracting fields from structured output.

```bash
# Print specific columns
awk '{print $1}' file.txt              # first column
awk '{print $1, $3}' file.txt          # first and third columns
awk '{print $NF}' file.txt             # last column

# Custom delimiter
awk -F':' '{print $1}' /etc/passwd     # split on colons, print first field (usernames)
awk -F',' '{print $2}' data.csv        # split on commas

# Filter rows
awk '$3 > 100' data.txt                # rows where column 3 > 100
awk '/error/' log.txt                  # rows containing "error"
awk '$1 == "192.168.244.129"' log.txt  # rows where column 1 is this IP

# Count and summarize
awk '{sum += $1} END {print sum}' numbers.txt    # sum a column
awk 'END {print NR}' file.txt                     # count lines
```

### awk for security

```bash
# Extract usernames from /etc/passwd (users with bash shells)
awk -F':' '$7 ~ /bash/ {print $1}' /etc/passwd

# Parse Apache logs — top 10 IPs by request count
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# Find large data transfers (potential exfiltration)
awk '$10 > 1000000 {print $1, $7, $10}' access.log
# Column 10 is typically bytes sent in Apache combined log format

# Parse nmap greppable output — hosts with port 445 open
awk '/445\/open/' scan.gnmap | awk '{print $2}'
```

---

## sed — Stream Editor (find and replace)

```bash
# Replace text in a file
sed 's/old/new/' file.txt              # first occurrence per line
sed 's/old/new/g' file.txt             # all occurrences
sed -i 's/old/new/g' file.txt          # edit in place (changes the file)

# Delete lines
sed '/pattern/d' file.txt              # delete lines matching pattern
sed '1d' file.txt                      # delete first line
sed '$d' file.txt                      # delete last line

# Print specific lines
sed -n '5p' file.txt                   # print only line 5
sed -n '5,10p' file.txt               # print lines 5-10
```

### sed for security

```bash
# Remove comments and blank lines from config files (clean view)
sed '/^#/d; /^$/d' /etc/ssh/sshd_config

# Extract values from key=value config files
sed -n 's/.*password=\(.*\)/\1/p' config.txt

# Sanitize IPs in reports (anonymize)
sed 's/192\.168\.244\.\([0-9]*\)/10.0.0.\1/g' report.txt
```

---

## Combining Tools — Power Pipelines

### Find the top attackers in auth logs

```bash
grep "Failed password" /var/log/auth.log |
  grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" |
  sort |
  uniq -c |
  sort -rn |
  head -10
```

Pipeline breakdown:
```
grep "Failed password"     → filter to only failed login lines
grep -oE "[0-9]+..."      → extract just the IP address
sort                       → sort IPs alphabetically (needed for uniq)
uniq -c                    → count consecutive duplicates
sort -rn                   → sort by count, highest first
head -10                   → show top 10
```

### Find all credentials in a web directory

```bash
find /var/www/ -type f \( -name "*.php" -o -name "*.conf" -o -name "*.ini" -o -name "*.txt" -o -name "*.xml" \) -exec \
  grep -lniE "password|passwd|credential|secret|api.?key|token" {} \;
```

### Extract all unique IPs from any text file

```bash
grep -oE "\b[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\b" file.txt | sort -u
```

### Monitor a log file for attack patterns in real time

```bash
tail -f /var/log/auth.log | grep --color "Failed\|Accepted\|Invalid"
```

### Process nmap XML output

```bash
# Extract all open ports and services from XML
grep -oP 'portid="\K[^"]+' scan.xml | sort -n | uniq
grep -oP 'name="\K[^"]+' scan.xml | sort | uniq
```

---

## Practice Exercise

### Create a sample log file and practice extracting data

```bash
# Generate a fake auth log
cat << 'EOF' > practice_log.txt
Jul 31 10:01:23 server sshd[1234]: Failed password for admin from 192.168.1.100 port 45678 ssh2
Jul 31 10:01:24 server sshd[1235]: Failed password for admin from 192.168.1.100 port 45679 ssh2
Jul 31 10:01:25 server sshd[1236]: Failed password for root from 10.10.14.5 port 54321 ssh2
Jul 31 10:01:26 server sshd[1237]: Accepted password for john from 192.168.1.50 port 22334 ssh2
Jul 31 10:01:27 server sshd[1238]: Failed password for invalid user test from 172.16.0.1 port 33445 ssh2
Jul 31 10:02:01 server sshd[1239]: Failed password for admin from 192.168.1.100 port 45680 ssh2
Jul 31 10:02:02 server sshd[1240]: Failed password for admin from 192.168.1.100 port 45681 ssh2
Jul 31 10:02:03 server sshd[1241]: Failed password for root from 192.168.1.100 port 45682 ssh2
Jul 31 10:02:04 server sshd[1242]: Accepted password for admin from 192.168.1.100 port 45683 ssh2
Jul 31 10:03:01 server sshd[1243]: Failed password for admin from 10.10.14.5 port 54322 ssh2
Jul 31 10:03:02 server sshd[1244]: Failed password for root from 10.10.14.5 port 54323 ssh2
Jul 31 10:03:03 server sshd[1245]: Failed password for user1 from 10.10.14.5 port 54324 ssh2
EOF

# Practice these queries:

# 1. How many failed login attempts total?
grep -c "Failed password" practice_log.txt

# 2. Which IP had the most failed attempts?
grep "Failed password" practice_log.txt | grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" | sort | uniq -c | sort -rn

# 3. Which usernames were targeted?
grep "Failed password" practice_log.txt | grep -oP "for (\w+)" | sort | uniq -c | sort -rn

# 4. Were any logins successful? From where?
grep "Accepted" practice_log.txt

# 5. Did the brute forcer eventually succeed? (same IP: failed then accepted)
grep "192.168.1.100" practice_log.txt

# 6. What invalid usernames were tried?
grep "invalid user" practice_log.txt | grep -oP "user \K\w+"

# Cleanup
rm practice_log.txt
```
