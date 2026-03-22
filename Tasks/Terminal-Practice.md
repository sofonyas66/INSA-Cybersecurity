# Assignments — Terminal Practice Log

**Track:** Cybersecurity  

---

## 📋 Assignment 1 — Linux Folder Structure Challenge

**Given by:** Mentor, Day 2  
**Task:** Recreate a given folder structure using **one command**

### The Structure
```
lab/
├── red_team/
│   ├── logs/
│   └── scripts/
└── blue_team/
    ├── configs/
    └── backups/
```

### My Solution
```bash
mkdir -p lab/{red_team/{logs,scripts},blue_team/{configs,backups}}
```

### Verification
```bash
tree lab
# or
find lab -type d
```

---

## 📋 Assignment 2 — File Creation Challenge

**Task:** Create multiple files using one command

### Solution
```bash
# Create test1.txt through test5.txt
touch test{1..5}.txt

# Verify
ls -la test*.txt
```

---

## 📋 Assignment 3 — Permissions Drill

**Task:** Practice `chmod` with different scenarios

### Exercises

```bash
# 1. Make a script executable by owner only
chmod 700 myscript.sh

# 2. Set a config file to owner read/write, others read-only
chmod 644 config.txt

# 3. Verify the permissions visually
ls -la myscript.sh config.txt
```

### Permission Reference
```
chmod 777 → rwxrwxrwx (everyone: all)     ⚠️ Dangerous
chmod 755 → rwxr-xr-x (standard script)
chmod 644 → rw-r--r-- (standard file)
chmod 700 → rwx------ (owner only)
chmod 400 → r-------- (read only, e.g. SSH keys)
```

---

## 📋 Assignment 4 — Bandit Progress (OverTheWire)

**URL:** https://overthewire.org/wargames/bandit/

| Level | Status | Key Concept | Command Used |
|-------|--------|-------------|-------------|
| 0 | ✅ | SSH login | `ssh bandit0@bandit.labs.overthewire.org -p 2220` |
| 0→1 | ✅ | Read a file | `cat readme` |
| 1→2 | ⏳ | Dashed filename | `cat ./-` |
| 2→3 | ⏳ | Spaces in name | `cat "spaces in this filename"` |
| 3→4 | ⏳ | Hidden files | `ls -a` then `cat .hidden` |
| 4→5 | ⏳ | Human-readable | `file ./-file*` |
| 5→6 | ⏳ | find by properties | `find . -size 1033c` |

---

## 📋 Assignment 5 — Command Mastery Checklist

Practice each of these until you can do them without looking at notes:

### Navigation
- [ ] `pwd` — print current directory
- [ ] `cd /path` — change directory
- [ ] `cd ..` — go up one level
- [ ] `cd ~` — go home
- [ ] `ls -la` — list all files with details

### File Management
- [ ] `mkdir -p path/sub/sub` — create nested directories
- [ ] `touch file.txt` — create file
- [ ] `cp src dest` — copy
- [ ] `mv src dest` — move/rename
- [ ] `rm -rf folder` — force delete
- [ ] `cat file` — view file
- [ ] `echo "text" > file` — write to file

### Permissions
- [ ] `chmod 755 file` — set permissions with octal
- [ ] `chmod +x file` — add execute
- [ ] `chown user:group file` — change owner

### Search & Filter
- [ ] `grep "pattern" file` — search in file
- [ ] `grep -r "pattern" /dir` — recursive search
- [ ] `find / -name "*.txt"` — find by name
- [ ] `find / -perm -4000` — find SUID files
- [ ] `strings binary | grep flag` — extract and filter

### System
- [ ] `ps aux` — view running processes
- [ ] `sudo -l` — check sudo permissions
- [ ] `printenv` — view environment variables

---

## 📋 Assignment 6 — Vim Editor Practice

Get comfortable with Vim since it's always available on servers.

```bash
# Practice sequence:
vim practice.txt

# Inside vim:
# Press i          → Insert mode
# Type: "Hello from INSA Lab"
# Press Esc        → Normal mode
# Type :wq         → Save and quit

# Verify:
cat practice.txt
```

---

## 📋 Assignment 7 — GitHub Consistency

to track progress

### Daily Git Workflow
```bash
# First time: clone your repo
git clone https://github.com/sofonyas66/INSA-Cybersecurity.git
cd INSA-Cybersecurity

# After adding or editing notes:
git add .
git commit -m "Add Day 00 notes and Flag-Hunting-00 write-up"
git push origin main
```

### Commit Message Format
```
Add [what you added]
Update [what you changed]
Fix [what you corrected]
```

Examples:
```
Add Day 0 notes on power tools
Update Flag-Hunting-00 with Flag 0 solution
Fix permission table in Day 0 notes
```

---

## 📝 Personal Notes & Reflections

- The "one command" folder structure challenge showed the power of Brace Expansion
- Need more practice with `find` flags — there are a lot of options

---
