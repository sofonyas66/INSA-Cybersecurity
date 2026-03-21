# Linux Research Quiz — Detailed Notes

Based on your presentation slide, here's a thorough breakdown of each topic:

---

## 1. Linux User Management

### Users vs Root (`user d/t root`)

| Regular User | Root User |
| --- | --- |
| Limited permissions | Unrestricted access |
| Home dir: `/home/username` | Home dir: `/root` |
| Can't modify system files | Can modify anything |
| UID ≥ 1000 | UID = 0 |

### Key Commands

```bash
# Create a user
sudo useradd -m username
sudo passwd username

# Delete a user
sudo userdel -r username

# List users
cat /etc/passwd

# Switch to another user
su - username
```

### Accessing Root / SuperUser DO (`sudo`)

```bash
sudo command          # Run single command as root
sudo -i               # Open root shell (interactive)
sudo su               # Switch to root user
whoami                # Check current user
id                    # Show UID, GID, groups
```

`sudo` works by checking `/etc/sudoers` — only listed users can use it.

```bash
sudo visudo           # Safely edit sudoers file
usermod -aG sudo username   # Add user to sudo group (Debian/Ubuntu)
usermod -aG wheel username  # Add user to wheel group (Arch/RHEL)
```

---

## 2. Package Installation on Linux

### `apt` — Debian/Ubuntu/Parrot/Kali

```bash
sudo apt update               # Refresh package list
sudo apt install <pkg>        # Install
sudo apt remove <pkg>         # Remove
sudo apt upgrade              # Upgrade all packages
```

### `pkg` — FreeBSD / Termux (Android)

```bash
pkg install <pkg>
pkg update && pkg upgrade
```

### `pacman` — Arch Linux / Manjaro

```bash
sudo pacman -Syu              # Sync & upgrade
sudo pacman -S <pkg>          # Install
sudo pacman -R <pkg>          # Remove
sudo pacman -Ss keyword       # Search
yay -S <pkg>                  # AUR helper (like you use on your Arch setup)
```

---

## 3. Script Installation

Running scripts to install software:

```bash
# Make a script executable
chmod +x install.sh

# Run it
./install.sh

# Or run directly with bash
bash install.sh

# Run as root
sudo bash install.sh
```

**Common pattern** (like your dotfiles `install.sh`):

```bash
#!/bin/bash
echo "Installing..."
sudo apt install zsh git curl -y
chsh -s $(which zsh)
```

---

## 4. Software Installation Methods (Summary)

| Method | Command Example | Distro |
| --- | --- | --- |
| Package manager | `apt install vim` | Debian-based |
| Pacman/yay | `pacman -S vim` | Arch-based |
| Compile from source | `./configure && make && sudo make install` | Any |
| `.deb` file | `sudo dpkg -i file.deb` | Debian-based |
| `.tar.gz` | Extract → build | Any |
| Snap | `sudo snap install vlc` | Ubuntu |
| Flatpak | `flatpak install flathub org.app` | Any |

---

## 5. Shell

### Types of Shells

| Shell | Notes |
| --- | --- |
| `sh` | Original Bourne shell |
| `bash` | Most common, default on many distros |
| `zsh` | What you use with Oh My Zsh + Powerlevel10k |
| `fish` | User-friendly, modern |
| `dash` | Lightweight, fast scripting |

```bash
echo $SHELL         # See your current shell
cat /etc/shells     # List installed shells
chsh -s /bin/zsh    # Change default shell
```

---

## 6. Shell Features: OR, Piping, AND (`or, piping, and d/t`)

### AND operator `&&` — run second command only if first succeeds

```bash
sudo apt update && sudo apt upgrade
```

### OR operator `||` — run second command only if first fails

```bash
ping -c1 google.com || echo "No internet"
```

### Piping `|` — pass output of one command as input to next

```bash
ls -la | grep ".sh"
cat /etc/passwd | grep root
ps aux | grep firefox
```

### Combining all three

```bash
sudo apt update && sudo apt install nmap || echo "Install failed"
```

### Redirection (related `d/t`)
