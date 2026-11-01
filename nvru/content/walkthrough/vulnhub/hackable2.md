+++
title = "Hackable II - Walkthrough"
date = "2026-07-27"
author = "nvru"
description = "Walkthrough for the Hackable II VulnHub machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "ftp",
  "apache",
  "anonymous-ftp",
  "file-upload",
  "reverse-shell",
  "sudo",
  "python",
  "privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.80.46.142`
- **Target Machine IP:** `10.80.46.210`

The objective was to enumerate the target, abuse anonymous FTP file uploads to gain initial access through a web-accessible upload directory, and escalate privileges by abusing a misconfigured `sudo` rule allowing Python execution as root.

![Machine information](/images/vulnhub/hackable2/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.80.46.210 -- -A -oA nmap/rust
```

### Results

```text
Open 10.80.46.210:21
Open 10.80.46.210:22
Open 10.80.46.210:80
```

A full Nmap scan was then performed.

```bash
nmap -sSCV -A -oN nmap/ports 10.80.46.210 -p21,22,80
```

```text
PORT   STATE SERVICE VERSION

21/tcp open  ftp     ProFTPD
| ftp-anon: Anonymous FTP login allowed
|_-rw-r--r--   1 0 0 109 Nov 26 2020 CALL.html

22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu

80/tcp open  http    Apache 2.4.18 (Ubuntu)
```

Only three services were exposed, making them the primary focus for enumeration.

---

## Enumeration

### FTP

Anonymous FTP authentication was enabled.

![FTP Login](/images/vulnhub/hackable2/ftp.png)

Only a single file named `CALL.html` was present.

Inspecting the file did not immediately reveal anything useful.

![CALL.html](/images/vulnhub/hackable2/cat_ftp.png)

While testing the FTP service, it was discovered that anonymous users could also upload files.

At this point, I suspected the uploaded files might be served by the web server.

---

### HTTP

Browsing to port **80** showed the default Apache page.

Viewing the page source revealed a hidden hint encouraging directory brute forcing.

![HTTP Source](/images/vulnhub/hackable2/http_src.png)

---

### Directory Enumeration

To locate hidden directories, I used Feroxbuster.

```bash
feroxbuster \
-u http://10.80.46.210 \
-w ~/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt \
-x txt,html,js,php,png,jpg,jpeg,xml,sql
```

The scan quickly identified a directory exposing uploaded FTP files.

![Directory Enumeration](/images/vulnhub/hackable2/dir_brute.png)

The previously uploaded test file (`CALL1.html`) was accessible through the browser, confirming that files uploaded via FTP were served directly by Apache.

![FTP WEB](/images/vulnhub/hackable2/it_worked.png)

This provided an ideal path for uploading a web shell.

---

## Initial Access

A reverse shell payload was uploaded through FTP and executed from the discovered web directory.

The payload connected back successfully, resulting in a shell as:

```text
www-data
```

![Reverse Shell Upload](/images/vulnhub/hackable2/upload_revshell.png)

---

## Privilege Escalation

### www-data → shrek

During local enumeration, I discovered a hidden file named:

```text
.runme.sh
```

![Found .runme.sh](/images/vulnhub/hackable2/fnd_runme.png)

Reading the file revealed what appeared to be credentials:

```text
cf4c2232354952690368f1b3dfdfb24d:onion
```

![Contents of .runme.sh](/images/vulnhub/hackable2/cat_runme.png)

Using these credentials allowed me to switch to the `shrek` user.

```text
shrek:onion
```

```bash
su shrek
```

![Switch to shrek](/images/vulnhub/hackable2/brute_su_shrek.png)

---

### shrek → root

The first step was checking sudo permissions.

```bash
sudo -l
```

The output showed that `shrek` could execute **Python 3.5** as root without supplying a password.

This is dangerous because Python can execute arbitrary operating system commands through the `os` module.

The following commands were sufficient to obtain a root shell.

```python
import os
os.system("/bin/bash")
```

Executing the payload resulted in a root shell.

![Root Shell](/images/vulnhub/hackable2/su_root.png)

---

## Credentials

```text
shrek:onion
```

---

## Flags

The root flag was successfully retrieved.

![Root Flag](/images/vulnhub/hackable2/root_flag.png)
