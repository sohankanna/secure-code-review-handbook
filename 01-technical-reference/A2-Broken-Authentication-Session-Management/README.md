# A2: Broken Authentication & Session Management — Technical Reference

## 1. Executive Summary
According to the **OWASP Code Review Guide 2.0 (p. 59)**, authentication and session management are the primary shields for protecting confidential files and user data. Authentication identifies *who* the user is, while Session Management maintains that identity across multiple web requests.

A failure in this category allows attackers to compromise passwords, keys, or session tokens, or to exploit implementation flaws to assume other users' identities (identity theft).

---

## 2. Authentication: The Gateway (PDF p. 59-60)
Authentication is not just a login page; it is a complex flow that must be protected at every step.

### A. Information Leakage in Login Flows
A common mistake is providing overly descriptive error messages.
*   **Vulnerable Pattern:** "Login failed: User 'admin' exists but password was incorrect."
*   **The Flaw:** This allows an attacker to "harvest" valid usernames. 
*   **Reviewer Action:** Ensure the application returns a generic message like *"Invalid username or password"* regardless of which part of the credential was wrong.

### B. Password Complexity and Storage
The guide (p. 60) establishes specific baselines for "Secure" passwords:
1.  **Minimum Length:** Passwords shorter than **10 characters** are considered weak.
2.  **Passphrases:** Should be encouraged over complex, hard-to-remember strings.
3.  **Storage:** Credentials must **never** be stored in clear text. They must be hashed (using algorithms like Argon2 or bcrypt) and salted.

### C. Out-of-Band (OOB) & MFA (PDF p. 64-65)
The guide identifies three ways to authenticate:
1.  **Something you know:** Password, PIN.
2.  **Something you have:** Mobile phone (SMS), RSA token, Cash card.
3.  **Something you are:** Biometrics (Fingerprint, Iris scan).

**Reviewer Insight:** When reviewing Multi-Factor Authentication (MFA), look for **Rate Limiting**. If an attacker can guess a 6-digit SMS code 1 million times without being blocked, the OOB channel is useless.

---

## 3. Session Management: The Persistent Identity (PDF p. 66)
Since HTTP is a stateless protocol, the application uses a **Session ID** to remember the user. If an attacker steals this ID, they *become* that user without ever needing a password.

### A. The Anatomy of a Secure Session ID
For a session ID to be "Review-Ready," it must meet these criteria:
*   **Length/Entropy:** Must be at least **128 bits** (16 bytes) long to prevent brute-force guessing. It must provide at least **64 bits of entropy** (randomness).
*   **Meaninglessness:** The ID content must be random. If an ID looks like `UserID_Timestamp_Role`, an attacker can predict the next ID in the sequence.
*   **Generic Naming:** Change default framework names (like `JSESSIONID` or `PHPSESSID`) to something generic like `id` to avoid giving away the server technology to attackers.

### B. The "Cookie Flag" Trinity
As a reviewer, you must check the code that sets cookies for three specific flags:
1.  **Secure Flag:** Forces the cookie to be sent *only* over encrypted (HTTPS) connections.
2.  **HttpOnly Flag:** Prevents JavaScript from accessing the cookie, which stops **Session Hijacking via XSS**.
3.  **SameSite Attribute:** (Modern addition) prevents Cross-Site Request Forgery (CSRF).

---

## 4. Common Attack Vectors to Review (PDF p. 68-69)
1.  **Session Hijacking:** Stealing a valid session-id via XSS or sniffing unencrypted traffic.
2.  **Session Fixation:** An attacker "fixes" a session ID on a victim's browser *before* they log in. If the application doesn't change the ID upon login, the attacker can use the same ID to access the account.
    *   **The Fix:** Always call `session.invalidate()` (Java) or `Session.Abandon()` (.NET) upon successful login and issue a brand-new ID.
3.  **Session Elevation:** When a user upgrades from a Guest to a User (or User to Admin), but the Session ID remains the same.

---

## 5. Termination & Expiration (PDF p. 67-68)
Sessions should not last forever. Review the code for:
*   **Idle Timeouts:** Automatically ending the session after 15-30 minutes of inactivity.
*   **Absolute Timeouts:** Ending the session after a set time (e.g., 8 hours) regardless of activity.
*   **Proper Logout:** The "Logout" button must actually destroy the session on the **Server Side**, not just delete the cookie on the client.

---

## 6. Secure Code Reviewer’s Checklist (A2)
Use these specific "Search and Verify" tasks:

- [ ] **Protocol Check:** Are all login pages and session-related actions forced over **TLS/SSL**?
- [ ] **Case Sensitivity:** Are usernames case-insensitive? (e.g., 'Smith' and 'smith' should be the same user to prevent confusion).
- [ ] **Lockout Policy:** Does the code implement temporary account lockouts after 5-10 failed attempts?
- [ ] **Session Re-generation:** Search for the login function. Does it explicitly destroy the old session and create a new one?
- [ ] **Cookie Flags:** Check the `web.config` or code-behind. Are `requireSSL="true"` and `httpOnlyCookies="true"` set?
- [ ] **CAPTCHA Logic:** Is the CAPTCHA randomized in length and font? Does it block the user after 3-5 failed guesses?
- [ ] **Password Reset:** Does the "Forgot Password" link expire after a short time (e.g., 24-48 hours)? Is the user notified via email when their password changes?

---

## 7. Platform Specific Red Flags (Sample Analysis)

### .NET (Sample 8.1 / 8.5)
*   **Red Flag:** Missing `RequireHttpsAttribute()` in the `global.asax` file.
*   **Configuration:** Look for `<sessionState timeout="15" />` in `web.config`. If the timeout is missing, it defaults to 20 or 30 minutes, which may be too long for high-security apps.

### Java (Sample 8.3 / 8.6)
*   **Requirement:** In `web.xml`, ensure the `<cookie-config>` section has `<secure>true</secure>`.
*   **Requirement:** Check for `<session-timeout>1</session-timeout>` (note: Java measures this in minutes).

### PHP (Sample 8.4 / 8.9)
*   **Red Flag:** `session.use_trans_sid = 1`. This is dangerous because it puts the Session ID in the URL.
*   **Requirement:** `session.use_only_cookies = 1` must be set in `php.ini`.

---
*Reference: OWASP Code Review Guide 2.0 (Pages 58-70)*
