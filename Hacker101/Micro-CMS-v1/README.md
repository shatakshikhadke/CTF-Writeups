# Hacker101 — Micro-CMS v1

**Platform:** Hacker101 CTF
**Challenge:** Micro-CMS v1
**Category:** Web Security

## 🧩 About the Challenge

Micro-CMS v1 looked pretty simple at first. It was basically a small CMS where I could view pages, create pages and edit them.

I started by just looking around and trying to understand how the application worked instead of immediately throwing complicated payloads at it. I changed page numbers, checked different URLs, looked at the responses, and tested what happened when I modified the input.

That eventually led me to three flags covering:

* SQL Injection
* Stored Cross-Site Scripting (XSS)
* Broken Access Control

---

# 🔎 Starting the Recon

The first thing I noticed was that the application used page IDs directly in the URL.

For example:

```text
/page/1
/page/2
/page/create
```

There was also an edit functionality:

```text
/page/edit/<id>
```

I started changing the page numbers and checking what happened.

Some of the responses I got were:

| Endpoint       | Response      |
| -------------- | ------------- |
| `/page/1`      | Accessible    |
| `/page/2`      | Accessible    |
| `/page/3`      | 404           |
| `/page/4`      | 403 Forbidden |
| `/page/edit/4` | Accessible    |

The difference with page 4 stood out to me, so I kept that in mind and continued testing.

---

# 🚩 Flag 0 — SQL Injection

### How I approached it

Since the page ID was being passed directly through the URL, I wanted to see how the application handled something that wasn't a normal number.

I started with a very simple test:

```text
/page/edit/1'
```

I wasn't expecting a lot from just adding a quote, but the application's behaviour changed and it revealed **Flag 0**.

That immediately made me think that the page ID might be reaching a SQL query without being handled safely.

### What I found

The vulnerable functionality was:

```text
/page/edit/<id>
```

The simple test:

```text
/page/edit/1'
```

was enough to show that the input was affecting the backend database operation.

My testing path was basically:

```text
Normal page ID
      ↓
/page/edit/1
      ↓
Modify the ID
      ↓
/page/edit/1'
      ↓
Unexpected behaviour
      ↓
Flag 0
```

### Vulnerability

**SQL Injection**

The likely underlying problem is that user-controlled input is being incorporated into a database query without proper parameterization.

### What I learned

This was a good reminder that even something as simple as a page number is still **user input**.

A parameter being expected to contain an integer doesn't make it automatically safe.

A proper implementation should use parameterized queries/prepared statements and validate the ID server-side.

---

# 🚩 Flag 1 — Stored Cross-Site Scripting

This was the part where I started experimenting more with the page creation functionality.

The application allowed me to create pages and enter content into the title and body. Since the application mentioned Markdown, I wanted to see what happened with HTML and JavaScript.

I started with simple tests rather than trying to guess the final payload immediately.

For example:

```html
<img src=x onerror=alert(1)>
```

I also tried:

```html
<button onclick="alert('xss')">click</button>
```

Some things didn't behave exactly as expected, so I started paying more attention to **what was actually being stored and what happened when I opened the page again**.

That was the important clue.

I could create a page with my input, and the content remained there when the page was loaded again.

So I started thinking about it as:

```text
My input
   ↓
Page created
   ↓
Input gets stored
   ↓
Page opened again
   ↓
Stored content is rendered
   ↓
Browser interprets it
```

After testing the stored content, I was able to trigger the behaviour that revealed **Flag 1**.

### Vulnerability

**Stored Cross-Site Scripting (XSS)**

The important part here was that the payload wasn't only reflected in the immediate request. It was stored as part of the page and then processed when the stored page was viewed.

### What this taught me

One thing I understood better from this challenge is that XSS isn't limited to:

```html
<script>alert(1)</script>
```

Blocking `<script>` tags doesn't necessarily make an application safe.

HTML event handlers and other browser-executable contexts can also become dangerous if user-controlled HTML is rendered without proper sanitization or output encoding.

---

# 🚩 Flag 2 — Broken Access Control

I had actually noticed something suspicious about page 4 earlier during reconnaissance.

When I visited:

```text
/page/4
```

I received:

```text
403 Forbidden
```

So I assumed that page 4 was restricted.

But then I tried:

```text
/page/edit/4
```

and the edit page was accessible.

That didn't make much sense.

If I couldn't access the page itself, why could I access the functionality for editing that same page?

### My thought process

I compared the two endpoints:

```text
/page/4
    ↓
403 Forbidden

/page/edit/4
    ↓
Accessible
```

That suggested that the access-control checks weren't being applied consistently.

This led me to **Flag 2**.

### Vulnerability

**Broken Access Control**

The application was restricting access to one endpoint while leaving related functionality accessible.

The important lesson for me here was that authorization needs to be checked on the actual operation being requested.

Just protecting:

```text
/page/4
```

doesn't automatically protect:

```text
/page/edit/4
```

### How it should be fixed

The application should perform server-side authorization checks for every sensitive operation.

Access-control rules should be applied consistently to viewing, editing, deleting and other operations involving the same resource.

---

# 🧠 What I Took Away From This Challenge

### 1. Start with the application, not the exploit

I didn't begin this challenge knowing exactly which payload would work.

I started by looking at the URLs and asking simple questions:

> What happens if I change the page number?

> Why does this page give me a 403?

> Why can I edit something I can't view?

> What happens if I put a quote here?

That approach ended up being enough to uncover the vulnerabilities.

---

### 2. Small changes can reveal a lot

The SQL injection started with something as simple as:

```text
'
```

I think this was one of the more useful parts of the challenge because it showed me why basic manual testing is still important.

You don't always need a complicated payload to discover that something is wrong.

---

### 3. Compare how different endpoints behave

The `/page/4` and `/page/edit/4` difference was probably my biggest clue for the access-control issue.

Looking at individual pages in isolation wouldn't have made the problem as obvious.

Comparing related endpoints made the inconsistency stand out.

---

### 4. Follow the input

For the XSS part, I had to stop thinking only about the payload itself and look at what happened to my input after I submitted it.

The useful flow was:

```text
Input
  ↓
Storage
  ↓
Retrieval
  ↓
Rendering
  ↓
Browser execution
```

Understanding that flow made it much easier to recognise why the vulnerability was **stored XSS**.

---

### 5. Manual testing is actually useful

For this challenge, most of my progress came from:

* Changing URL parameters
* Trying different page IDs
* Checking HTTP responses
* Inspecting the page
* Testing controlled HTML/JavaScript input
* Looking at what happened after the input was stored

It was less about using a huge toolset and more about noticing small inconsistencies and following them.

---

# 🛠️ Tools Used

* Browser
* Browser Developer Tools
* Network tab
* View Page Source
* Manual URL manipulation
* Basic HTML/JavaScript testing
* Hacker101 CTF environment

I solved the challenge through manual testing rather than relying on Burp Suite.

---

# 📌 Final Takeaway

Micro-CMS v1 initially looked like a very basic CMS, but once I started changing inputs and comparing how the different endpoints behaved, a few interesting things started appearing.

The three flags I found were:

| Flag       | Vulnerability         | How I got there                             |
| ---------- | --------------------- | ------------------------------------------- |
| **Flag 0** | SQL Injection         | Manipulating the page ID                    |
| **Flag 1** | Stored XSS            | Testing and storing HTML/JavaScript content |
| **Flag 2** | Broken Access Control | Comparing `/page/4` with `/page/edit/4`     |

For me, the biggest takeaway wasn't just learning three vulnerability names. It was getting more comfortable with the process of **looking at an application, noticing something that doesn't make sense, and testing that observation instead of immediately moving on.**

> **Disclaimer:** All testing documented here was performed against the intentionally vulnerable Hacker101 CTF environment for educational purposes.
