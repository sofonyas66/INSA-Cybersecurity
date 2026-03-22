# Day 01 — Linux Introduction & Orientation

**Date:** Saturday, March 7, 2026  

---

## 🏛️ Program Orientation

### Rules & Protocol
- Attendance is tracked every Saturday and Sunday
- Progress is monitored via **GitHub** — notes, assignments, and challenges must be pushed regularly
- Recommended note-taking tools: **Obsidian** or **Notion**
---

## 🐧 The Linux Philosophy

### What is Linux?
- **Linux is a Kernel** — the core that bridges software and hardware
- A "Linux OS" is actually a **Distribution** (distro) = Kernel + GNU Tools + Desktop Environment
- Examples: Kali Linux, Parrot OS, Ubuntu, Arch Linux

### The Golden Rule
> **"In Linux, everything is a file."**

This applies to:
- Regular files and directories
- Hardware devices (e.g., `/dev/sda` is your hard drive)
- Running processes (accessible via `/proc`)
- Network connections

---

## 📂 Linux File System Hierarchy

The file system starts from **root** (`/`). Every directory branches from here.

| Directory | Description |
|-----------|-------------|
| `/` | Root — the top of the entire filesystem |
| `/bin` | Essential user binaries (e.g., `ls`, `cp`, `cat`) |
| `/sbin` | System binaries — used by admins (e.g., `reboot`, `ifconfig`) |
| `/boot` | Bootloader and kernel files (GRUB lives here) |
| `/dev` | Device files — hardware is represented as files here |
| `/etc` | System-wide configuration files |
| `/home` | Personal directories for each user |
| `/lib` | Shared libraries needed by `/bin` and `/sbin` |
| `/media` | Auto-mount point for removable drives (USB, CD) |
| `/mnt` | Temporary manual mount point |
| `/opt` | Optional/third-party software packages |
| `/tmp` | Temporary files — cleared on reboot |
| `/usr` | User utilities, applications, and secondary hierarchy |
| `/var` | Variable data — logs, databases, mail spools |
| `/var/log` | System and application logs |
| `/root` | Home directory for the root superuser |

### Security Context (Red vs Blue Team)

| Directory | Team | Why |
|-----------|------|-----|
| `/var/log` | 🔵 Blue | Monitor for intrusion attempts and anomalies |
| `/etc` | 🔵 Blue | Watch for unauthorized config changes |
| `/etc/shadow` | 🔵 Blue | Contains hashed passwords — must be protected |
| `/tmp` | 🔴 Red | World-writable, perfect for dropping malicious scripts |
| `/home` | 🔴 Red | Primary target for user data exfiltration |
| `/root` | 🔴 Red | Achieving access here = full system compromise |

---

## 🎮 OverTheWire — Bandit

Bandit is a wargame that teaches terminal basics through a series of levels. Each level requires finding a "password" to SSH into the next one.

**Connect:**
```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
# Password: bandit0
```

### Quick Level Guide

| Level | Key Concept | Key Command |
|-------|-------------|-------------|
| 0 → 1 | Reading a file | `cat readme` |
| 1 → 2 | Dashed filename | `cat ./-` |
| 2 → 3 | Spaces in filename | `cat "spaces in this filename"` |
| 3 → 4 | Hidden files | `ls -a` then `cat .hidden` |

---

## 📝 Key Takeaways — Day 1

- Linux is a kernel, not an OS. Distributions are built on top of it.
- The file system hierarchy is the foundation of everything in security — know where files live.
- `/tmp` is the attacker's first stop; `/var/log` is the defender's first stop.
- Bandit is the best way to build terminal muscle memory fast.

---
