# A6: Sensitive Data Exposure — Technical Reference

## 1. Executive Summary
**Sensitive Data Exposure** occurs when an application fails to adequately protect information such as passwords, credit card numbers, health records, or personal data (PII).

According to the **OWASP Code Review Guide 2.0 (p. 117)**, attackers do not "hack" cryptography; they hack the **implementation** of cryptography. They steal keys, intercept unencrypted traffic, or exploit weak algorithms (like MD5 or DES).

**The Reviewer's Mandate:**
You must identify **what** data is sensitive, **where** it is stored/sent, and **how** it is protected. You must distinguish between **Encoding** (reversible/insecure) and **Encryption** (secure).

---

## 2. The Three States of Data
Code Reviewers must check for protection in three distinct states:

1.  **Data at Rest:** Data sitting in a database, file system, or backup tape.
    *   *Defense:* Symmetric Encryption (AES) or Hashing.
2.  **Data in Transit:** Data moving over the network (Internet or Intranet).
    *   *Defense:* TLS/SSL (Transport Layer Security).
3.  **Data in Process:** Data currently in the application's memory.
    *   *Defense:* Secure memory management (e.g., using `SecureString` in .NET to prevent swapping to disk).

---

## 3. Cryptography Definitions (The "Vocabulary of A6")
Page 119 of the guide provides a critical table. A developer confusing these terms is a major red flag.

| Term | Definition | Key Characteristic | Security Value |
| :--- | :--- | :--- | :--- |
| **Encoding** | Transforming data format (e.g., Base64, Hex). | **Reversible** without a key. | **None.** Hides data from eyes, not attackers. |
| **Encryption** | Transforming data using a mathematical algorithm and a secret key. | **Reversible** ONLY with the key. | **High** (Confidentiality). |
| **Hashing** | Transforming data into a fixed-length "fingerprint" (Digest). | **One-Way** (Not reversible). | **High** (Integrity / Passwords). |
| **Salt** | Random data added to input before hashing. | Unique per user. | Prevents **Rainbow Table** attacks. |
| **Entropy** | Randomness. | Unpredictable. | Essential for key generation. |

---

## 4. Cryptographic Algorithms: The Good and The Bad
The guide (p. 123, 129) is explicit about which algorithms are broken.

### 🚫 The "Broken" List (Fail the Review)
*   **Symmetric:** `DES` (Too weak), `RC4` (Biased), `3DES` (Legacy/Slow).
*   **Hashing:** `MD5` (Collisions found), `SHA-0`, `SHA-1` (Deprecated by NIST/FIPS).
*   **Randomness:** `java.util.Random`, `Math.random()` (Predictable).

### ✅ The "Approved" List (Pass the Review)
*   **Symmetric:** `AES` (128-bit minimum, 256-bit preferred).
*   **Hashing:** `SHA-256`, `SHA-512`.
*   **Randomness:** `java.security.SecureRandom` (Java), `RNGCryptoServiceProvider` (.NET).

---

## 5. Key Management (The "Crown Jewels")
If the encryption key is stored in the source code, the encryption is useless.

**Reviewer Check (PDF p. 122):**
> "In any cryptographic system the protection of the key is the most important aspect."

*   **Vulnerable:** Hardcoded keys (`String key = "12345";`).
*   **Vulnerable:** Keys stored in the same database table as the encrypted data.
*   **Secure:** Keys stored in a Hardware Security Module (HSM), Windows DPAPI, or a separate Key Vault configuration.

---

## 6. Reducing the Attack Surface (PDF p. 131)
The best way to protect sensitive data is **not to have it**.

**Reviewer Questions:**
*   Does the application need to store the Credit Card number, or can it just store the "Token" from the payment processor?
*   Does the app need the user's Social Security Number / Tax ID forever, or just for the initial verification?
*   **Logging:** Are logs scrubbing sensitive data? (Reviewing `log4j` or `Logger` calls is critical here).

---

## 7. Secure Code Reviewer’s Checklist (A6)

### Data Handling
- [ ] **Identification:** Has all PII (Personally Identifiable Information) been identified?
- [ ] **Need:** Is the data absolutely necessary? Can it be truncated or tokenized?
- [ ] **Autocomplete:** Is `autocomplete="off"` set on sensitive HTML forms? (Prevents browser caching).

### Encryption (At Rest)
- [ ] **Algorithm:** Is `AES` used? (Not DES/RC4).
- [ ] **Key Size:** Is the key at least 128-bit (preferably 256-bit)?
- [ ] **IV (Initialization Vector):** Is a random IV used for every encryption operation? (Reusing IVs breaks encryption security).
- [ ] **Key Storage:** Are keys kept out of the source code?

### Passwords
- [ ] **Hashing:** Are passwords Hashed? (Never Encrypted, Never Plaintext).
- [ ] **Salting:** Is a unique, random salt used for every user?
- [ ] **Algorithm:** Is a slow algorithm used? (Argon2, PBKDF2, bcrypt) to prevent GPU brute-forcing.

### Transport (In Transit)
- [ ] **HTTPS:** Is TLS enforced for all pages?
- [ ] **Certificates:** Are self-signed certificates blocked in production?
- [ ] **Downgrade:** Is the server configured to disable SSL v2/v3?

---
*Reference: OWASP Code Review Guide 2.0 (Pages 117-132)*
