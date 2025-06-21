# ✅ Brute Force Protection (DVWA - Impossible Level)


## 📄 Executive Summary

This report documents the successful brute-force of DVWA’s login form under its highest security setting ("Impossible"), which employs CSRF protection. By automating token extraction in Burp Suite, the CSRF mechanism was bypassed, resulting in successful credential brute-forcing.


### Security Mechanism:

- The login form uses a **CSRF token (`user_token`)** that changes with every request, effectively blocking automated brute-force attempts unless the token is dynamically updated.
    

---

## 🎯 Objective

The goal was to brute-force the `admin` user's password on DVWA's **Brute Force** module, under **Impossible** security level, which includes CSRF token validation and session enforcement mechanisms.

---

## 🧪 Attack Workflow Summary

### 1️⃣ Initial Observation

A login attempt using an incorrect password returns either:

- `Login failed`
    
- `Incorrect CSRF token`
    

This inconsistency revealed that the token was validated and expired between sessions, requiring live extraction.

---

### 2️⃣ Request Interception (Burp Suite)

Captured a valid login attempt in Burp Suite. The POST request format:

![Burp Intruder Settings](screenshots/burp_intruder_settings.png)

### 🔐 Identified Dynamic Parameter:

- `user_token` — a CSRF token unique per session/request.
    

---

### 3️⃣ Configuring Burp Intruder

![Burp Intruder Settings](screenshots/burp_intruder_settings.png)

**Attack Type:** Pitchfork  
**Payload Position 1:** `password`  
**Payload Position 2:** `user_token`

**Passwords:** Loaded from `/usr/share/wordlists/sectlists/Passwords/cirt-default-passwords.txt`
**Payload Type:** Simple list

![Payload 1](screenshots/payload1.png)



**Token:** Extracted dynamically from server responses using **Grep - Extract**
**Payload Type:** Recursive Grep

![Payload 2](screenshots/payload2.png)



---

### 4️⃣ Token Extraction Logic

Set Grep - Extract to scan each server response for:

![Grep Extract Settings](screenshots/grep_extract_settings.png)

`<input type="hidden" name="user_token" value="...">`

Burp dynamically parsed and injected the new CSRF token into every subsequent request.

---

### 5️⃣ Match Term for Results

![Grep Match Settings](screenshots/grep_match_settings.png)

Configured Grep - Match for `"failed"` (referring to **"Login Failed"**) and `"incorrect"`(referring to **"CSRF Token Incorrect"**) to filter failed attempts and highlight potential successes.

---

### 6️⃣ Attack Launch

- Burp iterated over password/token combinations.
    
- Each request retrieved a valid CSRF token and injected it.
    
- Responses were analyzed for anomalies in length and content to identify successful logins.
    

---

## 🔬 Forensic Packet Analysis

### 📷 Screenshot A – Full Intercepted Session

![Wireshark Intercept All Packets](screenshots/wireshark_intercept_all_packets.png)

Captured full TCP session, showing:

- TCP handshake
    
- POST login attempt with valid token
    
- HTTP `302` redirect to `login.php`
    
- Follow-up `GET` request returning HTTP `200 OK`
    

---

### 📷 Screenshot B – Packet Highlighted

#### ✅ Key Packet: HTTP 200 OK

![Wireshark Intercept Specific Packet](screenshots/wireshark_intercept_specific.png)

- **Packet 6297** in stream
    
- Response header confirms successful page load:
    
    
    ```psqlg
    HTTP/1.1 200 
    OK Content-Type: text/html 
    Server: Apache/2.4.25 (Debian)
    ```
    
- Packet 6297 showed a 200 OK response. The HTTP body length (1132 bytes) matched a successful login response. No error message like "Login failed" or "Invalid CSRF token" was found.


#### 🔍 Response Breakdown:

- Server accepted session
    
- Returned proper content
    
- No "Login failed" message
    
- CSRF token accepted

![Pitchfork Results](screenshots/pitchfork_results.png)
    

---

## 💥 Impact

- Bypassed CSRF protection via token extraction automation.
    
- Able to brute-force user accounts even on the **"impossible"** security level.
    
- Demonstrates how lack of rate-limiting or CAPTCHA makes CSRF ineffective alone.
    

---

## 🔧 Exploit Mechanism Breakdown

| Component         | Description                                                                        |
| ----------------- | ---------------------------------------------------------------------------------- |
| `user_token`      | Extracted from previous responses using Grep-Extract                               |
| `password`        | Fetched from `/usr/share/wordlists/sectlists/Passwords/cirt-default-passwords.txt` |
| HTTP Status `200` | Indicated success; absence of `Login failed` flag                                  |
| POST/GET pattern  | Confirmed by analyzing packet sequence in Wireshark                                |

**Resulting in:**

| Indicator          | Value                                |
| ------------------ | ------------------------------------ |
| Successful payload | `password` (row 833)                 |
| Status Code        | `200 OK`                             |
| Token Reused?      | No – dynamic update via Grep Extract |
| Bypass Successful? | ✅ Yes                                |

---

## 🛡️ Recommendations

- **[HIGH]** Implement brute-force detection and rate-limiting mechanisms.

- **[HIGH]** Add CAPTCHA to login forms on sensitive security levels.

- **[MEDIUM]** Invalidate CSRF tokens after single use.

- **[LOW]** Ensure login failures return consistent and generic responses.