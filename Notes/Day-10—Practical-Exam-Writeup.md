# Day 10 — Practical Exam Writeup
## CTF Challenge: Black Box Penetration Test

> Given only an IP address, find all hidden flags using real security tools and techniques.

**Target**: `ec2-3-227-254-50.compute-1.amazonaws.com` (`3.227.254.50`)  
**Flags found**: 9/11  

---

## Methodology

When given only an IP, the approach follows a structured recon workflow:

```
Host Discovery → Port Scan → Service Enum → Vuln Scan → Exploitation → Flag Hunting
```

---

## Step 1 — Reconnaissance (Nmap)

### Service & Version Detection
```bash
nmap -sV -Pn 3.227.254.50
```

**Result:**
```
PORT      STATE  SERVICE  VERSION
21/tcp    open   ftp      vsftpd 3.0.5
22/tcp    open   ssh      OpenSSH 8.7 (protocol 2.0)
80/tcp    open   http     Apache httpd 2.4.41 (Ubuntu)
1022/tcp  open   ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
30000/tcp closed ndmps
```

**Answer to Q1**: 4 open ports  
**Answer to Q2**: Services listed above with versions

### Vulnerability Scan
```bash
nmap --script vuln -Pn 3.227.254.50 -oN vuln.txt
```

**Notable findings:**
- `/robots.txt` discovered on port 80
- No CSRF/XSS/DOM vulnerabilities found automatically

**Answer to Q3 & Q4 — Known CVEs:**

| Service | CVE | Description |
|---------|-----|-------------|
| vsftpd 3.0.5 | CVE-2021-3618 | ALPACA attack — traffic hijacking |
| vsftpd 3.0.5 | CVE-2015-1419 | File permission restriction bypass |
| OpenSSH 8.7 | CVE-2023-38408 | RCE via ssh-agent |
| OpenSSH 8.7 | CVE-2023-51385 | OS command injection |
| Apache 2.4.41 | CVE-2021-41773 | Path traversal + RCE (critical) |
| Apache 2.4.41 | CVE-2021-42013 | Path traversal bypass (critical) |
| OpenSSH 8.2p1 | CVE-2020-15778 | Command injection via scp |
| OpenSSH 8.2p1 | CVE-2021-28041 | Double free memory vulnerability |

---

## Step 2 — Web Enumeration

### robots.txt
```bash
curl http://3.227.254.50/robots.txt
```
```
Disallow: /messages/user
```

### Exploring the disallowed path
```bash
curl http://3.227.254.50/messages/
curl http://3.227.254.50/messages/user
```

**Found an internal message:**
```
Hello abebe,
We have told you multiple times to make your password strong and told 
you not to use the word password as part of your password.
Adding the year to it does not make it strong, so change it immediately!!!
- Dev Team
```

**Credentials discovered**: `abebe : password2024`  
The message revealed a weak password pattern — username + "password" + year.

### Directory Fuzzing
```bash
ffuf -u http://3.227.254.50/FUZZ -w /usr/share/dirb/wordlists/common.txt -mc 200,301
```
```
index.html    [200]
messages      [301]
robots.txt    [200]
```

---

## Step 3 — FTP Access

Using the credentials discovered from the web message:
```bash
ftp 3.227.254.50
# Name: abebe
# Password: password2024
```

Login successful! 

### FTP Enumeration
```bash
ftp> ls -la          # root directory: 3 hidden files + ftp/ folder
ftp> cd ftp
ftp> ls -la          # contents below
```

**FTP root directory contents (Answer to Q5: 4 files):**
```
.hidden.txt          (40 bytes)
Initial_Access.txt   (16 bytes)
challenge.jpeg       (77168 bytes)
readme.txt           (0 bytes)
captured/            (directory)
logs/                (directory)
```

```bash
ftp> cd captured && ls -la
# analyst_note
# bonus.pcapng
# captured.pcap

ftp> cd ../logs && ls -la
# SSH.log
```

Downloaded everything:
```bash
ftp> mget *
```

---

## Step 4 — Flag Hunting

### 🚩 Flag 1 — Initial Access
```bash
cat Initial_Access.txt
```
```
flag{ftp_access}
```
**Technique**: Plain text file on FTP server

---

### 🚩 Flag 2 — Hidden File (Base64)
```bash
cat .hidden.txt
# RkxBR3tIaWRkZW5fRmxhZ3NfSGF2ZV9HZW1zfQo=

cat .hidden.txt | base64 -d
```
```
FLAG{Hidden_Flags_Have_Gems}
```
**Technique**: Base64 encoded content in hidden file

---

### 🚩 Flag 3 — Image Metadata
```bash
exiftool Challenge.jpeg
```
```
Comment: FLAG{Check_Metadata}
```
**Technique**: EXIF metadata inspection with exiftool

---

### 🚩 Flag 4 — Steganography
```bash
steghide extract -sf Challenge.jpeg
# Passphrase: password2024

cat flag4.txt | base64 -d
```
```
FLAG{steghide_protected}
```
**Technique**: Data hidden inside JPEG using steghide, encoded in Base64

---

### 🚩 Flag 5 — File in File (Binwalk)
```bash
binwalk Challenge.jpeg
# Found: ZIP archive at offset 77192

binwalk -e Challenge.jpeg
cd _Challenge.jpeg.extracted/12D88/
cat flag3.txt
```
```
d1db559832a70a5e4b48a0f21cc8da04
```
**Technique**: ZIP archive appended to end of JPEG, extracted with binwalk. The flag is an MD5 hash.

---

### 🚩 Flag 6 — ICMP Covert Channel
```bash
wireshark captured/captured.pcap
# Filter: icmp
# Click first ICMP packet → inspect Data field
```
```
FLAG{CURIOUS_PING}
```
**Technique**: Secret message hidden inside ICMP ping packet payload. The analyst note said "the first ping request feels unnecessary" — that was the hint.

---

### 🚩 Flag 7 — DNS Exfiltration
```bash
wireshark captured/captured.pcap
# Filter: dns
# Inspect query names
```

Flag was split across 3 DNS subdomains:
```
FLAG{THIS_.example.com
DNS_CONS.example.com
PIRACY}.example.com
```
```
FLAG{THIS_DNS_CONSPIRACY}
```
**Technique**: Data exfiltrated by encoding flag across multiple DNS query subdomains — a real-world C2 technique called DNS tunneling.

---

### 🚩 Flag 8 — HTTP Plaintext Credentials
```bash
strings captured/captured.pcap | grep -oE "FLAG\{[^}]+\}|flag\{[^}]+\}"
```
```
username=student&password=FLAG{FOLLOW_THE_STREAM}
```
**Technique**: Credentials sent in plaintext over HTTP POST request on a non-standard port. Visible in Wireshark by following TCP stream 7. This is why HTTPS matters.

---

### 🚩 Flag 9 — SSH Log Analysis
```bash
grep -A 5 "Accepted" logs/SSH.log
```
```
Dec 24 09:32:20 LabSZ sshd[24680]: Accepted password for fztu from 119.137.62.142 port 49116 ssh2
Dec 24 09:32:20 LabSZ sshd[24680]: SUCCESS: RkxBR3tTU0hfTG9naW5fRm91bmR9Cg==

echo "RkxBR3tTU0hfTG9naW5fRm91bmR9Cg==" | base64 -d
```
```
FLAG{SSH_Login_Found}
```
**Technique**: Flag hidden as Base64 encoded string in SSH log entry immediately after a successful brute-force login. The log also showed a real SSH brute force attack — hundreds of failed attempts from multiple IPs before `fztu` was compromised from `119.137.62.142`.

---

## Summary

| # | Flag | Location | Technique |
|---|------|----------|-----------|
| 1 | `flag{ftp_access}` | `Initial_Access.txt` | Plain text |
| 2 | `FLAG{Hidden_Flags_Have_Gems}` | `.hidden.txt` | Base64 |
| 3 | `FLAG{Check_Metadata}` | `Challenge.jpeg` | EXIF metadata |
| 4 | `FLAG{steghide_protected}` | `Challenge.jpeg` | Steganography |
| 5 | `d1db559832a70a5e4b48a0f21cc8da04` | `Challenge.jpeg` | Binwalk + ZIP |
| 6 | `FLAG{CURIOUS_PING}` | `captured.pcap` | ICMP payload |
| 7 | `FLAG{THIS_DNS_CONSPIRACY}` | `captured.pcap` | DNS tunneling |
| 8 | `FLAG{FOLLOW_THE_STREAM}` | `captured.pcap` | HTTP plaintext |
| 9 | `FLAG{SSH_Login_Found}` | `SSH.log` | Log analysis + Base64 |

---

## Key Lessons

**1. Enumeration is everything**  
The FTP credentials came from a web page hidden by `robots.txt`. Always check `robots.txt` — it often reveals paths the admin doesn't want you to see.

**2. Weak passwords are exploitable**  
`password2024` was guessable from context alone. The dev team's warning message actually revealed the password pattern.

**3. Plaintext protocols are dangerous**  
HTTP on non-standard ports leaks credentials. Always use HTTPS. Wireshark showed the login in clear text.

**4. Steganography hides in plain sight**  
`Challenge.jpeg` contained 3 separate flags — metadata, steghide data, and an appended ZIP. Always run `exiftool`, `steghide`, and `binwalk` on any image file.

**5. DNS can be used as a covert channel**  
Attackers split data across DNS queries to exfiltrate information — it looks like normal DNS traffic at first glance.

**6. Logs tell the story**  
The SSH log showed a full brute force attack with hundreds of failed attempts followed by one success. Event ID equivalent in Linux — always grep for `Accepted` to find successful logins.
