+++
title = "Sysadmins - Walkthrough"
date = "2026-08-16"
author = "nvru"
description = "Walkthrough for the Sysadmins machine."
tags = [
    "ctf",
    "walkthrough",
    "linux",
    "ftp",
    "snmp",
    "nginx",
    "information-disclosure",
    "credential-reuse",
    "cve-2025-32463",
    "privilege-escalation"
]
draft = false
+++

## Overview

- **Platform:** HackSmarter
- **Difficulty:** `Medium`
- **OS:** `Linux`
- **Attacker IP:** `10.200.82.76`
- **Target IP:** `10.1.142.188`

The objective was to enumerate the target, obtain initial access, and escalate privileges to root.

---

## Enumeration

### Port Scanning

#### RustScan TCP

```bash
rustscan -a 10.1.142.188 -- -sC -sV -A -oA nmap/rust
```

#### Results

```text
21/tcp   open  ftp     vsftpd 3.0.5
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp   open  http    nginx 1.24.0
```

#### Nmap UDP

```bash
nmap --top-ports 100 -sU 10.1.142.188
```

##### Result

```text
161/udp  open  snmp    net-snmp SNMPv3
```

The scan revealed the following services:

| Port    | Service | Version         |
| ------- | ------- | --------------- |
| 21      | FTP     | vsftpd 3.0.5    |
| 22      | SSH     | OpenSSH 9.6p1   |
| 80      | HTTP    | nginx 1.24.0    |
| 161/udp | SNMP    | net-snmp SNMPv3 |

I started by enumerating the anonymous FTP service.

---

### FTP Enumeration

Anonymous login was enabled and contained a single file.

```bash
wget ftp://10.1.142.188/data_breach_notification.txt
```

The downloaded file was a notification informing users that a recent breach had leaked passwords and referenced a Pastebin page containing the compromised credentials.

```text
https://pastebin.com/mqPMU1cF
```

I downloaded the leaked password list.

```bash
wget https://pastebin.com/raw/mqPMU1cF -O compromised_password
```

The file only contained passwords, so I continued gathering usernames before attempting any authentication attacks.

---

### Web Enumeration

Browsing the web application showed a company website for system administration services.

![Homepage](/images/hsm/sysadmins/http.png)

While browsing the site I identified the internal domain name.

![Domain](/images/hsm/sysadmins/domain.png)

```text
sysadmins.hsm
```

The contact page did not expose any obvious vulnerabilities.

![Contact Page](/images/hsm/sysadmins/contact_page.png)

Continuing enumeration revealed the company team members.

```text
waserby
helena
peter
```

![Team Page](/images/hsm/sysadmins/team_page.png)

With a list of usernames and leaked passwords, SNMP became the most interesting target.

---

### SNMP Enumeration

The service was running **SNMPv3**, which limited the available brute-force tools.

I initially tested `snmpwn`, which successfully identified a valid username.

```text
waserby
```

![Valid User](/images/hsm/sysadmins/found_user.png)

Since the tool was unreliable, I switched to **legba** to perform a password-only attack.

```bash
./legba snmp3 \
    --target 10.1.142.188 \
    --username waserby \
    --password compromised_password
```

The attack successfully recovered the SNMP password.

```text
waserby:butterfly
```

![Bruteforce](/images/hsm/sysadmins/brute_snmp.png)

Using the recovered credentials with `snmpwalk` exposed additional information.

```bash
snmpwalk -v3 -u waserby -A butterfly -l authNoPriv 10.1.142.188
```

![SNMPWalk](/images/hsm/sysadmins/snmpwalk_cmd.png)

The SNMP data disclosed SSH credentials for **helena**.

![SSH Credentials](/images/hsm/sysadmins/helena_creds.png)

---

## Initial Access

Using the recovered credentials:

```bash
ssh helena@sysadmins.hsm
```

Password:

```text
PerfectIsTheEnemyOfDone223!
```

![SSH Access](/images/hsm/sysadmins/ssh_helena.png)

Initial access was successfully obtained as **helena**.

---

## Post-Exploitation

After obtaining a shell I performed basic local enumeration.

### Sudo Permissions

```bash
sudo -l
```

![Sudo](/images/hsm/sysadmins/sudo_helena.png)

No immediately useful sudo permissions were available.

Only one normal user existed on the system.

![Users](/images/hsm/sysadmins/only_helena.png)

No interesting SUID binaries were present.

```bash
find / -perm -4000 -type f 2>/dev/null
```

![SUID](/images/hsm/sysadmins/no_suid.png)

No exploitable Linux capabilities were found.

```bash
getcap -r / 2>/dev/null
```

![Capabilities](/images/hsm/sysadmins/no_cap.png)

Since `curl` was installed, I transferred **LinPEAS** to the target for further enumeration.

![curl](/images/hsm/sysadmins/curl_exist.png)

LinPEAS reported the installed version of **sudo** as vulnerable to **CVE-2025-32463**.

---

## Privilege Escalation

Searching for a proof of concept led me to the following repository:

- https://github.com/pr0v3rbs/CVE-2025-32463_chwoot

![PoC](/images/hsm/sysadmins/google_cve.png)

Running the exploit successfully resulted in a root shell.

![Root](/images/hsm/sysadmins/root.png)

The machine was successfully compromised.

---

## Credentials

```text
SNMP
waserby:butterfly

SSH
helena:PerfectIsTheEnemyOfDone223!
```

---

## Flags

### User

```text
f47e15d6b346cb443b77f71fb2b2d946
```

### Root

```text
2c0ae869620220c3ee66afb9c91e9ed7
```

---

## Attack Path

```text
Port Scan
    ↓
Anonymous FTP
    ↓
Information Disclosure (Password Leak)
    ↓
Web Enumeration
    ↓
Username Enumeration
    ↓
SNMPv3 Password Bruteforce
    ↓
Sensitive Data Exposure (SSH Credentials)
    ↓
SSH Access
    ↓
Local Enumeration
    ↓
Vulnerable sudo (CVE-2025-32463)
    ↓
Privilege Escalation
    ↓
Root
```

---

## References

- [PEASS-ng – LinPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS)
- [CVE-2025-32463 Proof of Concept (chwoot)](https://github.com/pr0v3rbs/CVE-2025-32463_chwoot)
- [Legba](https://github.com/evilsocket/legba)
