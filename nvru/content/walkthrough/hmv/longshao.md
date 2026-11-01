+++
title = "Longshao - Walkthrough"
date = "2026-07-30"
author = "nvru"
description = "Walkthrough for the Longshao machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "apache",
  "php",
  "information-disclosure",
  "ssh",
  "kernel-exploit",
  "CVE-2026-46331",
  "easy",
  "privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.33.146.142`
- **Target Machine IP:** `10.33.146.133`

The objective was to enumerate the target, identify exposed services, gain initial SSH access using disclosed credentials, and escalate privileges by exploiting **CVE-2026-46331** affecting the Alpine Linux kernel.

![Machine information](/images/hmv/longshao/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.33.146.133 -- -sC -sV -A -oN nmap/rust
```

### Results

```text
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 10.3 (protocol 2.0)
80/tcp open  http    syn-ack Apache httpd 2.4.67 ((Unix))
|_http-server-header: Apache/2.4.67 (Unix)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Maze 内部管理系统 - 登录
```

Only two services were exposed:

- **SSH** on port **22**
- **Apache HTTP Server** on port **80**

The web application became the primary focus for further enumeration.

---

## Enumeration

### Web Application

Browsing to the web server presented a simple login page.

![Login page](/images/hmv/longshao/http.png)

Inspecting the page source did not reveal anything useful, and there were no hidden comments or JavaScript files of interest.

The `robots.txt` file also did not contain any interesting entries.

![robots.txt](/images/hmv/longshao/robots.png)

Using the Wappalyzer browser extension showed the application was powered by **PHP 8.2.31** running behind Apache.

![Wappalyzer](/images/hmv/longshao/wapp.png)

After searching for publicly known vulnerabilities affecting this PHP version, nothing useful was found.

---

### Directory Enumeration

Since the login page did not provide an obvious attack surface, I performed directory brute-forcing.

```bash
gobuster dir \
-u http://10.33.146.133/ \
-w /opt/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt \
-x php,html,js,xml,db,sql,txt,png
```

A hidden page was discovered:

```text
/dashboard.php
```

![Directory enumeration](/images/hmv/longshao/dir_brute.png)

Visiting the page displayed a warning message intended for administrators. More importantly, it accidentally disclosed SSH credentials.

![Dashboard](/images/hmv/longshao/dashboard.png)

---

### Information Disclosure

The exposed page contained valid SSH credentials:

```text
Username: baolong
Password: jinhua
```

This information disclosure provided direct access to the target system.

---

## Initial Access

Using the recovered credentials, SSH authentication succeeded immediately.

```text
Username: baolong
Password: jinhua
```

![SSH Login](/images/hmv/longshao/ssh_baolong.png)

---

## Privilege Escalation

### baolong → baolong

The first step after obtaining a shell was checking for common privilege escalation vectors.

```bash
sudo -l
```

The user had no useful sudo permissions.

![sudo](/images/hmv/longshao/sudo.png)

I also checked the user's groups and searched the `/home` directory for other user accounts.

Although multiple home directories existed, they were empty.

![Home directories](/images/hmv/longshao/home.png)

To verify whether those users actually existed, I checked `/etc/passwd`.

```bash
cat /etc/passwd | grep sh$
```

![User enumeration](/images/hmv/longshao/check_users.png)

Nothing useful was obtained from user enumeration.

---

### Interesting Files

While enumerating the filesystem, I noticed two unusual files located in the root directory.

![Interesting files](/images/hmv/longshao/two_files.png)

One of them was an extremely large file that could be read by the current user.

Because of its size, I copied it to my attacking machine for offline analysis instead of processing it locally.

```bash
scp baolong@10.33.146.133:/empty .
```

![Large file](/images/hmv/longshao/huge.png)

Although these files were interesting, they ultimately were not required for privilege escalation.

---

### Kernel Vulnerability (CVE-2026-46331)

Checking the operating system version revealed the machine was running Alpine Linux with kernel version **7.0.10**.

```bash
uname -a
```

![Kernel version](/images/hmv/longshao/uname.png)

Searching for public exploits targeting this kernel version led to **CVE-2026-46331**.

Reference:

- https://github.com/yanxinwu946/CVE-2026-46331

The proof-of-concept repository even included examples using the same kernel version.

![Affected targets](/images/hmv/longshao/cve_targets.png)

After compiling and executing the exploit, root privileges were obtained successfully.

![Exploit](/images/hmv/longshao/exploit.png)

---

### Root Access

After obtaining a root shell, I navigated to the root user's home directory and found a file containing the root password.

![Password file](/images/hmv/longshao/pass.png)

Using the recovered password, I authenticated as the root user.

![Root login](/images/hmv/longshao/su.png)

---

## Credentials

```text
baolong:jinhua

root:chaojiheianwudilong
```

---

## Flags

```text
user:flag{user-3408c2a9ca636da4a40f054eea401fd9}

root:flag{root-e0bf0dabcccb7d4519c0ad4b431aff16}
```
