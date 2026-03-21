# Day 03 — Linux Power Tools & Privilege Escalation

**Date:** TBD  
**Track:** Cybersecurity  
**Session:** Full Day

---

## 🔁 Morning Revision

- Reviewed Linux directory hierarchy
- Reviewed file permissions and `chmod`
- Discussed how attacker and defender logic maps to directory structure

---

## 🔐 Privilege Escalation — Concept Overview

**Privilege Escalation** (PrivEsc) is the process of gaining higher permissions than you were originally granted.

### Two Types

| Type | Description |
|------|-------------|
| **Horizontal** | Accessing another user's account at the same level |
| **Vertical** | Escalating from a regular user to `root` |

### Why It Matters
- Red Team: PrivEsc is the goal after initial access
- Blue Team: Preventing PrivEsc is a critical hardening task

### Common PrivEsc Vectors
```bash
# Check what you can run as sudo
sudo -l

# Find SUID binaries (run as root even if you're not)
find / -perm -4000 -type f 2>/dev/null

# Look for world-writable scripts run by root
find / -writable -type f 2>/dev/null

# Check for cron jobs
cat /etc/crontab
ls -la /etc/cron.*
```

---

## 🛠️ Power Tools

### A. `sudo` — Superuser Do

```bash
sudo command              # Run a command as root
sudo -l                   # List what you're allowed to run as sudo
sudo su                   # Switch to root shell
```

**Best Practice:** Never work directly as `root`. Use `sudo` only when needed to avoid accidental system damage.

**Security Note:** If `sudo -l` shows `(ALL) NOPASSWD: ALL`, that's a serious misconfiguration — any command can be run as root without a password.

---

### B. `grep` — Global Regular Expression Print

`grep` searches for patterns inside files. One of the most important tools for both defense (log analysis) and offense (finding sensitive data).

```bash
grep "pattern" filename           # Basic search
grep -i "pattern" filename        # Case-insensitive
grep -r "pattern" /directory      # Recursive (search entire folder)
grep -v "pattern" filename        # Invert match (show lines without pattern)
grep -n "pattern" filename        # Show line numbers
grep -l "pattern" /directory      # Show only filenames that match

# Combine with pipe for filtering:
ls /etc | grep net                # Find files in /etc containing "net"
ps aux | grep ssh                 # Find SSH-related processes
cat /var/log/syslog | grep error  # Find errors in system log
```

**CTF Tip:**
```bash
strings binary_file | grep -i flag    # Find flags in a binary
grep -r "INSA{" /                     # Search entire system for flag format
```

---

### C. `find` — Locate Files by Attribute

While `grep` searches **inside** files, `find` searches **for** files by their properties.

```bash
find /path -name "filename"           # Find by name
find /path -name "*.txt"              # Find all .txt files
find /path -type f                    # Find files only (not directories)
find /path -type d                    # Find directories only
find /path -perm 777                  # Find files with specific permissions
find /path -perm -4000                # Find SUID files
find /path -user root                 # Find files owned by root
find /path -mmin -10                  # Modified in last 10 minutes
find /path -size +10M                 # Files larger than 10MB
find / -name "flag*" 2>/dev/null      # Search whole system, suppress errors
```

**Combining find with exec:**
```bash
find / -name "*.log" -exec cat {} \;  # Read every .log file found
```

---

### D. `ps` and `top` — Process Monitoring

```bash
ps aux                     # Snapshot of all running processes
ps aux | grep apache        # Find specific process
top                        # Real-time CPU and RAM usage
htop                       # Improved version of top (if installed)
kill -9 PID                # Force kill a process by its ID
```

**Columns in `ps aux`:**
| Column | Meaning |
|--------|---------|
| USER | Who is running it |
| PID | Process ID |
| %CPU | CPU usage |
| %MEM | Memory usage |
| COMMAND | The actual command |

**Security Use:**  
After running a suspicious file, immediately check `ps aux` to see if it spawned any background processes.

---

### E. `strings` — Extract Human-Readable Text from Binaries

```bash
strings filename               # Print readable strings from any file
strings filename | grep flag   # Filter for specific patterns
strings filename | less        # Browse output page by page
```

**Why it matters:**  
Binaries can contain passwords, flags, URLs, or command strings that are invisible if you just `cat` the file.

---

## 📝 Text Editors

### Nano — Easy and Accessible
```bash
nano filename.txt    # Open or create file

# Inside nano:
# Ctrl + O  →  Save (Write Out)
# Ctrl + X  →  Exit
# Ctrl + K  →  Cut line
# Ctrl + U  →  Paste line
```

### Vim — Powerful but Has a Learning Curve
```bash
vim filename.txt     # Open or create file

# Vim has two modes:
# Normal Mode  →  Navigate, delete, copy
# Insert Mode  →  Type text

# Key commands:
# i          →  Enter Insert Mode
# Esc        →  Return to Normal Mode
# :w         →  Save
# :q         →  Quit
# :wq        →  Save and Quit
# :q!        →  Force quit without saving
# dd         →  Delete current line
# yy         →  Copy current line
# p          →  Paste
```

### VS Code — For Project-Level Work
Best used for larger codebases, scripts, and when working with a GUI. Not available in a pure terminal environment.

---

## 🔄 The `strings` Command — Deep Dive

`strings` is one of the first tools used in **binary analysis** and **CTF challenges**.

```bash
# Basic usage
strings suspicious_binary

# With line numbers
strings -n 8 binary    # Only show strings >= 8 characters

# Search for encoded data
strings binary | grep "==$"    # Base64 strings often end with ==

# Decode Base64 after finding it
echo "ZmxhZ3t0ZXN0fQ==" | base64 -d
```

---

## 📝 Key Takeaways — Day 3

- `sudo` is powerful — always audit what users can run with `sudo -l`
- `grep` is your search engine for file contents; `find` is your search engine for the filesystem
- SUID binaries are a common PrivEsc path — always run `find / -perm -4000` during enumeration
- `strings` is often the fastest way to extract flags from unknown files
- Knowing your text editor (especially Vim) in a headless environment is non-negotiable

---

## ✅ Homework

- [ ] Find a SUID binary on your Kali install and research if it's exploitable
- [ ] Use `grep -r` to search `/etc` for any file containing the word "password"
- [ ] Practice Vim — open a file, edit it, save and quit without using nano
- [ ] Push this week's notes and challenge write-ups to GitHub
