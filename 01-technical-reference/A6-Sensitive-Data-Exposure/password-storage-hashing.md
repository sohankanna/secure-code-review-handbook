# Password Storage & Hashing: Defense Against Offline Attacks

## 1. Executive Summary
Storing passwords in **Plain Text** is a catastrophic failure. If the database is stolen, every account is immediately compromised.

To protect passwords, we use **Cryptographic Hashing**. Unlike encryption, hashing is **One-Way**. You cannot "decrypt" a hash to get the password back. You can only hash an input and see if it matches the stored fingerprint.

However, simple hashing is no longer enough due to **Rainbow Tables**. To defeat them, we must use **Salting**.

**The Reviewer's Mandate:**
You are checking for three specific things:
1.  **Algorithm Strength:** Is the math strong? (SHA-256 vs MD5).
2.  **Uniqueness:** Is a unique random Salt used for *every* user?
3.  **Entropy:** Is the random number generator cryptographically secure?

---

## 2. The Mechanics of Hashing (The "Fingerprint")
According to the guide (p. 127), a hash function takes an arbitrary block of data (the password) and returns a fixed-size bit string (the "Message Digest").

### Key Properties for Reviewers
1.  **One-Way:** It should be mathematically impossible to reverse the process.
2.  **Deterministic:** The same input always produces the exact same output.
3.  **Collision Resistant:** Two different passwords should never produce the same hash.

### Algorithm Selection (The "Approved" List)
The guide (p. 129) references **FIPS 180-4**. As a reviewer, you must flag weak algorithms.

| Algorithm | Status | Reviewer Action |
| :--- | :--- | :--- |
| **MD5** | **BROKEN** | **High Severity Finding.** MD5 has collision vulnerabilities. It is fast, which helps attackers brute-force it. |
| **SHA-1** | **WEAK** | **High Severity Finding.** Deprecated by NIST. |
| **Windows LanMan** | **BROKEN** | **Critical Severity.** Splits passwords into 7-char chunks. Trivial to crack. |
| **SHA-256** | **SECURE** | **Pass.** Standard approved algorithm. |
| **SHA-512** | **SECURE** | **Pass.** Stronger, produces a longer digest. |

---

## 3. The Threat: Rainbow Tables (Pre-Computed Attacks)
To understand why we need "Salt," you must understand the **Rainbow Table**.

### The "Phone Book" Analogy
Imagine an attacker wants to crack a database of hashed passwords.
*   **Brute Force:** Taking a hash, then guessing "aaaaa", "aaaab", "aaaac", hashing them, and checking if they match. This takes CPU power and time.
*   **Rainbow Table:** The attacker pre-calculates the hash for every possible password (e.g., 1-10 characters) and saves them in a giant lookup table (like a Phone Book).
    *   *Password:* `qwerty` -> *Hash:* `65e84be33532...`
    *   *Password:* `password` -> *Hash:* `5f4dcc3b5aa7...`

When the attacker steals your database, they don't guess. They just look up the hash `5f4dcc3b5aa7...` in their "Phone Book" and see that the password is `password`. This happens instantly.

---

## 4. The Defense: Salting (Making Hashes Unique)
**Reference:** PDF Page 128 (Working with Salts).

A **Salt** is random data added to the password *before* it is hashed.
`Hash = SHA256( Salt + Password )`

The guide outlines three scenarios to explain why this works (Table 20 Analysis):

### Scenario A: No Salt (Vulnerable)
*   **User 1 Password:** `password` -> **Hash:** `B109F3...`
*   **User 2 Password:** `password` -> **Hash:** `B109F3...`
*   **The Flaw:** Identical passwords produce identical hashes. If the attacker cracks User 1, they have also cracked User 2. The Rainbow Table works perfectly.

### Scenario B: Fixed Salt (Vulnerable)
The developer hardcodes a salt in the application: `String salt = "WindowCleaner";`
*   **User 1:** `password` + `WindowCleaner` -> **Hash:** `E6F9DC...`
*   **User 2:** `password` + `WindowCleaner` -> **Hash:** `E6F9DC...`
*   **The Flaw:** The hashes are still identical! The attacker just generates a *new* Rainbow Table where every entry ends with "WindowCleaner". Once they generate this table (which takes a few days), they can crack the entire database instantly.

### Scenario C: Unique Random Salt (Secure)
The application generates a random salt for **every single user** and stores it in the database next to the hash.
*   **User 1 Salt:** `a0w8hs...`
*   **User 2 Salt:** `8ash87...`
*   **User 1 Hash:** SHA256(`password` + `a0w8hs...`) -> `5AA762...`
*   **User 2 Hash:** SHA256(`password` + `8ash87...`) -> `8058D4...`

**The Result:** Even though both users have the password "password", their hashes are completely different. The Rainbow Table is useless. The attacker must calculate hashes individually for *every single user*, which is computationally impossible for large databases.

---

## 5. Code Review: Generating Secure Salts
**Reference:** PDF Page 129 (Sample 12.4).

Generating the salt requires **Entropy** (Randomness). If you use a weak random number generator, the attacker can guess the salt.

### Vulnerable Logic
```csharp
// RED FLAG: Do not use System.Random for security!
Random r = new Random(); 
byte[] salt = new byte[16];
r.NextBytes(salt);
```

### Secure Logic (C# .NET)
The guide provides the following secure implementation using `RNGCryptoServiceProvider`.

```csharp
// Sample 12.4: Secure Salt Generation
private byte[] GetSalt(string input) {
    byte[] saltBytes;
    int saltSize = 16; // Minimum recommended size (128 bits)

    // 1. Use the Cryptographically Secure Generator
    RNGCryptoServiceProvider rng = new RNGCryptoServiceProvider();
    
    // 2. Create the container
    saltBytes = new byte[saltSize];
    
    // 3. Fill with non-zero secure bytes
    rng.GetNonZeroBytes(saltBytes); 
    
    // ... Combine salt with input (password) ...
    return dataWithSaltBytes;
}
```

**Reviewer Checklist for Salt Generation:**
1.  **Generator:** Is `RNGCryptoServiceProvider` (.NET) or `SecureRandom` (Java) used?
2.  **Size:** Is the salt at least **128 bits** (16 bytes)? The guide warns: *"The bits are not costly... use a value of 256-bit salt value."*
3.  **Uniqueness:** Is a new salt generated for every user creation and every password change?

---

## 6. Code Review: Hashing Implementation
**Reference:** PDF Page 130 (Sample 12.5).

Once you have the Salt and the Password, you must combine them and hash them.

### Secure Logic (C# .NET)
The guide demonstrates reading the algorithm from a config file (allowing updates without recompiling) and computing the hash.

```csharp
// Sample 12.5 Concept
private string computeHash(HashAlgorithm myHash, string input) {
    // 1. Convert string to bytes (Always check Encoding!)
    byte[] data = Encoding.UTF8.GetBytes(input);
    
    // 2. Perform the Hash
    byte[] hashData = myHash.ComputeHash(data);
    
    // 3. Convert back to Hex String for storage
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < hashData.Length; i++) {
        sb.Append(hashData[i].ToString("x2"));
    }
    return sb.ToString();
}
```

**Reviewer Red Flags:**
*   **Hardcoded Algorithm:** `HashAlgorithm.Create("MD5")` -> **Fail**.
*   **Encoding Issues:** Using `ASCII.GetBytes` might fail for complex passwords/international characters. `UTF8` is preferred.
*   **Truncation:** Ensure the database column is wide enough to hold the full hash (e.g., SHA-512 requires 128 characters in Hex). If the DB truncates it, you lose security.

---

## 7. Compliance & Export Laws (PDF p. 127)
A unique note in the OWASP guide regarding legal compliance:
> "If the entire message is hashed instead of a digital signature... the NSA considers this a quasi-encryption and State controls would apply."

**Reviewer Note:** If you are reviewing code that will be exported internationally, verify if the hashing logic triggers export control regulations. (Consult your legal team, but flag it in the review).

---

## 8. Secure Code Reviewer’s Checklist (A6 - Passwords)

- [ ] **No Plaintext:** Confirm absolutely ZERO passwords are stored in plaintext.
- [ ] **Algorithm:** Verify SHA-256 or SHA-512 is used. (Flag MD5/SHA-1).
- [ ] **Salt Existence:** confirm a Salt column exists in the User table.
- [ ] **Salt Uniqueness:** Verify the code generates a *new* salt every time `UpdatePassword()` is called.
- [ ] **Salt Generation:** Verify `SecureRandom` / `RNGCryptoServiceProvider` is used.
- [ ] **Salt Length:** Verify salt is > 16 bytes (128 bits).
- [ ] **Performance:** (Modern Check) While the guide focuses on SHA-2, modern best practice suggests slow algorithms like **PBKDF2**, **Bcrypt**, or **Argon2**. If the app uses SHA-256, it is "Compliant" with this guide, but recommending a work-factor (slow) algorithm is a "Best Practice" finding.

---
*Reference: OWASP Code Review Guide 2.0 (Pages 127-130)*
