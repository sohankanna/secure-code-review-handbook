# MVC Authorization Flaws: The "View-Based" Security Trap

## 1. Executive Summary
In an MVC architecture, the application is split into three components:
1.  **Controller:** Receives the request and decides what to do.
2.  **Model:** The business logic and data.
3.  **View:** What the user sees (HTML/UI).

**The Vulnerability:**
Developers often place the "Security Check" in the **View** layer to hide sensitive buttons (like "Delete User") from non-admins. However, if the **Controller** does not *also* check permissions before executing the action, an attacker can simply type the URL directly (Force Browsing) to execute the command.

**The Golden Rule:**
> "Much of the business logic processing (for a request) is done **before** the 'view' is executed." (PDF p. 134)

---

## 2. The Architecture of Failure (Figure 11 Analysis)
The guide provides a diagram (Figure 11 on Page 135) that illustrates exactly how this vulnerability works. Let's break down the flow of a **Vulnerable Application**.

### The Vulnerable Workflow
1.  **The Request:** An attacker sends a request to `/app/deleteUser?id=123`.
2.  **The Controller:** Receives the request. It looks up the Action Mapping.
3.  **The Instantiation:** The Controller creates the `DeleteAction` class.
4.  **The Execution (THE BUG):** The Controller calls `execute()` on the Action Class.
    *   *Note:* The user is deleted from the database **RIGHT HERE**.
5.  **The View Selection:** The Action returns "Success," and the Controller selects `success.jsp`.
6.  **The View Rendering:** The JSP page loads. Inside the JSP, there is a tag: `<if user="admin">Show Success</if> <else>Show Access Denied</else>`.
7.  **The Result:** The attacker sees an "Access Denied" page. They think they failed. **However, the user was already deleted in Step 4.**

### The "Ticket Taker" Analogy
Imagine a movie theater.
*   **Secure:** The Ticket Taker stands at the front door. You cannot enter the theater without a ticket.
*   **Vulnerable (View-Based Auth):** There is no Ticket Taker at the door. You walk in, sit down, and watch the movie. At the very end, a staff member asks to see your ticket. If you don't have one, they yell "Get Out!"
    *   *Result:* You were yelled at, but you still watched the movie.

In code, executing the business logic (deleting the user) is "watching the movie." Showing the error page is "yelling get out."

---

## 3. Vulnerable Code Patterns to Review

### A. The ASP.NET "PostBack" Flaw (PDF p. 135)
In classic ASP.NET WebForms, developers often mix UI logic with backend logic.

**Vulnerable Code:**
```csharp
void Page_Load(object sender, EventArgs e) {
    
    // Developer creates a UI element for Admins only
    if (User.IsInRole("Admin")) {
        DeleteButton.Visible = true;
    } else {
        DeleteButton.Visible = false; // "Security by Hiding"
    }

    // VULNERABILITY: Handling the Action
    // If the attacker sends a POST request simulating a click on 'DeleteButton',
    // ASP.NET might execute this event handler even if the button wasn't visible!
    if (IsPostBack && Request.Form["__EVENTTARGET"] == "DeleteButton") {
        DeleteUser(); // Executed without checking Role again!
    }
}
```

**The Reviewer's Fix:**
The `DeleteUser()` function (or the block calling it) must explicitly check `User.IsInRole("Admin")` **inside the execution block**, not just in the UI rendering block.

### B. The "Struts" Mapping Flaw (Sample 13.1)
Frameworks like Struts use XML files to map URLs to code. A common vulnerability is **Unused** or **Test** actions left in the configuration.

**Vulnerable `struts-config.xml`:**
```xml
<action-mappings>
    <!-- Secure Mapping -->
    <action path="/changePassword" type="com.site.ChangePasswordAction" />

    <!-- VULNERABLE MAPPING (Leftover from Dev) -->
    <!-- The developer assumes no one knows this URL exists -->
    <action path="/InsecureDesign/action/AddUserDetails" 
            type="Action.UserAction" />
</action-mappings>
```

**The Attack:**
An attacker guesses or "force browses" to `/InsecureDesign/action/AddUserDetails`.
Because the `UserAction` class likely doesn't have security checks (assuming it was just for testing), the attacker can add an admin user to the system.

**Reviewer Action:**
Review `web.xml`, `struts-config.xml`, or Spring `@Controller` mappings. Identifying "Test", "Debug", or "Old" endpoints is a critical part of the review.

---

## 4. Remediation: Where to put the checks?
To prevent this, authorization must happen at the **Controller** or **Filter** level, *before* any resource is touched.

### 1. The Intercepting Filter Pattern
Instead of checking permissions inside every function, use a Filter that sits in front of the Controller.

**Secure Flow:**
1.  **Request:** `/admin/deleteUser`.
2.  **Security Filter:** Intercepts request. Checks URL `/admin/*`. Checks User Role.
    *   *If Fail:* Redirect to Login or 403. Stop Processing.
    *   *If Pass:* Forward to Controller.
3.  **Controller:** Executes Logic.

### 2. Controller-Level Annotations
Modern frameworks allow you to "decorate" methods with security rules. This ensures the check runs before the code.

**.NET MVC (Secure):**
```csharp
[Authorize(Roles = "Admin")] // This runs BEFORE DeleteUser()
public ActionResult DeleteUser(int id) {
    db.Delete(id);
    return View("Success");
}
```

**Java Spring (Secure):**
```java
@PreAuthorize("hasRole('ADMIN')") // This runs BEFORE deleteUser()
public void deleteUser(int id) {
    userDao.delete(id);
}
```

---

## 5. Reviewer’s "MVC" Checklist

### Step 1: Map the URLs
*   Look at the configuration files (`web.xml`, `route.config`).
*   List every URL that performs a sensitive action (Edit, Delete, Add, Transfer).

### Step 2: Trace the Execution
*   For each sensitive URL, find the **Controller/Action Class**.
*   **Critical Question:** Does the very first line of the function check the user's permission?
*   **Critical Question:** Is there a "Filter" or "Attribute" on top of the function?

### Step 3: Check the View
*   Look at the HTML/JSP/ASPX files.
*   If you see logic like `<% if (admin) { %> <button>Delete</button> <% } %>`, immediately go back to Step 2 and verify the Controller.
*   If the Controller relies *only* on the View not rendering the button, **Flag it as A7 (High Severity).**

---
*Reference: OWASP Code Review Guide 2.0 (Pages 134-136, Figure 11)*
