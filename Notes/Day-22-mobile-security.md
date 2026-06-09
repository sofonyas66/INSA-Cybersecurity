# Day 22 — Mobile Security

Topics: Android Security, APK Structure, Static Analysis, Dynamic Analysis, JADX, APKTool, ADB, Genymotion, Frida, Burp Suite, Mobile Pentesting

**Date:** Sunday May 31, 2026

---

## What is Mobile Application Security?

Mobile application security penetration testing is the practice of assessing and identifying vulnerabilities within mobile applications. Mobile apps often handle sensitive information such as user credentials, financial data, and personal information. A compromised mobile app can lead to severe security breaches — identity theft, unauthorized financial transactions, and loss of user trust.

Penetration testing involves simulating attacks on an app from an attacker's perspective. The goal is to uncover security flaws before malicious individuals can exploit them.

---

## What is an APK?

APK stands for **Android Package Kit** — the installation file format used by Android, similar to how `.exe` is used on Windows.

```
Windows → .exe
Android → .apk
```

An APK contains:

- Application code (`classes.dex`)
- Resources (`res/`)
- Images and icons
- `AndroidManifest.xml` — describes structure, components, and permissions
- Libraries (`lib/`)
- Certificates (`META-INF/`)
- Precompiled resources (`resources.arsc`)
- Raw assets (`assets/`)

APK files can be analyzed and reverse engineered.

---

## Android Platform Overview

Android is an open-source mobile OS developed by Google. It runs on a Linux-based kernel and uses a sandboxed application model for security.

### Android Architecture (bottom to top)

| Layer | Description |
|---|---|
| Linux Kernel | Base of Android — manages hardware, memory, processes. Includes Low Memory Killer, wake locks, Binder IPC |
| HAL (Hardware Abstraction Layer) | Standard interface for interacting with hardware components |
| Android Runtime (ART) | Executes apps. Replaced older Dalvik VM. Apps compile to Dalvik bytecode then run here |
| Core Libraries | Fundamental Java libraries for app development |
| Java API Framework | APIs for app development and system interactions |
| System Apps | Pre-installed apps — Dialer, Camera, Calendar, etc. |

---

## Application Sandboxing & Permissions

Each Android app runs in its own isolated environment:

- Every app gets a unique **UID** (User ID) when installed
- App data is stored at `/data/data/<package_name>` — only accessible to that app
- **SELinux** enforces mandatory access control policies at runtime

Apps must declare permissions in `AndroidManifest.xml` to access hardware or other app data:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

### Permission Protection Levels

| Level | Description |
|---|---|
| `normal` | Automatically granted, no user confirmation needed |
| `dangerous` | Access to sensitive data — user must explicitly grant at runtime |
| `signature` | Only granted to apps signed with the same certificate |
| `signatureOrSystem` | Granted to same-cert apps or system apps |

---

## Mobile Application Components

Android apps are built from these main components:

- **Activities** — UI screens
- **Services** — background tasks
- **Broadcast Receivers** — listen for system/app events
- **Content Providers** — share data between apps
- **Intents** — messages that trigger components (explicit and implicit)

---

## Mobile Pentesting Methodology

```
APK
 ↓
JADX (Static Analysis)
 ↓
Understand Code / Find Vulnerabilities
 ↓
Genymotion (Install & Run App)
 ↓
Burp Suite (Intercept Network Traffic)
 ↓
Frida / Objection (Runtime Analysis)
```

### Testing Types

| Type | Description |
|---|---|
| Black-box | No prior knowledge of the app |
| White-box | Full access to source code and internals |
| Gray-box | Partial info provided (e.g. credentials only) |

---

## Static Analysis

Static analysis means examining an application **without running it**.

**Purpose:**
- Understand app behavior and logic
- Review decompiled source code
- Identify hardcoded secrets, insecure configs, weak cryptography
- Discover hidden functionality

### Tools

#### JADX — Decompiler

Converts APK files into readable Java code.

```bash
jadx -d output_folder/ target.apk
```

**What to search for in JADX:**

```text
# API endpoints
https://
http://

# Authentication logic
login
signin
authenticate

# Hardcoded secrets
apikey
api_key
token
secret
password

# Insecure storage
MODE_WORLD_READABLE
getSharedPreferences

# Weak crypto
MD5
SHA1
DES
ECB
```

Always open `AndroidManifest.xml` and look for:
- `android:exported="true"` — component accessible from outside the app
- `android:debuggable="true"` — dangerous in production
- `android:allowBackup="true"` — app data can be extracted via ADB
- `android:usesCleartextTraffic="true"` — allows HTTP (unencrypted)
- Declared permissions and deep links

#### APKTool — Decompiler to Smali

Decodes APK resources and decompiles to smali (low-level bytecode representation).

```bash
apktool d target.apk -o output_folder/
```

Use APKTool when you need to:
- View and modify `AndroidManifest.xml`
- Read smali code
- Rebuild/repackage a modified APK (`apktool b output_folder/`)

**JADX vs APKTool:**

```
JADX    → Java code (more readable)
APKTool → Smali code + resources (needed for rebuilding)
```

---

## Dynamic Analysis

Dynamic analysis means testing the application **while it is running**.

**Purpose:**
- Observe real runtime behavior
- Monitor network traffic
- Test security controls in action
- Identify vulnerabilities that static analysis can miss

### Tools

#### Genymotion — Android Emulator

A virtual Android phone on your computer. Used to install and run APKs for testing.

```
APK → Genymotion → Run App → Test & Observe
```

#### ADB (Android Debug Bridge)

Controls Android devices (real or emulated) from the terminal.

```bash
adb devices                        # list connected devices
adb install app.apk                # install APK
adb shell                          # open device shell
adb shell am start -n pkg/.Activity  # launch specific activity
adb logcat                         # view real-time logs
adb shell run-as <pkg> ls /data/data/<pkg>/  # browse app files
```

#### Burp Suite — Traffic Interception

Intercepts HTTP/HTTPS traffic between the app and the server.

```
App → Burp Suite → Server
```

Questions to ask when intercepting:
- Is sensitive data encrypted in transit?
- Are tokens or credentials exposed in requests?
- Can requests be modified to bypass logic?

#### Frida — Runtime Hooking

Hooks into running application functions to observe or modify behavior at runtime.

Use cases:
- Bypass root detection
- Disable SSL pinning
- Monitor function calls
- Modify return values

#### Objection — Frida Wrapper

A higher-level tool built on Frida. Easier to use for common mobile testing tasks.

```bash
objection -g com.package.name explore
```

---

## Common Mobile Security Vulnerabilities

### Hardcoded Secrets
```java
String API_KEY = "sk-abc123secret";  // attacker extracts this from APK
```

### Insecure Data Storage
Sensitive data stored in plaintext in SharedPreferences, SQLite databases, or log files.

### Broken Authentication
Weak login protections, predictable tokens, poor session management.

### Broken Authorization (IDOR)
```
/user/1001  →  change to  /user/1002  →  access another user's data
```

### Excessive Permissions
```
Flashlight app requesting Contacts + Camera + Location permissions
```

### Insecure Network Communication
App sends sensitive data over HTTP instead of HTTPS — interceptable with Burp Suite or Wireshark.

### Insecure WebViews
WebView with JavaScript enabled + `addJavascriptInterface` can lead to RCE. Key settings to look for:
```java
webView.getSettings().setJavaScriptEnabled(true)
webView.addJavascriptInterface(new MyObject(), "Android")
webView.getSettings().setAllowFileAccess(true)
```

---

## OWASP Mobile Top 10 (2024)

1. Improper Credential Usage
2. Inadequate Supply Chain Security
3. Insecure Authentication/Authorization
4. Insufficient Input/Output Validation
5. Insecure Communication
6. Inadequate Privacy Controls
7. Insufficient Binary Protections
8. Security Misconfiguration
9. Insecure Data Storage
10. Insufficient Cryptography

---

## Lab — InjuredAndroid

InjuredAndroid is a deliberately vulnerable Android app used for practice (like DVWA but for mobile). It contains flag challenges solved through static and dynamic analysis.

**Install:**
```bash
adb install InjuredAndroid-1.0.12-release.apk
```

**Basic static analysis workflow:**
```bash
# Decompile
jadx -d injured_out/ InjuredAndroid-1.0.12-release.apk

# Check manifest
cat injured_out/resources/AndroidManifest.xml

# Search for secrets
grep -r "password\|secret\|api_key\|flag" injured_out/sources/ -i
```

---

## Homework

- Take a beginner course on Java
