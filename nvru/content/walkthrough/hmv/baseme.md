+++
title = "Baseme - Walkthrough"
date = "2026-07-22"
author = "nvru"
description = "Walkthrough for the Baseme machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "nginx",
  "base64",
  "ssh",
  "sudo",
  "privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.34`

The objective was to enumerate the target, identify exposed services, gain initial access by abusing Base64-encoded resources, and escalate privileges through a misconfigured sudo rule allowing access to sensitive files.

![Machine information](/images/hmv/baseme/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.34 -- -sC -sV -A -oA nmap/rust
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2
80/tcp open  http    nginx 1.14.2
```

The scan revealed two exposed services:

- SSH on port `22`
- HTTP on port `80`

The web application was the obvious starting point.

---

## Enumeration

### Web Enumeration

Visiting the web page displayed what appeared to be a Base64-encoded string.

![HTTP page](/images/hmv/baseme/http.png)

Inspecting the page source revealed the following HTML comment:

```html
<!-- iloveyou youloveyou shelovesyou helovesyou weloveyou theyhatesme -->
```

![Page source](/images/hmv/baseme/http_src.png)

Decoding the Base64 string displayed on the page revealed the following message:

```text
ALL, absolutely ALL that you need is in BASE64.
Including the password that you need :)
Remember, BASE64 has the answer to all your questions.

-lucas
```

The message introduced the username `lucas` and strongly hinted that everything required to solve the machine was Base64 encoded.

At this point, I attempted to authenticate over SSH using the words from the HTML comment as the password, including Base64-encoded variations, but none were successful.

---

### Directory Enumeration

Since the hint repeatedly referenced Base64, I generated a Base64-encoded wordlist from SecLists and used it for directory brute forcing.

First, I generated a list from the medium DirBuster wordlist:

```bash
for i in $(cat /opt/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt); do
    echo "$i" | base64
done > misc/dir_medium_64
```

Unfortunately, this did not reveal anything useful.

Next, I generated a Base64 version of the common wordlist:

```bash
for i in $(cat /opt/wordlists/seclists/Discovery/Web-Content/common.txt); do
    echo "$i" | base64
done > misc/common_64
```

This time, two interesting paths were discovered:

```text
aWRfcnNhCg==
cm9ib3RzLnR4dAo=
```

![Directory discovery](/images/hmv/baseme/gobus_common_64.png)

---

### Recovering the SSH Key

Requesting the first path returned a very large Base64 blob.

```bash
curl -s http://10.10.10.34/aWRfcnNhCg==
```

![Downloading the key](/images/hmv/baseme/curl_sshkey_64.png)

After decoding it, the content was identified as an SSH private key.

![Decoded SSH key](/images/hmv/baseme/curl_ssh.png)

The second path did not return any useful data.

![Other endpoint](/images/hmv/baseme/other_curl_nothing.png)

Since the earlier message mentioned the user `lucas`, I attempted to authenticate using the recovered key.

SSH accepted the key but requested its passphrase.

![SSH passphrase](/images/hmv/baseme/ssh_need_pass.png)

---

## Initial Access

Initially, I attempted to crack the passphrase using RockYou:

```bash
ssh2john creds/ssh_web > misc/ssh_hash
john --wordlist=~/wordlists/rockyou.txt misc/ssh_hash
```

Since the machine revolved around Base64, I instead generated a Base64-encoded version of the RockYou wordlist.

```bash
for i in $(cat ~/wordlists/rockyou-50.txt); do
    echo "$i" | base64
done > misc/rock_64

john --wordlist=misc/rock_64 misc/ssh_hash
```

![Passphrase recovered](/images/hmv/baseme/john_pass_suc.png)

The recovered passphrase successfully unlocked the private key.

Using the decrypted key, I authenticated as `lucas`.

![SSH login](/images/hmv/baseme/ssh_lucas.png)

---

## Privilege Escalation

### lucas → root

The first thing I checked was the user's sudo permissions.

```bash
sudo -l
```

![sudo permissions](/images/hmv/baseme/sudo_perm.png)

The user could execute:

```text
/usr/bin/base64
```

as any user without supplying a password.

Although `base64` cannot execute commands directly, it can read files that the invoking user normally cannot access.

I used it to read root's private SSH key:

```bash
sudo -u root /usr/bin/base64 /root/.ssh/id_rsa
```

![Reading root SSH key](/images/hmv/baseme/sudo_root_key.png)

After copying the output and decoding it locally, the root private key was recovered.

![Decoded root key](/images/hmv/baseme/decode_root_key.png)

Using the recovered key, I authenticated directly as `root`.

![Root SSH login](/images/hmv/baseme/ssh_root.png)

Root access was obtained.

---

## Credentials

```text
User: lucas

SSH Private Key:
aWxvdmV5b3UK
```

---

## Flags

```text
user:HMV8nnJAJAJA

root:HMVFKBS64
```
