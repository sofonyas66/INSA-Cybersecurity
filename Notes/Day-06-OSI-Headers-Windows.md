# Day 06 — OSI Encapsulation, Headers & Windows Analysis

**Date:** Sunday March 29, 2026

---

## Morning — OSI Model Deep Dive

---

## Encapsulation & Decapsulation

The process of how data moves through the 7 OSI layers.

### Encapsulation (Sending — top to bottom)

Each layer **adds its own header** to the data before
passing it to the layer below.

```
Application  → [        Data        ]
Presentation → [      Data          ]
Session      → [     Data           ]
Transport    → [ TCP Header | Data  ]        → Segment
Network      → [ IP Header | TCP | Data ]    → Packet
Data Link    → [ MAC | IP | TCP | Data | FCS ] → Frame
Physical     → 010110101010101011010101...    → Bits
```

This process of wrapping data with headers at each layer
is called **encapsulation**.

### Decapsulation (Receiving — bottom to top)

Each layer **reads and removes its own header** then
passes the rest up to the next layer.

```
Physical     → receives bits
Data Link    → reads MAC header, removes it
Network      → reads IP header, removes it
Transport    → reads TCP header, removes it
Session      → manages session
Presentation → decrypts/decodes
Application  → user receives data
```

This unwrapping process is called **decapsulation**.

### Why it matters for security
```
Each header can be:
→ Inspected  — firewalls read IP and TCP headers
→ Spoofed    — attacker fakes source IP in IP header
→ Analyzed   — Wireshark shows every header field
→ Attacked   — each layer has its own attack surface
```

---

## Headers — What's Inside

### IP Header (Layer 3)
```
┌────────────┬──────────────┬─────────────────┐
│  Version   │  Header Len  │    TTL          │
├────────────┴──────────────┴─────────────────┤
│         Source IP Address                   │
├─────────────────────────────────────────────┤
│       Destination IP Address                │
├─────────────────────────────────────────────┤
│              Protocol                       │
│  (TCP=6, UDP=17, ICMP=1)                   │
└─────────────────────────────────────────────┘
```

Key fields:
```
Source IP      → where packet came from
Destination IP → where packet is going
TTL            → Time To Live — max hops before dropped
Protocol       → what's inside (TCP, UDP, ICMP)
```

### TCP Header (Layer 4)
```
┌──────────────────┬──────────────────┐
│   Source Port    │  Destination Port│
├──────────────────┴──────────────────┤
│           Sequence Number           │
├─────────────────────────────────────┤
│         Acknowledgment Number       │
├──────────┬──────────────────────────┤
│  Flags   │  SYN ACK FIN RST PSH URG│
└──────────┴──────────────────────────┘
```

Key fields:
```
Source Port      → which port data is coming from
Destination Port → which service to deliver to
Sequence Number  → order of packets
Flags            → SYN, ACK, FIN, RST — control connection
```

### Data / Payload
```
The actual content being sent —
webpage HTML, file data, message text, etc.
Everything above the headers is payload.
```

---

## OSI Layer Attacks Summary

| Layer | Name | Attack | Description |
|-------|------|--------|-------------|
| 7 | Application | SQL Injection | Malicious SQL in input fields |
| 7 | Application | XSS | Inject scripts into web pages |
| 7 | Application | Phishing | Fake sites steal credentials |
| 6 | Presentation | SSL Stripping | Downgrade HTTPS to HTTP |
| 5 | Session | Session Hijacking | Steal session token |
| 4 | Transport | SYN Flood | Exhaust server with half-open connections |
| 4 | Transport | Port Scanning | Find open services |
| 3 | Network | IP Spoofing | Fake source IP address |
| 3 | Network | ICMP Flood | Ping flood DoS |
| 2 | Data Link | ARP Poisoning | Link attacker MAC to victim IP |
| 2 | Data Link | MAC Spoofing | Fake MAC address |
| 1 | Physical | Cable Tapping | Intercept physical signal |
| 1 | Physical | Jamming | Disrupt wireless signals |

---

## DNS Query Resolution Process (Detailed)

How your browser finds the IP for a domain:

```
Step 1 — Browser Cache
You visit google.com
Browser checks its own cache first
If found → use cached IP → done
If not found → continue

Step 2 — OS Cache
Checks local OS DNS cache
Linux: /etc/hosts file checked first
If found → use it → done
If not → continue

Step 3 — Router (Local DNS Resolver)
Asks your home router
Router checks its own cache
If found → returns IP → done
If not → continues to ISP

Step 4 — ISP DNS Server
Router asks ISP's DNS server
If cached → returns IP → done
If not → asks Root DNS

Step 5 — Root DNS Server
13 root servers worldwide
Doesn't know the IP but knows
who manages .com, .org, .et etc
Returns address of TLD server

Step 6 — TLD DNS Server
Top Level Domain server
.com server knows who manages google.com
Returns address of Google's DNS server

Step 7 — Authoritative DNS Server
Google's own DNS server
Knows the exact IP for google.com
Returns: 142.250.185.46

Step 8 — Response travels back
Each server caches the result
Your browser connects to the IP
Next time → cache is used → faster
```

### DNS Caching
```
Why cache?
→ Faster — no need to ask every time
→ Less load on DNS servers

TTL (Time To Live)
→ How long the cache is valid
→ After TTL expires → ask again
→ Usually 300 seconds to 86400 seconds
```

### DNS attacks
```
Cache Poisoning → inject fake DNS records into cache
                  victim gets wrong IP → goes to attacker site
DNS Spoofing    → intercept DNS query, return fake response
DNS Exfiltration→ hide data inside DNS queries (Challenge 2!)
```

---

## Afternoon — Windows Network Analysis

---

## Netstat — Network Statistics

Shows active network connections, open ports, and
listening services on a Windows machine.

```cmd
netstat -an        → all connections with IP and port
netstat -b         → show which program owns each connection
netstat -ano       → show Process ID (PID) too
```

Reading the output:
```
Proto  Local Address      Foreign Address    State
TCP    0.0.0.0:80         0.0.0.0:0          LISTENING
TCP    192.168.1.5:49200  142.250.1.1:443    ESTABLISHED
```

```
LISTENING   → port is open, waiting for connections
ESTABLISHED → active connection in progress
TIME_WAIT   → connection closing
CLOSE_WAIT  → remote side closed connection
```

Security use:
```
→ Find unexpected open ports
→ Spot connections to unknown IPs
→ Detect malware phoning home
→ Identify which program is using which port
```

---

## Windows Event Logs

Windows records everything that happens in Event Logs.
Location: `Event Viewer` → Windows Logs

### Log Categories
```
System          → OS events, driver errors
Security        → Login attempts, privilege use
Application     → App crashes, errors
Setup           → Installation events
Forwarded Events→ Logs from other machines
```

### Important Event IDs for Security

| Event ID | Description | Why it matters |
|----------|-------------|----------------|
| 4624 | Successful login | Track who logged in |
| 4625 | Failed login | Brute force detection |
| 4634 | Logoff | Session tracking |
| 4648 | Login with explicit credentials | Lateral movement |
| 4672 | Admin privileges assigned | Privilege escalation |
| 4688 | New process created | Malware execution |
| 4698 | Scheduled task created | Persistence mechanism |
| 4720 | User account created | Unauthorized account |
| 4732 | User added to admin group | Privilege escalation |
| 1102 | Audit log cleared | ⚠️ Attacker covering tracks |
| 104  | System log cleared | ⚠️ Attacker covering tracks |

### Key insight — Log Clearing
```
If an attacker clears the logs:
→ Event ID 1102 is still recorded
→ The act of clearing logs creates a log entry
→ "You can't hide the fact that you hid"
→ This is the first thing Blue Team checks
```

---

## SSH on Windows & Remote Connection

Windows now has SSH built in (since Windows 10).

```cmd
# Connect to another machine via SSH
ssh username@ip_address
```

This is the same SSH you use on Linux — same protocol,
same port 22, same commands.

Use case in class:
```
If you have someone's IP on the same network
you can SSH into their machine (if SSH is enabled)
This is how remote administration works
This is also how attackers move laterally
```

---

## Useful Windows Commands for Network Analysis

```cmd
ipconfig              → show IP address (like ip addr on Linux)
ipconfig /all         → detailed network info
ipconfig /flushdns    → clear DNS cache
ping 8.8.8.8          → test connectivity
tracert google.com    → trace route to destination
netstat -an           → show all connections
nslookup google.com   → DNS lookup
arp -a                → show ARP table (MAC to IP)
```

---

## Linux vs Windows Equivalents

| Task | Linux | Windows |
|------|-------|---------|
| Show IP | `ip addr` | `ipconfig` |
| Show connections | `ss -tuln` | `netstat -an` |
| DNS lookup | `nslookup` / `dig` | `nslookup` |
| Ping | `ping` | `ping` |
| Trace route | `traceroute` | `tracert` |
| Show processes | `ps aux` | Task Manager |
| Remote login | `ssh user@ip` | `ssh user@ip` |

---

## Key Takeaways — Day 6

- Encapsulation adds headers going down, decapsulation removes them going up
- Every header field is a potential attack surface
- DNS caching speeds up browsing — cache poisoning exploits this
- Windows Event ID 1102 proves log clearing happened — you can't hide it
- Netstat is the first tool to run when investigating a Windows machine
- SSH works the same on Windows and Linux

---

## ✅ Homework

- [ ] Complete the Windows Event Log challenge
- [ ] Read: https://gist.github.com/githubfoam/69eee155e4edafb2e679fb6ac5ea47d0
- [ ] nmap
