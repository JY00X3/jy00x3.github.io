---

title: "Race Condition in Organization Membership — Legacy API Abuse and Sensitive Data Exposure"

date: 2026-08-04

categories: [Bug Bounty, Web, API Security, Business Logic]

tags: [race-condition, turbo-intruder, api, bola, idor, access-control, business-logic, legacy-api]

image:
    path: /assets/PII/cover.jpg

---
![Desktop View](/assets/PII/cover.jpg){: width="700" height="400" .normal }

## 1. Introduction

While testing the organization's user-management functionality, I identified a business-logic restriction limiting organizations to **five members**. Each additional member required an additional payment of **$50**.

This made the membership functionality an interesting target for testing, particularly for race conditions and business-logic bypasses.

The initial objective was to understand how the application enforced the membership limit and whether the same controls were consistently applied across different API versions.

The investigation eventually revealed a chain involving:

* Legacy API functionality
* Inconsistent authorization controls
* Race-condition-based membership-limit bypass
* Mass account addition
* Sensitive user information disclosure

The complete attack chain ultimately allowed more than **200 users** to be added to an organization that was intended to be limited to five members.

---

## 2. Mapping the Modern API Workflow

I began by observing the normal process used by the application when connecting a user to an organization.

The workflow involved several API requests under `/api/v2`.

At a high level, the process was:

```text
Save User Data
      |
      v
Assign UUID
      |
      v
Authorization Check
      |
      v
Connect User to Organization
```

### 2.1 Saving User Data and Assigning a UUID

The first stage saved the user's information and generated an identifier that was subsequently used by the membership workflow.

```http
POST /api/v2/...
```

The assigned UUID became an important object identifier for subsequent authorization testing.

### 2.2 Authorization Check

The application then performed an authorization check to determine whether the user could be added to the organization.

```http
POST /api/v2/...
```

This indicated that the modern implementation was explicitly checking whether the requested membership operation was permitted.

### 2.3 Connecting the User

The final stage connected the user to the organization:

```http
POST /api/v2/connect
```

At this point, the application appeared to enforce the organization's membership restriction.

Rather than immediately attempting to bypass the restriction, I continued investigating how the functionality was implemented and whether older versions of the API exposed alternative functionality.

---

## 3. Historical API Documentation

Because the application was using versioned APIs, I searched historical documentation using the Wayback Machine.

This revealed an older API implementation containing:

```http
POST /api/v1/add-user
```

Unlike the newer workflow, the legacy endpoint accepted a user UUID directly.

This was significant because it provided an alternative path for performing the same general membership operation without necessarily going through the complete `/api/v2` workflow.

The next step was therefore to understand how the UUID was generated and whether arbitrary user accounts could be targeted.

---

## 4. UUID Analysis

During testing, I observed that the UUID generation process appeared to incorporate information related to the user's registration, including:

```text
Signup Time
+
First Name
+
6 Random Characters
```

I attempted to construct the UUID of a specific target user using the observed pattern.

The attempt was unsuccessful.

This indicated that simply understanding the apparent UUID structure was insufficient to reliably identify a specific account.

I therefore moved away from targeted UUID prediction and focused on the behavior of the legacy endpoint itself.

---

## 5. Testing the Legacy Add-User Endpoint

I tested randomly generated UUID values against:

```http
POST /api/v1/add-user
```

The legacy functionality allowed valid account identifiers to be processed, but the organization's five-member restriction was still enforced.

At this stage, the situation was:

```text
Maximum Members: 5

Sequential Requests:
5 Members → Additional Requests Rejected
```

The endpoint itself was therefore not sufficient to bypass the membership restriction through sequential requests.

This led to the next question:

> Is the five-member limit enforced atomically, or is it implemented as a check followed by a separate database operation?

---

## 6. Identifying the Race Condition

The membership operation appeared to follow a logical sequence similar to:

```text
1. Check current member count
2. Verify that the limit has not been reached
3. Add the new member
4. Update the organization membership state
```

This type of implementation can become vulnerable if multiple requests perform the validation step before any of them commits the resulting state change.

Conceptually:

```text
Request A ─┐
Request B ─┤
Request C ─┤
Request D ─┤──> Check member count
Request E ─┘
                 |
                 v
          Multiple requests
          observe valid state
                 |
                 v
          Multiple additions
```

The security property that needed to be tested was therefore the **atomicity of the membership limit**.

---

## 7. Exploiting the Race Condition

I used Burp Suite Turbo Intruder to send concurrent requests against the legacy membership endpoint.

The purpose was to determine whether simultaneous requests could bypass the membership invariant.

The organization was initially restricted to:

```text
5 members
```

Under concurrent execution, however, the application processed a large number of membership operations before the membership state could be consistently enforced.

The resulting state exceeded:

```text
200 members
```

despite the intended five-member limit.

The important observation was that the issue was not caused by modifying the membership-limit parameter.

Instead, the vulnerability resulted from **concurrent execution of otherwise valid requests**.

This is characteristic of a race condition / time-of-check-to-time-of-use (TOCTOU) vulnerability.

---

## 8. Business Logic Impact

The race condition had a direct business impact because the organization's membership model required payment for additional members.

The intended model was:

```text
5 Members
    |
    v
Additional Member
    |
    v
$50 Additional Cost
```

The race condition allowed the membership constraint to be bypassed and more than 200 accounts to be added.

Therefore, the vulnerability affected both:

* The application's technical access-control model
* The organization's subscription and billing logic

The issue demonstrated that the membership quota was enforced at the application level but was not sufficiently protected against concurrent requests.

---

## 9. Legacy Endpoint Discovery After Membership Bypass

After establishing the membership-limit bypass, I continued reviewing the legacy API functionality.

I identified three additional endpoints associated with organization members.

These endpoints exposed different categories of user information.

The three categories were:

```text
1. First Name / Last Name / Contact Information

2. 2FA Status / 2FA Method

3. Connected Organizations
```

This significantly expanded the impact of the initial race condition.

---

## 10. User Profile and Contact Information

The first legacy endpoint exposed basic identity information associated with a member.

The returned data included:

```text
First Name
Last Name
Contact Information
```

The concern was that the endpoint could be queried for accounts made accessible through the vulnerable membership workflow.

This transformed the issue from a simple quota bypass into a potential unauthorized information-disclosure vulnerability.

---

## 11. Two-Factor Authentication Information

The second endpoint exposed security-related account metadata.

The response included:

```text
2FA Status
2FA Method
```

For example:

```text
2FA Status: Enabled
2FA Method: Authenticator App
```

Although this information did not expose the authentication secret itself, it disclosed the security configuration of the account.

This represents sensitive security metadata that should only be accessible to appropriately authorized users.

---

## 12. Organization Membership Information

The third endpoint exposed information about organizations associated with the account.

Conceptually, the response could identify relationships such as:

```text
User
 |
 +-- Organization A
 |
 +-- Organization B
 |
 +-- Organization C
```

This disclosed additional information about the user's organizational relationships.

Combined with the profile and security endpoints, this created a broader user-information disclosure issue.

---

## 13. Complete Attack Chain

The complete attack chain can be summarized as follows:

```text
Organization limited to 5 members
                |
                v
Additional members require $50
                |
                v
Map /api/v2 membership workflow
                |
                +--> Save user data
                |
                +--> Assign UUID
                |
                +--> Authorization check
                |
                +--> /api/v2/connect
                |
                v
Review historical API documentation
using the Wayback Machine
                |
                v
Discover /api/v1/add-user
                |
                v
Analyze UUID generation
                |
                v
Attempt targeted UUID generation
                |
                v
Targeted UUID attempt fails
                |
                v
Test random UUIDs
                |
                v
Five-member limit remains enforced
                |
                v
Race-condition testing
                |
                v
Turbo Intruder
                |
                v
Concurrent requests
                |
                v
More than 200 users added
                |
                v
Discover legacy member endpoints
                |
                +--> Name / Contact Information
                |
                +--> 2FA Status / 2FA Method
                |
                +--> Connected Organizations
```

---

## 14. Impact

The complete chain demonstrated the ability to:

* Bypass the intended five-member organization limit.
* Add more than 200 users through concurrent requests.
* Circumvent the business logic requiring payment for additional members.
* Abuse legacy API functionality.
* Access member profile and contact information.
* Retrieve 2FA status and authentication method.
* Identify organizations associated with users.

The finding therefore combined a **race condition**, **business-logic bypass**, and **unauthorized information disclosure**.

The most significant impact was the ability to violate an organization-level security and billing invariant at scale.

---

## 15. Root Cause

The primary technical issue was that the organization membership limit was not enforced as an atomic operation.

The vulnerable logic can be represented as:

```text
Read Member Count
       |
       v
Check Limit
       |
       v
Create Membership
       |
       v
Update State
```

When multiple requests were processed concurrently, several requests could pass the membership check before the underlying state was updated.

As a result, the application could transition from:

```text
5 Members
```

to:

```text
200+ Members
```

without correctly enforcing the business constraint.

The presence of the legacy `/api/v1/add-user` endpoint also provided an alternative path to the membership functionality.

---

## 16. Remediation

### 16.1 Enforce the Membership Limit Atomically

The membership limit should be enforced using an atomic database transaction or equivalent concurrency-control mechanism.

The operation should conceptually be:

```text
Validate Limit
      +
Reserve Membership Slot
      +
Create Membership
```

These operations must be protected from concurrent execution.

### 16.2 Use Database-Level Constraints

Where appropriate, enforce membership limits using database-level controls rather than relying exclusively on application logic.

Application-level checks alone are insufficient when multiple requests can execute simultaneously.

### 16.3 Decommission Legacy APIs

Unused `/api/v1` endpoints should be removed.

If legacy functionality must remain available, it should use the same authorization, validation, rate-limiting, and business-logic controls as the current API.

### 16.4 Enforce Authorization on Every Endpoint

Each member-data endpoint should independently verify that the authenticated user is authorized to access the requested account.

Authorization should not be implicitly granted because an account was previously added or referenced through another endpoint.

### 16.5 Protect Sensitive Security Metadata

Information such as:

```text
2FA Status
2FA Method
Organization Memberships
```

should only be returned to appropriately authorized users.

### 16.6 Implement Abuse Detection

The application should detect abnormal membership-creation patterns, particularly large numbers of requests originating from the same authenticated session within a short period.

---

## 17. Key Bug Bounty Lessons

### 17.1 Understand the Business Rule

The five-member limit and $50 additional-member cost immediately identified the functionality as a valuable business-logic target.

Whenever an application has quotas, limits, credits, payments, or resource allocations, test whether those constraints remain valid under concurrent execution.

### 17.2 Map the Entire Workflow

Instead of testing only the final `/api/v2/connect` request, I mapped the entire process:

```text
Save
 ↓
UUID Assignment
 ↓
Authorization
 ↓
Connect
```

Understanding the complete workflow made it easier to identify alternative implementations.

### 17.3 Investigate Historical APIs

The current API was not the only source of functionality.

Historical documentation revealed:

```text
/api/v1/add-user
```

This demonstrates why legacy endpoints and archived documentation can be valuable during API security assessments.

### 17.4 Test Invariants Under Concurrency

A useful question when testing business logic is:

> What must always remain true?

In this case:

```text
Organization Members <= Allowed Limit
```

The race-condition test demonstrated that this invariant could be violated.

### 17.5 Continue After the Initial Finding

After proving the membership-limit bypass, I reviewed related legacy functionality and identified additional endpoints exposing user information.

This expanded the finding from a single business-logic issue into a broader attack chain.

---

## 18. Conclusion

The assessment began with a straightforward business rule: organizations were limited to five members, with additional members requiring an additional $50 payment.

By mapping the modern `/api/v2` workflow, reviewing historical documentation, identifying the legacy `/api/v1/add-user` endpoint, analyzing UUID behavior, and testing the membership operation under concurrent execution, I identified a race condition that allowed more than 200 users to be added despite the intended membership restriction.

Further analysis of the legacy API revealed additional endpoints exposing:

```text
First Name / Last Name / Contact Information
            |
            v
2FA Status / 2FA Method
            |
            v
Connected Organizations
```

The final attack chain was:

```text
Business Logic Restriction
          |
          v
Legacy API Discovery
          |
          v
Race Condition
          |
          v
Membership Limit Bypass
          |
          v
200+ Unauthorized Additions
          |
          v
Legacy Member Endpoints
          |
          v
Sensitive Information Disclosure
```

The primary lesson is that business-logic controls must be enforced atomically and consistently across every API version.

A membership limit that works correctly for sequential requests is not sufficient if concurrent requests can violate the underlying invariant.

---
