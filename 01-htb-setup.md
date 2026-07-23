[🏠 Index](README.md) · [Next: Methodology Framework ▶](02-methodology-framework.md)

---

# 01 · HTB Setup & Environment Configuration

## 1.1 HackTheBox Account & VPN Setup

HackTheBox (HTB) provides a legal, controlled environment for practicing penetration testing against intentionally vulnerable machines. All attacks are authorized within the HTB platform.

1. Create a free account at: https://www.hackthebox.com
2. Navigate to: **Access → Labs → Download OpenVPN pack**
3. Select your server region (EU, US, AU, SG) — choose closest
4. Download the `.ovpn` configuration file

```bash
# Connect to HTB VPN
sudo openvpn --config your_username.ovpn

# Verify connection — your tun0 IP should be 10.10.14.x or 10.10.16.x
ip addr show tun0
ifconfig tun0

# Keep this terminal open — VPN must stay connected during attack
# Or run in background:
sudo openvpn --config your_username.ovpn --daemon

# Verify you can reach HTB network
ping 10.10.10.1 -c 3
```

## 1.2 Recommended Attacker Setup

| Tool | Version | Purpose | Install |
| --- | --- | --- | --- |
| Kali Linux | 2024.x | Primary attack platform | kali.org or HTB Pwnbox |
| Burp Suite | Community | Web app interception proxy | portswigger.net / pre-installed |
| nmap | 7.94+ | Port scanning and service detection | `sudo apt install nmap` |
| gobuster / ffuf | latest | Directory and vhost fuzzing | `sudo apt install gobuster ffuf` |
| impacket | latest | SMB/Kerberos/RPC tools | `pip3 install impacket` |
| pwncat-cs | latest | Advanced reverse shell handler | `pip3 install pwncat-cs` |
| linpeas | latest | Linux privilege escalation enum | github.com/carlospolop/PEASS-ng |
| pspy | latest | Process monitoring without root | github.com/DominicBreuker/pspy |
| chisel | latest | TCP tunneling for pivoting | github.com/jpillora/chisel |
| john / hashcat | latest | Password cracking | `sudo apt install john hashcat` |

## 1.3 Workspace Setup — HTB Standard Layout

```bash
# Create organized workspace for each machine
MACHINE='MachineName'
TARGET='10.10.11.xxx'
mkdir -p ~/htb/$MACHINE/{nmap,web,exploits,loot,screenshots,notes}
cd ~/htb/$MACHINE

# Start tmux with named windows
tmux new-session -s $MACHINE
# Ctrl+B, c = new window
# Ctrl+B, , = rename window
# Recommended layout:
#   Window 0: recon
#   Window 1: exploit
#   Window 2: listeners
#   Window 3: loot/notes

# Set target variable in every pane
export TARGET=10.10.11.xxx
echo $TARGET
```

## 1.4 Note-Taking Discipline

> [!TIP]
> **Take notes from minute one.** HTB machines often require connecting dots across multiple enumeration stages. Write down every open port, every service version, every username, every credential, every file path you find interesting. Use a tool like Obsidian, CherryTree, or a simple Markdown file. The flag you need is often hidden in information you found 30 minutes before you needed it.

```bash
# Recommended notes structure per machine
cat > notes.md << 'EOF'
# Machine: TargetName
# IP: 10.10.11.xxx
# OS: Linux
# Difficulty: Easy/Medium/Hard/Insane

## Open Ports
| Port | Service | Version |
| ---- | ------- | ------- |

## Credentials Found
| Source | Username | Password/Hash |

## Attack Path
1.

## Commands That Worked
EOF
```

---

[🏠 Index](README.md) · [Next: Methodology Framework ▶](02-methodology-framework.md)
