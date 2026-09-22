# Hacker101 — Micro-CMS v2

![Hacker101](https://img.shields.io/badge/Platform-Hacker101-red)
![Category](https://img.shields.io/badge/Category-Web%20Security-blue)
![Status](https://img.shields.io/badge/Status-In%20Progress-orange)

## 📌 Challenge Overview

**Platform:** Hacker101 CTF  
**Challenge:** Micro-CMS v2  
**Category:** Web Security  
**Focus:** SQL Injection, Authentication, Authorization, HTTP Methods  
**Tools:** Browser, Burp Suite, cURL, Kali Linux

Micro-CMS v2 is the second version of the Micro-CMS challenge.

After completing Micro-CMS v1, I expected this version to have better security because the challenge specifically says that the security issues from the previous version were fixed.

The changelog says:

> "By default, users need to be an admin to add or edit pages now."

That immediately made authentication and authorization the main things I wanted to investigate.

> ⚠️ **Note:** Actual flag values are intentionally redacted from this public write-up.

---

## 🔎 Reconnaissance

When I opened the challenge, the homepage contained a few interesting pages:

- `/page/1` — Micro-CMS Changelog
- `/page/2` — Markdown Test
- `/page/create` — Create a new page

The changelog was the first thing I checked because it gave me an idea of what had changed from v1.

The important points were:

- Authentication had been added.
- Users needed to be administrators to add or edit pages.
- The previous security flaws were supposedly fixed.

Instead of immediately trying random payloads, I decided to focus on the new authentication and authorization functionality.

---

# 🚩 Flag 0 — Authentication SQL Injection

## Initial Observation

When I tried to edit a page, I was redirected to:

```text
/login
```

So the application was now protecting the editing functionality with a login page.

I decided to test whether the login form was vulnerable to SQL injection.

I entered the following into the username field:

```text
' UNION SELECT 'pass' AS password FROM admins WHERE '1' = '1
```

For the password, I entered:

```text
pass
```

The idea was to manipulate the SQL query used by the authentication system so that the application would return a password value that I already knew.

## Result

The login was successfully bypassed.

I was authenticated as an administrator and could access functionality that was supposed to be restricted.

The application revealed:

```text
FLAG 0: [REDACTED]
```

## What Happened?

The login functionality was vulnerable to SQL injection.

The application was allowing user input to influence the SQL query used during authentication.

A secure application should use parameterized queries instead of directly inserting user input into SQL statements.

For example, an application might logically perform something like:

```sql
SELECT password
FROM admins
WHERE username = '<username>';
```

The injected input allowed me to alter the intended SQL logic.

## Vulnerability

**SQL Injection — Authentication Bypass**

## Impact

An attacker could potentially:

- Bypass authentication
- Gain administrator access
- Access restricted functionality
- Extract information from the database

## Recommended Mitigation

The application should use:

- Prepared statements
- Parameterized SQL queries
- Proper database permissions
- Input validation as an additional security layer

The main fix for SQL injection should be parameterized queries rather than attempting to blacklist individual characters or payloads.

---

# 🚩 Flag 1 — Direct HTTP Request / Page Editing

After bypassing the login and gaining administrator access, I started looking at the page editing functionality.

The relevant endpoint was:

```text
/page/edit/1
```

Instead of relying only on the browser interface, I wanted to see what happened if I interacted with the endpoint directly.

This is where cURL became useful.

## Testing the Endpoint

I sent a POST request directly to the page editing endpoint:

```bash
curl -X POST https://<INSTANCE>.ctf.hacker101.com/page/edit/1
```

The server immediately returned another flag.

```text
FLAG 1: [REDACTED]
```

This was interesting because I was directly interacting with the underlying HTTP endpoint instead of following the normal browser workflow.

## What I Learned

This showed me why it is important to inspect and test HTTP requests directly.

A web application may appear to restrict functionality through its interface, but the actual security controls need to be implemented on the server.

The server should independently verify:

- Whether the user is authenticated
- Whether the user has the required privileges
- Whether the requested operation is allowed
- Whether the HTTP method is valid
- Whether the request contains the required parameters

## Vulnerability

**Improper access control / insecure endpoint handling**

## Impact

Weak access control can potentially allow an attacker to:

- Access administrative functionality
- Modify application content
- Bypass frontend restrictions
- Perform actions outside their intended privilege level

## Recommended Mitigation

Authorization checks should be performed server-side on every sensitive endpoint.

The application should never rely only on:

- Hidden buttons
- Frontend restrictions
- Redirects
- Expected browser behavior

---

# 🔐 Inspecting the Authentication Session

While investigating the authenticated application, I used Burp Suite to inspect the HTTP requests.

The authenticated request contained a session cookie similar to:

```http
Cookie: l2session=...
```

I inspected the session information and found information indicating administrator privileges.

Conceptually, the decoded session information looked like:

```json
{
  "admin": true
}
```

I am not including the actual session cookie in this write-up because session tokens should never be published.

## Why This Caught My Attention

Whenever I see authorization information associated with a client-side session, I want to understand how the application protects it.

Some of the questions I considered were:

- Is the session signed?
- Is it encrypted?
- Can the client modify it?
- Does the server verify its integrity?
- Does the server blindly trust the `admin` value?
- Can the session be forged?

This became another interesting part of the investigation.

## Security Considerations

If an application stores authorization-related information in a client-side session, the information needs to have proper integrity protection.

The server should not blindly trust a client-controlled value such as:

```json
{
  "admin": true
}
```

without verifying that the session was legitimately issued by the server and has not been modified.

Session cookies should also use appropriate security attributes such as:

```text
HttpOnly
Secure
SameSite
```

---

# 🚩 Final Flag — Administrator Credential Discovery

After finding the earlier flags, I reached the final part of the challenge.

The hint was:

> "Credentials are secret, flags are secret. Coincidence?"

This made me think that the administrator credentials themselves were probably important.

At this point, I started investigating whether the administrator credentials could be discovered through the application's SQL injection vulnerability.

One approach is to test the login endpoint with `sqlmap`.

For example:

```bash
sqlmap -u "https://<INSTANCE>.ctf.hacker101.com/login" --data="username=abc&password=xyz" -p username --dbms=mysql --dump
```

The purpose of this is to determine whether the username parameter is injectable and, if it is, investigate the database contents.

The administrator credentials are the interesting target here.

## Current Status

I have not personally completed this final credential-discovery step on the current instance.

So I am intentionally leaving the final flag as:

```text
Final Flag: 🔄 Investigation
```

I would rather leave it incomplete than copy a credential or flag from another person's write-up and present it as something I discovered myself.

---

# 🧠 My Approach

My overall approach to Micro-CMS v2 was:

```text
Reconnaissance
      ↓
Read the changelog
      ↓
Identify authentication as an attack surface
      ↓
Test the login form
      ↓
SQL Injection
      ↓
Authentication bypass
      ↓
Admin access
      ↓
Inspect page editing functionality
      ↓
Test the HTTP endpoint directly
      ↓
Inspect the authenticated session
      ↓
Investigate administrator credentials
```

The main thing I learned from this challenge was to look beyond the normal application interface.

Instead of only asking:

> "What can I click?"

I started asking:

> "What HTTP request is actually being sent, and what happens if I interact with the endpoint directly?"

---

# 🛠️ Tools Used

| Tool | Purpose |
|---|---|
| Browser | Reconnaissance and application interaction |
| Burp Suite | HTTP request/response inspection |
| cURL | Direct interaction with endpoints |
| Kali Linux | Security testing environment |
| SQL Injection payloads | Authentication testing |
| sqlmap | SQL injection/database investigation |

---

# 🔍 Vulnerabilities Identified

## 1. SQL Injection

The login functionality was vulnerable to SQL injection.

### Impact

- Authentication bypass
- Potential database access
- Potential privilege escalation

### Mitigation

- Use prepared statements
- Use parameterized queries
- Never concatenate user input directly into SQL queries
- Apply least-privilege database permissions

---

## 2. Improper Access Control

The application exposed sensitive page functionality through an HTTP endpoint that could be interacted with directly.

### Impact

- Unauthorized access to functionality
- Potential content modification
- Privilege boundary bypass

### Mitigation

- Enforce authorization server-side
- Verify privileges on every sensitive endpoint
- Do not rely on frontend restrictions
- Use centralized authorization controls

---

## 3. Session Security

The authenticated session contained information associated with administrator privileges.

### Security Considerations

- Session data should have integrity protection
- Client-controlled authorization information should not be blindly trusted
- Privileges should be validated server-side
- Session cookies should use appropriate security attributes

---

# 📚 What I Learned

## 1. Read the Application Carefully

The changelog gave me useful information about where to look.

The statement about administrators being required to edit pages immediately made authentication and authorization interesting attack surfaces.

## 2. Don't Trust the Frontend

If a button is not visible, that does not mean the underlying endpoint is secure.

Testing the actual HTTP request gave me much more information than simply interacting with the interface.

## 3. Authentication and Authorization Are Different

Authentication answers:

> **Who are you?**

Authorization answers:

> **What are you allowed to do?**

A secure application needs both.

## 4. HTTP Requests Are Worth Inspecting

Using Burp Suite helped me understand what the application was actually sending.

This made it easier to move from simply using the website to understanding how the application worked underneath.

## 5. Form a Hypothesis Before Testing

Instead of randomly trying payloads, I started using information from the application itself.

For example, the statement:

> "Users need to be an admin to add or edit pages."

gave me a clear reason to investigate:

- Login
- Administrator privileges
- Page editing
- Direct HTTP requests
- Session behavior

## 6. Document What I Actually Solved

For this write-up, I have intentionally left the final flag as an investigation.

I don't want to claim a flag that I did not personally obtain.

These write-ups are meant to document what I actually tested, what worked, and what I learned from the process.

---

# 📊 Challenge Progress

| Flag | Technique | Status |
|---|---|---|
| Flag 0 | Authentication SQL Injection | ✅ Solved |
| Flag 1 | Direct HTTP Request / Access Control | ✅ Solved |
| Final Flag | Administrator Credential Discovery | 🔄 Investigation |

---

# 🎯 Key Takeaways

Micro-CMS v2 reinforced several important web security concepts:

- SQL Injection
- Authentication bypass
- Authorization
- HTTP request manipulation
- Session security
- Administrative privilege boundaries

The biggest practical lesson for me was to look beyond the interface.

The browser may appear to prevent an action, but the real security question is:

> **What does the server actually allow?**

My approach going forward is:

```text
Understand the application
        ↓
Identify trust boundaries
        ↓
Inspect HTTP requests
        ↓
Form a hypothesis
        ↓
Test the hypothesis
        ↓
Verify the result
        ↓
Document the finding
```

---


All testing was performed against the intentionally vulnerable Hacker101 CTF environment.

The techniques described here should only be used against systems where you have explicit authorization to conduct security testing.
