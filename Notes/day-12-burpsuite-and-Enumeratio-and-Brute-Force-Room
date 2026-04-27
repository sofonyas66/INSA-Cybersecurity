# Day 12 — burpsuite and Enumeratio & Brute Force Room

**Date:** Sunday April 26, 2026

## HTTP Methods

HTTP methods (also called verbs) define the intended action for a request.

| Method  | Purpose                                      | Has Body? |
|---------|----------------------------------------------|-----------|
| GET     | Retrieve a resource                          | No        |
| POST    | Submit data to the server                    | Yes       |
| PUT     | Replace a resource entirely                  | Yes       |
| PATCH   | Partially update a resource                  | Yes       |
| DELETE  | Remove a resource                            | No        |
| HEAD    | Like GET but returns headers only, no body   | No        |
| OPTIONS | Ask server which methods are allowed         | No        |

**Security relevance:**
- Servers should only allow the methods they need — an endpoint that accepts DELETE when it shouldn't is a misconfiguration
- Switching methods (GET → POST) can sometimes bypass weak input validation
- `OPTIONS` can reveal which methods are enabled — useful in recon

---

## HTTP Status Codes

Status codes tell you what happened with a request. Five classes:

| Range | Class         | Meaning                              |
|-------|---------------|--------------------------------------|
| 1xx   | Informational | Request received, continuing         |
| 2xx   | Success       | Request understood and fulfilled     |
| 3xx   | Redirection   | Further action needed                |
| 4xx   | Client Error  | Bad request from the client side     |
| 5xx   | Server Error  | Server failed to handle the request  |

**Ones you need to know:**

| Code | Meaning               | Notes                                                  |
|------|-----------------------|--------------------------------------------------------|
| 200  | OK                    | Normal success                                         |
| 201  | Created               | Resource was successfully created (e.g. after POST)    |
| 301  | Moved Permanently     | Permanent redirect — browser follows automatically     |
| 302  | Found (Temp Redirect) | Temporary redirect — common after login                |
| 400  | Bad Request           | Malformed syntax or invalid parameters                 |
| 401  | Unauthorized          | Not authenticated — need to log in                     |
| 403  | Forbidden             | Authenticated but no permission                        |
| 404  | Not Found             | Resource doesn't exist                                 |
| 405  | Method Not Allowed    | That HTTP method isn't permitted on this endpoint      |
| 429  | Too Many Requests     | Rate limiting in effect                                |
| 500  | Internal Server Error | Something broke on the server — useful for error recon |
| 503  | Service Unavailable   | Server overloaded or down                              |

**In web testing:** Response code differences during brute force or fuzzing are a key signal — a 302 where everything else returns 200 often means a valid credential or parameter.

---

## Parameters

Parameters are how data is passed to a web server. Three main locations:

**1. Query string (URL parameters) — GET**
```
GET /search?query=admin&page=1
```
- Visible in the URL
- Logged in browser history, server logs, proxies
- Never use for sensitive data

**2. Request body — POST/PUT/PATCH**
```
POST /login
Content-Type: application/x-www-form-urlencoded

username=admin&password=secret
```
- Not in the URL but still plaintext over HTTP (only encrypted over HTTPS)
- Can also be JSON: `{"username": "admin", "password": "secret"}`

**3. Path parameters**
```
GET /users/42/profile
```
- The `42` is a parameter embedded in the URL path itself
- Common in REST APIs — changing it to another ID is an IDOR test

**In Burp Suite:** All three are visible and editable in the Repeater/Intruder. Parameters are the primary attack surface for injection, IDOR, and brute force.

---

## Regular Expressions (Regex)

A pattern language for matching strings. Used heavily in web security for input validation, WAF rules, log analysis, and Burp Suite grep matching.

### Core syntax

| Pattern   | Matches                                          |
|-----------|--------------------------------------------------|
| `.`       | Any single character                             |
| `*`       | Zero or more of the previous                     |
| `+`       | One or more of the previous                      |
| `?`       | Zero or one of the previous (optional)           |
| `^`       | Start of string                                  |
| `$`       | End of string                                    |
| `[abc]`   | Any one of: a, b, or c                           |
| `[a-z]`   | Any lowercase letter                             |
| `[^abc]`  | Anything except a, b, or c                       |
| `\d`      | Any digit (0–9)                                  |
| `\w`      | Any word character (letters, digits, underscore) |
| `\s`      | Any whitespace                                   |
| `{n,m}`   | Between n and m repetitions                      |
| `(abc)`   | Capture group                                    |
| `a\|b`    | a or b                                           |

### Examples
```bash
# Find lines with "flag" in a file
grep "flag" response.txt

# Find BB{...} flag format
grep -oE "BB\{[^}]+\}" response.txt

# Match an email address pattern
grep -oE "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" file.txt

# Extract base64-looking strings (alphanumeric + = padding)
grep -oE "[A-Za-z0-9+/]{20,}={0,2}" headers.txt
```

**In Burp Suite:** Intruder's "Grep - Match" and "Grep - Extract" tabs use regex to flag interesting responses automatically during attacks.

---

## OWASP Top 10 (Common Web Vulnerabilities)

The OWASP Top 10 is the standard reference for the most critical web application security risks.

### A01 — Broken Access Control
Users can act outside their intended permissions.
- **Examples:** Accessing another user's account by changing an ID in the URL (IDOR), accessing admin pages without being an admin, elevation of privilege
- **Test:** Change path params, user IDs, role values and observe what the server returns

### A02 — Cryptographic Failures
Sensitive data exposed due to weak or missing encryption.
- **Examples:** Passwords stored in plaintext, HTTP instead of HTTPS, weak hashing (MD5, SHA1 for passwords), hardcoded secrets in client-side JS
- **Seen in class:** NexaCore CTF challenge — API token in JS file

### A03 — Injection
Untrusted data sent to an interpreter as part of a command or query.
- **Types:** SQL injection, command injection, LDAP injection, XPath injection
- **Example:** `' OR 1=1 --` in a login form that bypasses authentication
- **Why it works:** Server constructs a query using user input without sanitization

### A04 — Insecure Design
Flaws in architecture and design — not just implementation bugs.
- **Example:** A password reset that relies on a security question easily found on social media
- **Fix:** Threat modeling during design phase, not just patching after

### A05 — Security Misconfiguration
Default credentials, unnecessary features enabled, verbose error messages, missing hardening.
- **Examples:** Default admin:admin credentials, directory listing enabled, stack traces shown to users
- **Test:** Check `OPTIONS`, try default creds, look at error messages for framework/version info

### A06 — Vulnerable and Outdated Components
Using libraries, frameworks, or services with known CVEs.
- **Example:** Running an old version of a CMS with a published exploit
- **Fix:** Keep dependencies updated, track CVEs

### A07 — Identification and Authentication Failures
Broken login, session management, or credential handling.
- **Examples:** No brute force protection, weak session tokens, passwords not hashed, session fixation
- **Seen in class:** Juice Shop login brute force with Burp Intruder

### A08 — Software and Data Integrity Failures
Trusting code or data without verifying its integrity.
- **Example:** Auto-updating from an untrusted source, deserializing unvalidated data
- **Modern relevance:** Supply chain attacks (SolarWinds-style)

### A09 — Security Logging and Monitoring Failures
Not detecting or logging attacks, making incident response impossible.
- **Example:** No alerts on 1000 failed login attempts, logs deleted without detection
- **Seen in class:** Windows Event ID 1102 — log clearing as an attack indicator

### A10 — Server-Side Request Forgery (SSRF)
Server fetches a URL supplied by the user — attacker redirects it to internal resources.
- **Example:** `?url=http://169.254.169.254/latest/meta-data/` to reach AWS metadata endpoint
- **Impact:** Internal network scanning, credential theft from cloud metadata

---

## Authentication & Access Control

### Authentication vs Authorization
- **Authentication:** Are you who you claim to be? (identity — login)
- **Authorization / Access Control:** Are you allowed to do this? (permissions)
Both must be correct. You can be authenticated but still unauthorized.

### Common Authentication Weaknesses

**No rate limiting / lockout**
- Server allows unlimited login attempts
- Enables brute force and credential stuffing attacks
- **Fix:** Lock after N failures, CAPTCHA, rate limiting (HTTP 429)

**Weak credentials**
- Default passwords (`admin:admin`, `admin:password`)
- Short or simple passwords with no complexity enforcement

**Insecure session management**
- Predictable session tokens
- Session not invalidated after logout
- Token transmitted over HTTP (not HTTPS)

**Verbose error messages**
- "Invalid username" vs "Invalid password" — lets attacker enumerate valid usernames
- Both should return the same generic message: "Invalid credentials"

### Enumeration
Enumeration is discovering valid usernames, emails, or other identifiers before attempting to crack passwords.

**Indicators that enumeration is possible:**
- Different response message for wrong username vs wrong password
- Different response *length* (even with same message text)
- Different response *time* (server does more work for a valid username)
- Different HTTP status code

**In Burp Intruder:**
- Set the username as the payload position
- Load a wordlist
- Sort results by response length or status code — outliers are valid usernames

### Access Control Attacks

**IDOR (Insecure Direct Object Reference)**
- Directly referencing an object (ID, filename) the user shouldn't have access to
- Example: `/api/users/42/data` — change 42 to 43 to access another user's data
- **Test in Burp:** Modify path/query parameters and observe whether authorization is checked

**Privilege Escalation**
- Horizontal: accessing another user's data at the same privilege level (IDOR)
- Vertical: accessing functionality reserved for a higher role (e.g., regular user reaching admin panel)
- **Test:** Modify role/admin parameters in requests, try accessing `/admin` endpoints directly

---

## TryHackMe: Enumeration & Brute Force

**Room:** https://tryhackme.com/room/enumerationbruteforce

**Topics covered:**
- Username enumeration via response differences (message, length, timing)
- Password brute forcing against login forms
- Using Burp Suite Intruder for automated attacks
- Hydra for command-line brute forcing
- Identifying rate limiting and lockout behavior
- Cookie/session token analysis

**Core workflow practiced:**
1. Capture login request in Burp Proxy
2. Send to Intruder → mark username or password as payload position
3. Load wordlist → start attack
4. Sort by response length/status to find the valid credential
5. Confirm manually in Repeater

**Hydra equivalent (CLI):**
```bash
# Brute force HTTP POST login
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  <target-ip> http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid credentials"

# Enumerate usernames with a list
hydra -L usernames.txt -p dummy \
  <target-ip> http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid username"
```
