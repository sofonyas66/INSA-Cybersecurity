# Day 9 — Theory Exam Reflection

**Date:** Saturday April 18, 2026

> Topics I found tricky and what I learned from them

---

## 1. Subnetting — Usable Hosts

This was the part that caught me off guard. The formula is simple but easy to mess up under pressure.

**Formula:**
```
Usable hosts = 2^(32 - prefix) - 2
```

> We subtract 2 because one address is the **network address** and one is the **broadcast address**.

**Practice table I had to memorize:**

| CIDR | Total Addresses | Usable Hosts |
|------|----------------|--------------|
| /8   | 16,777,216     | 16,777,214   |
| /16  | 65,536         | 65,534       |
| /24  | 256            | 254          |
| /26  | 64             | 62           |
| /28  | 16             | 14           |
| /30  | 4              | 2            |

**How to calculate fast:**
- /26 → 32 - 26 = 6 → 2^6 = 64 → 64 - 2 = **62**
- /28 → 32 - 28 = 4 → 2^4 = 16 → 16 - 2 = **14**
- /30 → 32 - 30 = 2 → 2^2 = 4  → 4 - 2  = **2**

> ⚠️ /30 only gives 2 usable hosts — used for point-to-point links

---

## 2. Nmap Commands I Mixed Up

### Decoy Scan
Hides your real IP by sending scan packets from multiple fake (decoy) IPs.
```bash
nmap -D RND:10 <TARGET>        # 10 random decoy IPs
nmap -D 192.168.1.1,ME <TARGET> # specific decoy + your real IP labeled ME
```
> The target sees traffic from all those IPs and can't tell which is real.

### Firewall Evasion + Service Info
```bash
nmap -sS -sV -f --mtu 8 -T2 <TARGET>
```
| Flag | What it does |
|------|-------------|
| `-sS` | SYN stealth scan |
| `-sV` | Detect service versions |
| `-f` | Fragment packets (harder for firewall to inspect) |
| `--mtu 8` | Set custom packet size (evades DPI) |
| `-T2` | Slow timing (less noisy) |

### Bypass ICMP blocking (host treated as down)
```bash
nmap -Pn <TARGET>              # skip ping, assume host is up
nmap -PS80,443 <TARGET>        # TCP SYN ping on port 80/443 instead
nmap -PA <TARGET>              # TCP ACK ping
```

### Full stealth scan (all ports)
```bash
nmap -sS -T4 -p- <TARGET>
```

### Everything at once
```bash
nmap -A -Pn -T4 <TARGET> -oN output.txt
```

---

## 3. Vim — Execute System Command

This one I blanked on during the exam.

| Action | Command |
|--------|---------|
| Enter insert mode | `i` |
| Exit insert mode | `Esc` |
| Save and quit | `:wq` |
| Quit without saving | `:q!` |
| **Execute system command** | `:!command` |

**Examples:**
```vim
:!ls              " list files without leaving vim
:!whoami          " run whoami
:!bash            " spawn a shell (used in privilege escalation!)
```

> 💡 The `:!` prefix is how you run ANY shell command from inside vim. This is also a well-known **privilege escalation technique** — if vim can run as root via sudo, you can do `sudo vim` → `:!bash` → root shell.

---

## Key Takeaways

- Subnetting: always use `2^(32-prefix) - 2` — don't forget the `-2`
- Nmap decoys: `-D RND:10` or `-D ip1,ip2,ME`
- Firewall evasion: `-f` fragments packets, `--mtu` sets packet size, `-Pn` skips ping
- Vim system command: `:!command` — also a privesc vector worth knowing