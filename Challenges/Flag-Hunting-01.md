# Challenge Write-Up: Payload Flag Hunting #01

**Date:** March 8, 2026  
**Program:** INSA Cyber Talent Weekend Program  
**Track:** Cybersecurity  
**Difficulty:** Beginner — Intermediate  
**Flags Found:** 2/3

---

## 📋 Challenge Description

Mentors provided a **payload** (executable file). The task was to:

1. Run the payload
2. Observe its behavior
3. Find **3 hidden flags** that the payload dropped across the Linux system

No hints were given on where the flags were hidden.

---

## 🧠 Methodology

Before running any unknown file, I made a mental checklist:

```bash
# Check what type of file it is
file payload

# Look for readable text before running
strings payload

# Check permissions — can we even run it?
ls -la payload

# If needed, give execute permission
chmod +x payload
```

Then run it:
```bash
./payload
```

After execution, my approach was to search the most likely attacker-used directories first.

---

## 🚩 Flag 1 — `/tmp` Directory

**Status:** ✅ Found  
**Tool Used:** `ls -la`, `cat`  
**Environment:** Termux (phone) — no laptop available

### Reasoning
`/tmp` is the first place an attacker drops files because:
- It is **world-writable** — any user can write here
- Files in `/tmp` often blend in and get ignored
- The **Sticky Bit** (`t`) prevents other users from deleting each other's files

### Steps
```bash
# After running the payload, check /tmp immediately
ls -la /tmp

# Output included a suspicious file:
# -rw-r--r-- 1 root root 42 Mar 08 admin....txt

cat /tmp/admin....txt
```

### Result
```
FLAG: INSA{[redacted]}
```

### Lesson Learned
After executing any unknown binary, check `/tmp` first. Time-sort with `ls -lt /tmp` to see what was most recently created.

---

## 🚩 Flag 2 — `/var/log` Directory

**Status:** ✅ Found  
**Found by:** Teammate  
**Tool Used:** `ls -lt /var/log`, `grep`

### Reasoning
Malware often creates fake log files in `/var/log` to blend in with legitimate system logs. It's a more advanced hiding spot than `/tmp` because defenders spend time here and might overlook a new file that "looks like a log."

### Steps
```bash
# Sort by time to find most recently modified file
ls -lt /var/log | head -20

# Or search directly for flag content
grep -r "flag" /var/log
grep -r "INSA{" /var/log
```

### Result
```
FLAG: INSA{[redacted]}
```

### Lesson Learned
Sort logs by modification time (`ls -lt`) during incident response. An attacker's file will be among the newest entries.

---

## 🚩 Flag 3 — Unknown Location

**Status:** ⏳ In Progress

### Possible Locations to Investigate

Based on the pattern (going deeper with each flag), Flag 3 is likely hiding in one of these places:

**1. Environment Variables**
```bash
printenv | grep -i flag
env | grep -i flag
```

**2. Hidden Dotfiles in Home Directory**
```bash
ls -la ~
ls -la /home/user/
cat ~/.flag 2>/dev/null
```

**3. Full Filesystem Search**
```bash
find / -name "*flag*" 2>/dev/null
find / -name "*.txt" -newer /tmp/admin....txt 2>/dev/null
```

**4. Inside the Payload Binary Itself**
```bash
strings payload | grep -i flag
strings payload | grep "INSA{"
```

**5. Encoded/Obfuscated (Base64)**
```bash
strings payload | grep "==$"
# If found, decode:
echo "base64string==" | base64 -d
```

**6. Process or Memory**
```bash
ps aux | grep -i flag
cat /proc/[PID]/environ 2>/dev/null
```

---

## 📊 Summary

| Flag | Location | Method | Status |
|------|----------|--------|--------|
| Flag 1 | `/tmp/admin....txt` | `ls -la /tmp` → `cat` | ✅ Found |
| Flag 2 | `/var/log/` | `ls -lt` → `grep -r` | ✅ Found |
| Flag 3 | Unknown | Under investigation | ⏳ |

---

## 💡 Tools Used

| Tool | Purpose |
|------|---------|
| `ls -la` | List all files including hidden |
| `ls -lt` | Sort files by time — find recent drops |
| `cat` | Read file contents |
| `strings` | Extract readable text from binary |
| `grep -r` | Recursive search inside files |
| `find` | Search filesystem for files |
| `chmod +x` | Give execute permission |
| `printenv` | View environment variables |

---

## 🧩 Personal Note

This was my first real flag-hunting exercise. I solved Flag 1 using **Termux on my phone** before the class session ended — no laptop, just a phone and terminal knowledge. That experience made it clear that the tools matter less than understanding the system.

> *"Know the system — the flags will reveal themselves."*
