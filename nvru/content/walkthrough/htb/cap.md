````text
+++
title = "Lower2 - Walkthrough"
date = "2026-11-01"
author = "nvru"
description = "Walkthrough for the Vulnyx Lower2 machine."
tags = [
  "ctf",
  "walkthrough",
  "vulnyx",
  "linux",
  "nginx",
  "ssh",
  "telnet",
  "password-bruteforce",
  "shadow-group",
  "privilege-escalation",
  "easy"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.4`

The objective was to enumerate the target, identify exposed services, gain initial access through Telnet using a recovered password, and escalate privileges by abusing membership in the `shadow` group, which allowed modification of `/etc/shadow`.

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.4 -b 500 -- -sC -sV -A -oN nmap/init
````

### Results

text

```text
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 9.2p1 Debian 2+deb12u4 (protocol 2.0)
| ssh-hostkey:
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAEAAABBBIzUvGOaZF4gJoYBGR4NrMZOj32x98uVDUQ0dY0RENRdIyokD8RvJG8g9g71aoh/20m4mcEEdSyp+eE9ABu1kwk=
|   256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPrNZ9AQg+cgX4w0wabsDTAVeo9/VWThsF5efc2OzsFo
23/tcp open  telnet  syn-ack Netkit telnet-ssl telnetd
80/tcp open  http    syn-ack nginx 1.22.1
| http-methods:
|_  Supported Methods: GET HEAD
|_http-server-header: nginx/1.22.1
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Three services were exposed:

- SSH on port 22
- Telnet on port 23
- HTTP (nginx) on port 80

## Enumeration

### HTTP Enumeration

Browsing to the website on port 80 only displayed the default nginx landing page.

I tried directory brute forcing to discover hidden directories and files.

bash

```bash
feroxbuster -u http://10.10.10.4/ -w /usr/share/seclist/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt -x php,html,js,bak,py,sql,xml,phtml
```

Unfortunately, nothing useful was discovered, so I moved on to the other exposed services.

### SSH Enumeration

I tried connecting to SSH as `root`.

Interestingly, the server displayed a welcome message referencing the `b.taylor` user.

I also tried connecting as `b.taylor`, but SSH required a public key for authentication rather than accepting a password.

Since password authentication was not an option, I moved on to Telnet.

### Telnet Enumeration

Telnet was exposed on port 23, so I attempted to brute-force the service to find valid credentials.

The attempt successfully recovered a password for the `b.taylor` account.

The recovered credentials were:

text

```text
b.taylor:rockyou
```

I used these credentials to log in through Telnet and obtained an interactive shell as `b.taylor`.

## Initial Access

With the recovered credentials, I connected to the target through Telnet.

bash

```bash
telnet 10.10.10.4
```

After authenticating as `b.taylor`, I had access to a shell on the target machine.

## Privilege Escalation

### Local Enumeration

As usual, I started by checking the user's sudo permissions.

bash

```bash
sudo -l
```

There were no sudo permissions available.

I also checked for `doas`, but it was not available on the machine.

Next, I checked the groups that the current user belonged to.

bash

```bash
id
```

The output revealed something interesting: `b.taylor` was a member of the `shadow` group.

### Abusing the Shadow Group

On Linux systems, the `shadow` group is associated with `/etc/shadow`, which stores password hashes for local user accounts.

Since `b.taylor` belonged to this group, I checked the permissions on `/etc/shadow`.

bash

```bash
ls -l /etc/shadow
```

The file was owned by the `shadow` group, meaning the current user had the ability to modify it.

To confirm the finding, I searched for files writable by the `shadow` group.

bash

```bash
find / -writable -group shadow 2>/dev/null | grep -Ev "proc|home|sys|dev|tmp|run"
```

The search confirmed that `/etc/shadow` was writable by the current user.

### Changing the Root Password

Since I could modify `/etc/shadow`, I could replace the root password hash with one that I controlled.

I generated a new password hash using `mkpasswd`.

Alternatively, OpenSSL can be used:

bash

```bash
openssl passwd -6
```

I then replaced the existing root password hash in `/etc/shadow` with the newly generated hash.

After updating the file, I switched to the root account using `su`.

bash

```bash
su root
```

I entered the password associated with the new hash and successfully obtained a root shell.

## Credentials

text

```text
Telnet
    b.taylor:rockyou
```

## Privilege Escalation Summary

The privilege escalation path was straightforward:

1. Gain access as `b.taylor` through Telnet.
2. Enumerate the user's group memberships.
3. Discover that `b.taylor` belongs to the `shadow` group.
4. Identify that `/etc/shadow` is writable by the group.
5. Replace the root password hash.
6. Switch to root using `su`.

The misconfigured group permissions allowed a regular user to modify the system's password database and ultimately obtain root access.

## Flags

text

```text
user:

root:
```
