# Day 08 — Nmap & Networking Revision

**Date:** sunday April 5, 2026
---

## Overview

Mostly a revision session covering networking fundamentals
and Nmap from the previous class. One new topic introduced:
NSE scripting internals.

---

## Nmap Scripting Engine (NSE) — Under the Hood

### Scripts Are Written in Lua

All NSE scripts use the `.nse` file extension and are written
in the Lua programming language. Nmap embeds a full Lua 5.4
interpreter to execute them.

```bash
# Scripts live here on Linux:
ls /usr/share/nmap/scripts/

# Run a specific script:
sudo nmap --script=http-title 192.168.1.1

# Run all default scripts:
sudo nmap -sC 192.168.1.1

# Run a category:
sudo nmap --script=vuln 192.168.1.1
```

### Why Nmap Chose Lua

```
Embeddability → Lua is designed to be embedded inside
                other applications. It adds almost no
                overhead to Nmap's binary size.

Efficiency    → Lua supports coroutines — lightweight
                threads that allow many scripts to run
                in parallel without the memory cost of
                real OS threads.

Simplicity    → Easy to learn for anyone with basic
                scripting or C-style experience.
                Scripts are short and readable.
```

### How NSE Scripts Are Structured

Every NSE script has three required parts:

```lua
-- 1. Description and metadata
description = [[
  Checks if the target is vulnerable to X.
]]

-- 2. Rule: when should this script run?
portrule = function(host, port)
  return port.number == 80
end

-- 3. Action: what should it do?
action = function(host, port)
  return "Found something interesting"
end
```

### NSE Libraries

Nmap provides built-in Lua libraries that connect scripts
to Nmap's internal scanning engine:

```
http    → send HTTP requests, parse responses
smb     → interact with Windows file sharing
ssl     → inspect TLS/SSL connections
dns     → perform DNS queries
ftp     → interact with FTP services
ssh1/2  → SSH protocol interaction
mysql   → MySQL database queries
brute   → generic brute-force framework
```

### Script Categories (Revision)

```
default    → run with -sC or -A (safe, useful)
safe       → will not crash or harm target
vuln       → check for known vulnerabilities
auth       → test authentication
brute      → brute force credentials
discovery  → gather more information
intrusive  → may crash or affect target
malware    → detect backdoors
exploit    → actively exploit weaknesses
```

### Practical Examples

```bash
# Check for MS17-010 (EternalBlue / WannaCry)
sudo nmap --script=smb-vuln-ms17-010 192.168.1.1

# Enumerate HTTP directories
sudo nmap --script=http-enum 192.168.1.1

# Grab SSL certificate details
sudo nmap --script=ssl-cert -p 443 192.168.1.1

# Brute force SSH login
sudo nmap --script=ssh-brute 192.168.1.1

# Check anonymous FTP access
sudo nmap --script=ftp-anon -p 21 192.168.1.1

# Run multiple scripts at once
sudo nmap --script=http-title,http-headers 192.168.1.1

# Run all scripts in a category
sudo nmap --script=auth 192.168.1.1
```

### Writing a Simple NSE Script

```lua
-- simple_banner.nse
-- Grabs the banner from any open port

description = "Grabs service banner from open port"

portrule = function(host, port)
  return port.state == "open"
end

action = function(host, port)
  local socket = nmap.new_socket()
  socket:connect(host.ip, port.number)
  local status, response = socket:receive()
  socket:close()
  if status then
    return "Banner: " .. response
  end
end
```

---

## Revision Summary — Key Concepts

### Nmap Port States

```
open         → SYN-ACK received, service running
closed       → RST-ACK received, no service
filtered     → silence or ICMP prohibited, firewall blocking
open|filtered→ no response, ambiguous (common in UDP)
```

### Nmap Stealth Techniques Recap

```
-sS              → half-open SYN scan, no full handshake
-T0/-T1          → slow timing, evades rate-based IDS
-f / --mtu       → fragmented packets, evades DPI
-D RND:10        → decoy IPs flood the target log
--source-port 53 → spoof DNS port, bypasses some firewalls
-sI zombie       → idle scan, your IP never touches target
```

### OSI + Nmap Layer Mapping

```
Layer 7 Application  → NSE scripts (http, smb, ftp probes)
Layer 4 Transport    → TCP/UDP port scanning, SYN scan
Layer 3 Network      → IP spoofing, TTL-based OS fingerprint
Layer 2 Data Link    → (raw socket access via Scapy/libpcap)
```

---

## Key Takeaways — Week 4 Sunday

- NSE scripts are Lua files (.nse) — Lua was chosen for its
  speed, coroutine support, and ease of embedding
- Coroutines let many scripts run in parallel efficiently
- Scripts have three parts: description, rule, action
- NSE libraries (http, smb, ssl) connect Lua to Nmap internals
- `-sC` runs the default script set, same as `--script=default`

---

## Assignments Carried Over

- [ ] Complete NetFP tool — understand every component
- [ ] Research C2 frameworks and how beaconing looks in Wireshark
