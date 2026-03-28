# Day 05 — Packet Analysis & Wireshark

**Date:** March 29, 2026
**Track:** Cybersecurity
**Session:** Full Day

---

## Morning — Psychology Session

Topics discussed:
- What is **Talent** — a natural ability you are born with
- What is **Skill** — something developed through practice
- What is **Purpose** — the reason behind what you do

Key idea: Talent without skill is wasted. Skill without purpose has no direction.

*Homework: Reflect on your own talent, skill, and purpose for next class.*

---

## Afternoon — Introduction to Packet Analysis

---

## What is a Packet?

When data is sent over a network it is not sent all at once.
It is broken into small chunks called **packets**.

```
Large data
    ↓
[Packet 1][Packet 2][Packet 3][Packet 4]
    ↓
Travel across network independently
    ↓
Reassembled at destination
```

Why packets?
```
→ Easier to manage
→ If one packet is lost, only that one is resent
→ Multiple packets can travel different paths
→ More efficient use of network bandwidth
→ Error checking per packet
```

Structure of a packet:
```
┌─────────────┬──────────────┬─────────────┐
│   Header    │   Payload    │   Trailer   │
│ (source IP, │  (actual     │  (error     │
│  dest IP,   │   data)      │   check)    │
│  port...)   │              │             │
└─────────────┴──────────────┴─────────────┘
```

---

## DNS — Domain Name System

DNS translates human-readable domain names into IP addresses.

```
You type: claude.ai
    ↓
DNS resolves: claude.ai → 104.26.10.78
    ↓
Your browser connects to that IP
```

Why DNS?
```
Humans remember names → claude.ai
Computers need numbers → 104.26.10.78
DNS is the translator between the two
```

How DNS resolution works:
```
1. You type google.com
2. Browser checks local cache
3. Asks your Router (DNS resolver)
4. Router asks ISP DNS server
5. ISP asks Root DNS server
6. Root refers to .com DNS server
7. .com refers to Google's DNS server
8. Google's DNS returns IP address
9. Browser connects to that IP
```

Common DNS record types:
```
A     → domain to IPv4 address
AAAA  → domain to IPv6 address
MX    → mail server for domain
CNAME → alias (one name points to another)
TXT   → text info (used for verification)
PTR   → reverse lookup (IP to domain)
```

DNS uses:
```
Port 53 UDP → fast queries
Port 53 TCP → large responses, zone transfers
```

DNS attacks:
```
DNS Spoofing    → fake DNS response, redirect to wrong IP
Cache Poisoning → corrupt DNS cache with fake entries
DNS Amplification → DDoS using DNS servers
DNS Hijacking   → redirect DNS queries to attacker server
```

---

## TCP vs UDP — Quick Revision

```
TCP                          UDP
─────────────────────────────────────────
Connection oriented          Connectionless
Reliable delivery            No guarantee
Error checking               No error checking
Slower                       Faster
Web, email, file transfer    DNS, video, gaming
```

---

## The 3-Way Handshake

How TCP establishes a connection:

```
Client          Server
  │                │
  │──── SYN ──────▶│  "I want to connect"
  │                │
  │◀── SYN-ACK ───│  "Ok, I'm ready"
  │                │
  │──── ACK ──────▶│  "Great, let's go"
  │                │
  [Connection established]
```

```
SYN     → Synchronize — client starts connection
SYN-ACK → Server acknowledges and responds
ACK     → Client confirms — connection open
```

Attack on handshake:
```
SYN Flood → attacker sends thousands of SYN packets
            never sends final ACK
            server waits for each connection
            server runs out of resources → DoS
```

---

## Wireshark — Packet Analyzer

Wireshark is a tool that captures and analyzes network traffic in real time.

What you can see:
```
→ Every packet going in and out of your machine
→ Source and destination IP
→ Protocol used (TCP, UDP, DNS, HTTP...)
→ Port numbers
→ Packet content (if not encrypted)
→ Timing between packets
```

Basic filters:
```
ip.addr == 192.168.1.1     → filter by IP
tcp.port == 80              → filter by port
dns                         → show only DNS traffic
http                        → show only HTTP traffic
tcp                         → show only TCP traffic
udp                         → show only UDP traffic
ip.src == 192.168.1.5       → filter by source IP
ip.dst == 8.8.8.8           → filter by destination IP
```

Combining filters:
```
ip.addr == 192.168.1.1 && tcp.port == 80
dns || http
!(arp)                      → exclude ARP packets
```

Why Wireshark matters in security:
```
Blue Team → monitor traffic for suspicious activity
Red Team  → capture unencrypted credentials
           analyze how applications communicate
           find open ports and services
```

---

## Key Takeaways — Day 5

- Packets make network communication manageable and efficient
- DNS is the phone book of the internet — names to IPs
- The 3-way handshake is how every TCP connection starts
- Wireshark lets you see exactly what is happening on the network
- Unencrypted traffic (HTTP, FTP, Telnet) can be read directly in Wireshark

---

## ✅ Homework

- [ ] Learn Wireshark by yourself — filters, capture, analyze
- [ ] Complete the Wireshark challenge
- [ ] Reflect on talent, skill, and purpose for next class
