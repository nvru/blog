+++
title = "Who Wants to Be King? 1 - Walkthrough"
date = "2026-07-20"
author = "nvru"
description = "Walkthrough for the Who Wants to Be King? 1 machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "http",
  "ssh",
  "credentials",
  "password-reuse",
  "privilege-escalation",
   "easy"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.245.58.142`
- **Target Machine IP:** `10.245.58.27`

The objective was to enumerate the target, identify exposed services, gain initial access by recovering credentials from an exposed binary, and escalate privileges by discovering the root password through user artifacts.

![Machine information](/images/vulnhub/who_wants_to_be_king1/logscreen.png)

---

## Enumeration

### Port Scanning

#### RustScan

```bash
rustscan -a 10.245.58.27
```

Output:

```text
Open 10.245.58.27:22
Open 10.245.58.27:80
```

#### Nmap

```bash
sudo nmap -sSCV 10.245.58.27 -A -p22,80 -oN nmap/ports
```

Results:

```text
PORT   STATE SERVICE VERSION

22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4
80/tcp open  http    Apache httpd 2.4.41

http-title: Index of /
http-server-header: Apache/2.4.41 (Ubuntu)

Index of /

31K  skeylogger
```

Only SSH and HTTP were exposed. The web server had directory listing enabled and exposed a single file named **skeylogger**, making it the obvious target for further enumeration.

![HTTP Directory Listing](/images/vulnhub/who_wants_to_be_king1/http.png)

---

### Analyzing the Exposed Binary

The binary was downloaded for offline analysis.

```bash
wget http://10.245.58.27/skeylogger
```

Checking the file type:

```bash
file skeylogger
```

Output:

```text
skeylogger: ELF 64-bit LSB pie executable, x86-64, dynamically linked,
with debug_info, not stripped
```

Since the binary contained debug symbols, extracting printable strings was the next step.

```bash
strings -n 10 skeylogger
```

Among the output was a Base64-encoded string:

```text
ZHJhY2FyeXMK
```

Decoding it:

```bash
echo "ZHJhY2FyeXMK" | base64 -d
```

Output:

```text
dracarys
```

The decoded string looked like a password. From the VM login screen, the username **daenerys** was already known, making SSH the next thing to try.

---

## Initial Access

Using the recovered credentials:

```text
Username: daenerys
Password: dracarys
```

SSH authentication succeeded.

![SSH Login](/images/vulnhub/who_wants_to_be_king1/ssh_daenerys.png)

---

## Privilege Escalation

### Inspecting User Files

After gaining access as **daenerys**, I started the usual post-exploitation enumeration.

One of the first files inspected was the shell history.

```bash
cat ~/.bash_history
```

The history referenced files inside the user's `.local/share` directory.

![Bash History](/images/vulnhub/who_wants_to_be_king1/bash_hist.png)

Inside the directory was a ZIP archive.

After extracting it, I found a `note.txt` file containing only part of a message.

![Recovered Note](/images/vulnhub/who_wants_to_be_king1/note.png)

The remaining directories did not reveal anything useful, so I searched online for the missing part of the note.

---

### Recovering the Root Password

The incomplete message referenced a Game of Thrones character. Searching for the text revealed the missing name:

```text
Khal Drogo
```

This suggested the password:

```text
khaldrogo
```

![Recovered Password](/images/vulnhub/who_wants_to_be_king1/found_khal.png)

Looking back at `.bash_history`, I noticed the user had switched to **root** immediately after reading the note. That strongly suggested the recovered password belonged to the root account.

Using `su`:

```bash
su -
```

Password:

```text
khaldrogo
```

The password was correct, resulting in a root shell.

![Root Shell](/images/vulnhub/who_wants_to_be_king1/su_root.png)

---

## Credentials

```text
daenerys:dracarys

root:khaldrogo
```
