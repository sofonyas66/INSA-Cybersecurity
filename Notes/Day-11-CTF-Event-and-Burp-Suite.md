# Day 11 — CTF Event & Burp Suite

**Date:** Saturday April 25, 2026

## Morning Session: CTF Event

Participated in a Capture The Flag event. Found **2 flags**.

---

### Challenge 1: The Courier's Envelope (HTTP Header Steganography)

**Category:** Web / Recon  
**Flag:** `BB{h3ad3rs_ar3_fun_t0_3xpl0r3}`

**Core idea:** The flag was hidden in a custom HTTP response header (`x-session-flag`), base64-encoded. The visible page HTML was a decoy — the real secret was in the *envelope* (headers), not the body.

**Approach:**
1. Fetched the URL with `curl -s -D -` to dump all response headers
2. Spotted non-standard header `x-session-flag` with a base64-looking value
3. Decoded with `echo "QkJ7aDNhZDNyc19hcjNfZnVuX3QwXzN4cGwwcjN9" | base64 -d`

**Key takeaway:** Always inspect HTTP headers — not just the body. Custom `x-*` headers are worth examining, especially in CTFs. Base64 is a common encoding to recognize on sight.

> Full write-up in challenges repo.

---

### Challenge 2: NexaCore — Secrets in Client-Side Assets

**Category:** Web / Source Review  
**Flag:** `BB{wait_you_got_it_huh}`

**Core idea:** A developer left a hardcoded base64-encoded `apiToken` inside a JavaScript file (`js/main.js`) that was publicly accessible. The HTML source revealed the script tag.

**Approach:**
1. Viewed page source → found `<script src="js/main.js"></script>`
2. Fetched the JS file with `curl -s <url>/js/main.js`
3. Found `_legacyConfig` object with a commented-out `apiToken` value
4. Decoded with `echo "QkJ7d2FpdF95b3VfZ290X2l0X2h1aH0" | base64 -d`

**Key takeaway:** Client-side assets (JS, CSS, HTML comments, source maps) are fully visible to anyone. Secrets hardcoded there are never safe. Always check `.js` files when doing web recon.

> Full write-up in challenges repo.

---

## Afternoon Session: Burp Suite Practice

**Target:** OWASP Juice Shop (intentionally vulnerable web app)

### Topics Covered

**Proxy — Intercepting Requests**
- Configured browser to route traffic through Burp's proxy (default `127.0.0.1:8080`)
- Captured raw HTTP requests in the Proxy > Intercept tab
- Examined request structure: method, path, headers, cookies, body
- **Forwarding** a request: passes it to the server as-is, letting the page load normally
- **Dropping** a request: blocks it entirely — server never sees it; useful for blocking unwanted requests or understanding what's essential

**Repeater**
- Sent an intercepted request to Repeater (`Ctrl+R`)
- Manually modified parameters and resent without re-browsing
- Observed how the response (status code, body, length) changed depending on what was modified
- Key use: testing how the server handles different input values without automating anything

**Request Manipulation in Repeater**
- Changed parameter values (e.g., product IDs, user fields) → server returns different data or errors
- Sent the same request multiple times with different values → compared responses side by side
- Differences in status codes (200 vs 400 vs 500) and response length are meaningful signals

---

## Assignment Given

Burp Suite practical tasks covering:
1. Repeater manipulation — adding parameters, changing headers (`User-Agent`, `X-Test`), modifying cookies, switching HTTP methods
2. Intruder — brute-forcing a login parameter with a small wordlist, observing response differences
3. Proxy + cURL — routing curl traffic through Burp
4. Request replay with cURL — capturing in Burp, replaying via curl with modifications
5. cURL flags — `-X`, `-H`, `-d`, `-i`, `-L`, `--proxy`