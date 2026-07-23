[◀ Prev: Methodology Framework](02-methodology-framework.md) · [🏠 Index](README.md) · [Next: Web App Attacks ▶](04-web-app-attacks.md)

---

# 03 · Enumeration — Complete Methodology

## 3.1 Port Scanning — The Standard HTB Workflow

### Phase 1 — Fast All-Port Scan

```bash
# STEP 1: Fast full port scan — find ALL open ports first
sudo nmap -p- --min-rate 5000 -T4 $TARGET -oA nmap/allports

# Parse the results
grep '/tcp' nmap/allports.nmap | awk -F/ '{print $1}' | tr '\n' ',' | sed 's/,$//'

# Store open ports
PORTS=$(grep '/tcp.*open' nmap/allports.nmap | awk -F/ '{printf "%s,",$1}' | sed 's/,$//')
echo "Open ports: $PORTS"
```

### Phase 2 — Detailed Service Scan on Open Ports

```bash
# STEP 2: Detailed scan on discovered ports
sudo nmap -sV -sC -p$PORTS $TARGET -oA nmap/detailed

# Key flags:
#   -sV   version detection
#   -sC   default scripts (safe category)
#   -p    only scan discovered ports (faster)
#   -oA   output all formats (nmap, xml, gnmap)

# Read the output
cat nmap/detailed.nmap
```

### Phase 3 — Vulnerability Scripts (Hard/Insane)

```bash
# STEP 3: Run vulnerability scripts on specific services

# SMB vulnerability check
sudo nmap --script='smb-vuln-*' -p 445 $TARGET

# Web server enumeration
sudo nmap --script='http-enum,http-title,http-methods' -p 80,443,8080 $TARGET

# Full aggressive (noisy — use only on HTB)
sudo nmap -A -p$PORTS $TARGET -oA nmap/aggressive

# UDP scan (don't forget this — many HTB machines hide services on UDP)
sudo nmap -sU --top-ports 20 $TARGET
sudo nmap -sU -p 161,500,1194,53,123 $TARGET
```

## 3.2 Web Enumeration — Standard Methodology

### Initial Web Fingerprinting

```bash
# Fingerprint web server and technology
curl -I http://$TARGET              # Headers
curl -s http://$TARGET | head -50   # Page source
whatweb -a 3 http://$TARGET         # Technology detection
nikto -h http://$TARGET -output web/nikto.txt

# Check standard paths immediately
curl http://$TARGET/robots.txt
curl http://$TARGET/sitemap.xml
curl http://$TARGET/.well-known/
```

### Directory and File Fuzzing

```bash
# gobuster — directory fuzzing
gobuster dir -u http://$TARGET \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -x php,html,txt,bak,old,zip,conf,config,js,json \
  -t 50 -o web/gobuster.txt

# ffuf — faster, more flexible
ffuf -u http://$TARGET/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -e .php,.html,.txt,.bak,.old,.zip,.conf,.js,.json \
  -mc 200,301,302,401,403 -o web/ffuf.json -of json -t 50

# feroxbuster — recursive fuzzing (great for deep paths)
feroxbuster -u http://$TARGET -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  --auto-tune -x php,html,txt --output web/ferox.txt
```

### VHost and Subdomain Fuzzing

```bash
# Virtual host fuzzing — critical for HTB (many machines use vhosts)
ffuf -u http://$TARGET \
  -H 'Host: FUZZ.TARGET_DOMAIN' \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -mc 200,301,302 -fs [SIZE_OF_DEFAULT_PAGE]

# gobuster vhost
gobuster vhost -u http://$TARGET \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Add discovered vhosts to /etc/hosts
echo '10.10.11.xxx hostname.htb subdomain.hostname.htb' | sudo tee -a /etc/hosts
```

## 3.3 Service-Specific Enumeration

### FTP (21)

```bash
# Anonymous FTP check
ftp $TARGET
# username: anonymous   password: anonymous (or blank)

# nmap FTP scripts
nmap --script=ftp-anon,ftp-bounce,ftp-syst $TARGET -p 21

# Download all files
wget -m --no-passive ftp://anonymous:@$TARGET
```

### SSH (22)

```bash
# Banner grab / version
nc $TARGET 22
ssh $TARGET -v 2>&1 | head -30

# User enumeration (older OpenSSH versions)
ssh-audit $TARGET

# Try found credentials
ssh user@$TARGET -p 22
ssh -i id_rsa user@$TARGET   # If RSA key found
```

### SMB (445)

```bash
# Null session
smbclient -L //$TARGET -N
crackmapexec smb $TARGET
crackmapexec smb $TARGET -u '' -p '' --shares

# Enum4linux
enum4linux-ng -A $TARGET

# Access share
smbclient //$TARGET/ShareName -N
smbmap -H $TARGET
```

### SMTP / POP3 / IMAP (25, 110, 143)

```bash
# SMTP user enumeration
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/Names/names.txt -t $TARGET

# Manual SMTP
telnet $TARGET 25
# EHLO test
# VRFY user@domain

# Read emails if creds found
curl -u user:pass imap://$TARGET/INBOX
```

### SNMP (UDP 161)

```bash
# SNMP community string brute
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt $TARGET

# Walk SNMP tree
snmpwalk -v2c -c public $TARGET
snmpwalk -v1 -c public $TARGET 1.3.6.1.4.1.77.1.2.25    # Windows users
snmpwalk -v1 -c public $TARGET 1.3.6.1.2.1.25.4.2.1.2   # Running processes
```

## 3.4 Automated Enumeration Scripts

```bash
# AutoRecon — all-in-one enumeration (great starting point)
autorecon $TARGET --output autorecon/

# Reconnoitre
reconnoitre -t $TARGET -o reconnoitre/ --services

# nmapAutomator — quick and thorough
nmapAutomator.sh $TARGET All

# Note: Always review tool output manually.
# Automated tools miss context — your brain is the best tool.
```

---

[◀ Prev: Methodology Framework](02-methodology-framework.md) · [🏠 Index](README.md) · [Next: Web App Attacks ▶](04-web-app-attacks.md)
