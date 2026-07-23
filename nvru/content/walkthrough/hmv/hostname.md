+++
title = "Hostname - Walkthrough"
date = "2026-07-21"
author = "nvru"
description = "Walkthrough for the Hostname machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "nginx",
  "source-code-disclosure",
  "cronjob",
  "tar",
  "privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.33`

The objective was to enumerate the target, identify exposed services, gain initial access through information disclosed in the web application, and escalate privileges by abusing a vulnerable tar backup cron job.

![Machine information](/images/hmv/hostname/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.33 -- -sC -sV -A -oA nmap/rust
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5
80/tcp open  http    nginx 1.18.0
```

The scan revealed two services:

- SSH on port `22`
- HTTP (nginx) on port `80`

The web application was the obvious starting point.

---

## Enumeration

### Web Enumeration

Browsing the website presented a simple page asking for a secret.

![HTTP page](/images/hmv/hostname/http.png)

Inspecting the page source revealed two interesting pieces of information:

![Page source](/images/hmv/hostname/http_src.png)

- Username: `po`
- Base64 string:

```text
S3VuZ19GdV9QNG5kYQ==
```

![Source code](/images/hmv/hostname/http_src.png)

Decoding the Base64 string produced:

```text
Kung_Fu_P4nda
```

![Decoded value](/images/hmv/hostname/decode.png)

Since SSH was exposed, I first attempted to authenticate using these values, but the login failed.

![Failed SSH login](/images/hmv/hostname/no_ssh.png)

---

### Hidden Parameter

After reviewing the source code again, I discovered a hidden parameter named `secret`.

![Hidden parameter](/images/hmv/hostname/secret_param.png)

Sending a POST request with the recovered values produced the following JavaScript alert:

```bash
curl -s http://10.10.10.33/ \
    -d "secret=Kung_Fu_P4nda&username=po"
```

Response:

```html
<script type="text/javascript">
  alert("!ts-bl4nk");
</script>
```

![Alert response](/images/hmv/hostname/alert.png)

The alert revealed another password:

```text
!ts-bl4nk
```

---

## Initial Access

SSH access was obtained using:

```text
Username: po
Password: !ts-bl4nk
```

Using these credentials, I successfully authenticated over SSH.

![SSH login](/images/hmv/hostname/ssh_po.png)

---

## Privilege Escalation

### po → oogway

The first check was the user's sudo permissions.

```bash
sudo -l
```

![sudo permissions](/images/hmv/hostname/sudo_perm_po.png)

The user had no direct sudo permissions.

While enumerating the system, I checked `/etc/sudoers.d/` and noticed that the current user could read the configuration files.

```bash
ls -l /etc/sudoers.d
cat /etc/sudoers.d/po
```

![sudoers file](/images/hmv/hostname/sudoers.png)

The configuration allowed the user `po` to execute Bash as `oogway`, provided the hostname was `HackMyVm`.

The following command successfully spawned a shell as `oogway`:

```bash
sudo -h HackMyVm -u oogway /bin/bash
```

![oogway shell](/images/hmv/hostname/su_oogway.png)

---

### oogway → root

As `oogway`, I checked sudo permissions again, but the account required a password that was not known.

![sudo permissions](/images/hmv/hostname/sudo_perm_oogway.png)

During enumeration, I discovered the directory `/opt/secret`.

```bash
ls -la /opt/secret
```

The directory was empty, but the user had read, write, and execute permissions.

![Secret directory](/images/hmv/hostname/opt_sec.png)

This suggested it might be used by an automated process.

Checking the system cron jobs confirmed that a cron task executed every second and archived the contents of `/opt/secret` using `tar` with a wildcard (`*`).

![Cron job](/images/hmv/hostname/crontab.png)

Since the backup used a wildcard, it was vulnerable to the classic GNU tar wildcard injection technique.

Reference:

- https://dev.to/devon_argent_f9a11303298a/day-12-auditing-linux-privilege-escalation-vectors-gm3

I placed the required payload files inside `/opt/secret` to exploit the wildcard injection. When the cron job executed, it copied `/bin/bash` to `/tmp/shell` and set the SUID bit on the copied binary.

![Tar exploitation](/images/hmv/hostname/exp_tar.png)

After the cron job ran, the SUID binary was successfully created.

![SUID bash created](/images/hmv/hostname/it_worked.png)

Executing the SUID binary with the `-p` option spawned a root shell.

```bash
/tmp/shell -p
```

![Root ](/images/hmv/hostname/shell_p.png)

---

## Credentials

```text
Web:
secret=Kung_Fu_P4nda
username=po

SSH:
po:!ts-bl4nk
```

---

## Flags

```text
user:081ecc5e6dd6ba0d150fc4bc0e62ec50

root:d5806296126a30ceebeaa172ff9c9151
```
