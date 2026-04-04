# Nmap — Deep Dive Research

**Name:** Sofonyas Yared
**Date:** April 4, 2026

---

## Table of Contents

1. [What is Nmap and How Does It Actually Work?](#lesson-1)
2. [TCP Connect Scan (-sT) — The Loud Knock](#lesson-2)
3. [SYN Stealth Scan (-sS) — The Half-Open Ghost](#lesson-3)
4. [UDP Scan (-sU) — Talking to the Deaf](#lesson-4)
5. [OS Detection (-O) — Reading Fingerprints](#lesson-5)
6. [Service and Version Detection (-sV)](#lesson-6)
7. [Aggressive Scan (-A) — The All-In-One](#lesson-7)
8. [Timing Templates (-T0 to -T5)](#lesson-8)
9. [Xmas, FIN and NULL Scans — The Stealth Trio](#lesson-9)
10. [NSE — Nmap Scripting Engine](#lesson-10)
11. [Output Formats and Saving Results](#lesson-11)
12. [Target Specification and Port Ranges](#lesson-12)
13. [How Open, Closed and Filtered Ports Are Recognised](#lesson-13)
14. [ICMP — The Silent Messenger Nmap Exploits](#lesson-14)
15. [Firewall Evasion — Fragmentation, Decoys and Timing](#lesson-15)
16. [Active vs Passive Reconnaissance](#lesson-16)
17. [IDS and IPS Evasion — Staying Under the Radar](#lesson-17)
18. [Pivoting and Scanning Through Networks](#lesson-18)
19. [Nmap in a Full Penetration Test Workflow](#lesson-19)
20. [Nmap Cheat Sheet and Quick Reference](#lesson-20)

---

## Lesson 1 — What is Nmap and How Does It Actually Work?

### The Core Idea

Nmap (Network Mapper) is an open-source tool for network discovery and security auditing. It works by sending specially crafted packets to a target and analysing the responses. Every OS, every firewall, every service responds slightly differently to different kinds of packets. Nmap exploits those differences to build a complete picture of what is running on a network. Think of it as knocking on doors in different ways and listening carefully to what knocks back.

### Phase 1 — Host Discovery

Before Nmap probes a single port it must confirm the target is alive. By default it sends four different probes simultaneously: an ICMP echo request (ping), a TCP SYN to port 443, a TCP ACK to port 80, and an ICMP timestamp request. If any one of these gets a response the host is marked as "up". If all are silent Nmap skips it entirely. You override this with `-Pn` which tells Nmap "skip discovery, treat the host as alive no matter what."

```bash
sudo nmap 192.168.1.1          # discovers host first then scans
sudo nmap -Pn 192.168.1.1      # skip discovery, scan directly
```

### Phase 2 — Port Scanning

Once a host is confirmed alive, Nmap probes ports. By default it scans the 1,000 most common ports (not 1-1000, but a curated list of the most likely to be open). For each port it sends a packet, waits for a response, classifies the port, and moves on. The classification is: open, closed, or filtered.

### Why Root Privileges Matter

Many scan types require root because they send raw packets — packets constructed manually, bypassing the OS TCP/IP stack. Without root, Nmap falls back to `-sT` (connect scan) which uses the OS socket system. With root, Nmap can set any TCP flags, spoof source IPs, and fragment packets arbitrarily. This is the difference between a tourist and a locksmith.

```bash
sudo nmap -sS 192.168.1.1     # requires root
nmap -sT 192.168.1.1          # no root needed, less stealthy
```

### What Travels on the Wire

When you run `nmap 192.168.1.1` your machine constructs a raw IP packet containing a TCP segment with the SYN flag set, addressed to a port. That packet travels through your NIC, your router, across the network, and arrives at the target's interface. The target OS decides: is this port open? Is there a firewall rule? It responds (or doesn't). Nmap captures and interprets that response.

---

## Lesson 2 — TCP Connect Scan (-sT) — The Loud Knock

### What It Is

The TCP Connect Scan is the most basic and the loudest Nmap scan. It uses the operating system's own `connect()` system call to establish a full three-way handshake with every target port. It does not require root privileges because it uses normal sockets, but that convenience comes at a heavy cost: it is easily detected and leaves complete records in the target's logs.

```bash
nmap -sT 192.168.1.1
```

### The Full Three-Way Handshake

For every port probed, the following exchange happens completely:

```
Client  -->  SYN          -->  Server   (I want to connect)
Client  <--  SYN-ACK      <--  Server   (OK, I am ready)
Client  -->  ACK          -->  Server   (Connection established)
Client  -->  RST          -->  Server   (Nmap closes immediately)
```

### What Gets Logged on the Target

Because the handshake completes, the target OS and any application on that port sees a real connection. Web servers, SSH daemons, and firewalls all log the source IP, the timestamp, and the port. A Blue Team analyst reviewing logs will see hundreds of connection attempts from the same IP in rapid succession — an immediate red flag.

```bash
# What appears in /var/log/apache2/access.log on target:
192.168.1.5 - - [28/Mar/2026:09:00:01] "GET / HTTP/1.0" 200
192.168.1.5 - - [28/Mar/2026:09:00:01] "-" 400
```

### When to Use -sT

Use `-sT` when you do not have root privileges. Also use it when scanning through a SOCKS proxy or Tor, because raw packets cannot be forwarded through proxies but OS-level connections can.

```bash
nmap -sT -p 80,443,22 192.168.1.1
nmap -sT --proxies socks4://127.0.0.1:9050 192.168.1.1
```

---

## Lesson 3 — SYN Stealth Scan (-sS) — The Half-Open Ghost

### What Makes It Stealthy

The SYN scan — also called a half-open or stealth scan — never completes the three-way handshake. After receiving a SYN-ACK from an open port, Nmap immediately sends a RST (reset) packet instead of ACK. This tears down the connection before it fully forms. Because the connection never completes, many older logging systems and applications never record it. It is the default scan type when running as root.

```bash
sudo nmap -sS 192.168.1.1
```

### The Half-Open Exchange

```
# Open port:
Nmap   -->  SYN          -->  Target   (probing)
Nmap   <--  SYN-ACK      <--  Target   (port is open!)
Nmap   -->  RST          -->  Target   (goodbye)

# Closed port:
Nmap   -->  SYN          -->  Target
Nmap   <--  RST-ACK      <--  Target   (nothing here)
```

### Why It Still Gets Detected Today

Modern firewalls and IDS systems like Snort and Suricata are specifically tuned to detect half-open scans. They look for a high volume of SYN packets from one source with no corresponding ACK completions. The -sS is stealthy against application logs but not against dedicated network monitoring.

### Requires Root — Why

Sending a RST immediately after SYN-ACK is not something the OS does naturally. Nmap must construct and send that RST as a raw packet, bypassing the normal TCP stack. Root access is required to intercept the incoming SYN-ACK and send a custom RST before the OS responds.

```bash
sudo nmap -sS -p- 192.168.1.1       # all 65535 ports
sudo nmap -sS --open 192.168.1.1    # show only open ports
```

---

## Lesson 4 — UDP Scan (-sU) — Talking to the Deaf

### Why UDP Is Different

UDP is connectionless. There is no handshake, no SYN-ACK, no confirmation of delivery. Nmap sends a UDP packet to a port and waits. If the port is closed the OS sends back an ICMP "port unreachable" message. If the port is open most services simply ignore the empty packet — so Nmap gets nothing back. Silence could mean open or filtered.

```bash
sudo nmap -sU 192.168.1.1
sudo nmap -sU -p 53,67,123,161 192.168.1.1
```

### Why UDP Scans Are Slow

Three reasons combine to make UDP scanning brutally slow. First, there is no handshake so Nmap must wait for the full timeout period. Second, operating systems rate-limit ICMP responses — Linux by default sends at most one ICMP unreachable per second, meaning 65535 ports would take 18 hours. Third, Nmap must retransmit packets because UDP delivery is not guaranteed.

```bash
sudo nmap -sU --top-ports 20 192.168.1.1   # scan 20 most common UDP ports
sudo nmap -sU -T4 192.168.1.1              # faster but less accurate
```

### Port States in UDP

```
open           --> service responded with data
open|filtered  --> no response (could be either)
closed         --> ICMP port unreachable received
filtered       --> ICMP "admin prohibited" received
```

### Combining TCP and UDP

```bash
sudo nmap -sS -sU -p T:80,443,22,U:53,161 192.168.1.1
```

---

## Lesson 5 — OS Detection (-O) — Reading Fingerprints

### The Principle

Every operating system implements the TCP/IP specification slightly differently. Linux chooses different default TTL values, window sizes, and responses to malformed packets than Windows does. Nmap sends a series of carefully designed probe packets and compares the responses against a database of known OS fingerprints.

```bash
sudo nmap -O 192.168.1.1
sudo nmap -O --osscan-guess 192.168.1.1   # guess even if uncertain
```

### What Nmap Actually Measures

```
TTL value              --> Linux=64, Windows=128, Cisco=255
TCP window size        --> varies widely by OS
IP ID sequence         --> how the OS increments packet IDs
TCP options order      --> which options and in what order
Response to bad flags  --> how the OS handles illegal flag combos
ICMP response rate     --> how quickly ICMP errors are generated
Don't Fragment bit     --> whether the OS sets DF in IP header
```

### Accuracy and Limitations

OS detection requires at least one open and one closed port to be reliable. If the target is behind a NAT device or a load balancer, the fingerprint belongs to that device, not the actual server. Firewalls that normalise packets can defeat fingerprinting entirely.

```bash
sudo nmap -O -v 192.168.1.1    # verbose OS detection details
```

---

## Lesson 6 — Service and Version Detection (-sV)

### Beyond Open Ports

Knowing port 80 is open tells you a web server might be there. Knowing it runs Apache 2.4.51 on Ubuntu 20.04 tells you exactly which CVEs to check. Version detection sends additional probes to open ports and matches responses against the `nmap-service-probes` database which contains thousands of signatures.

```bash
sudo nmap -sV 192.168.1.1
sudo nmap -sV --version-intensity 9 192.168.1.1   # most aggressive
```

### Example Output

```
22/tcp  open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.6
80/tcp  open  http     Apache httpd 2.4.52 ((Ubuntu))
3306/tcp open mysql    MySQL 8.0.32-0ubuntu0.22.04.2
```

### Why This Matters

Every version number is a potential CVE lookup. Once you know Apache 2.4.52 is running, you search the National Vulnerability Database for known vulnerabilities in that exact version.

```bash
sudo nmap -sV --script=banner 192.168.1.1   # grab raw banners
```

---

## Lesson 7 — Aggressive Scan (-A) — The All-In-One

### What -A Enables

The `-A` flag enables four things simultaneously: OS detection (`-O`), version detection (`-sV`), script scanning (`--script=default`), and traceroute (`--traceroute`). It is the most comprehensive single-flag scan and also the noisiest.

```bash
sudo nmap -A 192.168.1.1
sudo nmap -A -p- 192.168.1.1      # aggressive scan of all ports
```

### Example Output

```
22/tcp open  ssh     OpenSSH 8.9 (protocol 2.0)
| ssh-hostkey:
|   256 a0:b3:4f:... (ECDSA)
80/tcp open  http    Apache 2.4.52
|_http-title: Company Intranet - Login
OS: Linux 5.15 (96% confidence)
```

### When to Use It

Use `-A` when doing a thorough audit of a small number of hosts where stealth is not required. The combination of OS probes, version probes, and NSE scripts generates a very distinctive traffic pattern that any IDS will flag immediately.

---

## Lesson 8 — Timing Templates (-T0 to -T5)

### The Six Timing Levels

```bash
nmap -T0 192.168.1.1   # paranoid  - one probe per 5 minutes
nmap -T1 192.168.1.1   # sneaky    - one probe per 15 seconds
nmap -T2 192.168.1.1   # polite    - one probe per 0.4 seconds
nmap -T3 192.168.1.1   # normal    - default
nmap -T4 192.168.1.1   # aggressive - faster, assumes good network
nmap -T5 192.168.1.1   # insane    - very fast, may miss results
```

### IDS Evasion with Timing

Most IDS systems use a detection window — they count how many events occur from one source within a time period. T0 and T1 spread events so far apart that the IDS window resets between probes and never triggers the threshold.

```bash
sudo nmap -T1 -sS 192.168.1.1   # slow stealth scan
```

---

## Lesson 9 — Xmas, FIN and NULL Scans — The Stealth Trio

### The RFC 793 Trick

The original TCP specification (RFC 793) says: if a closed port receives a packet with no SYN flag set, it MUST respond with RST. If an open port receives such a packet, it should silently drop it. These three scans exploit that rule.

### FIN Scan (-sF)

Sends a packet with only the FIN flag set. Closed port replies RST. Open port ignores it.

```bash
sudo nmap -sF 192.168.1.1
```

### NULL Scan (-sN)

Sends a packet with NO flags set at all.

```bash
sudo nmap -sN 192.168.1.1
```

### Xmas Scan (-sX)

Sends a packet with FIN, PSH, and URG flags all set — lit up like a Christmas tree.

```bash
sudo nmap -sX 192.168.1.1

# Port state interpretation for all three:
# No response       --> open OR filtered
# RST received      --> closed
# ICMP unreachable  --> filtered
```

### Limitations

Windows, Cisco IOS, and many modern systems respond with RST to everything regardless of state, making these scans useless against them. They remain useful against Linux and Unix-like systems.

---

## Lesson 10 — NSE — Nmap Scripting Engine

### What NSE Is

The Nmap Scripting Engine allows users to write and run Lua scripts that extend Nmap's capabilities far beyond port scanning. Over 600 scripts ship with Nmap.

```bash
sudo nmap --script=default 192.168.1.1
sudo nmap --script=vuln 192.168.1.1
```

### Script Categories

```
auth       --> test authentication mechanisms
brute      --> brute force passwords on services
default    --> safe scripts run with -A or -sC
discovery  --> gather additional information
dos        --> test denial of service (dangerous)
exploit    --> actively exploit vulnerabilities
malware    --> detect backdoors and malware
safe       --> unlikely to crash target
vuln       --> check for known vulnerabilities
```

### Practical Examples

```bash
# Check for EternalBlue (MS17-010) vulnerability
sudo nmap --script=smb-vuln-ms17-010 192.168.1.1

# Enumerate HTTP directories
sudo nmap --script=http-enum 192.168.1.1

# Brute force SSH
sudo nmap --script=ssh-brute 192.168.1.1

# Get SSL certificate info
sudo nmap --script=ssl-cert -p 443 192.168.1.1

# Check for anonymous FTP
sudo nmap --script=ftp-anon -p 21 192.168.1.1
```

---

## Lesson 11 — Output Formats and Saving Results

### Output Formats

```bash
-oN file.txt     --> Normal output (human readable)
-oX file.xml     --> XML output (machine parseable)
-oG file.gnmap   --> Grepable output
-oA basename     --> All three formats simultaneously
```

### Practical Usage

```bash
sudo nmap -sV -oA scan_results 192.168.1.0/24
# Creates: scan_results.nmap, scan_results.xml, scan_results.gnmap

# Grep the grepable output for open ports
grep 'open' scan_results.gnmap

# Feed XML into Metasploit
msf> db_import scan_results.xml
```

---

## Lesson 12 — Target Specification and Port Ranges

### Specifying Targets

```bash
nmap 192.168.1.1                    # single host
nmap 192.168.1.1-50                 # range
nmap 192.168.1.0/24                 # entire subnet
nmap scanme.nmap.org                # hostname
nmap -iL targets.txt                # read from file
nmap --exclude 192.168.1.5 192.168.1.0/24  # exclude a host
```

### Port Ranges

```bash
nmap -p 80 192.168.1.1              # single port
nmap -p 80,443,22 192.168.1.1      # multiple ports
nmap -p 1-1000 192.168.1.1         # range
nmap -p- 192.168.1.1               # all 65535 ports
nmap -p U:53,T:80,443 192.168.1.1  # UDP and TCP together
nmap --top-ports 100 192.168.1.1   # top 100 most common
```

---

## Lesson 13 — How Open, Closed and Filtered Ports Are Recognised

### Open Port

A service is actively listening and accepts connections. Evidence: SYN-ACK response.

```
Nmap  -->  SYN      -->  Target
Nmap  <--  SYN-ACK  <--  Target   --> OPEN
Nmap  -->  RST      -->  Target   (Nmap resets)
```

### Closed Port

Host is reachable but no service on that port. The OS responds with RST-ACK.

```
Nmap  -->  SYN      -->  Target
Nmap  <--  RST-ACK  <--  Target   --> CLOSED
```

### Filtered Port

Something between Nmap and the target is blocking the probe. Nmap gets silence or ICMP "admin prohibited."

```
Nmap  -->  SYN  -->  [FIREWALL DROPS IT]   --> FILTERED

OR

Nmap  -->  SYN  -->  Target
Nmap  <--  ICMP type 3 code 13             --> FILTERED
```

### Open|Filtered

Nmap received no response and cannot determine whether the port is open or filtered. Common in UDP scans.

```bash
sudo nmap -sU -p 53 192.168.1.1   # likely shows open|filtered
```

---

## Lesson 14 — ICMP — The Silent Messenger Nmap Exploits

### What ICMP Is

ICMP (Internet Control Message Protocol) is a Layer 3 protocol used for network diagnostics and error reporting. It is not a data transport protocol — it carries control messages. Ping uses ICMP. Traceroute uses ICMP. Nmap uses ICMP extensively for host discovery and for interpreting filtered ports.

### ICMP Message Types Nmap Uses

```
Type 0   Echo Reply        --> response to ping (host is alive)
Type 3   Dest Unreachable  --> port closed or filtered
  Code 3   Port unreachable  --> closed UDP port
  Code 13  Admin prohibited  --> filtered by firewall
Type 8   Echo Request      --> Nmap sends this for host discovery
Type 13  Timestamp Request  --> OS fingerprinting
```

### How ICMP Reveals Firewall Behaviour

When Nmap gets ICMP type 3 code 3 it knows the port is closed. When it gets ICMP type 3 code 13 it knows a firewall is actively blocking. When it gets nothing at all the firewall is silently dropping packets — which is actually better security practice.

### When ICMP Is Blocked

```bash
sudo nmap -PE -PP -PM -sn 192.168.1.0/24   # all ICMP discovery types
sudo nmap -Pn 192.168.1.1                  # skip ICMP, assume up
```

---

## Lesson 15 — Firewall Evasion — Fragmentation, Decoys and Timing

### Packet Fragmentation (-f)

Splits the TCP header across multiple tiny IP fragments. Many firewalls inspect individual fragments without reassembling them and never see the complete TCP header.

```bash
sudo nmap -f 192.168.1.1           # 8-byte fragments
sudo nmap -ff 192.168.1.1          # 16-byte fragments
sudo nmap --mtu 24 192.168.1.1     # custom size (multiple of 8)
```

### Decoy Scans (-D)

Nmap sends probes appearing to come from multiple IPs at once. The target sees your real IP mixed with fake decoy IPs.

```bash
sudo nmap -D 10.0.0.1,10.0.0.2,ME 192.168.1.1
sudo nmap -D RND:10 192.168.1.1   # 10 random decoys
```

### Source Port Manipulation

Some firewalls trust traffic from port 53 (DNS). Pretend your scan traffic comes from there.

```bash
sudo nmap --source-port 53 192.168.1.1
sudo nmap -g 80 192.168.1.1
```

### IP Spoofing (-S)

```bash
sudo nmap -S spoofed_ip -e eth0 192.168.1.1
```

### Slow Timing for IDS Evasion

```bash
sudo nmap -T0 192.168.1.1              # one probe per 5 minutes
sudo nmap --scan-delay 30s 192.168.1.1
```

### Combined Evasion

```bash
sudo nmap -sS -T1 -f -D RND:5 --source-port 53 -oA results 192.168.1.1
```

---

## Lesson 16 — Active vs Passive Reconnaissance

### The Fundamental Difference

Passive recon gathers information without sending a single packet to the target. Active recon sends packets directly. The target can detect active recon. Passive recon is completely invisible.

### Passive Reconnaissance Tools

```
whois domain.com       --> registration info, IP ranges
dig domain.com         --> DNS records
theHarvester           --> emails, subdomains, employee names
Shodan.io              --> internet-exposed ports and services
Censys.io              --> certificate and port data
crt.sh                 --> SSL certificate history
Google Dorks           --> site:target.com filetype:pdf
Wayback Machine        --> old site versions, leaked data
LinkedIn               --> tech stack, employee names
```

### Active Reconnaissance Tools

```
nmap            --> port scanning
ping / fping    --> host discovery
traceroute      --> network path
nikto           --> web vulnerability scan
gobuster        --> directory brute force
netcat          --> banner grabbing
```

### Transitioning from Passive to Active

```
Passive phase output --> Nmap target list
  Shodan shows port 22 open on 203.0.113.5
  whois shows IP range 203.0.113.0/24

Active phase begins:
  sudo nmap -sV -p 22,80,443 203.0.113.0/24
```

---

## Lesson 17 — IDS and IPS Evasion

### IDS vs IPS

An IDS (Intrusion Detection System) monitors traffic and alerts but does not block. An IPS (Intrusion Prevention System) sits inline and actively blocks matching packets. Evading an IPS is harder because it can drop your packets in real time.

### Evasion Techniques Combined

```bash
sudo nmap -sS -T1 -f -D RND:5 --source-port 53 192.168.1.1

# -sS          --> SYN scan (no full connection logged)
# -T1          --> slow timing, evades rate-based detection
# -f           --> fragmented packets, evades DPI
# -D RND:5     --> 5 random decoys, hides your real IP
# --source-port 53  --> looks like DNS traffic
```

---

## Lesson 18 — Pivoting and Scanning Through Networks

### What Pivoting Is

Pivoting means using a compromised machine as a relay to scan networks not directly accessible from your attack machine. You compromise a web server in the DMZ — that server can reach the internal network but your laptop cannot. You pivot through it.

### Nmap Through a Proxy

```bash
# Create SSH SOCKS proxy through compromised host
ssh -D 1080 user@compromised_host

# Scan internal network through the proxy
nmap -sT --proxies socks4://127.0.0.1:1080 10.0.0.0/24
```

---

## Lesson 19 — Nmap in a Full Penetration Test Workflow

### The Pentest Phases

```
Phase 1  Planning       --> define scope
Phase 2  Reconnaissance --> passive OSINT
Phase 3  Scanning       --> Nmap  <-- HERE
Phase 4  Exploitation   --> use findings
Phase 5  Reporting      --> document everything
```

### A Real Nmap Workflow

```bash
# Step 1: Fast ping sweep to find live hosts
sudo nmap -sn 192.168.1.0/24 -oG live_hosts.gnmap

# Step 2: Quick port scan of live hosts
sudo nmap -sS --open -T4 -iL live_hosts.txt -oA quick_scan

# Step 3: Deep scan of interesting hosts
sudo nmap -sS -sV -O -p- -T3 10.0.0.5 -oA deep_scan

# Step 4: Run vuln scripts on findings
sudo nmap --script=vuln -p 80,443,445 10.0.0.5

# Step 5: Import to Metasploit
msfconsole -q -x 'db_import deep_scan.xml; hosts; services'
```

---

## Lesson 20 — Nmap Cheat Sheet and Quick Reference

### Essential Scan Types

```bash
nmap -sS target    # SYN stealth (default, root required)
nmap -sT target    # TCP connect (no root needed)
nmap -sU target    # UDP scan
nmap -sN target    # NULL scan
nmap -sF target    # FIN scan
nmap -sX target    # Xmas scan
nmap -sn target    # Ping sweep (no port scan)
nmap -sI zombie target  # Idle/zombie scan
```

### Discovery and Detection

```bash
nmap -O target              # OS detection
nmap -sV target             # Version detection
nmap -A target              # All: OS+version+scripts+traceroute
nmap --script=vuln target   # Vulnerability scan
nmap -sY target             # SCTP INIT scan
nmap -sO target             # IP protocol scan
```

### Port Specification

```bash
nmap -p 80 target           # single port
nmap -p 1-1000 target       # range
nmap -p- target             # all 65535 ports
nmap --top-ports 100 target # top 100
nmap -p U:53,T:80 target    # UDP + TCP combined
```

### Timing

```bash
nmap -T0  # paranoid (slowest, most stealthy)
nmap -T1  # sneaky
nmap -T2  # polite
nmap -T3  # normal (default)
nmap -T4  # aggressive
nmap -T5  # insane (fastest)
```

### Evasion

```bash
nmap -f target                 # fragment packets
nmap -D RND:10 target          # 10 random decoys
nmap --source-port 53 target   # spoof source port
nmap -S spoofed_ip target      # spoof source IP
nmap --scan-delay 5s target    # slow down between probes
nmap --mtu 16 target           # custom fragment size
```

### Output

```bash
nmap -oN file.txt    # normal human readable
nmap -oX file.xml    # XML for tools/Metasploit
nmap -oG file.gnmap  # grepable
nmap -oA basename    # all three at once
```

### Useful Combinations

```bash
# Full stealth scan
sudo nmap -sS -T2 -f -D RND:5 --source-port 53 -oA results target

# Complete audit
sudo nmap -A -p- -T4 -oA full_audit target

# Fast live host discovery
sudo nmap -sn 192.168.1.0/24

# Top UDP services
sudo nmap -sU --top-ports 20 target

# Vulnerability check on web server
sudo nmap --script=vuln -p 80,443 target
```

### Port State Quick Reference

| Response | State |
|----------|-------|
| SYN-ACK | Open |
| RST-ACK | Closed |
| No response | Filtered |
| ICMP type 3 code 13 | Filtered |
| No response (UDP) | Open\|Filtered |

### ICMP Types Quick Reference

| Type | Meaning |
|------|---------|
| 0 | Echo Reply (host alive) |
| 3 | Destination Unreachable |
| 3/3 | Port Unreachable (closed UDP) |
| 3/13 | Admin Prohibited (firewall) |
| 8 | Echo Request (ping) |
| 11 | Time Exceeded (traceroute) |
| 13 | Timestamp Request (OS fingerprint) |

### TTL Values by OS

| OS | Default TTL |
|----|-------------|
| Linux | 64 |
| Windows | 128 |
| Cisco | 255 |
| FreeBSD | 64 |
| macOS | 64 |
