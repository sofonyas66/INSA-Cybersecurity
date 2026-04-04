# Day 07 — Tshark, C2 & Nmap Deep Dive

**Date:** Saturday April 4, 2026

---

## Morning — Wireshark Revisited & Tshark

### Wireshark's Downside
In the last session we used Wireshark for packet analysis. Today we
revisited its limitations:

```
Wireshark problems:
→ GUI overhead — takes seconds to load and apply filters
→ Slow on large capture files (100MB+ pcap files lag badly)
→ Not scriptable — cannot pipe output into other tools
→ Requires a desktop environment — useless on a headless server
→ Filters must be typed in a GUI box — no automation possible
```

### Tshark — The Terminal Alternative

Tshark is the command-line version of Wireshark. Same engine,
no GUI. Runs fast, scriptable, pipeable, and works over SSH
on remote servers.

```bash
# Basic capture
tshark -i eth0

# Read a pcap file
tshark -r capture.pcap

# Apply display filter (same syntax as Wireshark)
tshark -r capture.pcap -Y "http"
tshark -r capture.pcap -Y "dns"
tshark -r capture.pcap -Y "tcp.port == 80"

# Show specific fields only
tshark -r capture.pcap -T fields -e ip.src -e ip.dst -e tcp.dstport

# Count packets matching a filter
tshark -r capture.pcap -Y "http" | wc -l

# Extract only HTTP hosts
tshark -r capture.pcap -Y "http.request" -T fields -e http.host
```

### Power of Pipes with Tshark

The real advantage — combining tshark with Linux tools:

```bash
# Find top talkers (most active IPs)
tshark -r capture.pcap -T fields -e ip.src \
  | sort | uniq -c | sort -rn | head -10

# Extract all DNS queries
tshark -r capture.pcap -Y "dns.flags.response == 0" \
  -T fields -e dns.qry.name | sort -u

# Find all HTTP GET requests
tshark -r capture.pcap -Y "http.request.method == GET" \
  -T fields -e http.host -e http.request.uri

# Extract credentials from FTP (cleartext)
tshark -r capture.pcap -Y "ftp" \
  -T fields -e ftp.request.command -e ftp.request.arg

# Save filtered output to new pcap
tshark -r big_capture.pcap -Y "ip.addr == 192.168.1.5" \
  -w filtered.pcap
```

### Tshark vs Wireshark — When to Use Which

```
Wireshark → visual analysis, follow streams, decode protocols
Tshark    → automation, scripting, large files, remote servers,
            piping into grep/awk/sort, CTF challenges, reports
```

---

## C2 — Command and Control (Brief Introduction)

C2 (Command and Control) is the infrastructure an attacker uses
to communicate with and control compromised machines (called
"bots" or "agents").

```
Attacker Machine
      |
      | (encrypted channel — HTTP, DNS, ICMP, etc.)
      |
   C2 Server
      |
   ┌──┴──┐
Agent  Agent  Agent   (compromised machines)
```

### How C2 Works (High Level)

```
1. Attacker compromises a machine and installs an agent/implant
2. The agent connects OUT to the C2 server (bypasses firewall)
3. Attacker sends commands to C2 server
4. C2 server relays commands to agents
5. Agents execute commands and send results back
```

### Why C2 Uses Outbound Connections

Firewalls typically block inbound connections but allow outbound.
By having the agent initiate the connection to the C2 server,
the traffic looks like normal browsing. This is called a
"beacon" or "call home" pattern.

```
Common C2 channels used to blend in:
→ HTTPS (port 443) — looks like web browsing
→ DNS  — queries disguised as domain lookups
→ ICMP — hidden in ping payloads (like Challenge 5!)
```

### C2 Frameworks (Research These)

```
Metasploit    → most well known, open source
Cobalt Strike → commercial, used by red teams and APTs
Empire        → PowerShell based
Sliver        → modern open source alternative
Havoc         → newer framework
```

*Self-study task: Research how C2 frameworks work, how Blue
Teams detect them, and what "beaconing" looks like in Wireshark.*

---

## Afternoon — Nmap Deep Dive

---

## Port States — The Technical Difference

Every port Nmap probes ends up in one of these states.
Understanding the exact packet exchange that produces each
state is essential.

### Open
A service is actively listening and accepts connections.
Evidence: SYN-ACK response received.

```
Nmap  -->  SYN       -->  Target
Nmap  <--  SYN-ACK   <--  Target    ← OPEN
Nmap  -->  RST       -->  Target    (Nmap resets cleanly)
```

### Closed
Host is reachable but no service on that port.
The OS responds with RST-ACK.

```
Nmap  -->  SYN       -->  Target
Nmap  <--  RST-ACK   <--  Target    ← CLOSED
```

Closed ports are still useful — they confirm the host is alive.

### Filtered
Something between Nmap and the target is blocking the probe.
Nmap gets silence or an ICMP "admin prohibited" message.

```
Nmap  -->  SYN  -->  [FIREWALL DROPS]    ← FILTERED (silence)

OR

Nmap  -->  SYN  -->  Target
Nmap  <--  ICMP type 3 code 13           ← FILTERED (admin prohibited)
```

### Open|Filtered
Nmap cannot tell if the port is open or filtered.
Common in UDP scans where open ports often give no response.

### Unfiltered
Port is accessible but Nmap cannot determine open or closed.
Appears in ACK scans.

---

## ICMP & Host Discovery — How Nmap Decides to Scan a Host

Before scanning ports Nmap sends discovery probes to confirm
the host is alive. It uses ICMP for this.

```
Default discovery probes Nmap sends:
1. ICMP Echo Request  (ping)        → Type 8
2. TCP SYN to port 443
3. TCP ACK to port 80
4. ICMP Timestamp Request           → Type 13

If ANY of these get a response → host is UP → scan begins
If ALL are silent → host assumed DOWN → skipped
```

### ICMP Message Types Nmap Uses

```
Type 0   Echo Reply        → host responded to ping = alive
Type 3   Dest Unreachable  → port is closed or filtered
  Code 0   Net unreachable
  Code 1   Host unreachable
  Code 3   Port unreachable  → closed UDP port
  Code 13  Admin prohibited  → firewall blocking
Type 8   Echo Request      → Nmap sends this (ping)
Type 11  Time Exceeded      → used by traceroute
Type 13  Timestamp Request  → OS fingerprinting
```

### When ICMP Is Blocked

Many firewalls block ICMP entirely. When this happens
Nmap thinks the host is down and skips it. Fix with -Pn:

```bash
sudo nmap -Pn 192.168.1.1          # skip discovery, assume host is up
sudo nmap -PE -PP -PM 192.168.1.1  # try all ICMP probe types
```

---

## Firewall Bypassing — Fragmentation & Decoys

### Packet Fragmentation (-f)

Splits the TCP header across multiple tiny IP fragments.
Many firewalls inspect individual fragments without reassembling
them — they never see the complete TCP header and miss the scan.

```bash
sudo nmap -f 192.168.1.1           # 8-byte fragments
sudo nmap -ff 192.168.1.1          # 16-byte fragments
sudo nmap --mtu 24 192.168.1.1     # custom size (multiple of 8)
```

### Decoy Scans (-D)

Nmap sends probes appearing to come from multiple IPs at once.
The target sees your real IP mixed with fake decoy IPs.
Blue Team cannot identify the real source easily.

```bash
sudo nmap -D 10.0.0.1,10.0.0.2,ME 192.168.1.1
# ME = your real IP among the decoys

sudo nmap -D RND:10 192.168.1.1
# 10 automatically generated random decoy IPs
```

### Source Port Manipulation

Some firewalls trust traffic from certain ports (like DNS=53).
Pretend your scan traffic is coming from that port:

```bash
sudo nmap --source-port 53 192.168.1.1
sudo nmap -g 80 192.168.1.1       # -g is alias
```

### Combined Evasion

```bash
sudo nmap -sS -T1 -f -D RND:5 --source-port 53 192.168.1.1
# SYN scan + slow timing + fragmented + 5 decoys + DNS port spoof
```

---

## MTU & Fragmentation — Breaking Packets to Slip Past IDS

MTU (Maximum Transmission Unit) is the largest packet size
a network link can carry. Standard Ethernet MTU is 1500 bytes.

### Why MTU Matters for Evasion

IDS/IPS systems use Deep Packet Inspection (DPI) — they
reassemble and analyse complete packets. If you fragment
packets below the minimum size the IDS expects to inspect,
it may skip reassembly and let the fragments pass.

```bash
# Custom MTU — must be multiple of 8
sudo nmap --mtu 8 192.168.1.1     # extremely small fragments
sudo nmap --mtu 16 192.168.1.1
sudo nmap --mtu 32 192.168.1.1
```

### How It Works Technically

```
Normal packet:
[IP Header 20 bytes][TCP Header 20 bytes][Data]
                     ↑ IDS inspects this

Fragmented (--mtu 8):
Fragment 1: [IP Header][TCP bytes 1-8]
Fragment 2: [IP Header][TCP bytes 9-16]
Fragment 3: [IP Header][TCP bytes 17-20]

IDS sees three separate tiny fragments
May not reassemble → TCP flags invisible → scan passes
```

---

## Passive to Active — Transitioning Recon Phases

### Passive Reconnaissance (No contact with target)

Gather information without sending a single packet to the target.

```
Tools:
  whois domain.com       → registration info, IP ranges
  dig domain.com         → DNS records (A, MX, NS, TXT)
  theHarvester           → emails, subdomains, employee names
  Shodan.io              → internet-exposed ports and services
  Censys.io              → certificate and port data
  crt.sh                 → SSL certificate history → subdomains
  Google Dorks           → site:target.com filetype:pdf
  Wayback Machine        → old site versions, leaked data
  LinkedIn               → tech stack, employee names
```

### smap — Passive Nmap Alternative

smap uses Shodan's database to simulate port scan results
without sending any packets to the target. Completely invisible.

```bash
smap 192.168.1.1
# Returns open ports and services using Shodan data
# Target never sees a single packet from you
```

### Shodan.io

Shodan is a search engine for internet-connected devices.
It continuously scans the entire internet and indexes
open ports, service banners, and vulnerabilities.

```
Shodan searches:
  port:22 country:ET         → SSH servers in Ethiopia
  apache 2.4.50              → servers running vulnerable Apache
  org:"Ethio Telecom"        → assets owned by Ethio Telecom
  vuln:CVE-2021-44228        → Log4Shell vulnerable servers
  default password           → devices with default creds
```

### Active Reconnaissance (Direct contact with target)

Only after passive recon gives you a clear target picture:

```bash
# Step 1: Fast sweep — find live hosts
sudo nmap -sn 192.168.1.0/24

# Step 2: Quick scan — find open ports
sudo nmap -sS --open -T4 target

# Step 3: Deep scan — versions and OS
sudo nmap -sV -O -p- target

# Step 4: Vulnerability check
sudo nmap --script=vuln target
```

---

## Idle Scanning — Using a Zombie Host

An Idle Scan (also called a Zombie Scan) lets you scan a target
without sending a single packet from your own IP. You use a
"zombie" — a third idle host on the internet — to do the
scanning for you.

### How It Works

```
You  -->  sends spoofed SYN (pretending to be Zombie)  -->  Target
Target  -->  SYN-ACK  -->  Zombie  (if port is open)
Zombie's IP ID counter increments

You  -->  probe Zombie's IP ID  -->  Zombie
If IP ID increased → target port was open
If IP ID unchanged → target port was closed
```

### Why It's Powerful

Your IP never touches the target. The target's logs show only
the Zombie's IP. You are completely invisible.

```bash
# First find a good zombie (idle host with predictable IP ID)
sudo nmap --script=ipidseq 203.0.113.5

# Then perform the idle scan
sudo nmap -sI zombie_ip:port target_ip
sudo nmap -sI 203.0.113.5:80 192.168.1.1
```

### Requirements for a Good Zombie

The zombie must be truly idle (no active traffic) and must
use a predictable, incrementing IP ID sequence. Many modern
OS use random IP IDs, making them useless as zombies.

---

## SCTP & Exotic Protocol Scanning

### What is SCTP?

SCTP (Stream Control Transmission Protocol) is a transport
protocol like TCP and UDP but designed for telecom signalling.
Used in VoIP infrastructure, 4G/5G networks, and SS7.

```bash
# SCTP INIT scan (like SYN scan for SCTP)
sudo nmap -sY 192.168.1.1

# SCTP COOKIE-ECHO scan
sudo nmap -sZ 192.168.1.1
```

### Other Protocol Scans

```bash
# IP Protocol scan — find which IP protocols are supported
sudo nmap -sO 192.168.1.1
# Discovers if target supports TCP, UDP, ICMP, IGMP, etc.

# ACK scan — map firewall rules
sudo nmap -sA 192.168.1.1
# Unfiltered = RST received (packet reached host)
# Filtered   = no response or ICMP error

# Window scan — like ACK but checks TCP window field
sudo nmap -sW 192.168.1.1

# Maimon scan — FIN/ACK probe
sudo nmap -sM 192.168.1.1
```

---

## Output Mastery — Converting to XML and JSON

### Output Formats

```bash
nmap -oN output.txt      # normal (human readable)
nmap -oX output.xml      # XML (machine parseable)
nmap -oG output.gnmap    # grepable
nmap -oA basename        # all three at once
```

### Working with XML Output

```bash
# Generate XML
sudo nmap -sV -oX scan.xml 192.168.1.0/24

# Parse with grep (quick and dirty)
grep "portid" scan.xml | grep "state=\"open\""

# Convert XML to HTML report
xsltproc /usr/share/nmap/nmap.xsl scan.xml > report.html

# Import into Metasploit
msf> db_import scan.xml
```

### Converting to JSON

```bash
# Using python
python3 -c "
import xml.etree.ElementTree as ET, json
tree = ET.parse('scan.xml')
# parse and convert to dict
print('converted')
"

# Using nmap-parse-output tool
pip install nmap-parse-output
nmap-parse-output scan.xml json > scan.json
```

### Grepable Output Power

```bash
# Find all open port 22
grep "22/open" scan.gnmap

# Extract just IPs with port 80 open
grep "80/open" scan.gnmap | awk '{print $2}'

# Count hosts with specific port open
grep "443/open" scan.gnmap | wc -l
```

---

## Other Notable Tools Mentioned

### smap
Uses Shodan data to return port information passively.
No packets sent to target. Good for initial passive recon.

### Shodan.io
Internet-wide scanner and search engine. Indexes open ports,
service banners, device types, and vulnerabilities across
the entire internet. Free tier available, powerful paid API.

---

## Key Takeaways — Week 4 Saturday

- Tshark = Wireshark without the GUI — faster, scriptable, pipeable
- C2 infrastructure uses outbound connections to bypass firewalls
- Port states (open/closed/filtered) are determined by specific packet responses
- ICMP is how Nmap discovers hosts before scanning ports
- Fragmentation and decoys are the primary firewall bypass techniques
- Passive recon (Shodan, smap) leaves zero trace on the target
- Idle/Zombie scanning hides your IP entirely during active scanning
- Always save Nmap output in XML for reporting and tool integration

---

## Assignments

- [ ] **Build your own Nmap-like tool** — Python script that takes an IP
      and outputs: open ports, service banners, basic OS guess
- [ ] **Research C2** — how frameworks work, how beaconing looks in
      Wireshark, how Blue Teams detect C2 traffic
- [ ] Push completed work to GitHub via PR
