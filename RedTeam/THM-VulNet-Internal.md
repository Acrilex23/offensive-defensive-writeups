# [THM] VulNet: Internal — Write-up

<p align="center"><img src="image_7029c6.png" alt="VulNet Internal Banner" width="100%"></p>

## 🎯 Room Overview

* **Target:** `vulnet.thm`
* **Difficulty:** Medium
* **Objective:** Comprehensive enumeration and responsible exploitation targeting NFS, Redis, Rsync, and internal CI/CD infrastructure (TeamCity).
* **Skills Covered:** Service Enumeration, Credential Leakage, SSH Key Injection, Port Forwarding, SUID Abuse.

---

## 🔍 Phase 1: Enumeration & Reconnaissance

Starting off the enumeration phase with a thorough Nmap scan against the target. Let's break down what each flag is doing and why it matters.

```bash
nmap -p- -sS -sC -sV -O -T4 --min-rate 500 vulnet.thm -oN full.txt -vv

```

### Flag Breakdown

| Flag | Purpose |
| --- | --- |
| `-p-` | Tells Nmap to scan all 65535 ports rather than just the default top 1000, ensuring nothing is missed on unusual ports. |
| `-sS` | Performs a SYN scan, also known as a stealth scan, which sends SYN packets without completing the full TCP handshake, making it faster and less noisy than a full connection scan. |
| `-sC` | Runs Nmap's default scripts against the discovered ports, which can automatically pull useful information like service banners, authentication methods and potential misconfigurations. |
| `-sV` | Probes open ports to detect the exact service version running, giving us more precise information about what we are dealing with. |
| `-O` | Attempts to identify the operating system of the target based on how it responds to certain packets. |
| `-T4` | Sets the timing template to aggressive, speeding up the scan significantly while remaining reliable enough for most environments. |
| `--min-rate 500` | Ensures Nmap sends at least 500 packets per second, keeping the scan moving at a good pace. |
| `-oN full.txt` | Saves the output to a file called full.txt so we have a clean record to reference throughout the engagement. |
| `-vv` | Enables extra verbose output, letting us see results as they come in rather than waiting for the scan to finish. |

### Scan Results

```text
PORT      STATE    SERVICE     REASON         VERSION
22/tcp    open     ssh         syn-ack ttl 62 OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 56:3a:00:62:b1:f6:e6:f0:a6:0f:5c:0a:3f:ff:00:63 (RSA)
|   256 2d:68:41:3a:96:89:40:bd:f9:7b:61:75:aa:07:45:01 (ECDSA)
|_  256 7b:bd:5d:a9:eb:7e:81:5b:4e:52:ca:ce:15:a0:19:90 (ED25519)
111/tcp   open     rpcbind     syn-ack ttl 62 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100003  3,4         2049/tcp   nfs
139/tcp   open     netbios-ssn syn-ack ttl 62 Samba smbd 4
445/tcp   open     netbios-ssn syn-ack ttl 62 Samba smbd 4
873/tcp   open     rsync       syn-ack ttl 62 (protocol version 31)
2049/tcp  open     nfs         syn-ack ttl 62 3-4 (RPC #100003)
6379/tcp  open     redis       syn-ack ttl 62 Redis key-value store
9090/tcp  filtered zeus-admin  no-response
43057/tcp open     java-rmi    syn-ack ttl 62 Java RMI
46079/tcp open     nlockmgr    syn-ack ttl 62 1-4 (RPC #100021)
49739/tcp open     mountd      syn-ack ttl 62 1-3 (RPC #100005)
58101/tcp open     mountd      syn-ack ttl 62 1-3 (RPC #100005)
58515/tcp open     mountd      syn-ack ttl 62 1-3 (RPC #100005) 

```

The Nmap scan comes back with a rich set of results, revealing a target with a surprisingly large attack surface. Here is a quick overview of what we are working with.

The machine is running SSH on port 22, which could be useful later if we manage to obtain valid credentials. Ports 139 and 445 confirm the presence of Samba, a file sharing service that is often a goldmine for misconfigured shares and sensitive files. Port 111 exposes RPCBind alongside NFS on port 2049, meaning there may be network file shares we can mount and explore. Port 873 reveals Rsync, a file synchronisation service that if misconfigured could allow unauthenticated access to files on the server. Port 6379 is running Redis, an in-memory data store that is frequently left open without authentication. Port 43057 shows Java RMI, and port 9090 appears filtered. The remaining ports are related to mountd and nlockmgr, supporting the NFS service.

There is plenty to work with here. Given the number of potentially misconfigured services, we will start our investigation with Samba and work our way through the rest methodically.

---

## 🗺️ Phase 2: Enumeration Spree & Dead Ends

### 1. Samba Share Assessment (Null Session)

Using NetExec (nxc) to enumerate the Samba shares on the target with a null authentication attempt, meaning we are trying to list available shares without providing any credentials at all. The fact that this works is already a misconfiguration worth noting. Guest should work too and that is what we gonna use to connect.

```bash
nxc smb vulnet.thm -u '' -p '' --shares

```

Connecting to the shares share using smbclient as a guest user, we navigate through its contents and find two directories `temp` and `data`.

```bash
smbclient //vulnet.thm/shares -U "guest"

```

Inside the `temp` folder there is a file called `services.txt`, which we download immediately. Moving over to the `data` folder reveals two more files, `data.txt` and `business-req.txt`, both of which we grab using `mget` to pull everything down at once.

Three files in hand without needing a single valid credential let's open them up and see what information they contain.

In the services file we get our first flag, let's keep going!

After reviewing the downloaded files, none of them contain anything immediately actionable or useful for gaining further access to the machine. Rather than going down a rabbit hole and wasting valuable time, it's better to move on and shift focus to the other services we identified during the initial scan.

### 2. Redis Auditing (Port 6379)

Shifting focus over to port 6379, the Nmap scan confirmed that Redis is running on the target. Redis is an in-memory key-value store that when left unauthenticated and exposed can be a serious security risk. Let's connect to it and see what kind of information we can pull out of it.

```bash
redis-cli -h vulnet.thm

```

Redis is requiring credentials to connect, so a direct approach isn't going to work just yet. Rather than hitting a dead wall, the smarter move is to step back and look at the other services running on the machine. One of them may be leaking credentials or storing configuration files that could give us the access we need to authenticate with Redis.

### 3. Rsync Probing (Port 873)

Pivoting over to Rsync on port 873, a quick Nmap script scan reveals a module called `files` described as necessary home interaction — that's an interesting hint. However, attempting to connect and list the contents immediately prompts for a password, and without valid credentials the connection is rejected with an authentication error.

Two services down and still no credentials in hand. Time to rethink the approach and look elsewhere for a way to get that password.

### 4. Java RMI (Port 43057)

Shifting attention over to Java RMI on port 43057, the tool of choice here is `remote-method-guesser`, which first needs to be built from source using Maven before it can be used. After getting it compiled and running the enumeration against the target, the results confirm that this port is not an RMI registry, exposes no codebases, and has JEP290 installed which protects against deserialization attacks. With no clear attack surface here, Java RMI turns out to be another dead end and it's time to look elsewhere.

---

## 🔓 Phase 3: Initial Access Vector

### NFS Share Exploitation

Turning to NFS which was spotted during the initial scan, running `rpcinfo` confirms the service is active and `showmount` reveals that the `/opt/conf` directory is being exported and accessible to anyone. This is a significant misconfiguration that we can take full advantage of.

```bash
showmount -e vulnet.thm

```

Creating a mount point and mounting the share locally gives us direct access to the target's configuration files.

```bash
mkdir /mnt/nfs_loot
sudo mount -t nfs vulnet.thm:/opt/conf /mnt/nfs_loot -o nolock

```

Browsing through the mounted directory reveals several interesting folders, but the one that immediately stands out is the `redis` folder. Inside it sits the Redis configuration file, and a quick grep for the `requirepass` directive hands us exactly what we have been looking for — the Redis password.

```bash
grep "requirepass" /mnt/nfs_loot/redis/redis.conf

```

After all those dead ends, the NFS share turns out to be the key that unlocks everything.

### Pivoting to Redis & Credential Extraction

With the password in hand, connecting to Redis and authenticating goes smoothly this time. The `AUTH` command accepts the credentials and returns OK, confirming we now have authenticated access to the database. Let's start enumerating its contents and see what useful information is stored inside.

```bash
redis-cli -h vulnet.thm -a <PASSWORD>

```

Enumerating the keys stored in Redis reveals five entries, with `"internal flag"` immediately catching the eye.

```text
127.0.0.1:6379> keys *

```

Retrieving its value hands us the internal flag directly. It's worth noting that working out the correct syntax for this version of Redis took a bit of trial and error, as the `TYPE` command doesn't handle keys with spaces the way you might expect — wrapping the key name in quotes turned out to be the solution.

```text
127.0.0.1:6379> get "internal flag"

```

Digging into the `authlist` key reveals a list of Base64 encoded strings. Decoding one of them exposes credentials for Rsync, including a username of `rsync-connect` and a password. This is exactly the breakthrough needed to revisit the Rsync service that was blocked behind authentication earlier in the engagement.

```bash
echo "Y29tcGFjdF9iYXNlNjRfc3RyaW5n..." | base64 -d

```

### Exploiting Rsync for Foothold

Armed with the Rsync credentials, connecting to the `files` share this time goes through without any issues. The directory listing reveals three user folders `ssm-user`, `sys-internal`, and `ubuntu`. The most interesting one here is `sys-internal`, which looks like a real user's home directory and could contain sensitive files worth exploring further.

Using rsync with the `-avz` flags to download the entire `sys-internal` home directory to our local machine, giving us a complete copy of the user's files to examine offline. This is a clean and efficient way to pull everything down at once and go through the contents at our own pace without staying connected to the service.

```bash
rsync -avz rsync-connect@vulnet.thm::files/sys-internal ./rsync_loot

```

Inside we get our user flag!

#### SSH Key Injection Persistence

With the Rsync write access discovered, the approach here is clever: generating a fresh SSH key pair locally and then uploading the public key directly into the `sys-internal` user's `authorized_keys` file via Rsync. This effectively plants our key on the target without needing to know the user's password.

```bash
ssh-keygen -f id_rsa_temp
rsync -avz id_rsa_temp.pub rsync-connect@vulnet.thm::files/sys-internal/.ssh/authorized_keys

```

Connecting via SSH using the private key works immediately and we land a stable shell as `sys-internal` on the target. A much cleaner and more reliable foothold than a reverse shell.

```bash
ssh -i id_rsa_temp sys-internal@vulnet.thm

```

---

## 🚀 Phase 4: Privilege Escalation

### Local Enumeration & Port Forwarding

With a stable shell established, the next step is running LinPEAS to automate the privilege escalation enumeration. Setting up a quick Python HTTP server on the attacking machine allows us to serve the script directly to the target. From there, downloading it with `wget`, making it executable with `chmod`, and running it will give us a thorough overview of any potential privilege escalation vectors on the system.

After LinPEAS doesn't reveal anything immediately obvious, a manual look around the filesystem turns up something interesting: a directory called `TeamCity` sitting at the root level. TeamCity is a CI/CD build server by JetBrains that is often running internal web services and can be a goldmine for credentials and misconfigurations. This is definitely worth investigating further.

Running `ss -tno` to check active network connections reveals that TeamCity is listening internally on port `8111` but is not exposed externally. To access it from our attacking machine, we set up SSH local port forwarding with the `-L` flag, tunneling port `8111` through our existing SSH session. This allows us to reach the TeamCity web interface by simply pointing our browser to `localhost:8111`.

```bash
ssh -L 8111:127.0.0.1:8111 -i id_rsa_temp sys-internal@vulnet.thm

```

We grep the super user token from the logs to authenticate:

```bash
grep -i "Super user token" /TeamCity/logs/teamcity-server.log

```

### Abusing the CI/CD Pipeline (Root RCE)

To explore this we start creating a project and we create a build too.

Navigating to the TeamCity interface through the tunnel, we can create a new build configuration and add a build step. The key insight here is selecting **Command Line** as the runner type — since the TeamCity server is running as root, any command executed through a build step will run with root level privileges. This means we can use it to execute arbitrary system commands as root, making it our path to full privilege escalation.

Using the TeamCity build step to execute a `chmod u+s /bin/bash` command sets the SUID bit on bash, which means any user can then run bash with the permissions of its owner, in this case root.

We save and run the build. Once the build is triggered and the command executes, we can spawn a root shell directly from our `sys-internal` session.

### Spawning the Root Shell

Running `/bin/bash -p` takes advantage of the SUID bit we just set, spawning a bash shell that inherits root privileges. We are now fully root on the machine and the root flag is ours to collect.

```bash
/bin/bash -p

```

```text
whoami
root
cat /root/root.txt

```

A creative and satisfying path to privilege escalation abusing a CI/CD server running as root to manipulate file permissions and pop a root shell is a technique that really highlights the danger of misconfigured internal services.

---

## 📝 Conclusion & Key Takeaways

VulNet was a fantastic machine that truly tested patience and methodology from start to finish. What made this box stand out was the sheer number of services running on the target, each one requiring a different approach and mindset to evaluate properly.

The path to initial access was far from straightforward — after hitting dead ends with Samba, Redis, Rsync, Java RMI, and even LinPEAS, it was a combination of services working together that ultimately opened the door. The NFS share leaked Redis credentials, Redis leaked Rsync credentials, and Rsync write access allowed us to plant our SSH key and get a stable foothold on the machine.

The privilege escalation was particularly creative, abusing a TeamCity CI/CD server running internally as root to set the SUID bit on bash and spawn a root shell — a great reminder of how dangerous misconfigured internal services can be even when they are not directly exposed to the outside world.

Overall this machine covered an impressive range of techniques including NFS enumeration, Redis exploitation, Rsync abuse, SSH key injection, port forwarding, and CI/CD server abuse. It's a brilliant box for anyone looking to sharpen their enumeration skills and learn to think across multiple services simultaneously.

Thanks for reading and as always, any feedback is welcome. More write-ups are on the way!

```

```
