# Technical Security Repository: CTF Writeups & Labs

Welcome to my technical repository. This space documents the methodology, enumeration, and responsible exploitation of complex CTF machines and security challenges, covering offensive, defensive, and collaborative security perspectives.

🏆 **Ranked Top 3 in Portugal on TryHackMe**

---

## 📂 Repository Structure

### 🔴 Red Team

*   **[THM VulNet: Internal](./Red-Team/VulNet-Internal.md)**
    *   *Vectors:* NFS Enumeration, Redis Rogue Server RCE, CI/CD Abuse (TeamCity).
*   **[THM Gallery](./Red-Team/Gallery.md)**
    *   *Vectors:* SQL Injection, RCE via File Upload, Linux PrivEsc.
*   **[THM b3dr0ck](./Red-Team/b3dr0ck.md)**
    *   *Vectors:* Certificates, Custom Services, sudo misconfigurations, encoding analysis.
*   **[CWL CRTA Preparation](./Red-Team/CRTA-Prep.md)**
    *   *Vectors:* Command Injection, Network Pivoting (SSHuttle), AD Enumeration, NTDS Extraction, Cross-Domain Kerberos Ticket Forgery, Pass-the-Hash.
*   **[THM RazorBlack](./Red-Team/RazorBlack.md)**
    *   *Vectors:* NFS Enumeration, AS-REP Roasting, Kerberoasting, Password Reset Abuse, NTDS Dump, SeBackupPrivilege, Pass-the-Hash, DPAPI Decryption.
*   **[THM You Got Mail](./Red-Team/You-Got-Mail.md)**
    *   *Vectors:* OSINT, SMTP Brute Force (CeWL + Hydra), Phishing, Meterpreter RCE, WDigest Credential Dumping, hMailServer Config Analysis.
*   **[THM K2](./Red-Team/K2.md)**
    *   *Vectors:* JWT Hijacking, Union SQLi, BloodHound AD Analysis, Targeted Kerberoasting, GenericAll Abuse, RBCD Attack, S4U2Proxy Impersonation, WMIExec.
*   **[THM VulnNet: Roasted](./Red-Team/VulnNet-Roasted.md)**
    *   *Vectors:* SMB Null Sessions, Username Harvesting from Docs, AS-REP Roasting (GetNPUsers), WinRM Access, NTDS.dit Dump (`secretsdump`), Pass-the-Hash.
*   **[THM Reset](./Red-Team/Reset.md)**
    *   *Vectors:* Onboarding Document Harvesting, Malicious LNK Generation (`ntlm_theft`), NTLMv2 Capture (Responder), Constrained Delegation Abuse, S4U2Proxy (`getST`), WMIExec.
*   **[THM Relevant](./Red-Team/Relevant.md)**
    *   *Vectors:* SMB Guest Share Write Abuse, ASPX Web Shell Upload, IIS Web Root Share Mapping, SeImpersonatePrivilege Abuse, Token Impersonation (`PrintSpoofer`).
*   **[THM Windows Jump](./Red-Team/Windows-Jump.md)**
    *   *Vectors:* AutoAdminLogon Registry Exposure, Service Binary Hijacking, Payload Delivery (`certutil`), Scheduled Task Overwrite, Reverse Shell Persistence.
*   **[THM Dead Drop](./Red-Team/Dead-Drop.md)**
    *   *Vectors:* SQLi (`sqlmap`), Node.js Reverse Shell, APK Decompilation (`strings.xml`), Network Pivoting (Chisel SOCKS Proxy + Proxychains), BloodHound AD Abuse.

### 🔵 Blue Team

*   **[THM Warzone 1](./Blue-Team/Warzone-1.md)**
    *   *Focus:* Log Analysis, PCAP Network Forensics (Brim, NetworkMiner, Wireshark).
*   **[THM PrintNightmare](./Blue-Team/PrintNightmare.md)**
    *   *Focus:* ProcDot Analysis, CVE-2021-1675, DLL Injection via Print Spooler, Registry Forensics, PowerShell History Analysis, Anti-Forensics Detection.
*   **[THM Investigating Windows 2](./Blue-Team/Investigating-Windows-2.md)**
    *   *Focus:* Autoruns Analysis, WMI Persistence, Scheduled Task Abuse, Process Monitoring (Procmon/ProcExp), Loki IOC Scanning, YARA Rule Development.
*   **[THM Investigating Windows 3.x](./Blue-Team/Investigating-Windows-3.md)**
    *   *Focus:* Registry Persistence, Sysmon Event Analysis, PrintDemon (CVE-2020-1048), Empire C2 Detection, Process Injection (T1055), MITRE ATT&CK Mapping.
*   **[THM Chrome](./Blue-Team/Chrome.md)**
    *   *Focus:* .NET Decompilation (ILSpy), Static AES Key Recovery, DPAPI Master Key Cracking, Impacket-DPAPI Analysis, Chrome Local State Decryption, Mimikatz.
*   **[THM Dunkle Materie](./Blue-Team/Dunkle-Materie.md)**
    *   *Focus:* ProcDOT Analysis, Ransomware Forensics, Process Masquerading (typosquatting), C2 Traffic Analysis, Registry Forensics, Wallpaper Persistence.

### 🟣 Purple Team

*   **[THM TryHack3M: Subscribe](./Purple-Team/Subscribe.md)**
    *   *Attack:* Client-Side Hostname Restriction Bypass, Cookie Tampering, Hidden PHP Endpoints, Web-Based CLI Abuse, SQL Injection (`sqlmap`), Plaintext Credential Dumping.
    *   *Defense:* Splunk Log Analysis, Attacker User-Agent Identification, IP-Based Event Correlation, URI Analysis for Targeted Table Discovery.

---

## 🛠️ Core Toolkit

*   **Offensive:** Impacket, BloodHound, NetExec, Responder, Chisel, Burp Suite, Hydra, WMIExec, Evil-WinRM, Kerbrute, Hashcat, John the Ripper.
*   **Defensive / Forensic:** Splunk, Wireshark, Sysmon, ProcDot, Volatility, YARA, Brim, NetworkMiner, Autoruns, Process Monitor, Loki IOC Scanner, ILSpy.
