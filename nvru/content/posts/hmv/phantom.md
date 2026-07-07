---
title: "Phantom - Walkthrough"
date: "2026-07-03"
author: "nvru"
description: "Walkthrough for Phantom machine."
tags: ["ctf", "walkthrough", "linux", "web", "privilege-escalation"]
draft: false
---

![VM](/images/hmv/phantom/vm.png)

---

## Target Information

- **Attacker IP:** `10.162.21.142`
- **Target Machine:** `10.162.21.86`

---

## Enumeration

### Port Scanning

```bash
rustscan -a 10.162.21.86 -- -A -oN nmap/rust
```

### Nmap Results

```text
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
80/tcp open  http    syn-ack ttl 64 nginx 1.14.2
```

### SSH

- OpenSSH `8.4p1`
- Debian `5+deb11u3`

### HTTP

- **Server:** `nginx/1.14.2`
- **Title:** `MazeSec | 迷踪安全`

---

## Web Enumeration

The website did not reveal much useful information during the initial assessment.

![Web](/images/hmv/phantom/port_80_nginx.png)

Several names appeared throughout the website. Since SSH was exposed, I noted them as potential usernames for later attacks.

![About Hidden](/images/hmv/phantom/about_hidden.png)

The **About** page also contained another image that looked worth investigating.

---

## Steganography

Since the web application offered very little attack surface, I decided to inspect the available images for hidden content.

![Cat Image](/images/hmv/phantom/stego_cat.png)

The first extraction revealed another embedded file.

![Stego Stage 1](/images/hmv/phantom/stego_part1.png)

Extracting the second layer exposed yet another archive.

![Stego Stage 2](/images/hmv/phantom/stego_part2.png)

Finally, the last extraction produced a QR code along with another hidden file.

The QR code did not lead to any additional useful information.

Further analysis revealed another password-protected object hidden inside `666.jpg`.

Using one of the names discovered on the website as the password (**11104567**) successfully decrypted the archive and revealed:

![666 File](/images/hmv/phantom/stego_part3.png)

This value later proved to be an important clue during the exploitation phase.

---

## Host Discovery

The extracted information suggested the existence of a virtual host.

After adding the hostname to `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

![Hosts File](/images/hmv/phantom/etc_hosts.png)

The hostname resolved successfully, exposing a different web application.

---

## Login Page

Browsing to the newly discovered virtual host presented a login portal.

![Login Page](/images/hmv/phantom/login_page.png)

Since no credentials were available, the next step was attempting account registration.

---

## Registration

Attempting to register required entering a PIN before an account could be created.

![Registration](/images/hmv/phantom/register.png)

---

## Directory Brute Force

While investigating the backend, I also discovered an internal application token.

![Database Token](/images/hmv/phantom/token_db.png)

After entering the correct PIN, registration completed successfully and a normal user account was created.

![PIN](/images/hmv/phantom/token_sqlit3.png)

---

## User Access

After logging in, the user dashboard became accessible.

![User Dashboard](/images/hmv/phantom/user_access_web.png)

At first glance, nothing appeared particularly useful, so I began inspecting the application's requests using Burp Suite.

---

## JWT / Token Manipulation

After exploring the application, no obvious vulnerabilities were found. Since authentication relied on a JWT, I inspected the token using **jwt.io**.

The payload contained user-related information that appeared suitable for privilege escalation.

![JWT](/images/hmv/phantom/jwt_io.png)

I modified the token to impersonate an administrator. Before forwarding the request, the application required an additional password to validate the modified JWT.

After brute-forcing the password, I recovered:

```text
6jlRvvwpO4
```

![Brute Force](/images/hmv/phantom/brute_force_jwt.png)

With the correct password supplied, I replayed the request through Burp Suite.

---

## Administrator Access

Although the modified JWT was accepted, access to the administrator panel was still denied.

![Access Denied](/images/hmv/phantom/access_denied_web.png)

Further testing revealed that administrator functionality was restricted to requests originating from `localhost` (`127.0.0.1`). By intercepting the request in Burp Suite and modifying the required headers, I was able to bypass this restriction and gain access to the administrator panel.

![Burp Forward](/images/hmv/phantom/burp_forward.png)

![Administrator Access](/images/hmv/phantom/admin_access.png)

One of the administrator features exposed an **arbitrary file read** vulnerability through the `file_path` parameter. Since it accepted absolute file paths without validation, it was possible to read arbitrary files from the target system.

Using this functionality, I retrieved `/etc/passwd`.

![Reading /etc/passwd](/images/hmv/phantom/burp_passwd.png)

Inspecting `/etc/passwd` revealed the local user `12138` along with a partially masked password:

```text
To***21**
```

Earlier during reconnaissance, I had noted several names mentioned throughout the website as potential usernames. Combining one of those names, **Todd**, with the masked password pattern and the username identified in `/etc/passwd` resulted in the credential `Todd12138`.

These credentials were then used to authenticate successfully over SSH as `12138`.

![SSH Access](/images/hmv/phantom/ssh_as_12138.png)

---

## Privilege Escalation

The target kernel was significantly outdated, making it a good candidate for publicly available local privilege escalation exploits.

### Copy-Fail

- https://copy.fail/

The first exploit attempted was Copy-Fail.

![Copy-Fail](/images/hmv/phantom/copy_fail.png)

Unfortunately, the original exploit failed.

### Dirty Frag

- https://github.com/v4bel/dirtyfrag

Next, I attempted the Dirty Frag exploit.

The public exploit also failed against this target.

### Copy-Fail (C Implementation)

- https://github.com/tgies/copy-fail-c.git

Finally, I used a C implementation of the Copy-Fail exploit.

![Copy-Fail C](/images/hmv/phantom/copy_fail_c.png)

This version successfully exploited the vulnerable kernel and provided a root shell.

---

## Flags

### User Flag

The user flag was located inside the user's home directory.

![User Flag](/images/hmv/phantom/find_user_flag.png)

```text
flag{user-e962b914ff98c867373b60d34943b8f7}
```

### Root Flag

```text
flag{root-2bd1dc759be8c8940e35569092e7dac2}
```
