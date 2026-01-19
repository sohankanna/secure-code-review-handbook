
# Forgot Password & CAPTCHA: Secure Recovery and Automation Defense

## 1. Executive Summary
While the primary login screen is the "Front Door" of an application, the "Forgot Password" function is effectively the "Back Door" or the "Emergency Exit." Attackers love this feature because it often has weaker security controls than the main login.

Similarly, **CAPTCHA** (Completely Automated Public Turing test to tell Computers and Humans Apart) is a control used to prevent automated tools from brute-forcing these recovery mechanisms or flooding the system with junk data.

As a Secure Code Reviewer, you must ensure that the recovery process is not a bypass for authentication and that the CAPTCHA implementation is not easily defeated by scripts.

---

## 2. Forgot Password: The Logical vulnerabilities
The OWASP Guide (p. 61) warns that "Direct Authentication" patterns (where the system authenticates a user based on a secret question or immediately allows a password reset) are fraught with peril.

### A. The "Clear Text" Red Flag
If the application sends the **old password** to the user via email, this is a Critical Severity Finding.
*   **The Implication:** For the system to send the old password, it must have been stored in **Plain Text** or **Reversible Encryption**.
*   **The Standard:** Passwords must be hashed (one-way). Therefore, the system *cannot* know the old password. It can only allow the user to set a *new* one.

### B. User Enumeration (Information Leakage)
This is a logic flaw where the application reveals if a user exists.

**Vulnerable Logic:**
1.  User enters `attacker@evil.com`.
2.  System responds: *"Error: This email is not registered."*
3.  User enters `admin@company.com`.
4.  System responds: *"Success: Email sent."*

**The Risk:** An attacker can script this form to check thousands of emails to build a target list for phishing or brute force.

**Secure Logic:**
The response must always be consistent.
> "If an account matches the email address you entered, you will receive a reset link shortly."

### C. The "Shared Secret" Logic (Security Questions)
Using "Security Questions" (e.g., "What is your mother's maiden name?") is considered a "Shared Secret."
*   **The Flaw (PDF p. 62):** These secrets are often easily guessable via social media (OSINT).
*   **Reviewer Requirement:** "A shared secret must always be stored in hashed or encrypted format in a database."
*   **Transmission:** Never send these secrets back to the user in clear text.

---

## 3. Secure Design Pattern: The Reset Token
The Code Review Guide recommends a specific workflow for password recovery. When reviewing the code, trace this exact path:

1.  **User Request:** User enters username/email.
2.  **Verification:** System verifies user exists (silently).
3.  **Token Generation:** System generates a **high-entropy, random token** (not a sequential ID).
4.  **Storage:** System stores the token in the DB, hashed, with a **short expiration** (e.g., 20 minutes).
5.  **Transmission:** System emails a link containing the token to the registered address.
6.  **Redemption:** User clicks link. System verifies token validity and expiration.
7.  **Reset:** User provides a new password.
8.  **Invalidation:** System deletes the token immediately.

### Code Review Checklist for Reset Tokens:
*   **Randomness:** Is `SecureRandom` used? (Java) or `RNGCryptoServiceProvider` (.NET)?
*   **Storage:** Is the token hashed in the database? (If the DB is leaked, the tokens shouldn't be usable).
*   **Binding:** Is the token bound to the specific user? (Token A should only reset User A).

---

## 4. CAPTCHA: Defense Against Automation
Section 8.5 of the guide (p. 62) details CAPTCHA as an access control technique. It is used to stop "Credential Stuffing" (testing stolen passwords) and "Spamming."

### A. The Statistics of Guessing
The guide highlights a mathematical flaw in many CAPTCHA designs.
*   **Scenario:** A CAPTCHA asks the user to "Pick the cat" from 4 images.
*   **The Math:** An automated bot acts randomly. It has a **25% chance** of guessing correctly.
*   **The Vulnerability:** If the application allows the user (bot) to try 100 times, the bot will succeed ~25 times.
*   **The Fix:** "Do not allow the user to enter multiple guesses after an incorrect attempt." After a failure, the CAPTCHA image/logic must change completely.

### B. Visual Security Principles (What to check in Code/Config)
If the application generates its own text-based CAPTCHAs, the code must implement specific distortions to defeat OCR (Optical Character Recognition) bots.

**Review Check (PDF p. 62-63):**
1.  **Random Length:** Don't always use 6 characters. Use 4, then 7, then 5.
2.  **Random Size/Font:** Varying fonts confuse OCR algorithms.
3.  **Waving:** The text should follow a sine wave, not a straight line.
4.  **Anti-Segmentation (Critical):** "Lines must cross only some of the CAPTCHA letters."
    *   *Why?* Bots try to chop the image into single letters (Segmentation). If a line connects two letters, the bot thinks they are one shape and fails.
5.  **Background Noise:** Dots, specks, and lines in the background.

### C. Implementation Logic Flaws
**1. Client-Side Validation (The "Hidden Field" Trick)**
*   **Vulnerable Code:** The server sends the CAPTCHA answer in a hidden HTML field (`<input type="hidden" value="X79A">`) so the JavaScript can check it.
*   **The Attack:** The bot simply reads the hidden field and submits it.
*   **The Rule:** The answer must **never** leave the server. The client sends the user's guess, and the server compares it to the stored answer in the session.

**2. Replay Attacks**
*   **The Flaw:** An attacker solves one CAPTCHA manually, captures the request (Token + Answer), and replays it 1,000 times.
*   **The Fix:** CAPTCHA tokens must be **One-Time Use**. Once verified (pass or fail), invalidate the token.

---

## 5. Code Examples & Review Scenarios

### Scenario A: Secure Token Generation (C#)
This code snippet demonstrates how to generate a token for the "Forgot Password" link. It uses a cryptographically secure generator (CSPRNG), not `Random()`.

```csharp
using System.Security.Cryptography;

public string GenerateResetToken() {
    // 32 bytes provides 256 bits of entropy
    byte[] randomBytes = new byte[32];
    
    // Use RNGCryptoServiceProvider (Strong) not Random (Weak)
    using (var rng = new RNGCryptoServiceProvider()) {
        rng.GetBytes(randomBytes);
    }
    
    // Convert to a URL-safe string
    return Convert.ToBase64String(randomBytes)
        .Replace("+", "-").Replace("/", "_").Replace("=", "");
}
```

### Scenario B: Generic Error Handling (Java)
This logic ensures that an attacker cannot enumerate users via the Forgot Password form.

```java
public void handleForgotPassword(String email) {
    User user = userDAO.findByEmail(email);
    
    if (user != null) {
        String token = generateSecureToken();
        storeTokenHash(user, token);
        emailService.sendResetLink(user, token);
    } else {
        // LOGIC: Do nothing, or perhaps wait a random amount of time 
        // to mimic the time taken to send an email (Timing Attack Defense)
    }
    
    // ALWAYS return the same message to the UI
    displayMessage("If an account exists for " + email + ", instructions have been sent.");
}
```

---

## 6. Out-of-Band (OOB) Communication Risks
The guide (p. 63) warns about the channel used for recovery (Email/SMS).

**ZitMo (Zeus in the Mobile):**
Banking Trojans on mobile phones can intercept SMS messages.
*   **Reviewer Question:** Does the application allow the user to choose the delivery method?
*   **Risk:** If I am attacking a user's bank account on their PC, I might prefer the OTP (One Time Password) go to their Email (which I might have hacked) rather than their Phone (which I don't have).
*   **Secure Code:** Hard-code the delivery channel to the most secure verified option available, do not let the current user (attacker) switch channels easily.

---

## 7. Secure Code Reviewer’s Checklist (Forgot Password & CAPTCHA)

### Forgot Password Flow
- [ ] **No Old Passwords:** Does the system send a reset link/token, NEVER the old password?
- [ ] **Questions:** Are "Security Questions" treated as passwords? (Hashed in DB, not visible to support staff).
- [ ] **Expiration:** Do reset links expire after a short window (e.g., 20 mins)?
- [ ] **Session Kill:** Does resetting the password invalidate all active sessions for that user? (Crucial if the account was stolen).
- [ ] **Enumeration:** Is the error message identical for valid and invalid emails?

### CAPTCHA Implementation
- [ ] **Server-Side Check:** Is the validation performed 100% on the server?
- [ ] **No Replay:** Can the same CAPTCHA token be reused? (Test this!).
- [ ] **Visuals:** Does the image use rotation, scaling, and noise?
- [ ] **Complexity:** Is the logic complex enough to stop basic bots (no simple math like `1+1`)?
- [ ] **Alternatives:** Is there an audio alternative for accessibility (ADA Compliance)?

---
*End of Forgot Password & CAPTCHA Notes*
*Reference: OWASP Code Review Guide 2.0 (Sections 8.4 - 8.5)*
