# Cryptography Implementation: Algorithms, Libraries, and Entropy

## 1. The Core Principle: "Don't Roll Your Own"
The most important rule in cryptography is: **Never write your own encryption algorithm.**

Cryptography is math, not logic. A tiny mathematical error can render the entire system useless, but the code will still run without crashing. As a Code Reviewer, you must ensure developers are using **Standard, Validated Libraries** (FIPS 140-2 compliant) and not creating custom "scramble" functions.

### The Standard Libraries (Table 19, p. 121)
| Environment | Library to Use | Description |
| :--- | :--- | :--- |
| **Java** | **JCE** (Java Cryptography Extension) | The standard API (`javax.crypto.*`). Implementations include Bouncy Castle or Spring Security. |
| **.NET (C#)** | **System.Security.Cryptography** | The built-in namespace. Wrappers like `DPAPI` are preferred for key management. |
| **C/C++ (Win)** | **CryptoAPI** / **DPAPI** | Native Windows APIs. |
| **C/C++ (Linux)** | **OpenSSL** / **NSS** | Standard open-source libraries. |

---

## 2. Entropy: The Foundation of Security
Encryption is only as strong as the random numbers used to generate the keys. This concept is called **Entropy**.

### The Vulnerability: Weak Randomness
Standard random number generators are designed for **Speed** (e.g., for video games or simulations), not **Unpredictability**.
*   **Bad Functions:** `Math.random()`, `java.util.Random`, `System.Random`.
*   **The Risk:** These are "Pseudo-Random." If an attacker knows the time the number was generated (the "Seed"), they can mathematically predict the next number.

### The Solution: `SecureRandom` (CSPRNG)
You must use a **Cryptographically Secure Pseudo-Random Number Generator**.

#### Java Implementation (Sample 12.2 - PDF p. 123)
Reviewers must look for `java.security.SecureRandom`.

```java
import java.security.SecureRandom;

public class SecureRandomGen {
    public static void main(String[] args) {
        try {
            // 1. Initialize the Secure Generator (SHA512 based)
            SecureRandom secureRandom = SecureRandom.getInstance("SHA512");
            
            // 2. Generate Bytes (e.g., for a Salt or Key)
            byte[] bytes = new byte[512];
            secureRandom.nextBytes(bytes); // Fills the array with high-entropy noise
            
            // 3. Reseeding (Optional but recommended for long-running apps)
            // This adds new "noise" to the generator to prevent prediction
            byte[] seed = secureRandom.generateSeed(10); 
            secureRandom.setSeed(seed);
            
        } catch (NoSuchAlgorithmException e) {
            // Handle error
        }
    }
}
```

#### .NET Implementation (PDF p. 123)
In C#, look for `RNGCryptoServiceProvider`.

```csharp
using System.Security.Cryptography;

// 1. Create the provider
RNGCryptoServiceProvider rng = new RNGCryptoServiceProvider();

// 2. Create a buffer
byte[] randomBytes = new byte[32];

// 3. Fill the buffer with cryptographically strong bytes
rng.GetBytes(randomBytes);
```

---

## 3. Algorithm Selection: The Good, The Bad, and The Broken
A major part of the code review is checking the strings passed into the cryptographic functions. The guide (p. 123) provides clear distinctions.

### A. Symmetric Encryption (Shared Key)
Used for encrypting data at rest (database columns, files).

| Algorithm | Status | Reviewer Action |
| :--- | :--- | :--- |
| **AES** (Advanced Encryption Standard) | **Secure** | **Pass.** Ensure Key Size is 128 or 256 bits. |
| **DES** (Data Encryption Standard) | **Broken** | **Fail.** 56-bit key is too small. Crakable in minutes. |
| **3DES** (Triple DES) | **Legacy** | **Warning.** Slow and being phased out. Prefer AES. |
| **RC4** | **Broken** | **Fail.** Biased stream cipher. |

### B. Hashing (One-Way Fingerprint)
Used for passwords and integrity checks.

| Algorithm | Status | Reviewer Action |
| :--- | :--- | :--- |
| **SHA-256** / **SHA-512** | **Secure** | **Pass.** The current industry standard. |
| **MD5** | **Broken** | **Fail.** Collisions (two different inputs producing the same hash) are trivial to generate. |
| **SHA-1** | **Broken** | **Fail.** Deprecated by NIST/FIPS. |

### C. Modes of Operation (The "Cipher String")
When reviewing Java code, you will see a string like: `Algorithm/Mode/Padding`.
*   **Example:** `AES/CBC/PKCS5Padding`
*   **Mode:** `ECB` (Electronic Code Book) is **Dangerous**. It encrypts identical blocks of data into identical blocks of ciphertext (Pattern Leakage).
*   **Recommendation:** Use `CBC` (Cipher Block Chaining) or `GCM` (Galois/Counter Mode).

---

## 4. Coding Encryption in Java (The AES Example)
The guide provides a lengthy example on **Pages 125-126** demonstrating how to implement AES correctly using the JCE.

### Step-by-Step Code Review Analysis

#### 1. Key Generation
Do not use a password string directly as a key. Use a `KeyGenerator`.
```java
// SECURE: Generates a random 128-bit AES key
KeyGenerator keyGen = KeyGenerator.getInstance("AES");
keyGen.init(128); 
SecretKey secretKey = keyGen.generateKey();
```

#### 2. Cipher Initialization
The `Cipher` object is the engine that performs the work.
```java
// SECURE: Explicitly defining Algorithm, Mode, and Padding
Cipher aesCipher = Cipher.getInstance("AES/CBC/PKCS5Padding");

// Initialize for ENCRYPT_MODE
aesCipher.init(Cipher.ENCRYPT_MODE, secretKey);
```

#### 3. Encryption (The "DoFinal" Method)
```java
String clearText = "Hello World";
// Convert string to bytes (Always specify encoding like UTF-8!)
byte[] byteDataToEncrypt = clearText.getBytes("UTF-8");

// Perform the encryption
byte[] byteCipherText = aesCipher.doFinal(byteDataToEncrypt);

// Encode the result to Base64 so it can be stored as text
String strCipherText = new BASE64Encoder().encode(byteCipherText);
```

#### 4. Decryption
Decryption requires the exact same Key and (if using CBC mode) the same **Initialization Vector (IV)**.
```java
// Initialize for DECRYPT_MODE
aesCipher.init(Cipher.DECRYPT_MODE, secretKey, aesCipher.getParameters());

// Perform the decryption
byte[] byteDecryptedText = aesCipher.doFinal(byteCipherText);
String result = new String(byteDecryptedText);
```

---

## 5. .NET Data Protection API (DPAPI)
For Windows environments, the guide (p. 122) strongly recommends **DPAPI** instead of managing keys manually.

**Why DPAPI?**
*   It uses the User's Windows Login credentials (or the Machine's credentials) to derive the encryption key.
*   **Benefit:** The developer never handles the key. The OS handles it.

**Secure .NET Pattern:**
```csharp
// Encrypt data using the Current User's credentials
byte[] encryptedData = ProtectedData.Protect(
    originalBytes, 
    optionalEntropy, 
    DataProtectionScope.CurrentUser
);

// Decrypt
byte[] originalData = ProtectedData.Unprotect(
    encryptedData, 
    optionalEntropy, 
    DataProtectionScope.CurrentUser
);
```
**Reviewer Check:** If you see `System.Security.Cryptography.ProtectedData`, this is a **Pass**. If you see custom AES code where the key is hardcoded in a string, this is a **Fail**.

---

## 6. Common Vulnerabilities to Spot (Red Flags)

### A. Hardcoded Keys (PDF p. 122)
> "Exposure of the symmetric or private key means the encrypted data is no longer private."

**Vulnerable:**
```java
// FAIL: The key is in the source code
SecretKeySpec key = new SecretKeySpec("MySuperSecretKey".getBytes(), "AES");
```

### B. Insecure Randomness (PDF p. 123)
**Vulnerable:**
```java
// FAIL: Predictable random number
Random r = new Random();
int salt = r.nextInt(); 
```

### C. Weak Algorithms (PDF p. 123)
**Vulnerable:**
```java
// FAIL: DES is broken
Cipher c = Cipher.getInstance("DES"); 
```

---

## 7. Secure Code Reviewer’s Checklist (Cryptography)

- [ ] **Entropy:** Is `SecureRandom` (Java) or `RNGCryptoServiceProvider` (.NET) used for all security-critical random numbers?
- [ ] **Algorithm Standard:** Are only FIPS-approved algorithms used? (AES, SHA-256, RSA-2048+).
- [ ] **Mode of Operation:** Is `ECB` mode avoided? (Look for `AES/ECB/...` or just `AES` default).
- [ ] **Key Management:** Are keys loaded from a secure location (HSM, KeyVault, DPAPI) and **not** stored in the source code?
- [ ] **Exception Handling:** Are crypto exceptions caught properly? (e.g., `BadPaddingException` might indicate a Padding Oracle attack, though deep mitigation of this is advanced).
- [ ] **Encoding vs Encryption:** Ensure Base64 is not being used as a "security" measure. It is only for formatting.

---
*Reference: OWASP Code Review Guide 2.0 (Pages 122-126)*
