# Day 04 — Networking Basics

**Date:** March 22, 2026
**Track:** Cybersecurity
**Session:** Full Day

---

## What is a Network?

A network is a collection of devices connected together to share
resources and communicate with each other.

Examples:
- Two computers connected by a cable
- All devices connected to your home WiFi
- The entire internet

---

## Types of Networks

```
LAN — Local Area Network
      Small area: home, office, school
      Fast, privately owned
      Example: your home WiFi

MAN — Metropolitan Area Network
      City-wide: connects multiple LANs
      Example: university campus, city WiFi

WAN — Wide Area Network
      Global: connects cities and countries
      Example: The internet is the biggest WAN
```

---

## What is an IP Address?

IP = Internet Protocol Address
Like a home address but for devices on a network.
Every device needs one to communicate.

```
IPv4 → 192.168.1.1     (32-bit, ~4 billion addresses)
IPv6 → 2001:db8::1     (128-bit, virtually unlimited)
```

---

## Two Types of IP

```
Private → only works inside your LAN
          Not routable on the internet
          192.168.x.x / 10.x.x.x / 172.16.x.x

Public  → works on the internet
          Assigned by your ISP
          Unique globally
```

---

## IP Address Classes

```
Class A → 1.0.0.0   to 126.255.255.255
          Large networks (governments, big corps)
          Subnet mask: 255.0.0.0 (/8)
          Networks:  126
          Hosts:     16,777,214

Class B → 128.0.0.0 to 191.255.255.255
          Medium networks (universities, ISPs)
          Subnet mask: 255.255.0.0 (/16)
          Networks:  16,384
          Hosts:     65,534

Class C → 192.0.0.0 to 223.255.255.255
          Small networks (home, small business)
          Subnet mask: 255.255.255.0 (/24)
          Networks:  2,097,152
          Hosts:     254

Class D → 224.0.0.0 to 239.255.255.255
          Multicast — one to many
          Not for regular hosts

Class E → 240.0.0.0 to 255.255.255.255
          Reserved for research only
```

**Private ranges per class:**
```
A → 10.0.0.0
B → 172.16.0.0
C → 192.168.0.0
```

---

## Decimal to Binary — The Formula

Each octet = 8 bits. Bit position values:
```
128  64  32  16  8  4  2  1
```

**Method 1 — Subtraction:**
```
Convert 192:
192 - 128 = 64  → 1
64  - 64  = 0   → 1
0   < 32        → 0
...             → 0 0 0 0 0
Result: 11000000
```

**Method 2 — Divide by 2, read remainders bottom to top:**
```
192 ÷ 2 = 96  r 0
96  ÷ 2 = 48  r 0
48  ÷ 2 = 24  r 0
24  ÷ 2 = 12  r 0
12  ÷ 2 = 6   r 0
6   ÷ 2 = 3   r 0
3   ÷ 2 = 1   r 1
1   ÷ 2 = 0   r 1
Read bottom to top → 11000000
```

**Memorize these:**
```
128 = 10000000
192 = 11000000
224 = 11100000
240 = 11110000
255 = 11111111
0   = 00000000
```

**Check in terminal:**
```bash
python3 -c "print(bin(192))"
```

---

## Network and Host

Every IP address has two parts:
```
Network part → identifies the network
Host part    → identifies the device
```

The subnet mask tells you which is which:
```
IP:   192.168.1.25
Mask: 255.255.255.0

255 → Network
0   → Host

Network → 192.168.1
Host    → 25
```

**Special addresses in every network:**
```
First address → Network address  (not usable)
Last address  → Broadcast address (not usable)
In between    → Usable hosts

Example 192.168.1.0/24:
Network   → 192.168.1.0
First host→ 192.168.1.1
Last host → 192.168.1.254
Broadcast → 192.168.1.255
Hosts     → 254
```

---

## Network and Host Bits by Class

```
Class A:
First bit always = 0
0xxxxxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx
|_8 bits_|______24 bits host________|
Network bits = 8
Host bits    = 24
Hosts = 2^24 - 2 = 16,777,214

Class B:
First 2 bits always = 10
10xxxxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx
|___16 bits net___|___16 bits host__|
Network bits = 16
Host bits    = 16
Hosts = 2^16 - 2 = 65,534

Class C:
First 3 bits always = 110
110xxxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx
|______24 bits network_____|8 host|
Network bits = 24
Host bits    = 8
Hosts = 2^8 - 2 = 254
```

**Formula:**
```
Hosts = 2^(host bits) - 2
-2 is always for network and broadcast address
```

---

## Special IP Addresses

```
0.0.0.0
→ Unknown/unspecified address
→ Used when device has no IP yet (DHCP request)

127.0.0.0 - 127.255.255.255
→ Loopback (localhost)
→ 127.0.0.1 most common
→ Traffic never leaves your machine
→ Test: ping 127.0.0.1

169.254.0.0 - 169.254.255.255
→ APIPA — assigned when DHCP fails
→ Can only communicate on local network
→ If you see this — your DHCP is broken

255.255.255.255
→ Limited broadcast
→ Sends to ALL devices on local network

Private ranges (not routable on internet):
10.0.0.0    - 10.255.255.255      Class A
172.16.0.0  - 172.31.255.255      Class B
192.168.0.0 - 192.168.255.255     Class C
```

---

## IPv6 — The Next Generation

```
Why IPv6?
IPv4 = 32 bits  → ~4 billion addresses (ran out)
IPv6 = 128 bits → 340 undecillion addresses
```

**Format:**
```
8 groups of 4 hex digits separated by colons
2001:0db8:0000:0000:0000:ff00:0042:8329
```

**Shortening rules:**
```
Rule 1 — Remove leading zeros
0db8 → db8

Rule 2 — Replace consecutive zero groups with ::
2001:0db8:0000:0000:0000:ff00:0042:8329
→ 2001:db8::ff00:42:8329
```

**Special IPv6 addresses:**
```
::1          → Loopback (like 127.0.0.1)
::           → Unspecified (like 0.0.0.0)
fe80::/10    → Link-local (like 169.254.x.x)
ff00::/8     → Multicast
```

**IPv4 vs IPv6:**
```
Feature    IPv4            IPv6
Bits       32              128
Format     Decimal dots    Hex colons
Addresses  ~4 billion      340 undecillion
Security   Optional        Built-in (IPSec)
NAT        Required        Not needed
```

---

## OSI Model

OSI = Open Systems Interconnection
7 layers that standardize how networks communicate.

```
Layer 7 — Application
→ What user sees and interacts with
→ Protocols: HTTP, HTTPS, FTP, SSH, DNS
→ Example: browser sending HTTP request

Layer 6 — Presentation
→ Encrypts, decrypts, compresses data
→ Protocols: TLS/SSL, JPEG, ASCII
→ Example: HTTPS encrypting your password

Layer 5 — Session
→ Opens, manages, closes connections
→ Example: keeping you logged in while browsing

Layer 4 — Transport
→ Breaks data into segments
→ Protocols: TCP (reliable), UDP (fast)
→ Example: TCP confirms every packet arrived

Layer 3 — Network
→ Logical addressing and routing
→ Protocols: IP, ICMP, OSPF
→ Example: routers finding path across internet

Layer 2 — Data Link
→ Physical addressing (MAC addresses)
→ Devices: Switches
→ Example: switch forwarding frame by MAC

Layer 1 — Physical
→ Raw bits over physical medium
→ Devices: Cables, hubs, WiFi radio
→ Example: electrical signals over ethernet cable
```

**Real world — Telegram message:**
```
You type "Hello" and send
L7 → Telegram app processes it
L6 → MTProto encrypts it
L5 → Session keeps connection open
L4 → TCP breaks into segments
L3 → Adds your IP and Telegram server IP
L2 → Adds MAC address
L1 → Radio waves carry bits to router
     → travels across internet →
L1 → Server receives signal
...decapsulates back up to L7...
L7 → Friend receives "Hello"
```

**Memory trick (top to bottom):**
```
All People Seem To Need Data Processing
A   P       S     T  N      D    P
```

---

## MAC Address

MAC = Media Access Control
Unique hardware address burned into every network card at factory.

```
Format: 00:1A:2B:3C:4D:5E  (48 bits)

00:1A:2B  →  3C:4D:5E
|_OUI___|     |Device ID|
Manufacturer   Unique ID
```

**MAC vs IP:**
```
MAC                    IP
Hardware address       Logical address
Layer 2                Layer 3
Never changes          Can change
Used inside LAN        Used across internet
48 bits                32 bits
```

**Important:**
```
MAC addresses never cross routers
Only IP addresses travel across networks
Every time packet crosses a router
MAC address is replaced
IP address stays the same
```

**Check your MAC:**
```bash
ip link show
# look for "ether"
```

**Security concerns:**
```
MAC Spoofing   → fake MAC to bypass filtering
MAC Flooding   → flood switch, force broadcast
ARP Poisoning  → link attacker MAC to victim IP
                 → Man in the Middle attack
```

---

## Ports

A number that identifies a specific service on a device.
```
IP address → finds the device
Port       → finds the service

Like apartment building:
IP = building address
Port = apartment number
```

**Port ranges:**
```
0     - 1023   → Well known (system)
1024  - 49151  → Registered (applications)
49152 - 65535  → Dynamic (temporary/client)
```

**Must know ports:**
```
22   → SSH      (secure remote login)
23   → Telnet   (unsecure — avoid)
25   → SMTP     (send email)
53   → DNS      (domain to IP)
80   → HTTP     (web)
443  → HTTPS    (secure web)
21   → FTP      (file transfer)
3306 → MySQL    (database)
3389 → RDP      (remote desktop)
```

**Check open ports:**
```bash
ss -tuln
nmap 192.168.1.1
nmap -p 22 192.168.1.1
```

**Security:**
```
Red Team  → scan for open ports (nmap)
Blue Team → close unnecessary ports (ufw)

Dangerous open ports:
23   → Telnet (plaintext)
445  → SMB (EternalBlue)
3389 → RDP (brute force)
4444 → Metasploit default
```

---

## Key Takeaways — Day 4

- A network connects devices to share resources
- IP address identifies a device, port identifies a service
- MAC address works inside LAN, IP works across internet
- OSI model has 7 layers — each handles a specific job
- Subnetting splits networks for security and organization
- IPv6 was created because IPv4 addresses ran out

---

## Homework / Research

- [ ] Research OSI model in detail — push to GitHub
- [ ] Practice binary conversion for all IP classes
- [ ] Study subnetting — practice with ipcalc
- [ ] Learn what happens at each OSI layer during an attack
EOF
