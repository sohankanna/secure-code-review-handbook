# IIS & .NET Hardening: Configuration, Filtering, and Integrity

## 1. Executive Summary
In the Microsoft ecosystem, security is a partnership between the **Web Server (IIS)** and the **Application Framework (.NET)**. 

Unlike other languages where you might write code to filter bad characters, IIS allows you to configure a "Bouncer" at the front door (Request Filtering) that blocks malicious traffic **before your code even runs**.

**The Reviewer's Goal:**
You must verify that the "Bouncer" is configured correctly (`web.config`), the secrets are locked inside a safe (Config Encryption), and the code itself hasn't been tampered with (Strong Naming).

---

## 2. The Configuration Hierarchy (PDF p. 93)
To review .NET correctly, you must understand **Inheritance**. Settings flow down from the server to the folder.

### The Chain of Command
1.  **`machine.config`:** The Master File. Located in the Windows System folder. Settings here apply to **every** application on the server.
2.  **`applicationHost.config`:** The IIS Server Settings.
3.  **Root `web.config`:** The main configuration file for your specific application.
4.  **Sub-folder `web.config`:** You can place a `web.config` inside a specific folder (e.g., `/admin/web.config`) to override settings just for that folder.

**Reviewer Tip:** If you see a security setting in the Root `web.config`, check the sub-folders! A developer might have overwritten a secure setting with an insecure one in a subdirectory (e.g., turning off authentication for a testing folder).

---

## 3. Request Filtering: The "Bouncer" (PDF p. 99-103)
**Request Filtering** is a security module built into IIS 7+. It replaces the old "UrlScan" tool. It is configured in the `<system.webServer>` section.

### A. Double Encoded Requests (The Disguise Attack)
Attackers try to hide malicious characters by encoding them twice.
*   **Normal:** `.` (Dot)
*   **Encoded:** `%2E`
*   **Double Encoded:** `%252E` (The `%` becomes `%25`).

If the server decodes it once, it sees `%2E`. If the security filter checks `%2E`, it might pass. But when the application uses it, it decodes it *again* to `.`, allowing a path traversal attack.

**Secure Configuration (Sample 11.20):**
```xml
<security>
    <requestFiltering allowDoubleEscaping="false">
    </requestFiltering>
</security>
```
*   **Reviewer Check:** Ensure `allowDoubleEscaping` is **false**. If true, the application is exposed to advanced evasion attacks.

### B. High Bit Characters (PDF p. 102)
Standard English text uses "ASCII" (Low Bit). Malicious shellcode or buffer overflow payloads often use "High Bit" characters (non-standard symbols).

**Secure Configuration (Sample 11.21):**
```xml
<requestFiltering allowHighBitCharacters="false">
```
*   **Note:** If your application supports international languages (like Chinese or Arabic), you usually *cannot* enable this restriction.

### C. File Extensions (Whitelisting vs. Blacklisting)
You should never allow a user to request a sensitive file type, like `.config`, `.mdb` (Database), or `.exe`.

**Secure Configuration (Sample 11.22):**
```xml
<requestFiltering>
    <!-- Step 1: Default Deny (Block everything not listed) -->
    <fileExtensions allowUnlisted="false">
        <!-- Step 2: Whitelist (Only allow safe types) -->
        <add fileExtension=".html" allowed="true"/>
        <add fileExtension=".aspx" allowed="true"/>
        <add fileExtension=".jpg" allowed="true"/>
    </fileExtensions>
</requestFiltering>
```
**Reviewer Check:** If `allowUnlisted` is "true", check if there is a blacklist ( `<add fileExtension=".exe" allowed="false"/>`). **Whitelisting (AllowUnlisted=false)** is significantly more secure.

### D. Request Limits (Buffer Overflow Protection)
Attackers try to crash servers or bypass buffers by sending massive URLs or Query Strings.

**Secure Configuration (Sample 11.23):**
```xml
<requestLimits 
    maxUrl="260"               <!-- Max characters in the URL -->
    maxQueryString="2048"      <!-- Max characters in ?param=... -->
    maxAllowedContentLength="30000000" /> <!-- Max File Upload Size (bytes) -->
```
*   **Reviewer Check:** Are these limits reasonable? If `maxUrl` is set to `2,000,000`, that is suspicious and risky.

### E. HTTP Verbs (Method Restrictions)
Does your website need to accept `DELETE` or `PUT` requests? Probably not.

**Secure Configuration (Sample 11.24):**
```xml
<verbs allowUnlisted="false">
    <add verb="GET" allowed="true" />
    <add verb="POST" allowed="true" />
</verbs>
```

---

## 4. Encrypting Configuration Sections (PDF p. 104)
The `web.config` file often contains database passwords in the `<connectionStrings>` section. Storing these in plain text is a critical vulnerability.

### The Remediation: `aspnet_regiis`
Microsoft provides a built-in tool to encrypt sections of the config file. When the application runs, it automatically decrypts them in memory. An attacker stealing the file gets garbage.

**The Command:**
```bash
aspnet_regiis -pef "connectionStrings" "C:\Inetpub\wwwroot\MyApp"
```

**Reviewer Check:**
Open the `web.config` file.
*   **Vulnerable:**
    ```xml
    <connectionStrings>
        <add name="DB" connectionString="uid=admin;pwd=password123;..."/>
    </connectionStrings>
    ```
*   **Secure:**
    ```xml
    <connectionStrings configProtectionProvider="RsaProtectedConfigurationProvider">
        <EncryptedData>...</EncryptedData>
    </connectionStrings>
    ```

---

## 5. Strong Named Assemblies (PDF p. 106-109)
This section deals with the integrity of the compiled code (`.dll` or `.exe`).

### The Problem
How do you know that the `MyBankingLogic.dll` on the server is the real one you compiled, and not a malicious version swapped in by an attacker (DLL Hijacking/Spoofing)?

### The Solution: Strong Naming
A "Strong Name" digitally signs the assembly with a Private Key. It guarantees:
1.  **Uniqueness:** No two assemblies can have the same strong name.
2.  **Integrity:** If one bit of the code changes, the signature breaks.
3.  **Versioning:** Ensures the app uses the exact version intended.

**Reviewer Check:**
*   Check the Visual Studio project settings -> **Signing** Tab.
*   Look for `.snk` (Key) files in the source tree.
*   Look for the `[AssemblyKeyFile]` attribute in the code (Legacy).

---

## 6. Obfuscation & Round Tripping (PDF p. 110)
### The Problem: Round Tripping
.NET code is "Intermediate Language" (IL). It is very easy to reverse engineer.
1.  Attacker uses `Ildasm.exe` (Disassembler) to turn your `.dll` back into text.
2.  Attacker modifies the logic (e.g., changes `if (isAdmin)` to `if (true)`).
3.  Attacker uses `Ilasm.exe` (Assembler) to turn it back into a `.dll`.

This is called **Round Tripping**.

### The Solution: Obfuscation
Obfuscation tools (like Dotfuscator) scramble the code. They rename `CheckPassword()` to `a()`.
*   This does not stop reverse engineering, but it makes it significantly harder, time-consuming, and expensive for the attacker.
*   **Reviewer Check:** Ask if the build process includes an obfuscation step for Release builds.

---

## 7. Locking Trust Levels (PDF p. 106)
By default, .NET applications might run with "Full Trust," meaning they can read the registry, access arbitrary files, and call native Windows API functions.

**Secure Configuration (Sample 11.28):**
You can **Lock** the trust level in the configuration so that sub-applications cannot increase their own privileges.

```xml
<location path="MySecuredApp" allowOverride="false">
    <system.web>
        <!-- Restrict app to "Medium" trust (Cannot access Registry/System files) -->
        <trust level="Medium" />
    </system.web>
</location>
```

---

## 8. Secure Code Reviewer’s Checklist (IIS/.NET)

### `web.config` Scan
- [ ] **Debug Mode:** `<compilation debug="false">` (Must be false in production).
- [ ] **Errors:** `<customErrors mode="On">` (Prevent Yellow Screen of Death).
- [ ] **Cookies:** `<httpCookies httpOnlyCookies="true" requireSSL="true" />`.
- [ ] **Secrets:** Are `<connectionStrings>` and `<appSettings>` encrypted?

### Request Filtering (`system.webServer`)
- [ ] **Double Escaping:** `allowDoubleEscaping="false"`.
- [ ] **Extensions:** `allowUnlisted="false"` (Whitelist approach).
- [ ] **Verbs:** Are `PUT`, `DELETE`, and `TRACE` blocked?
- [ ] **Limits:** Are `maxUrl` and `maxQueryString` set to reasonable limits?

### Binary Checks
- [ ] **Signing:** Are assemblies Strong Named?
- [ ] **Obfuscation:** Is the production code obfuscated?

---
*Reference: OWASP Code Review Guide 2.0 (Pages 93-110)*
