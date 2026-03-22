# Day 02 — Linux Operations & File Permissions

**Date:** Sunday, March 8, 2026  

---

## 🔁 Revision — Day 1 Recap

- Revisited the Linux file hierarchy
- Reinforced the rule: **"In Linux, everything is a file"**
- Reviewed Red Team vs Blue Team directory relevance
- Discussed how understanding the file system is the foundation of both offense and defense

---

## 🔑 File Permissions — The 4-2-1 System

### Why Permissions Matter
File permissions control who can **read**, **write**, or **execute** a file. Misconfigured permissions are one of the most common causes of security vulnerabilities.

### The Three Tiers
Permissions are applied to three groups in order:

```
[Owner]  [Group]  [Others]
```

### The Octal (Numeric) System

| Permission | Symbol | Octal Value |
|------------|--------|-------------|
| Read       | `r`    | 4           |
| Write      | `w`    | 2           |
| Execute    | `x`    | 1           |
| No access  | `-`    | 0           |

You add the values together for each group:

| Octal | Meaning | Binary |
|-------|---------|--------|
| 7 | Read + Write + Execute | 111 |
| 6 | Read + Write | 110 |
| 5 | Read + Execute | 101 |
| 4 | Read only | 100 |
| 0 | No permission | 000 |

### Common Permission Sets

| Notation | Meaning | Use Case |
|----------|---------|----------|
| `755` | Owner: all / Group+Others: read+execute | Web server files, scripts |
| `644` | Owner: read+write / Group+Others: read | Config files, docs |
| `700` | Owner: all / Group+Others: none | Private scripts, keys |
| `777` | Everyone: all | ⚠️ Dangerous — avoid in production |
| `400` | Owner: read only | SSH private keys |

### How to Read Symbolic Permissions

```
-rwxr-xr-x
│└──┘└──┘└──┘
│  │    │    └── Others (r-x = 5)
│  │    └─────── Group  (r-x = 5)
│  └──────────── Owner  (rwx = 7)
└─────────────── File type (- = regular file, d = directory)
```

### Special Permissions (Advanced)

| Permission | Octal | Symbol | Effect |
|------------|-------|--------|--------|
| SUID | 4000 | `s` in owner execute | Runs with owner's privileges |
| SGID | 2000 | `s` in group execute | Runs with group's privileges |
| Sticky Bit | 1000 | `t` at end | Only owner can delete (used in `/tmp`) |

---

## 💻 Essential Terminal Commands

### Navigation
```bash
pwd                  # Print Working Directory — where am I?
ls                   # List files
ls -la               # List all files including hidden (dotfiles)
ls -lt               # List by time (newest first)
cd /path/to/dir      # Change directory
cd ..                # Go up one level
cd ~                 # Go to home directory
```

### File & Directory Management
```bash
mkdir foldername             # Create a directory
mkdir -p parent/child/sub    # Create nested directories
touch filename.txt           # Create empty file
cat filename.txt             # View file contents
cp source dest               # Copy file
mv source dest               # Move or rename file
rm filename.txt              # Delete file
rm -rf foldername            # Force delete directory (⚠️ careful!)
echo "text" > file.txt       # Write text to file
echo "text" >> file.txt      # Append text to file
```

### Permissions
```bash
chmod 755 script.sh          # Set permissions with octal
chmod +x script.sh           # Add execute permission (symbolic)
chmod -w file.txt            # Remove write permission
chown user:group file.txt    # Change file owner
```

### Brace Expansion — Creating Structures in One Command
```bash
# Create a full nested structure in one command:
mkdir -p project/{src,bin,lib,docs/{pdf,txt}}

# Create multiple files at once:
touch test{1..5}.txt
```
---

## 📝 Key Takeaways — Day 2

- Permissions are the first line of defense — a misconfigured `777` can expose an entire system.
- Brace expansion `{}` is a power-user move that saves time in real deployments.
- Thinking like a Red Team (where would I hide?) makes you a better Blue Team defender.

---

## ✅ Homework

- [ ] Practice `chmod` with different permission combinations
- [ ] Push today's notes to GitHub
- [ ] Try Bandit levels 5–10 (uses `find` and `grep` heavily)
