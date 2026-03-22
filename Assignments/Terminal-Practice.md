# Assignments — Terminal Practice Log

**Track:** Cybersecurity  
**Program:** INSA Cyber Talent Weekend Program  

---

## 📋 Assignment 1 — Linux Folder Structure Challenge

**Given by:** Mentor, Day 2  
**Task:** Recreate a given folder structure using **one command**

**Why This Matters:** Understanding directory structures is fundamental to navigating Linux systems and organizing security projects. Mastering `mkdir` with brace expansion saves time during incident response or lab setup.

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

**Key Concept:** Brace expansion `{a,b}` creates multiple paths in one command, and `-p` creates parent directories as needed.

### Verification
```bash
tree lab
# Output:
# lab/
# ├── red_team/
# │   ├── logs/
# │   └── scripts/
# └── blue_team/
#     ├── configs/
#     └── backups/

# or
find lab -type d
# Output shows all directories created
```

### Common Mistakes
- ❌ Forgetting the `-p` flag (creates errors if parent doesn't exist)
- ❌ Misplacing braces (order matters for nested structures)

---

## 📋 Assignment 2 — File Creation Challenge

**Task:** Create multiple files using one command

**Why This Matters:** Batch file creation is useful when setting up test environments, creating log files, or generating multiple configuration templates quickly.

### Solution
```bash
# Create test1.txt through test5.txt
touch test{1..5}.txt

# Verify
ls -la test*.txt
# Output:
# -rw-r--r-- 1 user group    0 Mar 22 10:15 test1.txt
# -rw-r--r-- 1 user group    0 Mar 22 10:15 test2.txt
# -rw-r--r-- 1 user group    0 Mar 22 10:15 test3.txt
# -rw-r--r-- 1 user group    0 Mar 22 10:15 test4.txt
# -rw-r--r-- 1 user group    0 Mar 22 10:15 test5.txt
```

### Common Mistakes
- ❌ Using `{1-5}` instead of `{1..5}` (incorrect syntax)
- ❌ Forgetting that `touch` creates files with 0 bytes

---

## 📋 Assignment 3 — Permissions Drill

**Task:** Practice `chmod` with different scenarios

**Why This Matters:** File permissions are critical in cybersecurity. Improper permissions can expose sensitive data (SSH keys, config files) or allow unauthorized execution of scripts. This is a top vulnerability in Linux environments.

### Exercises

```bash
# 1. Make a script executable by owner only
chmod 700 myscript.sh
# Result: rwx------ (only owner can read, write, execute)

# 2. Set a config file to owner read/write, others read-only
chmod 644 config.txt
# Result: rw-r--r-- (owner can modify, others can only view)

# 3. Verify the permissions visually
ls -la myscript.sh config.txt
# Output:
# -rwx------ 1 user group    1234 Mar 22 10:15 myscript.sh
# -rw-r--r-- 1 user group    5678 Mar 22 10:15 config.txt
```

### Permission Reference
```
chmod 777 → rwxrwxrwx (everyone: all)     ⚠️ DANGEROUS — Never use in production
chmod 755 → rwxr-xr-x (standard script)   ✅ Script: owner full, others execute
chmod 644 → rw-r--r-- (standard file)     ✅ Config: owner edit, others read
chmod 700 → rwx------ (owner only)        ✅ Sensitive: SSH keys, private scripts
chmod 400 → r-------- (read only)         ✅ Immutable: archived configs
```

### Security Tips
- 🔒 SSH keys should **always** be `600` or `400` (owner read/write or read-only)
- 🔒 Never use `777` — it's a security risk that exposes files to unauthorized access
- 🔒 Config files with passwords should be `600`

---

## 📋 Assignment 4 — Bandit Progress (OverTheWire)

**URL:** https://overthewire.org/wargames/bandit/

**Why This Matters:** OverTheWire's Bandit wargame teaches practical Linux command-line skills through realistic scenarios. Each level introduces techniques used in real penetration testing and incident response.

| Level | Status | Key Concept | Command Used | What You Learn |
|-------|--------|-------------|-------------|----------------|
| 0 | ✅ | SSH login | `ssh bandit0@bandit.labs.overthewire.org -p 2220` | Basics of remote access |
| 0→1 | ✅ | Read a file | `cat readme` | File viewing, finding hidden information |
| 1→2 | ⏳ | Dashed filename | `cat ./-` | Handling unusual filenames with `./` prefix |
| 2→3 | ⏳ | Spaces in name | `cat "spaces in this filename"` | Quoting/escaping special characters |
| 3→4 | ⏳ | Hidden files | `ls -a` then `cat .hidden` | Finding hidden files with `-a` flag |
| 4→5 | ⏳ | Human-readable | `file ./-file*` | Using `file` to identify file types |
| 5→6 | ⏳ | find by properties | `find . -size 1033c` | Using `find` with size criteria |

**Pro Tips:**
- Each level teaches a technique you'll use in penetration testing
- Don't skip ahead — earlier levels teach fundamentals
- Document each solution in your notes for future reference

---

## 📋 Assignment 5 — Command Mastery Checklist

Practice each of these until you can do them without looking at notes. These are the **most frequently used** commands in Linux security work.

### Navigation
- [ ] `pwd` — print current directory
- [ ] `cd /path` — change directory
- [ ] `cd ..` — go up one level
- [ ] `cd ~` — go to home directory
- [ ] `ls -la` — list all files with details (including hidden files)

### File Management
- [ ] `mkdir -p path/sub/sub` — create nested directories
- [ ] `touch file.txt` — create empty file
- [ ] `cp src dest` — copy file
- [ ] `mv src dest` — move/rename file
- [ ] `rm -rf folder` — force delete (⚠️ use carefully!)
- [ ] `cat file` — view entire file
- [ ] `echo "text" > file` — write/overwrite file
- [ ] `cat file | head -20` — view first 20 lines
- [ ] `tail -f logfile` — follow log in real-time (useful for monitoring)

### Permissions & Ownership
- [ ] `chmod 755 file` — set permissions with octal notation
- [ ] `chmod +x file` — add execute permission
- [ ] `chmod -x file` — remove execute permission
- [ ] `chown user:group file` — change owner and group
- [ ] `stat file` — view detailed file metadata including permissions

### Search & Filter
- [ ] `grep "pattern" file` — search in file
- [ ] `grep -r "pattern" /dir` — recursive search in directory
- [ ] `grep -i "pattern" file` — case-insensitive search
- [ ] `find / -name "*.txt"` — find by filename pattern
- [ ] `find / -perm -4000` — find SUID files (security risk check)
- [ ] `find / -type f -size +100M` — find large files
- [ ] `strings binary | grep flag` — extract readable strings and filter

### System & Processes
- [ ] `ps aux` — view all running processes
- [ ] `ps aux | grep process_name` — find specific process
- [ ] `sudo -l` — check your sudo permissions
- [ ] `printenv` — view environment variables
- [ ] `whoami` — check current user
- [ ] `id` — show user ID and group memberships
- [ ] `which command` — find command location
- [ ] `man command` — read the manual for any command

### Advanced (Next Level)
- [ ] `awk '{print $1}' file` — extract columns from text
- [ ] `sed 's/old/new/g' file` — find and replace in text
- [ ] `tmux` — terminal multiplexer for multiple sessions
- [ ] `screen` — alternative terminal multiplexer

---

## 📋 Assignment 6 — Vim Editor Practice

Get comfortable with Vim since it's **always available on servers**, even when GUI editors aren't.

**Why This Matters:** Server environments rarely have GUI tools. Learning Vim makes you productive on any Linux machine, whether it's a compromised server during incident response or a minimal Docker container.

### Practice Sequence

```bash
vim practice.txt

# Inside vim:
# Press i              → Insert mode
# Type: "Hello from INSA Lab"
# Press Esc            → Return to Normal mode
# Type :wq             → Save and quit (w=write, q=quit)

# Verify:
cat practice.txt
# Output: Hello from INSA Lab
```

### Essential Vim Commands

**Normal Mode (press Esc to enter):**
- `:w` — save file
- `:q` — quit
- `:wq` — save and quit
- `:q!` — quit without saving
- `/pattern` — search for text
- `dd` — delete line
- `yy` — copy line
- `p` — paste line
- `u` — undo
- `Ctrl+r` — redo

**Insert Mode (press `i`):**
- Type normally to edit
- Press `Esc` to return to Normal mode

### Pro Tip
The moment Vim feels uncomfortable is when you should push yourself to use it more. It becomes second nature quickly!

---

## 📋 Assignment 7 — GitHub Consistency

**Mentor requirement:** GitHub will be reviewed to track progress

**Why This Matters:** Version control is essential in professional security work. You'll use GitHub (or GitLab) to document findings, share playbooks, and collaborate on security tools.

### Recommended Git Workflow

```bash
# First time: clone your repo
git clone https://github.com/sofonyas66/INSA-Cybersecurity.git
cd INSA-Cybersecurity

# Create a feature branch for your work (best practice)
git checkout -b add/day-2-notes

# After adding or editing notes:
git add Assignments/Terminal-Practice.md
git commit -m "Add Day 2 terminal practice solutions"
git push origin add/day-2-notes

# Then create a Pull Request on GitHub for review
# This allows mentors to review your work before merging
```

### Why Feature Branches?
- ✅ Keeps work organized
- ✅ Allows code review before merging
- ✅ Prevents accidental commits to `main`
- ✅ Professional workflow (used in real teams)

### Commit Message Format

```
Add [what you added]
Update [what you changed]
Fix [what you corrected]
Remove [what you removed]
Refactor [what you restructured]
```

### Examples
```
Add Day 3 notes on power tools
Update Flag-Hunting-01 with Flag 3 solution
Fix permission table in Day 2 notes
Remove outdated Bandit level 0 notes
Refactor security tips section for clarity
```

### Commit Best Practices
- ✅ Write clear, descriptive messages
- ✅ Commit logical units of work (not everything at once)
- ✅ Use present tense ("Add" not "Added")
- ✅ Keep commits small and focused

---

## 📝 Personal Notes & Reflections

- **Day 2 Confidence Boost:** Solving Flag 1 on Termux (phone) before the session ended showed that I can solve problems independently, even with limited tools.

- **Brace Expansion Power:** The "one command" folder structure challenge demonstrated the power of brace expansion. This will save significant time in future lab setup and automation.

- **find Command Depth:** I realize there are many `find` flags (`-type`, `-size`, `-perm`, `-exec`, etc.). Planning to create a dedicated cheat sheet for this.

- **Vim Learning Curve:** Vim feels awkward now, but I understand it will be essential for server-side work. Committing to using it for all text editing in the lab.

- **GitHub Workflow:** Adding feature branches to my workflow will make my commit history cleaner and align with professional practices.

### Next Steps
- [ ] Complete Bandit levels 1→6 by end of weekend
- [ ] Create a Vim cheat sheet for quick reference
- [ ] Document `find` command examples with real-world use cases
- [ ] Review and update this file weekly with new learnings
