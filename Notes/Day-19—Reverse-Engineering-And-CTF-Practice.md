# Day 19 | Reverse Engineering & CTF Practice

**Date:** Saturday May 23, 2026
 
### Reverse Engineering Fundamentals

Reverse engineering is the process of analyzing a compiled binary to understand its logic, behavior, and internals — without access to the source code. It is a core skill in both offensive security (finding vulnerabilities) and CTF competitions.

### Key Tools

**Static Analysis** (no execution required):
- `file` — identifies binary type, architecture, and format
- `strings` — extracts printable strings from a binary (reveals hardcoded values, messages, flags)
- `objdump` — disassembles binary into assembly instructions; useful for reading control flow
- `readelf` — inspects ELF headers, sections, and symbols
- `binwalk` — scans for embedded files or known signatures within a binary

**Dynamic Analysis** (runtime inspection):
- `ltrace` — intercepts library calls (e.g. `strcmp`, `printf`, `fgets`) and shows arguments/return values at runtime
- `strace` — intercepts system calls; useful for understanding I/O and signal behavior
- `gdb` — GNU debugger; allows setting breakpoints, stepping through instructions, and inspecting registers and memory at runtime

### ELF Binary Structure

Linux executables follow the ELF (Executable and Linkable Format) structure:
- **Sections**: `.text` (code), `.rodata` (read-only data/strings), `.data` (initialized variables), `.bss` (uninitialized variables)
- **PLT/GOT**: used for dynamic linking — function calls to shared libraries go through the PLT (Procedure Linkage Table) which resolves addresses via the GOT (Global Offset Table)
- Stripped binaries have no symbol names, making analysis harder

### Assembly Concepts Reviewed

- Registers: `rax`, `rdi`, `rsi`, `rdx`, `rbx`, `rsp`, `rbp`, `rip`
- Calling convention (x86-64 Linux): arguments passed in `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`; return value in `rax`
- Common instructions: `mov`, `lea`, `cmp`, `test`, `je`/`jne`/`jbe`, `call`, `ret`, `push`, `pop`
- Bit operations: `xor`, `and`, `or`, `shl`, `shr`, `rol` (rotate left)

### Anti-Reversing Techniques

Binaries can be designed to mislead analysts:
- **Fake flags / misdirection**: the binary contains decoy strings or outputs that appear to be the answer but are not
- **Signal-based obfuscation**: using `SIGILL` (illegal instruction signal) to redirect execution through a signal handler, hiding the real logic from casual analysis
- **Self-modifying or runtime-computed values**: data that appears static in the binary but is computed or XOR-decoded at runtime
- **VM/bytecode interpreters**: the binary implements a small virtual machine that executes its own instruction set, making analysis non-trivial

### GDB Workflow

```bash
gdb ./binary
set debuginfod enabled off          # disable symbol downloading
handle SIGILL noprint nostop pass   # let signals pass to the program
break *0x401234                     # set breakpoint at address
run <<< "input"                     # run with piped input
x/s $rdi                            # examine register as string
x/16xb 0x404090                    # examine memory as hex bytes
info registers                      # dump all registers
continue                            # resume execution
```

### ltrace / strace Workflow

```bash
# See all library calls with arguments:
ltrace ./binary <<< "input" 2>&1

# See all system calls:
strace ./binary <<< "input" 2>&1 | grep -iE "read|write|open"
```

`ltrace` is especially useful for crackme-style challenges because it reveals `strcmp` calls and their exact arguments, even when the binary is stripped.

### XOR Encoding Pattern

A common pattern in CTF reversing challenges:

```
encoded_data XOR key = plaintext
```

If the key and encoded data are both stored in the binary's `.data` section, the plaintext can be recovered statically without running the binary.

---

## CTF Practice

Participated in the **Adwa CTF × SMU** event. Applied reverse engineering techniques alongside web exploitation skills across multiple challenges. Write-ups documented separately.

---

## Key Takeaways

- Always start with static analysis (`file`, `strings`, `objdump`) before running anything
- `ltrace` often reveals the answer faster than manual disassembly for simple crackmes
- Binaries can lie — signal handlers and fake output paths are common CTF tricks
- GDB is essential for understanding runtime behavior that static analysis cannot reveal
- The `.rodata` section is the first place to look for hardcoded strings and comparison targets
