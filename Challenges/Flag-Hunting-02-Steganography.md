# Challenge Write-Up: Hidden Messages — Steganography #02

**Date:** March 22, 2026
**Program:** INSA Cyber Talent Weekend Program
**Track:** Cybersecurity
**Flags Found:** 4/5

---

## Challenge Description

A JPEG image was provided with 4 flags hidden in different layers.
Hint: First 2 use strings/metadata, remaining 2 require steganography tools.

---

## Flag 1 — Metadata

**Tool:** `exiftool`
```bash
exiftool challenge.jpg
```

Found in the `Comment` field.

**FLAG{ALWAYS_Check_Metadata}**

---

## Flag 2 — Strings

**Tool:** `strings` + `grep`
```bash
strings challenge.jpg | grep "FLAG"
```

**FLAG{Strings_leak_information}**

---

## Flag 3 — Steghide (Password Protected)

**Tool:** `stegseek` (bruteforce), `steghide`
```bash
# Bruteforce the passphrase
stegseek challenge.jpg rockyou.txt

# Passphrase found: password123
# Extracted to: challenge.jpg.out

cat challenge.jpg.out
```

**FLAG{steghide_protected}**

---

## Flag 4 — Appended ZIP

**Tool:** `binwalk`, `dd`, `unzip`, `base64`
```bash
# Detect hidden file
binwalk challenge.jpg

# Extract ZIP at offset 75667
dd if=challenge.jpg bs=1 skip=75667 of=hidden.zip

# Unzip
unzip hidden.zip

# Decode double Base64
echo "Umt4QlIzdGhjSEJsYm1SbFpGOTZhWEI5Q2c9PQo=" | base64 -d | base64 -d
```

**FLAG{appended_zip}**

---

## Flag 5 — Unknown

**Status:** 🤔 To be revealed in class

---

## Summary

| # | Flag | Tool Used |
|---|------|-----------|
| 1 | FLAG{ALWAYS_Check_Metadata} | exiftool |
| 2 | FLAG{Strings_leak_information} | strings + grep |
| 3 | FLAG{steghide_protected} | stegseek + rockyou.txt |
| 4 | FLAG{appended_zip} | binwalk + dd + unzip + base64 |
| 5 | ??? | TBD |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `exiftool` | Read image metadata |
| `strings` | Extract readable text from binary |
| `grep` | Filter output for FLAG pattern |
| `binwalk` | Detect embedded files |
| `dd` | Extract bytes at specific offset |
| `unzip` | Extract ZIP archive |
| `base64 -d` | Decode Base64 (ran twice) |
| `stegseek` | Bruteforce steghide passphrase |

---

## Lessons Learned

- Always check metadata first — `exiftool` is fast and reveals a lot
- `strings` leaks readable content from any binary file
- Files can be appended to images — `binwalk` detects them
- Steganography hides data inside files — `stegseek` + `rockyou.txt` cracks weak passwords
- Encoded data can be layered — always try decoding more than once
