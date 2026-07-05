# Technical Security Repository: CTF Writeups & Labs

Welcome to my technical repository. This space documents the methodology, enumeration, and responsible exploitation of complex CTF machines and security challenges, covering offensive, defensive, and collaborative security perspectives.

🏆 **Ranked Top 3 in Portugal on TryHackMe**

---

## 📂 Repository Structure

### 🔴 Red Team

*   **[THM VulNet: Internal](./Red-Team/VulNet-Internal.md)**
    *   *Vectors:* NFS Enumeration, Redis Rogue Server RCE, CI/CD Abuse (TeamCity).
*   **[CWL CRTA Preparation](./Red-Team/CRTA-Prep.md)**
    *   *Vectors:* Command Injection, Pivoting (SSHuttle), AD Enumeration, NTDS Extraction, Cross-Domain Ticket Forgery.
*   **[THM VulnNet: Roasted](./Red-Team/VulnNet-Roasted.md)**
    *   *Vectors:* SMB Null Sessions, AS-REP Roasting, WinRM, NTDS.dit Dump (`secretsdump`).
*   **[THM Dead Drop](./Red-Team/Dead-Drop.md)**
    *   *Vectors:* SQLi, Node.js Reverse Shell, Chisel SOCKS Proxy, BloodHound AD Abuse.

### 🔵 Blue Team

*   **[THM PrintNightmare](./Blue-Team/PrintNightmare.md)**
    *   *Focus:* ProcDot Analysis, CVE-2021-1675, DLL Injection, PowerShell History Analysis.
*   **[THM Investigating Windows 3.x](./Blue-Team/Investigating-Windows-3.md)**
    *   *Focus:* Registry Persistence, Sysmon Event Analysis, Process Injection (T1055).
*   **[THM Dunkle Materie](./Blue-Team/Dunkle-Materie.md)**
    *   *Focus:* Ransomware Forensics, Process Masquerading, C2 Traffic Analysis.

### 🟣 Purple Team

*   **[THM TryHack3M: Subscribe](./Purple-Team/Subscribe.md)**
    *   *Attack:* Client-Side Bypass, Hidden PHP Endpoints, SQL Injection.
    *   *Defense:* Splunk Log Analysis, Attacker User-Agent Identification, Event Correlation.

---

## 🛠️ Core Toolkit

*   **Offensive:** Impacket, BloodHound, NetExec, Responder, Chisel, Burp Suite, Hydra.
*   **Defensive / Forensic:** Splunk, Wireshark, Sysmon, ProcDot, Volatility, YARA.
