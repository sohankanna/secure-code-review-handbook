# Java Framework Configuration: Struts, Java EE, and Annotations

## 1. Executive Summary
In the Java ecosystem, security is often "Configured," not "Coded."

*   **Programmatic Security:** Writing `if (user.role == "admin")` inside the Java class.
*   **Declarative Security:** Writing `<url-pattern>/admin/*</url-pattern>` and `<role-name>admin</role-name>` inside an XML file.

**The Reviewer's Challenge:**
You cannot simply read the `.java` files. You must analyze the deployment descriptors (`web.xml`, `struts-config.xml`, `ejb-jar.xml`) to understand the security posture. A missing line in XML can leave an entire administrative section of a website open to the public.

---

## 2. Apache Struts Configuration (`struts-config.xml`)
**Reference:** PDF Page 83 (Section 11.1)

Apache Struts is a classic MVC framework. It separates the Model, View, and Controller. The "Brain" of a Struts application is the `struts-config.xml` file.

### A. The Action Mappings (The Traffic Cop)
The `action-mappings` section defines how URLs map to Java classes.

**Sample Configuration (Sample 11.1):**
```xml
<action-mappings>
    <action path="/login" type="test.struts.LoginAction">
        <forward name="valid" path="/jsp/MainMenu.jsp" />
        <forward name="invalid" path="/jsp/LoginView.jsp" />
    </action>
</action-mappings>
```

**Reviewer Analysis:**
*   **The Path:** `/login` is the URL the user visits.
*   **The Type:** `test.struts.LoginAction` is the Java class that handles the request.
*   **The Review:** You must find `LoginAction.java` and review it. The XML tells you *where* the logic lives. If you see a sensitive path (e.g., `/admin/deleteUser`) mapped to a class, verify that the class performs authorization checks.

### B. The Validator Plug-In (The Hidden Defense)
One of the most confusing parts of Struts for new reviewers is **Input Validation**. You might look at the Java code (`LoginAction.java`) and see **NO** validation logic. You might be tempted to flag this as a vulnerability.

**Wait! Check the XML first.**
Struts uses a "Validator Engine" configured in XML.

**Secure Configuration:**
```xml
<plug-in className="org.apache.struts.validator.ValidatorPlugIn">
    <set-property property="pathnames" 
                  value="/WEB-INF/validator-rules.xml, /WEB-INF/validation.xml"/>
</plug-in>
```

**Reviewer Insight:**
*   If this `<plug-in>` tag exists, validation is likely happening in `validation.xml`.
*   If this tag is **missing**, and the Java code has no validation, then the application is **Vulnerable** (A1 - Injection).

---

## 3. Java EE Declarative Security (`web.xml`)
**Reference:** PDF Pages 84-85 (Section 11.2)

The `web.xml` file is the heart of a Java Web Application. It defines the "Security Constraints" that the Application Server (Tomcat, JBoss, WebLogic) enforces before the request even reaches your code.

### A. The Security Constraint Structure
A security constraint is composed of three parts:
1.  **Web Resource Collection:** *What* are we protecting? (URL patterns).
2.  **Auth Constraint:** *Who* can access it? (Roles).
3.  **User Data Constraint:** *How* must they access it? (SSL/Encryption).

### B. Analyzing a Secure Configuration (Sample 11.2)
```xml
<security-constraint>
    <!-- 1. WHAT: Protect the Sales Info pages -->
    <web-resource-collection>
        <web-resource-name>SalesInfo</web-resource-name>
        <url-pattern>/salesinfo/*</url-pattern>
        <http-method>GET</http-method>
        <http-method>POST</http-method>
    </web-resource-collection>

    <!-- 2. WHO: Only Managers -->
    <auth-constraint>
        <role-name>manager</role-name>
    </auth-constraint>

    <!-- 3. HOW: Encrypted Transport (SSL) -->
    <user-data-constraint>
        <transport-guarantee>CONFIDENTIAL</transport-guarantee>
    </user-data-constraint>
</security-constraint>
```

**Reviewer Checklist for `web.xml`:**
*   **Completeness:** Does the `<url-pattern>` cover all sensitive files? (e.g., `/admin/*`).
*   **Methods:** Are HTTP methods restricted?
    *   *Risk:* If the config says `<http-method>GET</http-method>` but *omits* `POST`, an attacker might be able to `POST` to the URL and bypass the security check entirely.
*   **Transport:** Look for `<transport-guarantee>CONFIDENTIAL</transport-guarantee>`.
    *   *Red Flag:* `NONE` implies HTTP (Clear text).
    *   *Green Flag:* `CONFIDENTIAL` implies HTTPS (SSL/TLS).

---

## 4. EJB Security (`ejb-jar.xml`)
**Reference:** PDF Pages 86-87 (Sample 11.3, 11.4)

Enterprise JavaBeans (EJBs) are server-side components that handle business logic. Their security is defined in `ejb-jar.xml`.

### Method-Level Permissions
Unlike `web.xml` (which protects URLs), `ejb-jar.xml` protects **Java Methods**.

**Secure Configuration:**
```xml
<method-permission>
    <role-name>employee</role-name>
    <method>
        <ejb-name>AardvarkPayroll</ejb-name>
        <method-name>findByPrimaryKey</method-name>
    </method>
</method-permission>
```
*   **Analysis:** Only a user with the role `employee` can call the function `findByPrimaryKey` on the `AardvarkPayroll` bean.

**Reviewer Red Flag (`<unchecked>`):**
```xml
<method-permission>
    <unchecked/> <!-- DANGER: Public Access -->
    <method>
        <ejb-name>EmployeeServiceHelp</ejb-name>
        <method-name>*</method-name>
    </method>
</method-permission>
```
*   **Analysis:** The `<unchecked/>` tag means **Public Access**. Anyone can call these methods. Ensure sensitive functions (like `deleteUser`) are never marked as unchecked.

---

## 5. Java Annotations (Modern Configuration)
**Reference:** PDF Page 88 (Section 11.3)

In modern Java (Java EE 6+ / Spring), developers often move configuration out of XML and into the Java code using **Annotations** (lines starting with `@`).

### Key Security Annotations to Review
The guide lists five critical annotations you must understand:

1.  **`@DeclareRoles("admin", "user")`**: Defines the roles used in the class.
2.  **`@RolesAllowed("admin")`**: The "Whitelist". Only users with this role can run the method.
3.  **`@PermitAll`**: The "Public" switch. No security check is performed.
4.  **`@DenyAll`**: The "Kill Switch". No one can call this method (often used to lock down specific methods in a class where the rest are public).
5.  **`@RunAs("admin")`**: **DANGER**. This method executes with elevated privileges, regardless of who called it.

### Code Review Example (Sample 11.5)
```java
public class Movies {
    
    // SECURE: Only Employees and Managers can add data
    @RolesAllowed({"Employee", "Manager"})
    public void addMovie(Movie movie) { ... }

    // SECURE: Only Managers can delete data (Higher Privilege)
    @RolesAllowed({"Manager"})
    public void deleteMovie(Movie movie) { ... }

    // RISK: Public Access. Reviewer must ensure this method is truly safe.
    @PermitAll
    public List<Movie> getMovies() { ... }
}
```

**Reviewer Strategy:**
*   Scan all Controller/Service classes.
*   Look for methods that **change data** (`add`, `update`, `delete`).
*   Ensure they have `@RolesAllowed` and NOT `@PermitAll`.

---

## 6. Programmatic Security (The Fallback)
**Reference:** PDF Pages 91-92 (Section 11.5)

Sometimes XML and Annotations are not flexible enough. Developers may write security logic inside the code.

### The J2EE Security APIs
Look for these functions in the source code:
1.  **`request.getRemoteUser()`**: Returns the username.
2.  **`request.isUserInRole("admin")`**: Checks permissions.
3.  **`request.getUserPrincipal()`**: Gets the security object.

**Secure Code Pattern (Sample 11.9 Concept):**
```java
if (request.isUserInRole("admin")) {
    // Perform sensitive task
    deleteDatabase();
} else {
    // Log Security Alert
    logger.error("Unauthorized attempt by " + request.getRemoteUser());
    throw new SecurityException("Access Denied");
}
```

**Reviewer Warning:**
If you see Programmatic Security (`if user == admin`) mixed with Declarative Security (`web.xml`), ensure they do not contradict each other. Declarative security (XML) is usually processed *before* the code runs.

---

## 7. Secure Code Reviewer’s Checklist (Java Config)

### `struts-config.xml`
- [ ] **Validation:** Is the `ValidatorPlugIn` defined?
- [ ] **Action Mappings:** Do sensitive paths (`/admin`) map to secure Action Classes?

### `web.xml`
- [ ] **Transport:** Is `CONFIDENTIAL` (SSL) enforced for sensitive URL patterns?
- [ ] **Methods:** Are `HTTP Methods` (GET/POST) explicitly defined? (Prevents method hopping).
- [ ] **Roles:** Are `<auth-constraint>` tags used to lock down directories?
- [ ] **Error Pages:** Are `<error-page>` tags defined to prevent stack trace leakage? (See A2).

### `ejb-jar.xml`
- [ ] **Permissions:** Are sensitive Beans protected by `<method-permission>`?
- [ ] **Public Access:** Search for `<unchecked/>`. Is it used on sensitive methods?

### Source Code (`.java`)
- [ ] **Annotations:** verify usage of `@RolesAllowed` vs `@PermitAll`.
- [ ] **Logic:** Search for `isUserInRole`. Is the role name hardcoded? (e.g., `isUserInRole("admin")`). Hardcoding is fragile; ensure it matches the `web.xml` role definitions.

---
*Reference: OWASP Code Review Guide 2.0 (Pages 83-92)*
