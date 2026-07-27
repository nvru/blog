+++
title = "Noob - Walkthrough"
date = "2026-07-24"
author = "nvru"
description = "Walkthrough for the Noob machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "http",
  "ssh",
  "symlink",
  "initial-access",
  "privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.37`

The objective was to enumerate the target, identify exposed services, gain initial access by recovering an exposed SSH private key from the web server, and escalate privileges by abusing the web server's ability to access files through symbolic links.

![Machine information](/images/hmv/noob/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.37 -- -sC -sV -A -oA nmap/rust
```

### Results

```text
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2
65530/tcp open  http    Golang net/http server
```

Only two services were exposed:

- SSH on port **22**
- A Golang HTTP server on port **65530**

With a very small attack surface, the web service became the primary focus.

---

## Enumeration

### HTTP

Browsing to the web service returned only a simple **404** page with no additional information.

![HTTP](/images/hmv/noob/http.png)

I also checked for a `robots.txt` file, but none existed.

![Robots](/images/hmv/noob/robots.png)

---

### Directory Enumeration

Since the default page was unhelpful, I performed directory brute forcing.

```bash
ffuf -u http://10.10.10.37:65530/FUZZ \
     -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

The scan revealed an `index` page.

```text
/index
```

![Found index](/images/hmv/noob/found_index.png)

Opening the page displayed only a short message.

```text
Hi, You are close!
```

![Index page](/images/hmv/noob/index.png)

Continuing the enumeration revealed another interesting directory.

```text
/nt4share
```

![NT4Share](/images/hmv/noob/n4share.png)

---

### Exposed SSH Key

Inside `/nt4share`, I found an SSH private key (`id_rsa`).

![SSH key](/images/hmv/noob/ssh_key_web.png)

The corresponding public key was also available.

![Public key](/images/hmv/noob/pub_ssh_web.png)

Inspecting the public key revealed the username:

```text
adela
```

---

## Initial Access

Using the recovered private key, I authenticated through SSH.

```bash
chmod 600 id_rsa
ssh -i id_rsa adela@10.10.10.37
```

The login was successful.

![SSH as adela](/images/hmv/noob/ssh_adela.png)

---

## Privilege Escalation

### Enumeration

The first step was checking for common privilege escalation vectors.

I attempted to check sudo permissions.

```bash
sudo -l
```

However, `sudo` was not installed.

I also checked for `doas`, but it was unavailable.

![sudo / doas](/images/hmv/noob/sudo_doas.png)

Next, I searched for Linux capabilities.

```bash
getcap -r / 2>/dev/null
```

No interesting capabilities were found.

![Capabilities](/images/hmv/noob/cap.png)

Finally, I searched for SUID binaries.

```bash
find / -perm -4000 -type f 2>/dev/null
```

Nothing useful was present.

![SUID binaries](/images/hmv/noob/suid.png)

---

### Reading Root Files via Symlink

Since the traditional privilege escalation vectors failed, I considered the behavior of the Golang web server.

If the web service was running as **root**, it might follow symbolic links and expose files outside of its intended directory.

I created a symbolic link pointing to `/root`.

```bash
ln -s /root root
```

Requesting the linked directory through the web server exposed the contents of `/root`.

Among the files was the root user's SSH private key.

![Root SSH key](/images/hmv/noob/root_ssh_key.png)

After downloading the key, I authenticated directly as root.

```bash
chmod 600 root_id_rsa
ssh -i root_id_rsa root@10.10.10.37
```

The login succeeded, resulting in full system compromise.

![Root shell](/images/hmv/noob/ssh_root.png)

---

## Credentials

```text
adela:<SSH private key>

root:<SSH private key>
```

---

## Flags

```text
user:HMVVCXUIUHFDSUIYREWDS432

root:HMVA97DSA8732HJGDSA78623
```
