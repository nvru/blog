+++
title = "Helix - Walkthrough"
date = "2026-07-05"
author = "nvru"
description = "Walkthrough for the Helix machine."
tags = ["ctf", "walkthrough", "linux", "snmp", "ssh", "privilege-escalation", "dirtyfrag"]
draft = false
+++

## Overview

- **Attacker IP:** `10.162.21.142`
- **Target Machine IP:** `10.162.21.133`

The objective was to enumerate the target, identify exposed services, gain initial access, and escalate privileges.

![Machine information](/images/hmv/helix/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.162.21.133 -- -A
```

### Results

```text
Open 10.162.21.133:22
[~] Starting Script(s)
[>] Running script "nmap -vvv -p {{port}} -{{ipversion}} {{ip}} -A" on ip 10.162.21.133

PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)

Service Info:
OS: Linux
CPE: cpe:/o:linux:linux_kernel
```

### Notes

RustScan insisted SSH was the only TCP service. I reran it several times with the same result, so either RustScan was having a bad day, there really weren't any other TCP ports, or the machine was trolling me.

---

### Full UDP Nmap Scan

Running this Nmap command to search for all UDP ports.

```bash
nmap 10.162.21.133 -A -p- -sSCV -oN nmap/init_udp_all
```

### Results

```text
PORT    STATE         SERVICE REASON              VERSION
22/tcp  open          ssh     syn-ack ttl 64      OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)
68/udp  open|filtered dhcpc   no-response
161/udp open          snmp    udp-response ttl 64 SNMPv1 server; net-snmp SNMPv3 server (public)

snmp-netstat:
  TCP  0.0.0.0:22           0.0.0.0:0
  TCP  10.162.21.133:22     10.162.21.142:54996
  TCP  10.162.21.133:22     10.162.21.142:55012
  UDP  0.0.0.0:161          *:*
  UDP  10.0.2.15:68         *:*
  UDP  10.162.21.133:68     *:*

snmp-sysdescr:
Linux helix 6.12.74+deb13+1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64

System uptime:
9m13.72s (55372 timeticks)

Interfaces:
- lo
- enp0s3
- enp0s8

Running processes:
(systemd, cron, dbus-daemon, snmpd, sshd, dhcpcd, etc.)

snmp-info:
enterprise: net-snmp
snmpEngineBoots: 16
snmpEngineTime: 9m14s
```

The full process list returned by Nmap is omitted here since it is not particularly useful, but it confirmed that SNMP was exposing a large amount of system information.

---

## SNMP Enumeration

Using the default community string:

```bash
snmpbulkwalk -c public -v2c 10.162.21.133
```

### Output

```text
SNMPv2-MIB::sysDescr.0 = STRING: Linux helix 6.12.74+deb13+1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64
SNMPv2-MIB::sysObjectID.0 = OID: NET-SNMP-MIB::netSnmpAgentOIDs.10
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (161140) 0:26:51.40
SNMPv2-MIB::sysContact.0 = STRING: Me:lixeh22 ## maybe they creds for ssh
SNMPv2-MIB::sysName.0 = STRING: helix
SNMPv2-MIB::sysLocation.0 = STRING: Sitting on the Dock of the Bay
SNMPv2-MIB::sysServices.0 = INTEGER: 72
SNMPv2-MIB::sysORLastChange.0 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORID.1 = OID: SNMP-FRAMEWORK-MIB::snmpFrameworkMIBCompliance
SNMPv2-MIB::sysORID.2 = OID: SNMP-MPD-MIB::snmpMPDCompliance
SNMPv2-MIB::sysORID.3 = OID: SNMP-USER-BASED-SM-MIB::usmMIBCompliance
SNMPv2-MIB::sysORID.4 = OID: SNMPv2-MIB::snmpMIB
SNMPv2-MIB::sysORID.5 = OID: SNMP-VIEW-BASED-ACM-MIB::vacmBasicGroup
SNMPv2-MIB::sysORID.6 = OID: TCP-MIB::tcpMIB
SNMPv2-MIB::sysORID.7 = OID: UDP-MIB::udpMIB
SNMPv2-MIB::sysORID.8 = OID: IP-MIB::ip
SNMPv2-MIB::sysORID.9 = OID: SNMP-NOTIFICATION-MIB::snmpNotifyFullCompliance
```

The interesting finding is:

```text
Me:lixeh22
```

Maybe they creds for ssh.

![SNMP enumeration](/images/hmv/helix/snmp.png)

---

## Initial Access

### SSH Credentials

```text
Me:lixeh22
```

SSH login works.

![SSH as me user](/images/hmv/helix/ssh_as_me_user.png)

---

## Privilege Escalation

I checked the available local users. Only `root` and `me` were present in `/etc/passwd`.

![Contents of /etc/passwd](/images/hmv/helix/etc_passwd.png)

Since there were no obvious misconfigurations to abuse, I checked the kernel version. The system was running an older kernel, making kernel exploits the next thing to investigate.

![Old kernel](/images/hmv/helix/old_kernel.png)

Two exploits came to mind: **Dirty PageTable (DirtyFrag)** and **Copy-on-Write Fail (CopyFail)**. DirtyFrag looked like the better candidate for this kernel version, so I tested it first.

It worked immediately and provided a root shell.

- `https://github.com/v4bel/dirtyfrag`

![DirtyFrag exploit](/images/hmv/helix/dirty_frag.png)

---

## Credentials

```text
ssh
Username: Me
Password: lixeh22
```

---

## Flags

```text
user
410e2fb19f61cbce630520b66368cf11d04934650ffecaac322c0acd6d7511ae

root
3faf135eff748a201ff898b011c202bafb52ab89168ab90bae804d584a5e8565
```
