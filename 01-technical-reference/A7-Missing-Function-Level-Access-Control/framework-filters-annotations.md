# Framework Filters & Annotations: The Automated Gatekeeper

## 1. Executive Summary
In modern web frameworks, authorization logic is rarely written inside every single business method. This is because it is repetitive, error-prone, and hard to maintain.

Instead, frameworks provide two primary mechanisms:
1.  **Filters/Interceptors:** Code that runs **before and after** every HTTP request.
2.  **Annotations/Attributes:** Metadata tags placed directly on controllers or methods that declare security requirements.

**The Reviewer's Goal:**
Your job is to find the **Central Security Point**. If you cannot find a central filter, you must assume there is no security and manually check every single method. If you *do* find a central filter, you must verify it is configured to **Default Deny** and that no sensitive endpoints are accidentally excluded.

---

## 2. The Intercepting Filter Pattern (The "Front Door")
This is the most common and robust way to handle authorization. A single piece of code (the Filter) checks every incoming request.

### A. The "Register Global Filters" Method (.NET MVC)
**Reference:** PDF Page 136 (Sample 13.2)

The guide highlights the `RegisterGlobalFilters` method in `Global.asax.cs` as the central point for .NET MVC security.

**Secure Configuration (Default Deny):**
```csharp
// Sample 13.2: Global.asax.cs
public static void RegisterGlobalFilters(GlobalFilterCollection filters)
{
    // 1. This adds a global Error Handler
    filters.Add(new HandleErrorAttribute());

    // 2. THIS IS THE SECURITY GATEKEEPER
    // By adding the AuthorizeAttribute globally, NO request is allowed
    // unless the user is authenticated.
    filters.Add(new System.Web.Mvc.AuthorizeAttribute());
}
```

**Reviewer Insight:**
If `filters.Add(new AuthorizeAttribute())` is present, the application is **Secure by Default**. Your job now is to look for **Exceptions** (public pages).

### B. The `AllowAnonymous` Attribute (.NET MVC)
**Reference:** PDF Page 137 (Sample 13.3)

Once you have a "Default Deny" policy, you must explicitly mark public pages.

**Secure Code (Public Page):**
```csharp
// SECURE: This action is exempt from the Global Authorize filter.
[AllowAnonymous]
public ActionResult LogMeIn(string returnUrl) {
    // ... Login Logic ...
}

// SECURE: Because of the Global Filter, this action is protected
// without needing any attributes.
public ActionResult UserDashboard() {
    // ... User-specific logic ...
}
```

**Reviewer Checklist:**
1.  Find `Global.asax`. Is `AuthorizeAttribute` registered globally?
2.  If Yes, search the codebase for `[AllowAnonymous]`.
3.  For every method marked `[AllowAnonymous]`, verify that it is truly a public function (Login, Home Page, Forgot Password). If you see `[AllowAnonymous]` on `DeleteUser()`, it is a **Critical Finding**.

---

## 3. The Annotation/Attribute Pattern (Method-Level)
If a global filter is not used, the developer must "decorate" every sensitive method with an attribute.

### A. .NET `Authorize` Attribute
This is the most common way to implement RBAC in .NET.

**Secure Code:**
```csharp
public class AdminController : Controller {
    
    // This entire controller is now locked down.
    // Only users with the "Admin" role can access it.
    [Authorize(Roles = "Admin")]
    public ActionResult Index() {
        return View();
    }
    
    [Authorize(Roles = "Admin")]
    public ActionResult DeleteUser(int id) {
        // ...
    }
}
```

**Reviewer Red Flags:**
*   **Missing Attribute:** A sensitive method without `[Authorize]` is completely open.
*   **Simple `[Authorize]`:** If it just says `[Authorize]` without `Roles`, it only checks for **Authentication** (Is the user logged in?), not **Authorization** (Is the user an admin?).
*   **Split Logic:** If `[Authorize]` is on the `GET` method but missing on the corresponding `POST` method, an attacker can bypass the check by POSTing directly.

### B. Java Annotations (`@PreAuthorize`, `@RolesAllowed`)
Java has several standards for security annotations.

**1. JSR 250 (Java EE Standard):**
```java
import javax.annotation.security.RolesAllowed;

public class MyService {
    @RolesAllowed("Admin")
    public void deleteData() { ... }
}
```

**2. Spring Security (More Powerful):**
Spring uses "Spring Expression Language" (SpEL) which allows for more complex rules.

```java
import org.springframework.security.access.prepost.PreAuthorize;

public class MyController {
    
    // RBAC Check
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser() { ... }
    
    // ACL-Style Check (Ownership)
    @PreAuthorize("#invoice.owner == authentication.principal.username")
    public void viewInvoice(Invoice invoice) { ... }
}
```

**Reviewer Checklist (Java):**
1.  Grep for `@RolesAllowed`, `@Secured`, and `@PreAuthorize`.
2.  If you find a sensitive method **without** one of these annotations, it is likely vulnerable.
3.  Pay close attention to `@PermitAll`. This is the Java equivalent of `[AllowAnonymous]`.

---

## 4. XML-Based Configuration (The "Old School" Method)
As covered in the Java Config guide, frameworks like Java EE and Struts can define security in XML. The server/framework acts as a giant "Filter" based on these rules.

### Reviewing `web.xml`
**Secure Code:**
```xml
<security-constraint>
    <web-resource-collection>
        <url-pattern>/admin/*</url-pattern> <!-- Intercept everything in /admin -->
    </web-resource-collection>
    <auth-constraint>
        <role-name>administrator</role-name> <!-- Check the role -->
    </auth-constraint>
</security-constraint>
```

**Reviewer Insight:**
This is very powerful. The server enforces this before your Java code even runs. If this is in place, you can be reasonably sure that all URLs under `/admin/` are protected. Your job is to check if any sensitive logic was accidentally placed *outside* of a protected URL pattern.

---

## 5. Summary: The Reviewer's Workflow
When checking for A7 using this "Framework-First" approach, follow this path:

### Step 1: Look for the Global Filter
*   **.NET:** Check `Global.asax` for `filters.Add(new AuthorizeAttribute())`.
*   **Java:** Check `web.xml` for a "Security Filter" mapping (`<filter-mapping>`).

### Step 2: If Global Filter Exists (Default Deny)
*   **Your Job:** Search for **Exclusions**.
*   **Search For:** `[AllowAnonymous]` (.NET) or `@PermitAll` (Java).
*   **Verify:** Every excluded method must be a legitimate public page.

### Step 3: If No Global Filter Exists (Default Allow)
*   **Your Job:** Assume **Nothing is Secure**. You must check every single sensitive method.
*   **Search For:** **Inclusions**.
*   **Search For:** `[Authorize]` or `@PreAuthorize`.
*   **Verify:** Every sensitive method must have an authorization attribute. Any method without one is a vulnerability.

### Step 4: Final Check
*   **Client-Side Logic:** No matter how good the server-side filters are, if the JavaScript contains `if (isAdmin)` logic, it can leak information about what endpoints exist (even if the user can't access them).
*   **Complete Mediation:** Even with filters, if a multi-step process exists, ensure the check is performed on every step, not just the first one.

---
*Reference: OWASP Code Review Guide 2.0 (Pages 136-137)*
