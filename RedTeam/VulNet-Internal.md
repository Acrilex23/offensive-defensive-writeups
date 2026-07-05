Starting off the enumeration phase with a thorough Nmap scan against the target. Let's break down what each flag is doing and why it matters.

nmap -p- -sS -sC -sV -O -T4 --min-rate 500 vulnet.thm -oN full.txt -vv

    p- tells Nmap to scan all 65535 ports rather than just the default top 1000, ensuring nothing is missed on unusual ports.

    sS performs a SYN scan, also known as a stealth scan, which sends SYN packets without completing the full TCP handshake, making it faster and less noisy than a full connection scan.

    sC runs Nmap's default scripts against the discovered ports, which can automatically pull useful information like service banners, authentication methods and potential misconfigurations.

    sV probes open ports to detect the exact service version running, giving us more precise information about what we are dealing with.

    O attempts to identify the operating system of the target based on how it responds to certain packets.

    T4 sets the timing template to aggressive, speeding up the scan significantly while remaining reliable enough for most environments.

    -min-rate 500 ensures Nmap sends at least 500 packets per second, keeping the scan moving at a good pace.

    oN full.txt saves the output to a file called full.txt so we have a clean record to reference throughout the engagement.

    vv enables extra verbose output, letting us see results as they come in rather than waiting for the scan to finish.

PORT      STATE    SERVICE     REASON         VERSION
22/tcp    open     ssh         syn-ack ttl 62 OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 56:3a:00:62:b1:f6:e6:f0:a6:0f:5c:0a:3f:ff:00:63 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDBttqsKNpG4Un/Eh3gSpI4tHwIU2WbAeWqDtDg6gx9APB4nyle60lKe1/hDteJjsYZw+UNnFHxsfRSqaF5lcK74KhimBaLdAQ5mwUOFsA/dsMKxyr03UZp9JV9IreXhBt+HSswO1NSgFKRbSdoDMD8efyj4DoAsFIFR3i0Yzk3Az/ro+vwEdJsFvNtW+GZ1ycXI4B/JmvLC5MAGH+Ayl9E7HnBGRfbSmYhBOUqLv6I7u92LCsoJBcpAr8y5Qs8HpVwDW9coIBKiU4BvZncpXIVaX5NomIvq078Q+PGOlwbi5yjeK6SwCRqRnwxM8ATVJ8hBR+MAF5An4OupWmoVNEkS/P/GW44vGWhPQB3P2CyUaRA9vF6JLSJcLcep8VOEkbYIKphmwzPiywiufQV/MvtapAtE4xfgKvY668L7NYrA2oE5/7HjbnodJFAZ7oFqS3tcGXDw3lD41uDTaNcai/lSvNMJPjVAE6wBVXS/Bp88RjD54WZ6DwFlYs2avdfCM8=
|   256 2d:68:41:3a:96:89:40:bd:f9:7b:61:75:aa:07:45:01 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBI+/Tx6HevQ/H3lQu7siSDiMhA2fBFNI4Sk2oWdHP8mvZM6R+rS2HWz4d5aoM1VIwfukdmg/vD8g2CYsy7UzrA4=
|   256 7b:bd:5d:a9:eb:7e:81:5b:4e:52:ca:ce:15:a0:19:90 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIF5ROgESOzhTJdprdLQj6cSBU0hJNejZ1qgPhIviUrgx
111/tcp   open     rpcbind     syn-ack ttl 62 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      51475/udp   mountd
|   100005  1,2,3      56393/udp6  mountd
|   100005  1,2,3      58515/tcp   mountd
|   100005  1,2,3      60037/tcp6  mountd
|   100021  1,3,4      36109/tcp6  nlockmgr
|   100021  1,3,4      38131/udp   nlockmgr
|   100021  1,3,4      46079/tcp   nlockmgr
|   100021  1,3,4      51510/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
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

The Nmap scan comes back with a rich set of results, revealing a target with a surprisingly large attack surface. Here is a quick overview of what we are working with.

The machine is running SSH on port 22, which could be useful later if we manage to obtain valid credentials. Ports 139 and 445 confirm the presence of Samba, a file sharing service that is often a goldmine for misconfigured shares and sensitive files. Port 111 exposes RPCBind alongside NFS on port 2049, meaning there may be network file shares we can mount and explore. Port 873 reveals Rsync, a file synchronisation service that if misconfigured could allow unauthenticated access to files on the server. Port 6379 is running Redis, an in-memory data store that is frequently left open without authentication. Port 43057 shows Java RMI, and port 9090 appears filtered. The remaining ports are related to mountd and nlockmgr, supporting the NFS service.

There is plenty to work with here. Given the number of potentially misconfigured services, we will start our investigation with Samba and work our way through the rest methodically.

Using NetExec (nxc) to enumerate the Samba shares on the target with a null authentication attempt, meaning we are trying to list available shares without providing any credentials at all. The fact that this works is already a misconfiguration worth noting. Guest should work too and that is what we gonna use to connect.

Connecting to the shares share using smbclient as a guest user, we navigate through its contents and find two directories temp and data.

Inside the temp folder there is a file called services.txt, which we download immediately. Moving over to the data folder reveals two more files, data.txt and business-req.txt, both of which we grab using mget to pull everything down at once.

Three files in hand without needing a single valid credential let's open them up and see what information they contain.

In the services file we get our first flag, lest keep going

After reviewing the downloaded files, none of them contain anything immediately actionable or useful for gaining further access to the machine. Rather than going down a rabbit hole and wasting valuable time, it's better to move on and shift focus to the other services we identified during the initial scan.

Shifting focus over to port 6379, the Nmap scan confirmed that Redis is running on the target. Redis is an in-memory key-value store that when left unauthenticated and exposed can be a serious security risk. Let's connect to it and see what kind of information we can pull out of it.

Redis is requiring credentials to connect, so a direct approach isn't going to work just yet. Rather than hitting a dead wall, the smarter move is to step back and look at the other services running on the machine. One of them may be leaking credentials or storing configuration files that could give us the access we need to authenticate with Redis.

Pivoting over to Rsync on port 873, a quick Nmap script scan reveals a module called files described as necessary home interaction — that's an interesting hint. However, attempting to connect and list the contents immediately prompts for a password, and without valid credentials the connection is rejected with an authentication error.

Two services down and still no credentials in hand. Time to rethink the approach and look elsewhere for a way to get that password.

Shifting attention over to Java RMI on port 43057, the tool of choice here is remote-method-guesser, which first needs to be built from source using Maven before it can be used. After getting it compiled and running the enumeration against the target, the results confirm that this port is not an RMI registry, exposes no codebases, and has JEP290 installed which protects against deserialization attacks. With no clear attack surface here, Java RMI turns out to be another dead end and it's time to look elsewhere.

Turning to NFS which was spotted during the initial scan, running rpcinfo confirms the service is active and showmount reveals that the /opt/conf directory is being exported and accessible to anyone. This is a significant misconfiguration that we can take full advantage of.

Creating a mount point and mounting the share locally gives us direct access to the target's configuration files. Browsing through the mounted directory reveals several interesting folders, but the one that immediately stands out is the redis folder. Inside it sits the Redis configuration file, and a quick grep for the requirepass directive hands us exactly what we have been looking for — the Redis password.

After all those dead ends, the NFS share turns out to be the key that unlocks everything.

With the password in hand, connecting to Redis and authenticating goes smoothly this time. The AUTH command accepts the credentials and returns OK, confirming we now have authenticated access to the database. Let's start enumerating its contents and see what useful information is stored inside.

Enumerating the keys stored in Redis reveals five entries, with "internal flag" immediately catching the eye. Retrieving its value hands us the internal flag directly. It's worth noting that working out the correct syntax for this version of Redis took a bit of trial and error, as the TYPE command doesn't handle keys with spaces the way you might expect — wrapping the key name in quotes turned out to be the solution.

Digging into the authlist key reveals a list of Base64 encoded strings. Decoding one of them exposes credentials for Rsync, including a username of rsync-connect and a password. This is exactly the breakthrough needed to revisit the Rsync service that was blocked behind authentication earlier in the engagement.

Armed with the Rsync credentials, connecting to the files share this time goes through without any issues. The directory listing reveals three user folders ssm-user, sys-internal, and ubuntu. The most interesting one here is sys-internal, which looks like a real user's home directory and could contain sensitive files worth exploring further.

Using rsync with the -avz flags to download the entire sys-internal home directory to our local machine, giving us a complete copy of the user's files to examine offline. This is a clean and efficient way to pull everything down at once and go through the contents at our own pace without staying connected to the service.

We get our user flag

With the Rsync write access discovered, the approach here is clever generating a fresh SSH key pair locally and then uploading the public key directly into the sys-internal user's authorized_keys file via Rsync. This effectively plants our key on the target without needing to know the user's password.

Connecting via SSH using the private key works immediately and we land a stable shell as sys-internal on the target. A much cleaner and more reliable foothold than a reverse shell.

With a stable shell established, the next step is running LinPEAS to automate the privilege escalation enumeration. Setting up a quick Python HTTP server on the attacking machine allows us to serve the script directly to the target. From there, downloading it with wget, making it executable with chmod, and running it will give us a thorough overview of any potential privilege escalation vectors on the system.

After LinPEAS doesn't reveal anything immediately obvious, a manual look around the filesystem turns up something interesting a directory called TeamCity sitting at the root level. TeamCity is a CI/CD build server by JetBrains that is often running internal web services and can be a goldmine for credentials and misconfigurations. This is definitely worth investigating further.

Running ss -tno to check active network connections reveals that TeamCity is listening internally on port 8111 but is not exposed externally. To access it from our attacking machine, we set up SSH local port forwarding with the -L flag, tunneling port 8111 through our existing SSH session. This allows us to reach the TeamCity web interface by simply pointing our browser to localhost:8111.

We grep the super user token to authenticate

To explore this we start creating a project

And we create a build too

Navigating to the TeamCity interface through the tunnel, we can create a new build configuration and add a build step. The key insight here is selecting Command Line as the runner type — since the TeamCity server is running as root, any command executed through a build step will run with root level privileges. This means we can use it to execute arbitrary system commands as root, making it our path to full privilege escalation.

Using the TeamCity build step to execute a chmod u+s /bin/bash command sets the SUID bit on bash, which means any user can then run bash with the permissions of its owner in this case root. Once the build is triggered and the command executes, we can spawn a root shell directly from our sys-internal session.

We save and run

Running /bin/bash -p takes advantage of the SUID bit we just set, spawning a bash shell that inherits root privileges. We are now fully root on the machine and the root flag is ours to collect.

A creative and satisfying path to privilege escalation abusing a CI/CD server running as root to manipulate file permissions and pop a root shell is a technique that really highlights the danger of misconfigured internal services.

Root flag

VulNet was a fantastic machine that truly tested patience and methodology from start to finish. What made this box stand out was the sheer number of services running on the target, each one requiring a different approach and mindset to evaluate properly.

The path to initial access was far from straightforward — after hitting dead ends with Samba, Redis, Rsync, Java RMI, and even LinPEAS, it was a combination of services working together that ultimately opened the door. The NFS share leaked Redis credentials, Redis leaked Rsync credentials, and Rsync write access allowed us to plant our SSH key and get a stable foothold on the machine.

The privilege escalation was particularly creative, abusing a TeamCity CI/CD server running internally as root to set the SUID bit on bash and spawn a root shell — a great reminder of how dangerous misconfigured internal services can be even when they are not directly exposed to the outside world.

Overall this machine covered an impressive range of techniques including NFS enumeration, Redis exploitation, Rsync abuse, SSH key injection, port forwarding, and CI/CD server abuse. It's a brilliant box for anyone looking to sharpen their enumeration skills and learn to think across multiple services simultaneously.

Thanks for reading and as always, any feedback is welcome more write-ups are on the way!
