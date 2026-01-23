# RBAC vs. ACL Implementation: Logic & Review Strategies

## 1. Executive Summary
Authorization is the process of deciding if User A is allowed to perform Action B on Object C.

*   **RBAC (Role-Based):** Authorization is based on the **Group** the user belongs to.
    *   *Analogy:* "I have a 'Staff' key card. It opens the front door and the break room."
*   **ACL (Access Control List):** Authorization is based on specific permissions assigned to a specific **Object**.
    *   *Analogy:* "There is a guest list on the VIP room door. If your name is on the list, you get in."

**The Reviewer's Goal:**
Verify that the logic matches the business requirement. Using RBAC when granular privacy is needed (e.g., patient records) causes data leaks. Using ACLs for broad system admin tasks causes maintenance nightmares.

---

## 2. Role-Based Access Control (RBAC)
**Reference:** PDF Page 134.

RBAC is the most common model for web applications. It simplifies management by grouping users.

### A. The Hierarchy
1.  **Users** are assigned to **Roles** (Many-to-Many).
2.  **Roles** are assigned to **Permissions** (Many-to-Many).
3.  **Code** checks the **Role** (or Permission).

### B. Implementation Logic
In code, RBAC usually looks like a check against a session property or a database lookup.

**Secure Code Pattern (.NET):**
```csharp
// The code asks: "Does the user wear the 'Manager' hat?"
if (User.IsInRole("Manager")) {
    ShowReports();
} else {
    throw new SecurityException("Access Denied");
}
```

**Secure Code Pattern (Java):**
```java
// The code asks: "Is this user in the 'Admin' group?"
if (request.isUserInRole("admin")) {
    deleteDatabase();
}
```

### C. Reviewer Red Flags for RBAC
1.  **Hardcoded Users:**
    *   **Vulnerable:** `if (username == "Steve") { allow(); }`
    *   **Why:** If Steve leaves the company, you have to recompile the code.
2.  **Role Explosion:**
    *   **Warning:** Creating roles like `Manager_North_Read`, `Manager_North_Write`, `Manager_South_Read`.
    *   **Why:** This indicates the developer is trying to force RBAC to do the job of an ACL. It leads to complexity and bugs.
3.  **Role Hierarchy Failures:**
    *   **Vulnerable:** `if (role == "Admin")`.
    *   **Secure:** `if (user.HasPermission("Edit"))`.
    *   **Why:** If you add a "SuperAdmin" role later, the code checking strictly for "Admin" will block them. Ideally, code checks for *capabilities* (permissions), and the database maps Roles to Capabilities.

---

## 3. Access Control Lists (ACLs)
**Reference:** PDF Page 134 ("This is a more logical modeling...").

ACLs are required when users need to share specific data with other specific users (e.g., "Share this Google Doc with Bob"). RBAC cannot handle this because you can't create a new "Role" for every single document.

### A. The Structure
An ACL is a table (or list) associated with a specific **Object ID**.
*   **Object:** Document #55.
*   **List:**
    *   User A: Read/Write
    *   User B: Read Only
    *   User C: No Access

### B. Implementation Logic
ACL checks are almost always performed **Database-Side** or in a **Service Layer**. They cannot be static annotations.

**Secure Code Pattern (Generic):**
```java
// 1. Get the Object ID from the user (The Request)
int docId = request.getParameter("doc_id");

// 2. Get the User ID (The Session)
int userId = session.getAttribute("user_id");

// 3. Ask the ACL Service
// "Does User X have 'Write' permission on Document Y?"
if (aclService.hasPermission(userId, docId, "WRITE")) {
    document.write(data);
} else {
    throw new AccessDeniedException();
}
```

### C. Reviewer Red Flags for ACLs
1.  **Missing The Check (IDOR):**
    *   **Vulnerable:** The code gets the `doc_id` and immediately runs `SELECT * FROM docs WHERE id=?`.
    *   **Why:** This assumes possession of the ID equals permission to view. This is the definition of **A4 - IDOR**.
2.  **Horizontal Privilege Escalation:**
    *   **Vulnerable:** User A changes the ID to User B's file. If the ACL check is weak (e.g., only checks "Is User Logged In"), User A sees the file.
3.  **Performance Shortcuts:**
    *   **Vulnerable:** Developers skip ACL checks on "List" views to make the SQL query faster.
    *   **Result:** The user sees a list of *every* document, even ones they can't open.

---

## 4. RBAC vs. ACL: The Comparison Table
Use this table during your exam or review to determine if the architecture fits the business need.

| Feature | RBAC | ACL |
| :--- | :--- | :--- |
| **Focus** | Functionality (Can I load the "Edit" page?) | Data (Can I edit "Invoice #10"?) |
| **Granularity** | Coarse (Group Level) | Fine (Individual Object Level) |
| **Maintenance** | Low (Add user to group) | High (Manage list per object) |
| **Storage** | `User_Roles` Table | `Object_Permissions` Table |
| **Best For** | Admin Panels, HR Systems, Public Sites | Social Networks, File Systems, Cloud Storage |
| **Code Location** | Annotations / Top of Controller | Inside Business Logic / Database Queries |

---

## 5. The Hybrid Approach (Context-Aware)
Most complex enterprise applications use **Both**.
*   **RBAC** determines if you can reach the feature.
*   **ACL** determines which data you see inside that feature.

**Secure Hybrid Pattern:**
```csharp
// 1. RBAC Check (Function Level)
[Authorize(Roles="Manager")] 
public ActionResult EditReport(int reportId) {
    
    // 2. ACL Check (Data Level)
    if (!permissionService.CanUserEditReport(CurrentUser, reportId)) {
        return Forbidden();
    }
    
    // Proceed...
}
```

**Reviewer Insight:**
If you see an `@Authorize(Roles="Manager")` annotation, do **not** stop reviewing. You must look inside the method to ensure there is *also* a check to ensure the Manager is editing a report *from their own department*. Missing the second check is a logic flaw.

---

## 6. Secure Code Reviewer’s Checklist (Access Control)

- [ ] **Definition:** Does the design document clearly state if the module uses RBAC or ACL?
- [ ] **Hardcoding:** Grep for hardcoded usernames or role strings in the code logic.
- [ ] **Default Deny:** If the Role/ACL lookup fails (returns null or error), does the code **Block** access? (It should).
- [ ] **Complete Mediation:**
    *   Is the check performed on **GET** (View)?
    *   Is the check performed on **POST** (Update)?
    *   Is the check performed on **DELETE**?
- [ ] **Consistency:** Are roles named consistently? (e.g., don't mix "admin", "administrator", and "sysadmin").
- [ ] **Re-Validation:** If a user's role changes (e.g., fired), does the application re-check the database on the next request, or does it trust the stale Session Token?

---
*Reference: OWASP Code Review Guide 2.0 (Pages 134, 137)*
