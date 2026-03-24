# 30 Common Ports, Services & Attacks — Research

**Name:** Sofonyas Yared
**Program:** INSA Cyber Talent Weekend Program
**Track:** Cybersecurity
**Date:** March 22, 2026

---

## What is a Port?

A number that identifies a specific service on a device.

```
IP address → finds the device
Port       → finds the service on that device

Like an apartment building:
IP address = building address
Port       = apartment number number
```

Port ranges:
```
0     - 1023  → Well Known (system/reserved)
1024  - 49151 → Registered (applications)
49152 - 65535 → Dynamic (temporary/client)
```

---

## 30 Common Ports

| # | Port | Protocol | Service | Description | Typical Attack |
|---|------|----------|---------|-------------|----------------|
| 1 | 20 | TCP | FTP Data | File transfer data channel | Cleartext sniffing |
| 2 | 21 | TCP | FTP Control | File transfer commands | Brute force, anonymous login |
| 3 | 22 | TCP | SSH | Secure remote login | Brute force, weak keys |
| 4 | 23 | TCP | Telnet | Unsecure remote login | Sniffing (cleartext), MitM |
| 5 | 25 | TCP | SMTP | Send emails | Spam relay, spoofing |
| 6 | 53 | UDP/TCP | DNS | Domain to IP resolution | DNS spoofing, cache poisoning |
| 7 | 67 | UDP | DHCP Server | Assigns IP addresses | Rogue DHCP server |
| 8 | 68 | UDP | DHCP Client | Receives IP from server | DHCP starvation |
| 9 | 69 | UDP | TFTP | Simple file transfer | Unauthorized file access |
| 10 | 80 | TCP | HTTP | Web traffic unencrypted | XSS, SQLi, MitM |
| 11 | 110 | TCP | POP3 | Receive emails | Cleartext credential sniffing |
| 12 | 119 | TCP | NNTP | News transfer protocol | Unauthorized access |
| 13 | 123 | UDP | NTP | Network time sync | NTP amplification DDoS |
| 14 | 135 | TCP | RPC | Remote procedure call | MS exploits, worms |
| 15 | 139 | TCP | NetBIOS | Windows file sharing | Enumeration, relay attacks |
| 16 | 143 | TCP | IMAP | Email retrieval | Brute force, sniffing |
| 17 | 161 | UDP | SNMP | Network device management | Community string sniffing |
| 18 | 194 | TCP | IRC | Internet relay chat | Botnet C2 channel |
| 19 | 389 | TCP | LDAP | Directory services | Injection, enumeration |
| 20 | 443 | TCP | HTTPS | Secure web traffic | SSL stripping, cert spoofing |
| 21 | 445 | TCP | SMB | Windows file sharing | EternalBlue, ransomware |
| 22 | 514 | UDP | Syslog | System logging | Log injection, spoofing |
| 23 | 993 | TCP | IMAPS | Secure IMAP | Weak TLS exploitation |
| 24 | 995 | TCP | POP3S | Secure POP3 | Weak TLS exploitation |
| 25 | 1433 | TCP | MSSQL | Microsoft SQL Server | SQL injection, brute force |
| 26 | 1723 | TCP | PPTP | VPN protocol | Weak encryption attacks |
| 27 | 3306 | TCP | MySQL | MySQL database | SQL injection, brute force |
| 28 | 3389 | TCP | RDP | Remote desktop | Brute force, BlueKeep |
| 29 | 5900 | TCP | VNC | Remote desktop (graphical) | Brute force, no encryption |
| 30 | 8080 | TCP | HTTP Alt | Alternate web port | Same as port 80 attacks |

---

## Most Dangerous Open Ports

```
23   Telnet  → Everything sent in cleartext
               Replace with SSH (22)

445  SMB     → EternalBlue exploit used by
               WannaCry ransomware (2017)
               Affected 200,000+ computers

3389 RDP     → BlueKeep vulnerability
               Brute force target
               Never expose to internet

3306 MySQL   → Database directly exposed
               Should never be public facing

5900 VNC     → Often no authentication
               Full graphical access to machine
```

---

## Ports by Category

**Remote Access:**
```
22   SSH      → secure
23   Telnet   → insecure (avoid)
3389 RDP      → Windows remote desktop
5900 VNC      → graphical remote access
```

**Web:**
```
80   HTTP     → unencrypted
443  HTTPS    → encrypted
8080 HTTP Alt → development/proxy
```

**Email:**
```
25   SMTP     → send mail
110  POP3     → receive mail
143  IMAP     → receive mail (sync)
993  IMAPS    → secure IMAP
995  POP3S    → secure POP3
```

**File Transfer:**
```
20   FTP Data    → unencrypted
21   FTP Control → unencrypted
69   TFTP        → no authentication
445  SMB         → Windows shares
```

**Database:**
```
1433 MSSQL   → Microsoft SQL
3306 MySQL   → MySQL/MariaDB
```

**Network Services:**
```
53   DNS     → domain resolution
67   DHCP    → IP assignment
123  NTP     → time sync
161  SNMP    → device management
```

---

## Red Team — Ports to Target First

```
Port  Why
────────────────────────────────
21    Anonymous FTP login still common
22    Brute force SSH with weak passwords
23    Telnet — sniff credentials directly
80    Web app vulnerabilities
139/445 SMB — check for EternalBlue
3306  MySQL exposed to internet
3389  RDP brute force
5900  VNC with no password
8080  Dev servers left running
```

## Blue Team — Ports to Monitor/Block

```
→ Close all ports not in use
→ Never expose 23, 3306, 5900 to internet
→ Monitor 22, 3389 for brute force attempts
→ Alert on port 4444 (Metasploit default)
→ Alert on port 1337 (common backdoor)
→ Use firewall to whitelist only needed ports
```

---

## Scanning Ports

```bash
# Scan common ports
nmap 192.168.1.1

# Scan all 65535 ports
nmap -p- 192.168.1.1

# Scan specific port
nmap -p 22,80,443 192.168.1.1

# Detect service version
nmap -sV 192.168.1.1

# See open ports on your machine
ss -tuln
```
