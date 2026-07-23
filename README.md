<div align="center">

# 🐧 HackTheBox — Linux Machine CTF Guide

### Complete Methodology — Enumeration to Privilege Escalation

**`Easy`** • **`Medium`** • **`Hard`** • **`Insane`** — All Levels Covered

`Enumeration → Foothold → Lateral → PrivEsc → Root`

</div>

---

> [!WARNING]
> **Authorized Use Only.** This material is intended for use against **HackTheBox lab machines** and other explicitly authorized, controlled environments for educational purposes. All techniques documented here are for legal penetration-testing practice only. Do not use them against systems you do not own or have written permission to test.

---

## 📖 About

A complete, hands-on methodology reference for rooting HackTheBox Linux machines across every difficulty tier. Each chapter is a standalone Markdown file, wired together with prev/index/next navigation for easy reading on GitHub.

**Version 1.0 · 2026**

## 🎯 Difficulty Overview

| Tier | Typical Chain | Key Concepts |
| --- | --- | --- |
| 🟢 **Easy** | 1–2 step foothold, simple privesc | Public CVE, default creds, simple misconfiguration |
| 🟡 **Medium** | 2–3 step foothold, chained privesc | Custom web exploit, password cracking, service abuse |
| 🔴 **Hard** | 3–4 step chain, non-obvious privesc | Custom exploit, obscure service, complex logic chains |
| 🟣 **Insane** | Multi-stage, research required | Near-zero-day, binary exploitation, kernel attacks |

## 📚 Table of Contents

| # | Chapter | Focus |
| --- | --- | --- |
| 01 | [HTB Setup & Environment Configuration](01-htb-setup.md) | Account, VPN, tooling, workspace |
| 02 | [Universal HTB Methodology Framework](02-methodology-framework.md) | Five-phase cycle, decision trees |
| 03 | [Enumeration — Complete Methodology](03-enumeration.md) | Ports, web, service-specific enum |
| 04 | [Web Application Attack Techniques](04-web-app-attacks.md) | SQLi, LFI/RFI, cmd injection, SSRF, SSTI |
| 05 | [Initial Access — Gaining a Foothold](05-initial-access.md) | Reverse shells, stabilisation, transfers |
| 06 | [Linux Privilege Escalation](06-linux-privesc.md) | sudo, SUID, cron, caps, PATH, Docker/LXD |
| 07 | [Easy Machines — Methodology](07-easy-machines.md) | Standard workflow, common vectors |
| 08 | [Medium Machines — Methodology](08-medium-machines.md) | Deeper enum, chained privesc |
| 09 | [Hard Machines — Methodology](09-hard-machines.md) | Source audit, advanced privesc |
| 10 | [Insane Machines — Methodology](10-insane-machines.md) | Binary/kernel exploitation |
| 11 | [Post-Exploitation & Flag Capture](11-post-exploitation.md) | Flags, loot, persistence |
| 12 | [Password Attacks & Credential Hunting](12-password-attacks.md) | Hash cracking, brute force |
| 13 | [Service Exploitation Reference](13-service-exploitation.md) | DB, Redis, Elasticsearch, Docker API |
| 14 | [Pivoting & Tunneling](14-pivoting-tunneling.md) | SSH, Chisel, Ligolo-ng |
| 15 | [Custom Exploit Development Basics](15-exploit-development.md) | CVE hunting, modifying exploits |
| 16 | [Tools & Quick Reference](16-tools-reference.md) | Burp, pwncat, Metasploit |
| 17 | [Master Cheatsheet — All Levels](17-master-cheatsheet.md) | Decision trees, quick refs |

## 🧰 Core Toolkit

`nmap` · `ffuf` / `gobuster` / `feroxbuster` · `Burp Suite` · `sqlmap` · `impacket` · `linpeas` · `pspy` · `chisel` / `ligolo-ng` · `john` / `hashcat` · `pwncat-cs` · `GTFOBins`

## 🧭 The Golden Rules

1. **Enumerate fully before exploiting** — ~80% of the answer is in enumeration.
2. **Take notes on everything** — usernames, ports, file paths, credentials, error messages.
3. **When stuck, enumerate harder** rather than switching exploits.
4. **Difficulty ≈ obviousness** — on Insane, nothing is obvious.
5. **Version-match every service** — CVEs are version-specific.
6. **Always try credential reuse** across all services and all users.
7. **After a shell**, run `linpeas` immediately *and* `pspy` in the background.
8. **Submit flags immediately** — don't lose them in a crash or reset.

---

<div align="center">

**Happy Hacking on HackTheBox! 🐧⚔️**

</div>

