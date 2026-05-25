# Day 20 — Reverse Engineering Deep Dive

Topics: RE Methodology, CPU Architecture, x86 Assembly, PE/ELF Format, Static Analysis, Dynamic Analysis, Ghidra, x64dbg, Binary Patching, CrackMe Challenges, Anti-Reversing Techniques, Malware Analysis Intro

**Date:** Sunday May 24, 2026

---

## What is Reverse Engineering?

Reverse engineering is the process of analyzing a compiled binary to understand its logic, structure, and behavior — without access to the original source code.

**The RE Pipeline:**
```
Binary Executable → Disassemble / Decompile → Analyze Code → Understand Logic
```

**Three core questions:**
- What does this program **DO**?
- **How** does it do it?
- Can we **modify** it?

**Why it matters:**
- Malware analysis — understand how malicious software operates
- Vulnerability research — find flaws before attackers do
- CTF challenges — crackmes and binary exploitation
- Digital forensics — investigate incidents and recover evidence
- Software interoperability — enable compatibility between proprietary systems
- Legacy code recovery — recover functionality with no source available

---

## Ethical & Legal Boundaries

Always get **explicit written permission** before reversing any software you don't own.

| ✅ Legal & Ethical | ❌ Potentially Illegal |
|---|---|
| Malware you captured yourself | Commercial software (check ToS) |
| CTF / crackme challenges | DRM circumvention |
| Your own software | Competitor products |
| Licensed bug bounty targets | Systems without owner consent |

**Key laws:** CFAA (US), DMCA §1201 (anti-circumvention), EU NIS Directive, local cybercrime laws.

---

## CPU Architecture & Registers

The CPU fetches, decodes, and executes instructions. Registers are its fastest memory slots.

| Register | Role |
|---|---|
| EAX | Accumulator / return value |
| EBX | Base register / general purpose |
| ECX | Counter for loops |
| EDX | Data / I/O pointer |
| ESP | Stack pointer (top of stack) |
| EBP | Base pointer (stack frame) |
| EIP | Instruction pointer (next instruction) |

**Instruction Execution Cycle:**
1. **FETCH** — read instruction from memory at EIP
2. **DECODE** — CPU decodes the opcode
3. **EXECUTE** — ALU performs the operation
4. **WRITE-BACK** — result stored in register or memory
5. **UPDATE EIP** — point to next instruction

**Memory Layout:**

| STACK | HEAP |
|---|---|
| LIFO structure | Dynamic allocation (malloc/new) |
| Local variables, function calls | Programmer-managed |
| Fixed size | Variable size |
| Grows **DOWN** ↓ | Grows **UP** ↑ |

---

## Binary & Executable Formats

### ELF (Linux)
```bash
file binary        # identifies ELF 32/64-bit, stripped/not stripped
readelf -h binary  # ELF header info
```

### PE Format (Windows .exe / .dll)
Every Windows executable follows the Portable Executable format:

| Section | Contents |
|---|---|
| DOS Header | MZ magic bytes, legacy DOS stub |
| PE Header (COFF) | Machine type, section count, timestamp |
| Optional Header | ImageBase, EntryPoint, subsystem, imports |
| `.text` | Executable code — **this is what we reverse** |
| `.data` | Initialized global variables, strings |
| `.rdata` | Constants, import table, debug info |
| `.rsrc` | Icons, dialogs, version info |

> `MZ` at offset 0 is how the OS recognizes a valid Windows executable. The Entry Point lives in `.text`.

---

## x86 Assembly — Core Instructions

| Instruction | Example | Meaning |
|---|---|---|
| `MOV` | `MOV EAX, 5` | Copy value into register or memory |
| `PUSH` | `PUSH EAX` | Put value on top of stack; ESP decreases |
| `POP` | `POP EBX` | Remove top of stack into register |
| `CALL` | `CALL 0x401000` | Call a function (pushes return address) |
| `JMP` | `JMP 0x40108A` | Unconditional jump |
| `CMP` | `CMP EAX, 0` | Compare two values (sets CPU flags) |
| `JE` | `JE 0x40108A` | Jump if Equal (ZF=1) |
| `JNE` | `JNE 0x40108A` | Jump if Not Equal (ZF=0) |
| `NOP` | `NOP` | No operation (opcode `0x90`) — used to neutralize instructions |

### Reading Assembly: Simple Function

```c
// C source (what we DON'T have)
int add(int a, int b) {
    int result = a + b;
    return result;
}
```

```asm
; Compiled x86 assembly (what we SEE)
add:
  push ebp           ; save caller's stack frame
  mov  ebp, esp      ; set new frame base
  mov  eax, [ebp+8]  ; load argument 'a'
  add  eax, [ebp+12] ; add argument 'b'
  pop  ebp           ; restore caller's frame
  ret                ; return (result in EAX)
```

Key points:
- Every function starts with `push ebp` / `mov ebp, esp` — the **function prologue**
- `[ebp+8]` = first argument, `[ebp+12]` = second argument (32-bit cdecl/stdcall)
- Return value is always in **EAX**

---

## RE Workflow — The Full Methodology

```
1. Reconnaissance  →  file, DIE, exiftool, hashes
2. Static Analysis →  strings, imports, Ghidra (no execution)
3. Dynamic Analysis → x64dbg, breakpoints, watch registers
4. Identify Logic  →  main functions, crypto, network calls, file ops
5. Document        →  addresses, renamed functions, IOCs
6. Report / Patch  →  write report or patch per goal
```

---

## RE Tools

| Tool | Type | Notes |
|---|---|---|
| **Ghidra** | Disassembler + Decompiler | Free (NSA). Best free IDA alternative |
| **IDA Pro** | Disassembler | Industry standard, commercial |
| **x64dbg** | Debugger | Modern Windows debugger, great UI |
| **OllyDbg** | Debugger (32-bit) | Classic, legacy malware analysis |
| **Radare2** | Framework | CLI, scriptable, multi-architecture |
| **Binary Ninja** | Disassembler | Modern UI, great HLIL decompiler |
| **DIE** | Identifier | Detects packers, compilers, protectors |
| **PE-bear** | PE Viewer | Inspect PE headers without disassembling |

---

## Static Analysis Techniques

Extracting maximum information **without running the binary**.

```bash
# String extraction — hardcoded URLs, passwords, keys
strings malware.exe | grep "http"

# Import analysis — which Windows APIs are used?
# CreateFile → file ops, WSAConnect → network
dumpbin /imports target.exe
```

**In Ghidra:**
- Right-click a function/string → **References** — shows everywhere it's called or used (XREF)
- **Function Graph** view — visual control flow graph (CFG) of all code paths

**What to look for:**
- Hardcoded strings (passwords, URLs, registry keys, error messages)
- Suspicious imports (`CreateRemoteThread`, `VirtualAlloc`, `WSAConnect`)
- High entropy sections → likely packed or encrypted

---

## Dynamic Analysis — Debugging Live

Running the binary and observing actual execution.

```
x64dbg workflow:
1. Load the binary
2. Set breakpoints at key functions (e.g., strcmp, main, WinMain)
3. Run → execution pauses at breakpoint
4. Watch registers change (EAX = return value, ECX = loop counter)
5. Step through instruction by instruction (F7 = step into, F8 = step over)
```

**Key concepts:**
- Breakpoints pause execution at a specific address
- Watch the stack window to see function arguments being pushed
- EIP always points to the **next instruction to execute**
- After `CMP`, check the flags register (ZF=1 means they were equal)

---

## Binary Patching

Modifying bytes in a compiled binary to change its behavior.

### Conditional Jump Opcodes
| Opcode | Mnemonic | Meaning |
|---|---|---|
| `0x74` | `JZ` / `JE` | Jump if Zero / Equal |
| `0x75` | `JNZ` / `JNE` | Jump if Not Zero / Not Equal |
| `0x90` | `NOP` | No operation — neutralize a jump entirely |

Flipping `JE` ↔ `JNE` is a **one-byte change** that reverses a conditional branch. This is the most common binary patch in crackme challenges.

**Export patched binary in Ghidra:** `File → Export Program → ELF` (or PE)

---

## CrackMe2 — Binary Patching Walkthrough

### Recon
```bash
file crackme2      # ELF 64-bit, not stripped
strings crackme2   # revealed: hidden12H, check_password, strcmp
```

The `H` after `hidden12` was noise — a REX.W prefix byte from an adjacent x86-64 instruction bleeding into `strings` output. Always verify strings in Ghidra before trusting them.

### Static Analysis in Ghidra

**`check_password` decompiled:**
```c
char local_12[10];
builtin_strncpy(local_12, "hidden123", 10);
iVar1 = strcmp(param_1, local_12);
return 0;   // always returns 0 — bug!
```

**`main` logic:**
```c
iVar1 = check_password(local_48);
if (iVar1 == 0)
    puts("Access Denied!");   // 0 treated as denied
else
    puts("Access Granted!");
```

### The Bug

`check_password` always returns `0` because both branches inside it returned `0`. `strcmp` itself returns `0` on a correct match, but the function never propagated that result correctly — **Access Granted was completely unreachable**.

### The Patch

In `main` at address `00101202`:

| Before | After |
|---|---|
| `74 11  JZ  LAB_00101215` | `75 11  JNZ  LAB_00101215` |

One byte: `0x74` → `0x75`. Now a correct password (`iVar1 == 0`) falls through to Access Granted instead of jumping to Denied.

```bash
# Verify the patch
./crackme3
# Password: hidden123  → Access Granted ✓
# Password: test       → Access Denied  ✓
```

### Key Takeaways
- `strings` can bleed adjacent bytes — always verify in Ghidra
- A function can have correct logic but wrong return values
- Minimal patches are best — one byte flip fixed the entire program
- `JZ` (`0x74`) and `JNZ` (`0x75`) differ by exactly one bit

---

## Anti-Reversing Techniques

How malware authors make analysis harder:

| Technique | How it works | Counter |
|---|---|---|
| **Packing** | Compresses/encrypts binary (UPX common) | DIE detects it; unpack first |
| **Obfuscation** | Junk code, dead code, misleading names | Patient static analysis |
| **Anti-Debugging** | `IsDebuggerPresent`, timing checks | Patch the check or use plugins |
| **VM Detection** | Checks for VMware/VirtualBox artifacts | Use barebone or masked VMs |
| **Encryption** | Strings/payloads decrypted at runtime | Breakpoint AFTER decryption |
| **Control Flow Flattening** | Replaces normal flow with dispatch table | Miasm or Hex-Rays to deobfuscate |

---

## Malware Analysis — Safe Lab Principles

**Rule #1: ALWAYS analyze in an isolated VM. Take a snapshot BEFORE execution.**

**Lab setup:**
- Isolated VM (VMware/VirtualBox), host-only network adapter
- Shared folders DISABLED (malware can escape via these)
- FakeNet-NG to simulate network services

**IOCs to collect:**
- File hashes (MD5, SHA256)
- Registry keys created/modified
- Files dropped on disk
- Network C2 addresses and domains
- Mutex names, process names

**Common malware behaviors:**
- Persistence (Run keys, services)
- Privilege escalation
- Credential theft
- Lateral movement
- C2 beacon communication

---

## Glossary

| Term | Definition |
|---|---|
| **Disassembly** | Converting machine code back to assembly language |
| **Decompilation** | Converting binary to higher-level pseudo-code (C-like) |
| **Opcode** | Numeric code for a CPU instruction (e.g., `0x55` = `PUSH EBP`) |
| **Entropy** | Measure of randomness — high entropy often indicates packing/encryption |
| **IOC** | Indicator of Compromise — observable evidence of malicious activity |
| **Sandbox** | Isolated execution environment for safe dynamic analysis |
| **XREF** | Cross-reference — all locations that reference a function or data |
| **CFG** | Control Flow Graph — visual map of all execution paths in a function |
| **NOP** | No Operation (`0x90`) — used to neutralize instructions |
| **ROP** | Return-Oriented Programming — exploitation using existing code gadgets |
| **Shellcode** | Raw machine code designed to be injected and executed in memory |

---

## Resources

- **Practice:** crackmes.one, Hack The Box Reversing track, picoCTF
- **Courses:** OpenSecurityTraining2 (ost2.fyi), Malware Unicorn workshops
- **Books:** *Practical Malware Analysis* (Sikorski & Honig), *Hacking: The Art of Exploitation* (Erickson)
- **Tools:** Ghidra (ghidra-sre.org), x64dbg (x64dbg.com), ANY.RUN sandbox
