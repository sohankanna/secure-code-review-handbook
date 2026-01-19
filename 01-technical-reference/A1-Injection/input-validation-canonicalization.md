# Input Validation & Canonicalization: The Shield and the Filter

## 1. Introduction: The "Castle Gatekeeper" Analogy
Imagine a medieval castle. 
*   **Input Validation** is the gatekeeper who looks at everyone trying to enter and asks: *"Are you a knight? Are you carrying a weapon? Is your name on the list?"* If they don't look right, they are kicked out.
*   **Canonicalization** is the process of making sure a spy hasn't put on a mask or a costume to look like someone else. It forces everyone to take off their masks so the gatekeeper sees their **true face** before deciding to let them in.

In code, **Input Validation** checks if the data "looks" right (is it a number? is it too long?). **Canonicalization** makes sure the data isn't "encoded" or "disguised" to bypass the security checks.

---

## 2. The "Golden Rule" of Secure Code Review (PDF p. 58)
If you remember only one thing for your exam, it should be the **Golden Rule**:
> **"All external input, no matter what it is, will be examined and validated."**

As a reviewer, you must never assume data is safe. This includes:
*   Data from web forms.
*   Data from cookies.
*   HTTP Headers (which attackers can easily change).
*   Data coming from other internal systems or databases.

---

## 3. Input Validation: The First Line of Defense
According to the guide (p. 54), input validation is more than just checking form fields. It is a technical control that mitigates **SQL Injection, Cross-Site Scripting (XSS), and Buffer Overflows.**

### A. Whitelisting vs. Blacklisting (The "Known Good" Approach)
The guide is very clear: **Whitelisting is superior.**

1.  **Blacklisting (The Failure):** The developer tries to make a list of "bad characters" to block (like `DROP`, `SELECT`, or `'`). 
    *   **The Problem:** Attackers are creative. If you block `DROP`, they will try `drOp` or use hex codes. You can never list every "bad" thing in the world.
2.  **Whitelisting (The Success):** The developer defines exactly what is allowed. 
    *   **The Logic:** "I only expect a 5-digit number for this Zip Code." Anything that isn't exactly 5 digits is immediately rejected.

### B. The Four Types of Validation Checks
When reviewing code, look for these four specific checks:
1.  **Type:** Is it an Integer when it should be an Integer?
2.  **Length:** Is the input 1,000 characters long when the database only holds 20? (Prevents Buffer Overflows).
3.  **Format:** Does a Date look like `YYYY-MM-DD`? Does an Email have an `@` symbol?
4.  **Range:** Is a "User Age" between 0 and 120? If the input is `-5`, the code should reject it.

---

## 4. Canonicalization: Unmasking the Attacker (PDF p. 55)
This is a high-level concept that is critical for Secure Code Review. 

**The Problem:** Attackers use "Equivalent Forms" to hide their attacks. 
For example, a full stop (a period `.`) can be written in many ways:
*   Standard: `.`
*   URL Encoded: `%2E`
*   Unicode: `U+002E`

If a developer writes a security filter that looks for the string `../` (to prevent an attacker from climbing out of a folder), the attacker might send `%2E%2E%2F`. The simple filter won't recognize it, but the operating system will, allowing the attacker to steal files.

### The Solution: "Reduce to the Simplest Form"
**Canonicalization** is the process of converting all those different versions into the "Standard" or "Canonical" name **BEFORE** the validation happens.

#### 🚩 Bad Example: Path Traversal (Sample 7.11)
```java
// VULNERABLE: Using getAbsolutePath()
public static void main(String[] args) {
    File x = new File("/cmd/" + args[1]);
    String absPath = x.getAbsolutePath(); // This does NOT resolve synonyms like "../"
}
```
In this code, if the user sends `../../etc/passwd`, the `getAbsolutePath()` function might keep those "dots," and the application will happily open a sensitive system file.

#### ✅ Good Example: Secure Path Resolution (Sample 7.12)
```java
// SECURE: Using getCanonicalPath()
public static void main(String[] args) throws IOException {
    File x = new File("/cmd/" + args[1]);
    String canonicalPath = x.getCanonicalPath(); // This resolves all dots and encodings!
}
```
By using `getCanonicalPath()`, the application "unmasks" the input. It turns `../../etc/passwd` into its real, absolute location. The code can then see that the file is *outside* the allowed `/cmd/` folder and block the request.

---

## 5. Data Validation vs. Business Validation (PDF p. 55)
As a code reviewer, you must distinguish between these two:

*   **Data Validation:** Technical checks (Is it a number? Is it too long?). This can often be done by automated tools.
*   **Business Validation:** Logic checks based on the app's purpose. 
    *   *Example:* If a user tries to withdraw $500 but only has $100 in their account, the technical "Data Validation" passes (it's a positive number), but the "Business Validation" must fail.
    *   **Reviewer Action:** You must manually check if the code verifies permissions and logic, not just character types.

---

## 6. Platform-Specific Validation (IIS & .NET)
The guide (p. 113) lists specific tools used in Microsoft environments that you should look for during a review:

| Validator | Purpose |
| :--- | :--- |
| **RequiredFieldValidator** | Ensures the user didn't leave a box empty. |
| **CompareValidator** | Checks one box against another (like "Password" and "Confirm Password"). |
| **RangeValidator** | Checks if a number or date is between a specific minimum and maximum. |
| **RegularExpressionValidator** | Uses "Regex" to ensure complex patterns (like phone numbers) are correct. |
| **CustomValidator** | Used for the "Business Validation" mentioned above. |

---

## 7. Output Encoding: The Final Safety Net
If validation fails or is missed, **Output Encoding** is the last line of defense. It ensures that if malicious data *does* get into the system, it is rendered harmlessly as "text" rather than "code" when shown to the user.

### .NET Request Validation (PDF p. 56)
The guide warns that while .NET has a built-in "Request Validation" feature, **it is too generic.**
> "The downside is too generic and not specific enough to meet all of our requirements... You can never use request validation for securing your application against cross-site scripting."

**Reviewer Insight:** If a developer says, *"I'm safe because I turned on the .NET default validation,"* they are wrong. You must check for manual **HTML Encoding** in the code-behind:

```csharp
// SECURE .NET Pattern
var encodedInput = Server.HtmlEncode(userInput);
```

---

## 8. Managed vs. Unmanaged Code (PDF p. 57)
In Java and .NET, most code is "Managed" (handled by a safe virtual machine). However, sometimes apps call "Native" code (C or C++). This is a **High Risk** area for injection and buffer overflows.

**Reviewer Action:**
Look for `native` keywords in Java or `[DllImport]` in C#. 
1.  **Rule:** Native methods should never be `public`. 
2.  **Fix:** Create a "Wrapper" method. The wrapper performs the strict Input Validation in the safe Managed environment *before* passing the data to the dangerous Native environment.

---

## 9. Secure Code Reviewer’s A1 Defense Checklist

### Phase 1: The "Entry Points" (Sources)
- [ ] Are all `HTTP Headers`, `Cookies`, and `Hidden Fields` treated as untrusted?
- [ ] Is validation performed on the **Server Side**? (Client-side validation in the browser is for "ease of use" and can be easily bypassed by attackers).

### Phase 2: The "Cleaning" (Interpretation)
- [ ] Is **Canonicalization** performed before validation?
- [ ] In Java, is `getCanonicalPath()` used for file operations instead of `getAbsolutePath()`?
- [ ] Is `JSON.parse()` used instead of `eval()`?

### Phase 3: The "Checks" (Validation)
- [ ] Is **Whitelisting** used for all alphanumeric input?
- [ ] Does the code check for **Length** to prevent memory corruption (Buffer Overflows)?
- [ ] Does the code check for **Type** and **Range**?
- [ ] Are there **Business Logic** checks (e.g., checking account balances or ownership of a record)?

### Phase 4: The "Exit" (Sinks)
- [ ] Is user input **HTML Encoded** before being displayed?
- [ ] Is user input **Parameterized** before being sent to the database?

---

## 10. Summary for the Exam
*   **Validation** is the gatekeeper; **Canonicalization** is the mask-remover.
*   **Whitelisting** is the only secure way to validate.
*   **Encoding** (like UTF-8) can be used by attackers to hide attacks; the code must resolve these back to plain text first.
*   **Security is Layered:** You need validation at the input, parameterization at the database, and encoding at the output. This is called **Defense in Depth.**

---
