# A5: Security Misconfiguration — Technical Reference

## 1. Executive Summary
Security Misconfiguration is the most commonly seen issue in web application security. It occurs when security settings are defined, implemented, and maintained by defaults rather than by strict, secure standards.

According to the **OWASP Code Review Guide 2.0 (p. 83)**, modern applications do not execute in isolation. They run inside:
1.  **Frameworks** (Struts, Spring, ASP.NET MVC).
2.  **Application Servers/Containers** (IIS, Tomcat, Jetty).
3.  **Operating Systems** (Windows, Linux).

**The Reviewer's Mandate:**
A Secure Code Review is **incomplete** if you only look at the source code (`.cs`, `.java`). You **must** review the configuration files (`web.config`, `pom.xml`, `web.xml`) because a single XML flag can disable all the security logic you just reviewed.

---

## 2. The Ecosystem of Misconfiguration
The guide emphasizes that security controls are hierarchical. A secure application running on an insecure server is still vulnerable.

### Layers of Review (PDF p. 83)
1.  **Network/OS:** Firewalls, VLANs, and OS Hardening (often outside scope of App Code Review, but critical context).
2.  **Web Server/Container:** Tomcat `server.xml` or IIS `ApplicationHost.config`.
3.  **Application Framework:** Struts Actions, Spring Security Config, ASP.NET HttpRuntime.
4.  **The Application Code:** Custom error handling and logic.

---

## 3. Key Review Areas: What to Look For

### A. Debug & Diagnostic Modes
**The Vulnerability:** Developers often leave "Debug Mode" enabled in production to make troubleshooting easier.
**The Risk:** This provides attackers with stack traces, variable values, and detailed system paths (Information Leakage).

**Java Check (general):**
Look for logging levels set to `DEBUG` or `TRACE` in `log4j.properties` or `logback.xml`.

**.NET Check (`web.config`):**
```xml
<!-- VULNERABLE: compiles code with debug symbols -->
<compilation debug="true">
```
*   **Fix:** Set `debug="false"` for production builds.

### B. Directory Listings
**The Vulnerability:** If the server is configured to list directory contents, an attacker can browse files (images, configs, backups) just by visiting a folder URL without a default page (like `index.html`).
**The Risk:** Exposure of source code, backup files (`.bak`), or config files.

### C. Default Accounts & Pages
**The Vulnerability:** Application servers often ship with default admin consoles (e.g., Tomcat Manager, WebLogic Admin).
**Reviewer Action:** Check configuration files for default user/password mappings (e.g., `tomcat-users.xml`) or ensure the build script removes sample applications (`/examples`, `/docs`).

---

## 4. The Framework Configuration Trap
Modern frameworks do "heavy lifting" for developers. If configured poorly, they expose the app.

### Apache Struts (PDF p. 83)
Struts relies heavily on `struts-config.xml`.
*   **Validator Engine:** The guide notes that Struts has a validation engine that requires **no code** in the Java Form Bean. It is enabled via XML.
*   **Reviewer Check:** If you look at the Java code and see no validation logic, **don't panic yet**. Check `validation.xml` and `struts-config.xml`. If the validation rules are missing there *too*, then you have a finding.

### Java EE Declarative Security (PDF p. 84)
Java EE allows security to be defined in XML (`web.xml`) rather than Java code. This is called **Declarative Security**.
*   **Reviewer Check:** Look for `<security-constraint>` tags.
*   **The Logic:** You can define that `/admin/*` requires the role `MANAGER` and transport `CONFIDENTIAL` (SSL) purely in XML.

**Secure XML Pattern:**
```xml
<security-constraint>
    <web-resource-collection>
        <url-pattern>/salesinfo/*</url-pattern>
        <http-method>GET</http-method>
    </web-resource-collection>
    <auth-constraint>
        <role-name>manager</role-name> <!-- Access Control -->
    </auth-constraint>
    <user-data-constraint>
        <transport-guarantee>CONFIDENTIAL</transport-guarantee> <!-- SSL Enforced -->
    </user-data-constraint>
</security-constraint>
```

---

## 5. .NET Specifics: `web.config` and Strong Naming
The guide dedicates significant space (p. 106-110) to .NET specifics.

### Hierarchy (PDF p. 93)
IIS configuration is hierarchical.
1.  `machine.config` (Server Root)
2.  `web.config` (Application Root)
3.  `web.config` (Sub-folder)

**Reviewer Tip:** A setting in a sub-folder `web.config` overrides the root. Always check for nested config files.

### Strong Named Assemblies (PDF p. 106)
**The Concept:** Signing an assembly (DLL/EXE) with a digital signature (Key Pair).
**The Benefit:**
1.  **Uniqueness:** Guarantees unique names.
2.  **Version Protection:** Ensures no one can swap your DLL with an older/malicious version.
3.  **Integrity:** The .NET framework verifies the code hasn't been tampered with since build.
**Reviewer Check:** Look for the use of `sn.exe` (Strong Name Tool) or the `Signing` tab in Visual Studio project settings.

### Obfuscation (PDF p. 110)
**Round Tripping:** The guide warns that .NET assemblies can be easily reverse-engineered using `Ildasm.exe` (Disassembler) and `Ilasm.exe` (Assembler).
**The Defense:** Use **Obfuscation** (e.g., Dotfuscator) to rename classes and methods to unreadable strings, making reverse engineering difficult.

---

## 6. Secure Code Reviewer’s A5 Checklist

### General Configuration
- [ ] **Debug Mode:** Is debugging disabled in production configs?
- [ ] **Stack Traces:** Are custom error pages configured to hide stack traces? (See A2/Error Handling).
- [ ] **Directory Browsing:** Is directory listing disabled?
- [ ] **Sample Apps:** Are default scripts and sample apps removed?

### Java / J2EE
- [ ] **web.xml:** Are `<security-constraint>` tags used to protect sensitive URL patterns?
- [ ] **Method Constraints:** Are HTTP methods (PUT, DELETE) restricted?
- [ ] **Struts:** Is the `ValidatorPlugIn` enabled in `struts-config.xml`?

### .NET / IIS
- [ ] **web.config:** Is `<compilation debug="false">` set?
- [ ] **Request Filtering:** Are dangerous file extensions (`.exe`, `.config`, `.mdb`) blocked?
- [ ] **Strong Naming:** Are production assemblies signed?
- [ ] **Obfuscation:** Is the release build obfuscated?
- [ ] **Encryption:** Are sensitive sections of `web.config` (like `connectionStrings`) encrypted using `aspnet_regiis`? (PDF p. 104).

---
*Reference: OWASP Code Review Guide 2.0 (Pages 83-110)*
