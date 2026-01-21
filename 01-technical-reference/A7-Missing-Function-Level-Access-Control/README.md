# A7: Missing Function Level Access Control — Technical Reference

## 1. Executive Summary
**Missing Function Level Access Control** occurs when an application verifies permissions in the User Interface (UI) but fails to enforce them on the Server.

According to the **OWASP Code Review Guide 2.0 (p. 134)**, developers often assume that if they hide a button (e.g., the "Admin Panel" link), a standard user cannot access that function.
*   **The Reality:** Attackers do not use the UI. They use tools like Burp Suite or `curl` to **Force Browse** directly to the URL (e.g., `/admin/deleteUser`).
*   **The Defect:** If the server does not perform an Authorization Check **inside the function itself**, the attack succeeds.

---

## 2. The Core Logic: Authentication vs. Authorization
The guide (p. 134) stresses that these are two distinct mechanisms. A Secure Code Reviewer must verify both.

| Concept | Question | Vulnerability |
| :--- | :--- | :--- |
| **Authentication (AuthN)** | "Who are you?" | A2 - Broken Authentication |
| **Authorization (AuthZ)** | "Are you allowed to do this?" | **A7 - Missing Function Level Access Control** |

**The Code Review Trap:**
You might see code that checks `if (user.isLoggedIn())`. This is an **Authentication** check. It prevents anonymous access, but it does **not** prevent a regular user from acting like an Admin. You must look for `if (user.hasRole("Admin"))`.

---

## 3. The MVC Design Flaw (The "View" Trap)
**Reference:** PDF Page 135 (Figure 11).

This is the most common architectural failure in authorization.
In a Model-View-Controller (MVC) architecture, the flow is:
**Request -> Controller -> Model -> View**.

### The Vulnerable Pattern
1.  **Controller:** Receives request `/deleteUser`. Executes logic. Deletes User.
2.  **View:** Checks `if (user.isAdmin)`. If true, show success message. If false, show "Access Denied" HTML.

**Why this fails:**
The **Action** (Deleting the user) happened in Step 1. The **Check** happened in Step 2.
Even though the attacker sees an "Access Denied" page, the user was already deleted.

**Reviewer Mandate:**
Authorization checks must happen **before** any business logic or database calls are executed.

---

## 4. Authorization Models (RBAC vs. ACL)
**Reference:** PDF Page 134.

### A. Role-Based Access Control (RBAC)
Users are assigned to **Roles** (Admin, Manager, Guest). Code checks the Role.
*   **Code:** `if (User.IsInRole("Manager")) { ... }`
*   **Reviewer Check:** Ensure roles are not hardcoded (e.g., `if user == "Steve"`). Ensure there is a system to manage/revoke roles (p. 137).

### B. Access Control Lists (ACLs)
Users are assigned specific permissions on specific objects.
*   **Code:** `if (User.HasPermission("Write", "File_A")) { ... }`
*   **Reviewer Check:** This is more granular than RBAC. Ensure the ACL table is checked on every request.

---

## 5. The "Golden Rule": Complete Mediation (PDF p. 137)
> "Follow the principle of 'complete mediation', where authorization is checked at every stage of a function."

**Scenario:** An application has a 4-step wizard to transfer money.
1.  `SelectAccount.jsp`
2.  `EnterAmount.jsp`
3.  `Confirm.jsp`
4.  `ExecuteTransfer.do`

**The Vulnerability:**
The developer checks permissions on Step 1.
The attacker skips directly to Step 4 (`POST /ExecuteTransfer.do`).
If Step 4 does not re-check permissions, the attack succeeds.

---

## 6. Secure Code Reviewer’s Checklist (A7)

### Architecture & Design
- [ ] **Placement:** Are checks performed in the **Controller/Business Logic**, not the View/JSP?
- [ ] **Default Deny:** Does the application deny access by default? (e.g., if the role check fails, does it stop execution immediately?).
- [ ] **Configuration:** Check `web.xml` (Java) or `web.config` (.NET). Are administrative folders (`/admin`) locked down by URL patterns?

### Code Logic
- [ ] **Granularity:** Is every function (GET and POST) protected?
- [ ] **Force Browsing:** Can a user access a "hidden" URL (e.g., `test_action.do` mentioned on p. 135) that was left over from development?
- [ ] **Hardcoding:** Are specific usernames hardcoded as admins? (e.g., `if (user == "admin")`). This is a fail.
- [ ] **Client-Side Logic:** Is there JavaScript that says `if (admin) { showButton(); }`? If the API endpoint doesn't check as well, it's vulnerable.

### Modern Frameworks
- [ ] **Decorators:** Are methods tagged with `[Authorize(Roles="Admin")]` (.NET) or `@PreAuthorize` (Java Spring)?
- [ ] **Public Access:** Search for `[AllowAnonymous]` or `@PermitAll`. Verify these are strictly for public pages (Login/Home).

---
*Reference: OWASP Code Review Guide 2.0 (Pages 133-138)*
