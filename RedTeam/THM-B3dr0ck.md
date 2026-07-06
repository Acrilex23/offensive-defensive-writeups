# [THM] Bedrock — Write-up

## 🎯 Room Overview

Bedrock was a brilliantly creative and fun machine that stood out from the rest thanks to its charming Flintstones theme woven throughout every step of the challenge.

* **Target:** `bedrock.thm`
* **Difficulty:** Medium
* **Objective:** Enumerate custom network services, extract SSL client certificates without authentication, pivot through multiple character accounts (Barney and Fred), and abuse encoding-based binaries to read root-owned secrets.
* **Skills Covered:** SSL/TLS Certificate Authentication, Custom Service Probing, Cryptographic Hash Analysis, SUID/Sudo Misconfiguration Abuse, Defensive Encoding Bypass.

---

## 🔍 Phase 1: Enumeration & Reconnaissance

### Initial Nmap Scan
The Nmap scan is run with the same reliable set of flags used throughout these write-ups, scanning all ports with service and version detection, OS fingerprinting, default scripts, and aggressive timing to get a comprehensive picture of the target as quickly as possible.

```bash
nmap -p- -sS -sC -sV -O -T4 --min-rate 500 bedrock.thm -oN full_scan.txt -vv

```

### Scan Results

```text
PORT      STATE SERVICE      VERSION
22/tcp    open  ssh          OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 15:8c:0c:ef:54:fe:d1:95:1b:17:25:f9:d6:ee:fd:b2 (RSA)
|   256 24:70:2a:06:87:7b:0e:00:56:81:c8:9b:10:26:88:32 (ECDSA)
|_  256 de:61:46:9c:52:65:c2:6f:3e:20:32:39:8d:b9:6e:ae (ED25519)
80/tcp    open  http         nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to [https://bedrock.thm:4040/](https://bedrock.thm:4040/)
4040/tcp  open  ssl/yo-main?
| tls-alpn: 
|_  http/1.1
| fingerprint-strings: 
|   GetRequest, HTTPOptions: 
|     HTTP/1.1 200 OK
|     Content-type: text/html
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <title>ABC</title>
|     </head>
|     <body>
|     <h1>Welcome to ABC!</h1>
|     <p>Abbadabba Broadcasting Company</p>
|     <p>We're in the process of building a website! Can you believe this technology exists in bedrock?!?</p>
|     <p>Barney is helping to setup the server, and he said this info was important...</p>
|     <pre>
|     Hey, it's Barney. I only figured out nginx so far, what the h3ll is a database?!?
|     Bamm Bamm tried to setup a sql database, but I don't see it running.
|     Looks like it started something else, but I'm not sure how to turn it off...
|     said it was from the toilet and OVER 9000!
|_    Need to try and secure
9009/tcp  open  pichat?
| fingerprint-strings: 
|   NULL: 
|     ____ _____ 
|     \    / / | | | | /  | _ \ / ____|
|      \  / /__| | ___ ___ _ __ ___ ___ | |_ ___ /  \ | |_) | | 
|       \/ / _ \|/ __/ _ \| '_ ` _ \ / _ \| __/ _ \ / /\ \|  _ <| | 
|       / / __/ | (_| (_) | | | | | | __/ | || (_) / ____ \| |_) | |____ 
|      /_/\___|_|\___\___/|_| |_| |_|\___|  \___\___/_/    \_\____/|______|
|_    What are you looking for?
54321/tcp open  ssl/unknown

```

### Attack Surface Analysis

The results reveal a pretty interesting and characterful machine.

* **Port 22:** Running SSH as expected, nothing unusual there.
* **Port 80/4040:** Port 80 is running nginx but immediately redirects to `https://bedrock.thm:4040`, so the real web interface is sitting on port 4040 over SSL. Accessing it reveals a page for a company called ABC (Abbadabba Broadcasting Company), which is clearly a fun Flintstones themed machine. The page itself leaks some useful hints, mentioning that Barney has been setting up the server and that a database was attempted but something else started in its place with a mention of being *"from the toilet and OVER 9000"*, which is a strong hint pointing us toward port 9009.
* **Port 9009:** Reveals what appears to be a custom chat service called *Welcome to ABC*, presenting an interactive prompt asking what we are looking for.
* **Port 54321:** Running an unknown SSL service that will need further investigation.

There is plenty of character in this machine and some solid clues hidden in plain sight. Let's start pulling on those threads.

---

## 🗺️ Phase 2: Custom Service Exploitation & Initial Access

### Certificate Leakage on Port 9009

Connecting to port 9009 via Telnet reveals an interactive service that turns out to be a certificate recovery tool. By simply asking for the private key and then the client certificate, the service hands both over without any authentication — a serious misconfiguration that gives us exactly what we need.

```bash
telnet bedrock.thm 9009

```

> **Interactive Prompt Capture:**
> * Query: `private key` -> *Outputs Barney's RSA Private Key*
> * Query: `certificate` -> *Outputs Barney's Client Certificate*
> 
> 

The private key and certificate appear to belong to Barney Rubble, which ties back to the hints dropped on the web page earlier. These credentials are likely what we need to authenticate against the mysterious SSL service running on port 54321.

### Authenticating to the Custom SSL Shell

Using OpenSSL to connect to port 54321, passing the client certificate and private key we retrieved from the chat service.

```bash
openssl s_client -connect bedrock.thm:54321 -cert barney.crt -key barney.key

```

Despite the self-signed certificate warning, the connection goes through and we are greeted with a *Yabba Dabba Do* themed banner confirming that Barney Rubble is authorized. We now have an authenticated shell prompt on this custom service — let's see what commands are available and what we can do from here.

---

## 🕵️ Phase 3: Lateral Movement (Barney -> Fred)

### Breaking the Password Hint

Running the `hint` command on the service reveals a password hint for Barney Rubble in the form of an MD5 hash. The next step is to crack this hash and use the recovered password to further our access on the machine.

After John the Ripper fails to crack the hash against the `rockyou.txt` wordlist, a simple but clever intuition kicks in: **what if the hash itself is the password?** Testing the MD5 hash string directly as Barney's password turns out to be exactly right. Sometimes the most obvious approach is the correct one, and it's a good reminder not to overlook the simple possibilities when the complex ones aren't working.

Logging into the system via SSH using the hash string as the password works, and we get our first flag here!

```bash
ssh barney@bedrock.thm

```

### Abusing `certutil` Sudo Rights

Running `sudo -l` reveals that Barney has permission to run `certutil` as root. This is a custom utility that generates certificates and private keys for users on the system.

Using it to generate credentials for Fred Flintstone, another user on the machine, hands us a fresh private key and certificate for his account.

```bash
sudo certutil --user fred --out fred_cert

```

With Fred's credentials in hand, we can now authenticate to the SSL service on port 54321 as Fred and see what additional access or privileges his account provides.

---

## 🚀 Phase 4: Privilege Escalation to Root

### Extracting Fred's Credentials

Connecting to the SSL service on port 54321 using Fred's freshly generated credentials works perfectly.

```bash
openssl s_client -connect bedrock.thm:54321 -cert fred.crt -key fred.key

```

Authenticated as Fred Flintstone, running the `hint` command this time reveals his password in plain text: `YabbaDabbaD0000!` — no cracking required. With Fred's password in hand, we can now attempt to switch to his account and check what privileges he has on the system.

```bash
su fred
# Input password: YabbaDabbaD0000!

```

We get Fred's flag too now!

### Exploiting Sudo Bypasses (`base64`/`base32`)

Running `sudo -l` as Fred reveals two interesting permissions: he can run both `base32` and `base64` against `/root/pass.txt` without a password.

While directly reading or catting the file is denied, using `sudo base64` to encode the file contents and then piping it through `base64 --decode` on our end reads the file perfectly, bypassing the permission restriction entirely.

```bash
sudo base64 /root/pass.txt | base64 --decode

```

The output returned is a Base32 encoded string that we will need to decode one more time to reveal the actual root password.

### Cracking the Final Secret

Taking the Base32 encoded output, I used CyberChef to handle the structural layer. Then, dropping the resulting clean string into CrackStation as suggested by the room hints decodes it and reveals the root password directly.

With the plaintext credential in hand, we can authenticate as root and fully own the machine.

```bash
su root

```

And just like that, we get the root flag!

---

## 📝 Conclusion & Key Takeaways

The path to root was a clever chain of steps that required thinking across multiple services and reading between the lines of the hints left on the web page.

1. **Insecure Certificate Access:** The custom chat service on port 9009 handing over certificates without any authentication was a great entry point, and the SSL authenticated shell on port 54321 added a layer of uniqueness rarely seen in CTF boxes.
2. **Horizontal Privilege Chains:** Privilege escalation through `certutil` was particularly clever — using Barney's sudo permissions to generate credentials for Fred.
3. **Data Encoding Abuse:** Leveraging Fred's sudo permissions to read a root-owned file through base64/base32 encoding actions was a satisfying and well-designed chain that rewarded careful enumeration and creative thinking.

Overall, Bedrock covered a solid range of techniques including SSL certificate authentication, custom service enumeration, hash analysis, sudo misconfiguration abuse, and encoding-based file reads. The Flintstones theme made it an enjoyable and memorable experience from start to finish.

Thanks for reading and as always any feedback is welcome — more write-ups are on the way!
