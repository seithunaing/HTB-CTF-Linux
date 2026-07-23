[◀ Prev: HTB Setup](01-htb-setup.md) · [🏠 Index](README.md) · [Next: Enumeration ▶](03-enumeration.md)

---

# 02 · Universal HTB Methodology Framework

## 2.1 The Five-Phase Attack Cycle

| Phase | Goal | Key Activities | Output |
| --- | --- | --- | --- |
| **1. Enumeration** | Map the attack surface | Port scan, service ID, web crawl, DNS, user enum | Target map, service versions, interesting paths |
| **2. Foothold** | Get initial shell access | Exploit service vulns, web app attacks, phishing creds | Reverse/bind shell, SSH access |
| **3. Lateral Move** | Expand access to new users | Credential reuse, internal service abuse, pivot | Additional user accounts, new network access |
| **4. Privilege Esc.** | Become root | SUID / sudo / cron / capabilities / kernel / service exploits | root shell, `/root/root.txt` |
| **5. Post-Exploit** | Collect flags and evidence | Read flags, dump creds, document findings | `user.txt` + `root.txt` submitted to HTB |

## 2.2 The Enumeration-First Rule

> [!IMPORTANT]
> **Enumerate more when stuck.** The most common mistake on HTB is rushing to exploitation before finishing enumeration. When you are stuck, go back and enumerate more thoroughly. Run all nmap scripts. Fuzz every web endpoint. Check every port. Read every config file you can access. The answer is almost always in the enumeration data — you just haven't found it yet.

## 2.3 HTB Machine Difficulty — What to Expect

| Difficulty | Typical Attack Chain | Key Concepts | Hints Allowed |
| --- | --- | --- | --- |
| **Easy** | 1–2 step foothold, simple privesc | Public CVE, default creds, simple misconfiguration | Common — HTB forum, Discord |
| **Medium** | 2–3 step foothold, chained privesc | Custom web exploit, password cracking, service abuse | Use sparingly |
| **Hard** | 3–4 step chain, non-obvious privesc | Custom exploit, obscure service, complex logic chains | Minimal — only for direction |
| **Insane** | Multi-stage, research required | Near-zero-day, binary exploitation, kernel attacks | Almost never — research independently |

## 2.4 When You Are Stuck — Decision Tree

| Stuck At | Ask Yourself | Try This |
| --- | --- | --- |
| No ports found | Did I scan all 65535 ports? UDP too? | `nmap -p- --min-rate 5000`; `nmap -sU` top ports |
| No web entry point | Did I fuzz directories? Check all ports for HTTP? | gobuster/ffuf all ports; check `robots.txt`, `/admin`, `/api` |
| Login form | Tried default creds? SQL injection? Username enum? | `admin:admin`, `admin:password`; SQLi in all fields |
| Have shell but no priv | Ran linpeas? Checked sudo? SUID? Cron? Caps? | Full linpeas; `pspy` for processes; `sudo -l` |
| linpeas shows nothing | Is it custom software? Check running processes? | `pspy`; `ps aux`; `netstat`; check `/opt` `/srv` `/var/www` |
| Credential found | Tried it everywhere? Password reuse on all accounts? | SSH, FTP, web login, `su -`, `sudo -l` with the credential |

---

[◀ Prev: HTB Setup](01-htb-setup.md) · [🏠 Index](README.md) · [Next: Enumeration ▶](03-enumeration.md)
