# Hacker101 — Micro-CMS v1

## 📌 Challenge Overview

**Platform:** Hacker101 CTF
**Challenge:** Micro-CMS v1
**Category:** Web Security
**Difficulty:** Beginner / Intermediate

Micro-CMS v1 is a vulnerable web application that simulates a basic content management system. The challenge involves analyzing its functionality, enumerating endpoints, manipulating parameters, and identifying vulnerabilities in how the application handles user input and authorization.

---

## 🎯 Objectives

The challenge involved identifying multiple vulnerabilities and using them to retrieve the challenge flags.

The main areas investigated were:

* Web application reconnaissance
* Endpoint enumeration
* Parameter manipulation
* SQL Injection
* Access control
* Stored Cross-Site Scripting (XSS)
* HTML/source-code inspection

---

# 🔎 Reconnaissance

The application exposed several page-related endpoints.

```text
/page/1
/page/2
/page/create
/page/edit/<id>
```

The application allowed users to:

* View existing pages
* Create new pages
* Edit existing pages
* Submit a title and body
* Render Markdown and HTML content

The page creation and editing forms contained `title` and `body` parameters and did not contain a hidden page ID field.

### Initial observations

During endpoint enumeration, different page IDs produced different responses:

| Endpoint       | Response   |
| -------------- | ---------- |
| `/page/1`      | Accessible |
| `/page/2`      | Accessible |
| `/page/3`      | 404        |
| `/page/4`      | 403        |
| `/page/edit/4` | Accessible |

This difference in behavior became important later in the challenge.

---

# 🚩 Flag 1 — SQL Injection

### Hint

> Look at the sequence of IDs.

The application used numerical page IDs in its URLs.

I tested how the application handled unexpected input in the page ID parameter.

The following request produced abnormal behavior:

```text
/page/edit/1'
```

Instead of behaving like a normal invalid page ID, the application returned a challenge flag.

### Vulnerability Identified

**SQL Injection**

The behavior indicated that the page ID was being incorporated into a backend database query without adequate parameterization or validation.

### Attack Surface

```text
/page/edit/<id>
```

### Key Observation

A single quote appended to the page ID altered the application's behavior:

```text
/page/edit/1'
```

This is a strong indication that user-controlled input was reaching an SQL query unsafely.

### Recommended Remediation

* Use prepared statements / parameterized queries.
* Validate page IDs as integers.
* Never concatenate user input directly into SQL queries.
* Perform validation server-side.

---

# 🚩 Flag 2 — Broken Access Control

During enumeration, page `4` returned:

```text
/page/4
```

with a:

```text
403 Forbidden
```

However, the corresponding edit endpoint was accessible:

```text
/page/edit/4
```

This exposed functionality that should have been protected by the same authorization controls.

### Vulnerability Identified

**Broken Access Control**

The application enforced access restrictions inconsistently between related endpoints.

### Attack Flow

```text
/page/4
     ↓
403 Forbidden

/page/edit/4
     ↓
Accessible
```

This demonstrates why authorization must be enforced independently on every sensitive endpoint.

### Recommended Remediation

* Perform server-side authorization checks on every endpoint.
* Apply access-control rules consistently across view, edit, delete, and administrative functionality.
* Do not rely on restrictions implemented only at the UI or one route.
* Centralize authorization logic where possible.

---

# 🚩 Flag 3 — Stored Cross-Site Scripting

The final challenge hint pointed toward:

> Stored XSS

The application allowed user-controlled content to be stored and subsequently rendered.

Initial testing showed that some JavaScript payloads were filtered or escaped, while HTML event-handler based payloads were processed by the browser.

For example:

```html
<img src=x onerror=alert(1)>
```

was used to test whether HTML event handlers could execute.

Another payload tested was:

```html
<button onclick="alert('xss')">click</button>
```

The important behavior was that the content was **stored by the application** and could later be encountered when the stored page was rendered.

The resulting page source contained the challenge flag.

### Vulnerability Identified

**Stored Cross-Site Scripting (XSS)**

### Why It Is Stored XSS

The vulnerability follows this flow:

```text
Attacker-controlled input
        ↓
Application stores input
        ↓
Stored content is rendered
        ↓
Browser interprets the HTML/JavaScript
        ↓
JavaScript executes
```

### Recommended Remediation

* Apply context-aware output encoding.
* Sanitize user-supplied HTML using a strict allowlist.
* Do not permit arbitrary JavaScript event handlers.
* Avoid rendering raw user-controlled HTML.
* Implement a strong Content Security Policy (CSP) as defense-in-depth.

---

# 🛡️ Security Lessons Learned

### 1. Input validation is essential

Even parameters expected to contain only numbers must be validated and safely handled.

### 2. Authorization must be enforced server-side

Restricting access to one endpoint does not automatically protect related functionality.

### 3. XSS is not limited to `<script>` tags

Blocking `<script>` tags alone is not sufficient. HTML event handlers and other browser-executable contexts can also introduce XSS.

### 4. Understand the complete data flow

For stored XSS, it is important to follow the lifecycle of the input:

```text
Input → Storage → Retrieval → Rendering → Execution
```

### 5. Manual testing remains valuable

Simple URL manipulation, source inspection, and controlled input testing revealed vulnerabilities that could easily be missed by looking only at the application's visible interface.

---

# 🧰 Tools Used

* Browser Developer Tools
* View Page Source
* Manual HTTP/URL manipulation
* HTML/JavaScript testing
* Hacker101 CTF environment

---

## 📚 Conclusion

Micro-CMS v1 provided hands-on experience with several common web application security vulnerabilities:

* **SQL Injection**
* **Broken Access Control**
* **Stored Cross-Site Scripting**

The challenge demonstrated how seemingly simple CMS functionality can become vulnerable when input validation, database security, authorization, and output encoding are not implemented correctly.

This write-up documents the methodology and security concepts used during the challenge. **Challenge flags are intentionally omitted from this public repository.**

---

> **Disclaimer:** All testing documented here was performed against the intentionally vulnerable Hacker101 CTF environment for educational purposes.
