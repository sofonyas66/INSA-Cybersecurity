# Challenge Write-Up: Payload Flag Hunting #01

**Date:** March 8, 2026  
**Program:** INSA Cyber Talent Weekend Program  
**Track:** Cybersecurity  
**Difficulty:** Beginner — Intermediate  
**Flags Found:** 2/3

---

## 📋 Challenge Overview

**What is Flag Hunting?**  
Flag hunting is a cybersecurity exercise where you execute a potentially malicious binary and locate hidden flags (text files containing sensitive information) that it drops across the Linux filesystem. This mimics real-world **incident response**, where analysts must find artifacts left behind by attackers.

**Challenge Objective:**
1. Execute a mysterious payload (executable file)
2. Observe its behavior without harming the system
3. Find **3 hidden flags** dispersed across different directories
4. Understand attacker tradecraft and common hiding spots

This exercise teaches defensive skills: recognizing where attackers hide data and how to systematically search a compromised system.

---

## 🧠 Pre-Execution Safety Checklist

Before running any unknown binary, always verify what you're running:

```bash
# Step 1: Identify the file type
file payload
# Example output: ELF 64-bit LSB executable, x86-64 architecture

# Step 2: Extract readable text from the binary (before execution!)
strings payload
# Shows any human-readable strings embedded in the file
# Look for clues, hints, or suspicious behavior

# Step 3: Check current permissions
ls -la payload
# Output: -rw-r--r-- 1 user user 12345 Mar 08 payload

# Step 4: Grant execute permission if needed
chmod +x payload
# Changes from rw-r--r-- to rwxr-xr-x (adds execute permission)

# Step 5: Execute the payload
./payload
# Run it and observe what it does
```

**Why These Steps Matter:**
- `file`: Confirms it's actually an executable (not a trick)
- `strings`: Shows embedded text without running potentially dangerous code
- `chmod +x`: Makes the file executable (required on Unix systems)
- Observation: Note any output, errors, or system changes

---

## 🚩 Flag 1 — `/tmp` Directory

**Status:** ✅ Found  
**Tool Used:** `ls -la`, `cat`  
**Environment:** Termux (phone) — no laptop available  
**Difficulty:** Easy

### Why `/tmp`?

Attackers use `/tmp` for several reasons:
- **World-writable**: Any user can create, read, or modify files here (permissions: `drwxrwxrwt`)
- **Temporary nature**: Files are often cleaned up, reducing detection time
- **Low suspicion**: Defenders expect `/tmp` to contain legitimate temporary data
- **Sticky Bit protection**: The `t` permission prevents users from deleting each other's files, so the flag won't be accidentally removed

### Steps to Find Flag 1

```bash
# Execute the payload first (from previous step)
./payload

# Immediately list /tmp with all files (including permissions)
ls -la /tmp
# Look for newly created files with suspicious names
# Example suspicious file: admin....txt (created on Mar 08 22:15)

# Read the file
cat /tmp/admin....txt
```

### Result

```
FLAG: INSA{[redacted]}
```

### Key Lesson

**Always check `/tmp` immediately after running an unknown binary.** Time-sort with `ls -lt /tmp` to see what was most recently created. This is the first place any analyst should look during incident response.

```bash
# Pro tip: Sort by modification time to spot recent additions
ls -lt /tmp | head -10  # Shows 10 most recently modified files
```

---

## 🚩 Flag 2 — `/var/log` Directory

**Status:** ✅ Found  
**Found by:** Teammate collaboration  
**Tool Used:** `ls -lt /var/log`, `grep -r`  
**Difficulty:** Medium

### Why `/var/log`?

This is a more sophisticated hiding spot than `/tmp`:
- **Expected location**: Defenders regularly monitor logs, so a fake log "blends in"
- **High-volume data**: Legitimate logs are constantly being written, making one extra file less noticeable
- **Disguise**: If the attacker names their file like a real log (`syslog`, `auth.log`, `daemon.log`), it looks legitimate
- **Persistence**: Logs are often archived, not deleted immediately

### Steps to Find Flag 2

```bash
# List all files in /var/log sorted by modification time (newest first)
ls -lt /var/log | head -20
# The attacker's file will appear among the most recently modified

# Alternative: Search directly for flag format
grep -r "INSA{" /var/log
# Recursively searches all files in /var/log for the flag pattern

# Or search for the word "flag"
grep -r "flag" /var/log
```

### Result

```
FLAG: INSA{[redacted]}
```

### Key Lesson

**In real incident response, always sort logs by modification time (`ls -lt`).** An attacker's newly created file will be recent, while legitimate logs have consistent timestamps. This is how you spot anomalies quickly.

---

## 🚩 Flag 3 — Unknown Location (In Progress)

**Status:** ⏳ Still investigating  
**Difficulty:** Hard

Based on the pattern of escalating difficulty (Flag 1 in a writable directory, Flag 2 in a monitored directory), Flag 3 is likely hidden in a more obscure or indirect location. Here are the most probable hiding spots:

### Potential Location 1: Environment Variables

Attackers can store data in environment variables that persist after execution:

```bash
# View all environment variables
printenv | grep -i flag
# or
env | grep -i flag

# Check a specific user's environment
echo $FLAG
```

### Potential Location 2: Hidden Dotfiles in Home Directory

Filenames starting with `.` are hidden by default (won't show in `ls` without `-a`):

```bash
# List all files including hidden ones
ls -la ~
ls -la /home/user/

# Check common hidden locations
cat ~/.flag 2>/dev/null
cat ~/.bashrc  # Attackers sometimes append commands here
cat ~/.bash_history
```

### Potential Location 3: Full Filesystem Search

Brute-force search for anything with "flag" in the name:

```bash
# Search the entire filesystem for files with "flag" in the name
find / -name "*flag*" 2>/dev/null
# 2>/dev/null redirects error messages to avoid clutter

# Search for files modified after Flag 1 was created
find / -name "*.txt" -newer /tmp/admin....txt 2>/dev/null

# Search by content instead of filename
find / -type f -exec grep -l "INSA{" {} \; 2>/dev/null
```

### Potential Location 4: Embedded in the Binary Itself

The flag might be encoded within the payload binary:

```bash
# Extract all readable strings from the binary
strings payload | grep -i flag
strings payload | grep "INSA{"

# Check for base64-encoded content
strings payload | grep "==$"  # Base64 ends with == or =
# If found, decode:
echo "base64string==" | base64 -d
```

### Potential Location 5: Process Memory or Runtime Data

The payload might write the flag to memory or a process-specific location:

```bash
# Check running processes
ps aux | grep payload

# View a process's environment variables
cat /proc/[PID]/environ 2>/dev/null
# (Replace [PID] with the actual process ID)

# Check memory maps
cat /proc/[PID]/maps
```

### Potential Location 6: Cron Jobs or Scheduled Tasks

The payload might register itself to run later:

```bash
# Check user cron jobs
crontab -l
cat /etc/cron.d/*
cat /var/spool/cron/crontabs/*

# Check systemd timers
systemctl list-timers
```

### Potential Location 7: Kernel or System Logs (dmesg)

Some payloads interact with the kernel:

```bash
# Check kernel ring buffer
dmesg | grep -i flag

# Check system log
cat /var/log/syslog | grep -i flag
```

---

## 📊 Progress Summary

| Flag | Location | Method | Status | Difficulty |
|------|----------|--------|--------|------------|
| Flag 1 | `/tmp/admin....txt` | `ls -la /tmp` → `cat` | ✅ Found | Easy |
| Flag 2 | `/var/log/` | `ls -lt` → `grep -r` | ✅ Found | Medium |
| Flag 3 | Unknown | Under investigation | ⏳ | Hard |

---

## 💡 Tools Reference

| Tool | Purpose | Example |
|------|---------|---------|
| `file` | Identify file type | `file payload` |
| `strings` | Extract readable text from binary | `strings payload \| grep flag` |
| `ls -la` | List all files including hidden | `ls -la /tmp` |
| `ls -lt` | Sort files by modification time | `ls -lt /var/log` |
| `cat` | Display file contents | `cat /tmp/admin....txt` |
| `grep -r` | Recursively search for text | `grep -r "INSA{" /var/log` |
| `find` | Search filesystem by name or properties | `find / -name "*flag*"` |
| `chmod +x` | Add execute permission | `chmod +x payload` |
| `printenv` | View environment variables | `printenv \| grep flag` |
| `ps aux` | List running processes | `ps aux \| grep payload` |
| `hexdump -C` | View raw bytes (for encoded data) | `hexdump -C payload` |
| `binwalk` | Detect hidden files in binaries | `binwalk payload` |

---

## 🔍 Advanced Techniques for Future Challenges

These tools go beyond the basics for more complex flag-hunting scenarios:

```bash
# Detect and extract hidden files within a file (steganography)
binwalk payload
binwalk -e payload  # Extracts embedded data

# View raw bytes to spot encoding patterns
hexdump -C payload | head -50

# Trace system calls made by the payload (see what it does)
strace ./payload 2>&1 | grep -E "open|write|mmap"

# Check file metadata (timestamps, permissions, etc.)
stat payload

# Search for specific patterns with regex
grep -r "^[A-Za-z0-9/+=]\{50,\}$" /tmp  # Looks for base64
```

---

## 📚 Key Cybersecurity Concepts

### 1. **Attacker Tradecraft**
Attackers follow predictable patterns:
- Use world-writable directories (`/tmp`)
- Disguise artifacts as legitimate files (`/var/log`)
- Hide data in less-obvious locations as detection increases
- Clean up after themselves to avoid forensic discovery

### 2. **Incident Response Methodology**
- **Triage**: Check high-risk, easy-to-access directories first
- **Deep dive**: Move to more obscure locations if initial search fails
- **Timeline analysis**: Sort by modification time to spot anomalies
- **Comprehensive search**: Use multiple tools (grep, find, strings) to cross-verify

### 3. **Linux Filesystem Knowledge**
Understanding the Linux filesystem layout is critical:
- `/tmp` — Temporary files (world-writable)
- `/var/log` — System and application logs
- `/home` — User home directories
- `/root` — Root user's home directory
- `/etc` — Configuration files
- `/dev` — Device files

---

## 🧩 Personal Reflection

This was my first real flag-hunting exercise. I solved Flag 1 using **Termux on my phone** before the class session ended — no laptop, just a terminal and foundational knowledge. This taught me that **tools don't matter as much as understanding the system**. Whether you use a phone or a high-end workstation, knowing where attackers hide and how systems work is what makes you effective.

> *"Know the system — the flags will reveal themselves."*

---

## 🎯 Next Steps

1. **Find Flag 3** using the investigation strategies above
2. **Document findings** with the same methodology (reasoning, steps, result)
3. **Reflect on lessons learned** for future incident response work
4. **Practice** similar challenges to build muscle memory for forensic investigation