# Authentication Logic Review: The Gatekeeper's Design

## 1. The Philosophy of Authentication
Authentication is the process of verifying a digital identity. In a secure code review, we are not just looking for "bugs" in the code; we are looking for flaws in the **logic flow**. 

The guide emphasizes that authentication is the "Gateway" to your application. If the logic here is flawed, every other security control (Authorization, Logging, Data Protection) becomes irrelevant because the attacker *becomes* the legitimate user.

### The "All or Nothing" Rule
Authentication must be binary. You are either fully authenticated, or you are not. Code reviews often find "gray areas" where a user is partially authenticated (e.g., they entered a correct username but failed the password, yet the system allows them to see a profile picture). This is a logic flaw.

---

## 2. Reviewing the Login Mechanism
The login screen is the most attacked surface of any application.

### A. Transport Layer Protection (TLS/SSL)
**The Vulnerability:** 
Some developers only secure the "action" URL (where the data is posted) but leave the login form page itself on HTTP.
*   **The Risk:** A Man-in-the-Middle (MitM) attacker can modify the form before the user types in it, changing the destination URL to a malicious server.

**Code Review Check:**
*   Ensure the **Login Page itself** is forced over HTTPS.
*   In .NET MVC, look for the `RequireHttpsAttribute` in the `Global.asax` file or controller.

**Secure Code Pattern (.NET Sample 8.1):**
```csharp
public static void RegisterGlobalFilters(GlobalFilterCollection filters) {
    // This forces SSL for the entire application, preventing the MITM scenario.
    filters.Add(new RequireHttpsAttribute()); 
}
```

### B. Username Case Sensitivity (Logic Flaw)
**The Vulnerability:** 
Treating `Smith` and `smith` as two different users.
*   **The Risk:** This causes massive confusion and potential account takeovers if an attacker registers a name similar to an administrator.
*   **The Fix:** Emails and Usernames should be **Case Insensitive**. The backend should normalize the string (e.g., `ToLower()`) before comparing it in the database.

### C. Information Leakage in Failure Messages
This is one of the most common findings in a code review. The application tells the attacker *too much*.

**Vulnerable Logic Example:**
```java
// VULNERABLE LOGIC
if (userExists(username)) {
    if (checkPassword(username, password)) {
        return "Login Success";
    } else {
        return "Error: Password Incorrect"; // LEAK: Tells attacker the User Exists!
    }
} else {
    return "Error: Username not found"; // LEAK: Tells attacker to try a different name.
}
```

**The Enumeration Attack:**
If the application behaves differently (different error message, or even taking longer to respond) for a valid user vs. an invalid user, an attacker can write a script to find every valid username in your system.

**Secure Logic Example:**
```java
// SECURE LOGIC
// Regardless of which check fails, the output is identical.
if (validateUser(username, password)) {
    return "Login Success";
} else {
    return "Error: Invalid Username or Password"; // GENERIC MESSAGE
}
```

---

## 3. Password Policy & Complexity Logic
You cannot control how smart your users are, but your code can enforce a baseline of "smart" behavior.

### A. Length & Complexity (Page 60)
The guide explicitly states: **"Passwords shorter than 10 characters are considered weak."**
*   **Reviewer Action:** Check the validation logic during account creation and password change.
*   **Regex Check:** Ensure the code enforces a mix of:
    *   Upper Case
    *   Lower Case
    *   Numbers
    *   Special/Punctuation Characters

### B. Password History (Reuse Prevention)
For internal enterprise systems, the code should prevent a user from setting their password to one they used previously.
*   **Logic Check:** Does the database store a hash history (e.g., the last 5 passwords)?
*   **Code Flow:** When `ChangePassword()` is called, the code must query the `PasswordHistory` table. If the new hash matches any of the last 5 hashes, reject the change.

---

## 4. Brute Force Prevention Logic
Attackers will try millions of passwords against a valid username. The application code **must** have logic to stop this.

### A. Account Lockout
**The Rule:** Implement a temporary lockout after a set number of failures (e.g., 5 attempts).

**Reviewer Checklist for Lockout Code:**
1.  **Counter:** Is there a counter in the database or cache that increments on every failed login?
2.  **Reset:** Does the counter reset to 0 after a *successful* login? (If not, a legitimate user who makes 1 mistake every day will eventually get locked out).
3.  **Duration:** Is the lockout temporary (e.g., 15 minutes)? Permanent lockouts can lead to **Denial of Service (DoS)** attacks, where a hacker intentionally locks out the admin by guessing wrong 5 times.

### B. Rate Limiting (Throttling)
Instead of locking the account, the code can artificially slow down.
**Logic:**
*   Attempt 1: Respond immediately.
*   Attempt 2: Wait 2 seconds.
*   Attempt 3: Wait 5 seconds.
*   Attempt 10: Wait 30 seconds.

This makes brute force attacks mathematically impossible to execute in a reasonable timeframe without denying access to the legitimate user.

---

## 5. The "Forgot Password" Mechanism (Section 8.4)
This is often the weakest link in authentication. It allows a user to recover an account without the primary credential (the password).

### A. Secure Recovery Logic
**Vulnerable Pattern (Direct Authentication):**
*   User enters email -> System sends the old password in clear text. (**CRITICAL FAIL**)

**Secure Pattern (Reset Token):**
1.  User enters email.
2.  System generates a high-entropy, random **Token**.
3.  System stores Token in DB with a short expiry (e.g., 20 minutes).
4.  System sends a Link containing the Token to the user's email.
5.  User clicks link -> System validates Token -> User sets **NEW** password.

### B. Enumeration Prevention
**The Flaw:** If a user enters an email into "Forgot Password," and the site says *"Email not found"*, the attacker knows that email is not registered.
**The Fix:** The system should always say: *"If that email exists in our system, we have sent a reset link."*

### C. Logic Traps to Avoid
*   **The "Double Dip":** Do not allow the "Forgot Password" feature to be used to spam a victim. Rate limit this interface just like the login page.
*   **Social Engineering:** Never allow a password reset via phone/chat unless the agent has a strict verification protocol.

---

## 6. CAPTCHA Design & Logic (Section 8.5)
CAPTCHA (Completely Automated Public Turing test to tell Computers and Humans Apart) prevents automated bots from flooding login or registration pages.

### Security Principles for CAPTCHA Code
If the application generates its own CAPTCHAs (rather than using Google reCAPTCHA), the code reviewer must ensure:
1.  **Random Length:** Don't always use 5 letters. Use 4-7. Fixed length allows attackers to optimize their OCR (Optical Character Recognition) bots.
2.  **Random Size/Font:** Use different fonts and sizes to confuse OCR.
3.  **Waving/Distortion:** The image should be warped or waved.
4.  **No Replay:** Once a CAPTCHA solution is used (successful or not), that specific image/token must be invalidated. An attacker shouldn't be able to solve one CAPTCHA and use the token 100 times.
5.  **Complexity:** Do not use simple logic like "What is 1+1?". Bots can solve math faster than humans.

---

## 7. Multi-Factor Authentication (MFA) & Out-of-Band (OOB)
**Section 8.6** describes "Out-of-Band" as communicating via a channel separate from the browser (e.g., SMS, Email, Hardware Token).

### Logic Review for MFA
When reviewing code that handles SMS or Token verification:
1.  **Origin of Destination:**
    *   **Vulnerable:** The user enters their username AND the phone number to send the SMS to. (The attacker will just enter their own phone number!).
    *   **Secure:** The user enters the username. The system looks up the **pre-registered** phone number in the database and sends the SMS there. The user never types the destination.

2.  **Token Entropy:**
    *   Ensure the PIN sent is random.
    *   Ensure the PIN has a short lifespan (e.g., 5 minutes).

3.  **Rate Limiting (Critical):**
    *   The interface that accepts the PIN must be rate limited.
    *   If the PIN is 4 digits (0000-9999), an attacker can guess it in seconds using a script. The code must block attempts after 3-5 wrong guesses.

4.  **Device Awareness:**
    *   The guide mentions **ZitMo (Zeus in the Mobile)** malware. If the user is logging in via a mobile phone, and the SMS is sent to that *same* phone, malware can intercept it.
    *   **Best Practice:** Where possible, separate the channels (e.g., Login on PC, receive SMS on Phone).

---

## 8. Summary Checklist for Reviewers

- [ ] **Protocol:** Is `RequireHttps` enabled globally?
- [ ] **Feedback:** Are error messages generic ("Invalid User/Pass")?
- [ ] **Storage:** Are passwords hashed (Argon2/Bcrypt) and salted? (Never plain text).
- [ ] **Lockout:** Is there logic to handle >5 failed attempts?
- [ ] **Recovery:** Does "Forgot Password" send a temporary token, not the old password?
- [ ] **MFA:** Does the code use the registered phone number, not a user-supplied one?
- [ ] **CAPTCHA:** Is the CAPTCHA invalidated immediately after use?

---
*Reference: OWASP Code Review Guide 2.0 (Sections 8.1 - 8.6)*
