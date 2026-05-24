## CTF Writeup – MedSupply Variant (Adwa CTF)

**Target:** `34.28.116.206`  
**Role started as:** Low‑privilege supplier  
**Flag captured:** `Adwa_CTF{H0r1z0nta1_pr1v_35c}`

---

### Step 1 – Initial Access (Supplier Registration)

- Registered a supplier account at `/register` with email `supplier_test@codoteam.com` and password `Test12345`.
- Logged in – the application required OTP verification.
- **OTP Bypass:** Intercepted the `/verify-otp` response and changed `200 OK` to `302 Found` with `Location: /dashboard`. This granted a fully authenticated session without providing a real OTP.

---

### Step 2 – Information Gathering & UUID Leak

- Visited `/public-feed` and viewed the page source.
- Found JavaScript calling `/api/public-orders`.
- Directly accessed `/api/public-orders` and obtained a JSON list of orders, including `requesting_hospital_id` for three hospitals:

| Hospital | UUID |
|----------|------|
| Addis General Hospital | `e376c9a4-682a-4307-b0e3-d9e29db1c80a` |
| Bahir Dar Clinic | `699730c1-312d-48d2-a72f-39ff607bec27` |
| Hawassa Medical Center | `b75cac4e-1aa2-4392-8d37-07a198e531e6` |

- Tried the internal API endpoint `/internal/api/account/details?user_id=<UUID>` but it returned only hospital data (no flags) – the endpoint was working but required a higher‑privileged session to reveal flags.

---

### Step 3 – Password Cracking to Escalate to Hospital Admin

- From the internal API we obtained the bcrypt password hash of Addis General Hospital:  
  `$2b$08$Ssg8oX5ljPVb5PgyI1sza.0eoJT1y.iDHzUqu6ZEvUZVeudT18L8G`
- Used **John the Ripper** with the `rockyou.txt` wordlist to crack the hash:
  ```bash
  john --format=bcrypt --wordlist=rockyou.txt addis.hash
  ```
- The password was cracked as: **`30secondstomars`**

---

### Step 4 – Login as Hospital Admin & Capture Flag

- Logged in to `http://34.28.116.206/login` with:
  - Email: `addis.general@medsupply.local`
  - Password: `30secondstomars`
- Upon successful login, the dashboard (`/dashboard`) displayed the flag directly on the page:

  **`Adwa_CTF{H0r1z0nta1_pr1v_35c}`**

- No other flags were found on this target.

---

## Summary of Vulnerability Exploited

| Phase | Vulnerability |
|-------|---------------|
| OTP bypass | Missing server‑side session validation after OTP step |
| UUID leak | Public feed exposed internal identifiers |
| Password hash leak | Internal API returned bcrypt hashes of hospital accounts |
| Weak password | Hospital used a common password from `rockyou.txt` |
| Horizontal privilege escalation | Low‑privilege supplier → hospital admin via cracked credentials |

---

## Tools Used

- Browser + Burp Suite (OTP bypass)
- John the Ripper (password cracking)
- Manual API exploration
