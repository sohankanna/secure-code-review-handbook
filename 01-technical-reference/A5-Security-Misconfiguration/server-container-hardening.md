# Server & Container Hardening: Tomcat, Jetty, JBoss, and WebLogic

## 1. Executive Summary
The Application Server (or "Container") sits between the Operating System and the Java Application. It handles network connections, thread management, and deployment. 

**The Reviewer's Role:**
Developers often treat the server configuration as an "Ops" problem. However, modern DevOps and "Infrastructure as Code" practices mean these configurations (`server.xml`, `jetty-web.xml`) are often committed to the source code repository. A Secure Code Reviewer must inspect these files to ensure the environment is hostile to attackers.

---

## 2. Apache Tomcat Hardening
**Reference:** PDF Page 89 (Section 11.4)

Apache Tomcat is the most widely used Java Servlet Container. Its primary configuration file is `server.xml`.

### A. The Shutdown Port
**Vulnerability:** Tomcat listens on a specific TCP port (default 8005) for a "SHUTDOWN" command. If an attacker can connect to this port (e.g., via internal network access) and send the shutdown string, they can turn off the server (DoS).

**Reviewer Check (`server.xml`):**
```xml
<!-- VULNERABLE DEFAULT -->
<Server port="8005" shutdown="SHUTDOWN">
```
*   **The Fix:** Change the port to `-1` to disable it, or change the shutdown string to a long, complex password.

### B. Connector Security (The Network Listener)
The `<Connector>` element defines how Tomcat listens for traffic.

**Security Parameters to Review (Table 12):**
1.  **`maxPostSize`:** Limits the size of a POST request.
    *   *Risk:* If set to `0` or negative, it might be unlimited, allowing attackers to fill memory with massive uploads (DoS).
2.  **`maxParameterCount`:** Limits the number of parameters (key/value pairs) in a request.
    *   *Risk:* Hash Collision DoS attacks rely on sending thousands of parameters. Ensure this is capped (e.g., 10,000).
3.  **`SSLEnabled` / `secure`:** Must be set to "true" for HTTPS connectors.

### C. Host & Context Security
The `<Host>` and `<Context>` elements define the application deployment settings.

**Critical Attributes:**
1.  **`autoDeploy`:** If "true", Tomcat automatically deploys any WAR file dropped in the webapps folder.
    *   *Risk:* If an attacker can write a file to that directory (via directory traversal or upload), they get instant code execution. **Set to "false" in production.**
2.  **`deployXML`:** If "true", the web application can supply its own `context.xml`.
    *   *Risk:* This overrides server settings. **Set to "false"** to enforce server-level security policies.
3.  **`crossContext`:** If "true", applications can access each other's resources.
    *   *Risk:* A breach in a low-security app (e.g., the "Help" portal) allows access to the high-security app (e.g., "Banking"). **Set to "false"**.
4.  **`allowLinking`:** If "true", allows Symlinks.
    *   *Risk:* Path Traversal attacks can escape the web root. **Set to "false"**.

---

## 3. Jetty Hardening
**Reference:** PDF Page 89-90 (Sample 11.6, 11.7)

Jetty is often used as an embedded server. It has specific features for limiting content and obfuscating secrets.

### A. DoS Prevention (Form Limits)
Jetty allows granular control over form content to prevent Hash Collision DoS attacks.

**Secure Configuration (`jetty-web.xml`):**
```xml
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
    <!-- Limit total size of form data (bytes) -->
    <Set name="maxFormContentSize">200000</Set>
    
    <!-- Limit number of keys (parameters) -->
    <Set name="maxFormKeys">200</Set>
</Configure>
```
**Reviewer Insight:** If these limits are missing or set excessively high, the application is vulnerable to availability attacks.

### B. Password Obfuscation
Storing plain text passwords in config files is a major risk. Jetty provides a utility to "Obfuscate" (scramble) passwords. While not true encryption, it prevents casual viewing (e.g., over-the-shoulder surfing).

**Implementation (Sample 11.7):**
```xml
<Set name="password">
    <Call class="org.eclipse.jetty.util.security.Password" name="deobfuscate">
        <Arg>OBF:1ri71v1r1v2n1ri71shq1ri71shs1ri71v1r1v2n1ri7</Arg> 
    </Call>
</Set>
```
**Reviewer Check:** Grep for `OBF:`. If you see plain text passwords in database connections, flag it.

---

## 4. JBoss (WildFly) Hardening
**Reference:** PDF Page 90 (Sample 11.8)

JBoss Application Server (now WildFly) supports "Password Masking" to hide credentials in XML files.

### A. Password Masking Annotation
JBoss uses a specific annotation to handle masked passwords injected into the code.

**Secure Code Pattern:**
```java
<annotation>
    @org.jboss.security.integration.password.Password(
        securityDomain="MASK_NAME",
        methodName="setPROPERTY_NAME"
    )
</annotation>
```
**Reviewer Check:** Ensure that `securityDomain` references a properly configured security domain in the JBoss configuration, and that the code uses this annotation rather than hardcoded strings.

---

## 5. Oracle WebLogic Hardening
**Reference:** PDF Page 91 (Table 13)

WebLogic uses the `weblogic.xml` deployment descriptor. This file contains proprietary security settings that extend the standard `web.xml`.

### Key Parameters to Review

| Parameter | Function | Security Implication |
| :--- | :--- | :--- |
| **`run-as-principal-name`** | Sets the identity for `run-as` roles. | **Risk:** If set to an admin user, a standard user might escalate privileges automatically. Verify this is strictly limited. |
| **`security-permission-spec`** | Defines Java Security Manager permissions. | **Check:** Does the app request `AllPermission`? It should request the *minimum* permissions needed (Least Privilege). |
| **`externally-defined`** | Maps roles to external systems (LDAP). | **Check:** Ensure mappings align with the corporate directory structure. |

---

## 6. Secure Code Reviewer’s Checklist (Containers)

### Tomcat (`server.xml`)
- [ ] **Shutdown:** Is the shutdown port disabled (`-1`) or protected with a strong password?
- [ ] **Connectors:** Are `maxPostSize` and `maxParameterCount` defined to prevent DoS?
- [ ] **Deploy:** Is `autoDeploy="false"` and `deployXML="false"` in production?
- [ ] **Isolation:** Is `crossContext="false"`?

### Jetty (`jetty-web.xml`)
- [ ] **DoS:** Are `maxFormContentSize` and `maxFormKeys` set to reasonable limits?
- [ ] **Secrets:** Are passwords obfuscated (using `OBF:` prefix) rather than plain text?

### WebLogic (`weblogic.xml`)
- [ ] **Permissions:** Review `<security-permission-spec>` for excessive privileges.
- [ ] **Principals:** Verify that `run-as` identities are not over-privileged (e.g., running as Root/Admin).

### General
- [ ] **Least Privilege:** Does the container run as a dedicated, low-privilege OS user (not Root/Administrator)?
- [ ] **Sample Apps:** Have the `/docs`, `/examples`, and `/manager` apps been removed?

---
*Reference: OWASP Code Review Guide 2.0 (Pages 89-91)*
