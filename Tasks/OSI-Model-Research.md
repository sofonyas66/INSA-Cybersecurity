# OSI Model — Research Document

**Date:** March 22, 2026  

---

## What is the OSI Model?

OSI stands for Open Systems Interconnection. It is a conceptual 
framework that standardizes how different network systems 
communicate with each other. Developed by ISO in 1984, it divides 
network communication into 7 layers, each with a specific role.

The goal is simple — allow different hardware and software systems 
to communicate regardless of their internal structure.

---

## The 7 Layers

### Layer 1 — Physical
The lowest layer. Deals with raw bits transmitted over a physical 
medium.

- Transmits: Bits (0s and 1s)
- Medium: Cables, fiber optic, radio waves
- Devices: Hubs, repeaters, network cables
- Protocols: Ethernet (physical), USB, Bluetooth

**Real example:**  
When you plug an ethernet cable into your laptop, the Physical 
layer is responsible for converting data into electrical signals 
and sending them through the cable.

**Security concern:**  
Physical access to cables = physical access to data. 
Server rooms must be locked.

---

### Layer 2 — Data Link
Responsible for node-to-node communication on the same network. 
Uses MAC addresses to identify devices.

- Transmits: Frames
- Addressing: MAC address (hardware address)
- Devices: Switches, network cards (NIC)
- Protocols: Ethernet, WiFi (802.11), ARP

**Real example:**  
Your laptop sends data to your router. The switch on your 
network reads the MAC address on the frame and forwards it 
to the correct device.

**Security concern:**  
MAC spoofing — attacker fakes their MAC address to bypass 
MAC filtering. ARP poisoning — attacker links their MAC to 
a legitimate IP to intercept traffic.

---

### Layer 3 — Network
Handles logical addressing and routing between different networks.

- Transmits: Packets
- Addressing: IP address (logical address)
- Devices: Routers
- Protocols: IP, ICMP, OSPF, BGP, RIP

**Real example:**  
You send a request from Addis Ababa to a server in the USA. 
Routers at Layer 3 read the destination IP address and decide 
the best path to forward the packet across multiple networks.

**Security concern:**  
IP spoofing — attacker fakes source IP. ICMP attacks — ping 
flood (DoS). Route hijacking — attacker redirects traffic.

---

### Layer 4 — Transport
Ensures complete data transfer. Breaks data into segments and 
reassembles them at the destination.

- Transmits: Segments
- Protocols: TCP, UDP
- Ports live here

**TCP vs UDP:**
```
TCP — Transmission Control Protocol
    → Connection oriented (3-way handshake)
    → Reliable, ordered delivery
    → Error checking
    → Slower
    → Use: web browsing, email, file transfer

UDP — User Datagram Protocol
    → Connectionless
    → No guarantee of delivery
    → No error checking
    → Faster
    → Use: video streaming, gaming, DNS, VoIP
```

**3-Way Handshake (TCP):**
```
Client → SYN     → Server
Client ← SYN-ACK ← Server
Client → ACK     → Server
Connection established
```

**Security concern:**  
SYN flood attack — attacker sends many SYN requests without 
completing handshake, exhausting server resources (DoS).
Port scanning — attacker probes open ports to find services.

---

### Layer 5 — Session
Manages sessions (connections) between applications. Opens, 
maintains, and closes communication sessions.

- Transmits: Data
- Protocols: NetBIOS, RPC, PPTP, SIP

**Real example:**  
When you log into a website and browse multiple pages, the 
Session layer keeps track of your login state so you don't 
have to log in again for every page.

**Security concern:**  
Session hijacking — attacker steals your session token to 
take over your logged-in session without needing your password.

---

### Layer 6 — Presentation
Translates data between the application and the network. 
Handles encryption, decryption, and compression.

- Transmits: Data
- Protocols: SSL/TLS, JPEG, MPEG, ASCII, UTF-8

**Real example:**  
When you visit an HTTPS website, the Presentation layer 
encrypts your data using TLS before sending it. The server 
decrypts it on the other end. Also converts file formats 
like JPEG images and MP4 videos.

**Security concern:**  
Weak encryption — using outdated SSL instead of TLS 1.3 
leaves data vulnerable to decryption attacks.

---

### Layer 7 — Application
The layer closest to the user. Provides network services 
directly to applications.

- Transmits: Data
- Protocols: HTTP, HTTPS, FTP, SSH, DNS, SMTP, POP3

**Real example:**  
You open a browser and type google.com. The Application 
layer handles the HTTP/HTTPS request your browser sends 
to Google's server.

**Common ports:**
```
HTTP  → 80
HTTPS → 443
FTP   → 21
SSH   → 22
DNS   → 53
SMTP  → 25
POP3  → 110
```

**Security concern:**  
SQL injection, XSS, phishing — most attacks target 
Layer 7 because it's directly exposed to users.
WAF (Web Application Firewall) defends at this layer.

---

## Data Flow — How Layers Work Together

**Sending data (top to bottom — Encapsulation):**
```
Application  → creates data
Presentation → encrypts data
Session      → adds session info
Transport    → breaks into segments, adds port
Network      → adds IP address (packet)
Data Link    → adds MAC address (frame)
Physical     → converts to bits, sends
```

**Receiving data (bottom to top — Decapsulation):**
```
Physical     → receives bits
Data Link    → reads MAC, removes frame header
Network      → reads IP, removes packet header
Transport    → reassembles segments
Session      → maintains session
Presentation → decrypts data
Application  → user sees the data
```

**Real world — opening google.com:**
```
1. You type google.com (Layer 7 — HTTP request)
2. TLS encrypts request (Layer 6)
3. Session maintained (Layer 5)
4. TCP breaks into segments (Layer 4)
5. IP address added — your IP to Google IP (Layer 3)
6. MAC address added (Layer 2)
7. Bits sent over cable/wifi (Layer 1)
         ↓ travels across internet ↓
8. Google receives bits (Layer 1)
9. Reads up through layers
10. Google's browser engine processes request (Layer 7)
11. Response travels back same way
```

---

## OSI vs TCP/IP Model

```
OSI Model          TCP/IP Model
─────────────────────────────────
Application   ┐
Presentation  ├──→ Application
Session       ┘
Transport     ───→ Transport
Network       ───→ Internet
Data Link     ┐
Physical      ┘──→ Network Access
```

TCP/IP is the model actually used on the internet today. 
OSI is the reference model used for teaching and 
troubleshooting.

---

## Security at Each Layer

```
Layer 7 Application  → WAF, IDS/IPS, antivirus
Layer 6 Presentation → TLS/SSL, certificates
Layer 5 Session      → Session timeouts, tokens
Layer 4 Transport    → Firewalls, port filtering
Layer 3 Network      → Router ACLs, IPSec, VPN
Layer 2 Data Link    → MAC filtering, VLANs, 802.1X
Layer 1 Physical     → Locked server rooms, cameras
```

---

## Memory Tricks

**Top to bottom:**
```
All People Seem To Need Data Processing
A  P       S      T  N      D    P
```

**Bottom to top:**
```
Please Do Not Throw Sausage Pizza Away
P      D  N   T     S       P     A
```

---

## Summary Table

| Layer | Name         | Unit    | Devices        | Protocols          |
|-------|--------------|---------|----------------|--------------------|
| 7     | Application  | Data    | —              | HTTP, FTP, SSH, DNS|
| 6     | Presentation | Data    | —              | TLS, JPEG, ASCII   |
| 5     | Session      | Data    | —              | NetBIOS, RPC       |
| 4     | Transport    | Segment | —              | TCP, UDP           |
| 3     | Network      | Packet  | Router         | IP, ICMP, OSPF     |
| 2     | Data Link    | Frame   | Switch, NIC    | Ethernet, ARP      |
| 1     | Physical     | Bit     | Hub, Cable     | Ethernet, USB      |
```

---