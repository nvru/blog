+++
title = "The Suidy - Walkthrough"
date = "2026-07-04"
author = "nvru"
description = "Walkthrough for The Suidy machine."
tags = ["ctf", "walkthrough", "linux", "web", "privilege-escalation"]
draft = false
+++

## Overview

- **Attacker IP:** 10.10.10.2
- **Target Machine IP:** 10.10.10.8

The objective was to enumerate the target, identify exposed services, and escalate privileges.

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.8 -- -oA nmap/rust
```

### Open Ports

```
Open 10.10.10.8:22
Open 10.10.10.8:80
```

---

### Nmap Scan

```bash
nmap -sSCV -p22,80 10.10.10.8 -A -oA nmap/22_80
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2
80/tcp open  http    nginx 1.14.2
```

---

## Web Enumeration (Port 80)

The web server returned a very minimal page:

```text
hi
```



![Port 80 home page](/images/hmv/suidy/hi_port_80.png)

No useful information was found in the page source.

![source code](/images/hmv/suidy/src_port_80.png)

### Robots.txt Discovery

During enumeration, `/robots.txt` was found.

Contents:

```
/hi
/....\..\.-\--.\.-\..\-.
/shehatesme
```

when trying to decode this dots it appear to be valid morse code after deleting the `\` but nothing usefull just `HIAGAIN`

```
HIAGAIN
```

After visiting `/shehatesme`, the page returned:

![shehatesme endpoint](/images/hmv/suidy/shehatesme.png)

found creds as theuser - thepass

---

## Privilege Escalation

### Initial Access Context

No sudo privileges were available for the current user.

![sudo check](/images/hmv/suidy/sudo_l.png)

---

### SUID Enumeration

A suspicious SUID binary was discovered and executed, which switched execution context to user `suidy`.

![SUID binary found](/images/hmv/suidy/suid_binary.png)

Further analysis showed no meaningful output from `strings`.

The binary was copied locally for deeper inspection.

![copy binary](/images/hmv/suidy/copy_host.png)

---

### Exploitation

When executed on the target machine, the binary switches the current user to `suidy`.

![switch to suidy user](/images/hmv/suidy/switch_suidy_user.png)

Static analysis showed that the program modifies the process UID and GID, setting both values to `1001`. While this appears to drop privileges to a low-privileged user, the binary’s execution flow still allows insecure privilege handling that can be abused to escalate access.

This behavior was confirmed through reverse engineering in Ghidra.

![analysis in Ghidra](/images/hmv/suidy/ghidra.png)

Final step:

- A small C program was written to replicate the same logic but force UID/GID to `0` (root)
- The program was compiled on the target system using `gcc` (which was available on the machine)
- Executing the binary spawned a root shell via `/bin/bash`

![exploit to root](/images/hmv/suidy/exploit_to_root.png)

---

## Credential Discovery

```
theuser:thepass
```

---

## Flags

```text
user flag : HMV2353IVI
root flag : HMV0000EVE
```
