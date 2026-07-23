+++
title = "Observer - Walkthrough"
date = "2026-07-23"
author = "nvru"
description = "Walkthrough for the Observer machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "http",
  "golang",
  "file-observation",
  "ssh",
  "privilege-escalation"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.36`

The objective was to enumerate the target, identify exposed services, gain initial access through the vulnerable HTTP file observer service, and escalate privileges through exposed credentials and sudo permissions.

![Machine information](/images/hmv/observer/vm.png)

---

## Port Scanning

### RustScan

```bash
rustscan -a 10.10.10.36 -- -sC -sV -A -oA nmap/rust
```

### Results

```text
PORT     STATE SERVICE REASON  VERSION
22/tcp   open  ssh     syn-ack OpenSSH 9.2p1 Debian 2 (protocol 2.0)

3333/tcp open  http    syn-ack Golang net/http server
```

The scan revealed two exposed services:

- SSH running on port `22`
- A custom Golang HTTP server running on port `3333`

The web service looked unusual, so further enumeration was performed.

---

## Enumeration

### HTTP Service

Accessing the service on port `3333` returned:

![HTTP page](/images/hmv/observer/http.png)

```text
OBSERVING FILE: /home/ NOT EXIST

<!-- OxKmRNwKDOUnXzmDikdJPukQLaxAHzHMV -->
```

The response suggested that the application was attempting to access files inside the filesystem.

The server appeared to be observing arbitrary file paths, so I attempted to discover readable files.

---

This section is already structured well. I would only make a few wording fixes to improve flow and avoid repeating the SSH discovery twice:

### File Enumeration

Since the application appeared to check file existence, I attempted to brute force common sensitive files such as:

```text
.bashrc
.ssh/id_rsa
.bash_history
```

Using a users wordlist from SecLists, I discovered a valid user:

```text
jan
```

![Found user jan](/images/hmv/observer/found_jan.png)

Further enumeration revealed that the user's SSH private key was accessible:

```text
/home/jan/.ssh/id_rsa
```

![SSH key discovery](/images/hmv/observer/jan_key.png)

After recovering the private key, I used it to authenticate as `jan` through SSH.

---

## Initial Access

Using the recovered SSH key:

```bash
ssh -i id_rsa jan@10.10.10.36
```

SSH access was successful.

![SSH login](/images/hmv/observer/ssh_jan.png)

---

## Privilege Escalation

### jan → root

The first step after gaining access was checking sudo permissions.

```bash
sudo -l
```

![sudo permissions](/images/hmv/observer/sudo_jan.png)

The user was allowed to execute:

```text
/usr/bin/systemctl -l status
```

as any user without a password.

Although `systemctl status` can sometimes be abused for privilege escalation through its pager functionality, I was unable to leverage it in this environment. Further enumeration was required to find another privilege escalation path.

---

### Manual Enumeration

I continued with manual enumeration.

Inside `/opt`, I found a file named:

```text
/opt/observer
```

![Observer binary](/images/hmv/observer/opt_observer.png)

The file did not reveal anything immediately useful.

---

### Abusing the File Observer

Because the HTTP service appeared to read files relative to paths, I tested whether symbolic links could expose protected directories.

Inside the home directory:

```bash
ln -sf /root root
```

This created a symbolic link pointing to `/root`.

![Symlink root](/images/hmv/observer/symlink_root.png)

After creating the link, the file observer could access files inside the root directory.

I attempted to retrieve:

```text
/root/.ssh/id_rsa
```

but the SSH key was not accessible.

However, the root flag was readable.

![Root flag](/images/hmv/observer/curl_flag.png)

---

### Recovering Root Credentials

I continued checking user files and inspected bash history.

```bash
cat ~/.bash_history
```

The history contained a suspicious string:

```text
fuck1ng0bs3rv3rs
```


![Root history](/images/hmv/observer/root_hist.png)

This appeared to be a password.

Using it with the root account:

```bash
su root
```

worked successfully.

![Root login](/images/hmv/observer/su_root.png)

---

## Credentials

```text
root:fuck1ng0bs3rv3rs
```

---

## Flags

```text
user:HMVdDepYxsi8VSucdruB3P7

root:HMVb6MPDxdYLLC3sxNLIOH1
```
