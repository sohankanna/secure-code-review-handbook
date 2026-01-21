# Attack Surface Reduction: Minimization, Backups, and Logging

## 1. Executive Summary
The **Attack Surface** of an application is the sum of all entry points, exit points, data storage, and running code. Every new feature, every new API endpoint, and every piece of data collected adds to this surface.

**The Reviewer's Mandate:**
You must adopt the philosophy of **"Less is More."**
*   If you don't collect the data, it can't be stolen.
*   If a feature is unused, it should be disabled.
*   If a log file exists, it must not become a vulnerability itself.

According to the guide (p. 131), Microsoft uses a metric called the **Relative Attack Surface Quotient (RSQ)** to track changes. If a code change increases the RSQ (e.g., opens a new port or accepts anonymous input), it requires higher scrutiny.

---

## 2. Data Minimization (The "Toxic Asset" Concept)
Data is not just an asset; it is a liability. The guide (p. 132) explicitly asks: *"Is the change storing PII... is all of the new information absolutely necessary?"*

### A. Collection Limits
**The Flaw:** Developers often collect everything "just in case" (e.g., Social Security Numbers for a mailing list).
**The Fix:**
*   **Tokenization:** Do not store Credit Card numbers (PANs). Send them to a payment provider and store only the **Token**.
*   **Truncation:** Store only the last 4 digits.
*   **Retention:** Implement an automated "Data Purge" feature. If a transaction is > 7 years old, delete it.

### B. Data Separation (Partitioning)
**The Vulnerability:** Storing "Trivial Data" (e.g., a list of store locations) in the same database table as "Critical Data" (e.g., Usernames and Passwords).
**The Risk:** An SQL Injection in the "Store Locator" feature gives the attacker access to the "Passwords" table because they live side-by-side.
**The Fix (p. 131):**
*   **Partitioning:** Move sensitive tables to a separate database or schema with stricter permissions.
*   **Least Privilege:** The "Store Locator" web user should **not** have `SELECT` permissions on the `Users` table.

---

## 3. The "Shadow" Attack Surface: Backups & Files
**Reference:** PDF Page 131.

> "Backups of code and data... are an important but often ignored part of a system's Attack Surface."

### The Scenario
You encrypt the production database with AES-256. You have a $1M Firewall.
However, a developer wrote a script that dumps the database to a `.sql` file every night for backup.
*   **Location:** `/var/www/backups/db.sql`
*   **Protection:** None.
*   **Attack:** An attacker guesses the URL or uses Directory Traversal to download the unencrypted text file.

### Reviewer Checklist for Backups
1.  **Encryption:** Are backups encrypted immediately upon creation?
2.  **Location:** Are backups moved **off** the web server immediately?
3.  **Test Data:** Do developers take production backups to their laptops for testing? (This expands the attack surface to every developer's laptop).

---

## 4. Secure Logging: Protecting the "Black Box"
**Reference:** Section 19 (PDF p. 160-163).

Logs are essential for detecting attacks, but they often leak sensitive data or become vectors for attack (Log Injection).

### A. What NOT to Log (PDF p. 162)
Reviewers must grep for logging statements (`log.info`, `System.out.println`, `Trace.Write`) and ensure **none** of the following are recorded:
*   **Session IDs:** (Allows Session Hijacking via log access).
*   **Passwords:** (Plain text credential theft).
*   **Connection Strings:** (Database credentials).
*   **PII:** (Social Security Numbers, Credit Cards).

### B. Log Injection (CRLF Injection)
**The Vulnerability:**
An attacker inputs: `UserLogin: Admin \n [INFO] UserLogin: Root \n`
If the application logs this directly:
```text
[INFO] UserLogin: Admin
[INFO] UserLogin: Root
```
The attacker has forged a log entry, making it look like "Root" logged in. This destroys the integrity of the audit trail.

**The Remediation (PDF p. 161):**
> "Perform sanitization on all event data to prevent log injection attacks e.g. carriage return (CR), line feed (LF)."

**Secure Code (Java):**
```java
// Sanitize before logging
String safeUser = userParams.replaceAll("[\r\n]", "_");
logger.info("Failed login: " + safeUser);
```

### C. Logging Granularity (PDF p. 160)
*   **Debug/Trace Mode:** These modes often print full stack traces and variable dumps.
*   **Reviewer Check:** Ensure the production configuration (`log4j.properties`, `web.config`) sets the logging level to `INFO` or `WARN`, never `DEBUG` or `TRACE`.

---

## 5. Feature & Interface Reduction
Every feature is a potential door for an attacker.

### A. HTTP vs. HTTPS (PDF p. 131)
> "Is the feature unnecessarily using HTTP instead of HTTPS?"

If a feature (like a "Contact Us" form) doesn't *seem* sensitive, developers might leave it on HTTP. However, an attacker can modify that form (MITM) to point to a malicious server or inject JavaScript. **Reduce the surface by enforcing HTTPS everywhere.**

### B. File Uploads (PDF p. 132)
File uploads are a massive attack surface (Malware hosting, Shell upload).
**Reviewer Questions:**
*   Is the upload feature authenticated?
*   Is there a **Rate Limit**? (DoS prevention).
*   Is there a **File Size Limit**?
*   Is the file extension checked via **Whitelisting**? (Allow `.jpg`, deny everything else).

### C. Administrative Interfaces
Admin panels (`/admin`, `/console`) should not be accessible to the entire internet.
**Surface Reduction:**
*   Restrict access by **IP Address**.
*   Require **VPN** access.
*   Rename default paths (e.g., change `/admin` to something obscure, though "Security by Obscurity" is not a primary defense, it reduces noise).

---

## 6. Secure Code Reviewer’s Checklist (A6 - Attack Surface)

### Data Handling
- [ ] **Retention:** Is there code to delete old data?
- [ ] **Necessity:** Does the app collect data it doesn't use?
- [ ] **Separation:** Is sensitive data kept in a separate DB partition?

### Logging
- [ ] **Sanitization:** Are CR/LF characters stripped from user input before logging?
- [ ] **Leakage:** Grep for "Password", "Key", "Token" in log statements.
- [ ] **Levels:** Is `DEBUG` disabled in production config?

### Features
- [ ] **Uploads:** Are file uploads strictly validated (Type/Size/Auth)?
- [ ] **Endpoints:** Are unused API endpoints or Struts Actions removed?
- [ ] **Search:** Are search functions parameterized to prevent SQLi? (Search is a high-risk surface).

### Backups
- [ ] **Storage:** Are database dumps stored outside the web root?
- [ ] **Encryption:** Are backups encrypted at rest?

---
*Reference: OWASP Code Review Guide 2.0 (Pages 130-132, 160-163)*
