+++
title = "Hunter - Walkthrough"
author = "nvru"
date = "2026-07-25"
description = "Walkthrough for the Hunter machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "flask",
  "username-enumeration",
  "ffuf",
  "werkzeug"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.200.74.40`
- **Target Machine IP:** `10.1.36.72`

The objective was to enumerate the exposed services, identify a valid username through the password reset functionality, and use the discovered account for the next stage of the attack.

---

## Port Scanning

### RustScan

The first step was identifying the services exposed by the target.

```bash
rustscan -a 10.1.36.72 -- -sC -sV -A -oN nmap/rust
```

### Results

```text
22/tcp open  ssh   OpenSSH 8.9p1 Ubuntu
80/tcp open  http  Werkzeug 3.1.4 (Python 3.10.12)
```

Only two services were exposed:

- **SSH** on port **22**
- **HTTP** on port **80**, served by a Python **Werkzeug** application.

The web application appeared to be the primary attack surface, so I started enumerating it.

---

## Enumeration

### HTTP Enumeration

Browsing to the website presented a simple Flask login portal. Running **Wappalyzer** identified the application as a Python **Flask** application served by **Werkzeug**.

![Wappalyzer Information](/images/hsm/hunter/wappla.png)

Navigating to the homepage displayed a basic login form with no additional functionality or publicly accessible information.

![Homepage](/images/hsm/hunter/http.png)

At first glance, there were no obvious clues exposed by the application, so I continued by inspecting the page source and enumerating the available endpoints.

---

### Reviewing the Source

Inspecting the page source revealed the names of the login parameters that would be useful later when fuzzing the application.

![Login Source](/images/hsm/hunter/src_login.png)

Submitting random credentials always produced the same response without any useful error messages or status code differences, making username enumeration through the login page ineffective.

---

### Directory Enumeration

I performed directory enumeration to discover additional endpoints.

```bash
feroxbuster \
-u http://10.1.36.72/ \
-w /usr/share/seclists/Discovery/Web-Content/common.txt
```

The scan identified only a few accessible pages:

- Login page
- Password reset page

![Directory Enumeration](/images/hsm/hunter/dir_brute.png)

Nothing else of interest was exposed.

---

### Password Reset Enumeration

The password reset page accepted only a username parameter.

![Reset Page](/images/hsm/hunter/reset_param.png)

Since the endpoint only required a username, it was a good candidate for username enumeration.

I fuzzed the parameter using a wordlist of usernames.

```bash
ffuf \
-u http://10.1.36.72/reset \
-w creds/users \
-d "username=FUZZ" \
-H "Content-Type: application/x-www-form-urlencoded"
```

The response behavior did not return a clear success or failure message. Instead, the difference was visible through the response timing.

Most usernames returned a delay between **200-300 ms**, while the username `joey` caused a much longer response time of around **1000 ms**.

This timing difference indicated that `joey` was a valid username.

```text
joey
```

![Valid Username](/images/hsm/hunter/joey.png)

This confirmed that the password reset endpoint leaked whether a supplied username existed, making it vulnerable to username enumeration through response-time analysis.

---

## Discovered Credentials

```text
Valid Username
--------------
joey
```
