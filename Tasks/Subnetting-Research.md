# Subnetting & Subnet Mask — Research


**Name:** Sofonyas Yared  
**Date:** March 22, 2026

---

## What is a Subnet Mask?

A 32-bit number that divides an IP address into
Network part and Host part.

Every device on a network needs:
- An IP address
- A subnet mask

Without a subnet mask the device doesn't know
which part of the IP is the network and which
part identifies the device.

---

## How it Works — AND Operation

To find the network address, AND the IP with the mask:

```
IP:   192.168.1.25
      11000000.10101000.00000001.00011001

Mask: 255.255.255.0
      11111111.11111111.11111111.00000000

AND:  11000000.10101000.00000001.00000000
    = 192.168.1.0  ← Network Address
```

Rule:
```
1 AND 1 = 1
1 AND 0 = 0
0 AND 0 = 0
```

---

## Default Subnet Masks by Class

| Class | Range | Default Mask | CIDR |
|-------|-------|--------------|------|
| A | 1-126 | 255.0.0.0 | /8 |
| B | 128-191 | 255.255.0.0 | /16 |
| C | 192-223 | 255.255.255.0 | /24 |

---

## CIDR Notation

Instead of writing the full mask we use /number
The number = count of 1 bits in the mask

```
255.255.255.0
11111111.11111111.11111111.00000000
└──────────24 ones──────────┘
= /24
```

---

## What is Subnetting?

Splitting one large network into smaller networks.

Why subnet?
```
→ Better organization
→ Reduce broadcast traffic
→ Improve security
→ Efficient use of IP addresses
→ Isolate departments/teams
```

---

## The Subnetting Formula

```
Subnets = 2^s
Hosts   = 2^h - 2

s = bits borrowed from host part
h = remaining host bits
-2 = subtract network and broadcast address
```

---

## Subnetting Example — 192.168.1.0/24

Original: /24 → 8 host bits → 254 hosts → 1 network

**Borrow 2 bits → /26:**
```
Subnets = 2^2 = 4
Hosts   = 2^6 - 2 = 62 per subnet
```

| Subnet | Network | First Host | Last Host | Broadcast |
|--------|---------|------------|-----------|-----------|
| 1 | 192.168.1.0 | 192.168.1.1 | 192.168.1.62 | 192.168.1.63 |
| 2 | 192.168.1.64 | 192.168.1.65 | 192.168.1.126 | 192.168.1.127 |
| 3 | 192.168.1.128 | 192.168.1.129 | 192.168.1.190 | 192.168.1.191 |
| 4 | 192.168.1.192 | 192.168.1.193 | 192.168.1.254 | 192.168.1.255 |

---

## CIDR Cheat Sheet

| CIDR | Subnet Mask | Hosts | Subnets from /24 |
|------|-------------|-------|-----------------|
| /24 | 255.255.255.0 | 254 | 1 |
| /25 | 255.255.255.128 | 126 | 2 |
| /26 | 255.255.255.192 | 62 | 4 |
| /27 | 255.255.255.224 | 30 | 8 |
| /28 | 255.255.255.240 | 14 | 16 |
| /29 | 255.255.255.248 | 6 | 32 |
| /30 | 255.255.255.252 | 2 | 64 |
| /32 | 255.255.255.255 | 1 | single host |

---

## Special Addresses in Every Subnet

```
First address → Network address (not usable)
Last address  → Broadcast address (not usable)
Everything in between → Usable hosts

Example 192.168.1.0/24:
192.168.1.0   → Network
192.168.1.1   → First usable host
192.168.1.254 → Last usable host
192.168.1.255 → Broadcast
```

---

## Real World Example — INSA Network

```
INSA has network: 10.0.0.0/8
They subnet it per department:

Cyber team   → 10.1.0.0/24  (254 hosts)
Dev team     → 10.2.0.0/24  (254 hosts)
Admin        → 10.3.0.0/24  (254 hosts)
Servers      → 10.4.0.0/24  (254 hosts)

Result:
→ Each team is isolated
→ Cyber team cannot directly reach Dev team
→ Security through network segmentation
→ Easier to monitor traffic per department
```

---

## Security Use of Subnetting

```
DMZ subnet     → public facing servers /28
LAN subnet     → internal users /24
Server subnet  → isolated /26
Guest WiFi     → isolated /25

Attacker on Guest WiFi
cannot reach Server subnet
even in the same building
```

---

## Step by Step — How to Subnet

Given: 192.168.10.0/24, need 4 subnets

**Step 1:** How many bits to borrow?
```
2^s >= 4
2^2 = 4 ✓
Borrow 2 bits
```

**Step 2:** New prefix
```
/24 + 2 = /26
```

**Step 3:** New mask
```
/26 = 11111111.11111111.11111111.11000000
    = 255.255.255.192
```

**Step 4:** Hosts per subnet
```
2^6 - 2 = 62 hosts
```

**Step 5:** Subnet increment
```
256 - 192 = 64
Subnets jump by 64
```

**Step 6:** List subnets
```
192.168.10.0   - 192.168.10.63
192.168.10.64  - 192.168.10.127
192.168.10.128 - 192.168.10.191
192.168.10.192 - 192.168.10.255
```

---

## Useful Commands

```bash
# See your IP and subnet
ip addr

# Calculate subnet details
ipcalc 192.168.1.0/24

# Install ipcalc
sudo apt install ipcalc    # Debian/Ubuntu/Kali
sudo pacman -S ipcalc      # Arch
```
```

---

When you're on PC:

```bash
vim Tasks/Subnetting-Research.md
# paste content
git add .
git commit -m "Add subnetting research"
git push
```
---