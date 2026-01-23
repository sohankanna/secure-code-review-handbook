# CSRF Token Implementation Patterns: A Deep Dive

## 1. Executive Summary
A simple "token" is not enough to stop CSRF. The way that token is **generated, stored, transmitted, and validated** determines its effectiveness. The goal of any token-based defense is to introduce a secret that the attacker on `evil.com` cannot guess or read, but which the victim's browser can submit automatically.

This document covers the three main patterns:
1.  **Synchronizer Token Pattern:** The "gold standard," stateful, and most secure.
2.  **Double Submit Cookies:** A stateless alternative, slightly less secure but easier for some architectures.
3.  **ASP.NET ViewStateUserKey:** A framework-specific implementation that binds the `ViewState` to a user's session.

---

## 2. The Synchronizer Token Pattern
**Reference:** PDF Page 142.

This is the most common and recommended pattern for CSRF defense. It is **stateful**, meaning the server must store the token for each user.

### The "Secret Handshake" Analogy
1.  When you enter a secret club (log in), the guard (server) whispers a secret password-of-the-day to you: **"Blue Eagle"**. He remembers this password.
2.  This secret is written on a hidden card in your pocket (the HTML form).
3.  When you want to do something important (like order a drink), you must tell the bartender the secret password.
4.  The bartender calls the guard and asks, "What's the password for this person?" The guard says, "Blue Eagle."
5.  If what you said matches what the guard remembers, your order is fulfilled. An outsider trying to yell an order from the street won't know the secret password.

### The Technical Logic Flow
1.  **Generation:** When a user logs in or requests a form, the server generates a cryptographically strong, random token (e.g., using `SecureRandom`).
2.  **Storage (Stateful):** The server stores this token in the user's **session data** on the server side.
3.  **Embedding:** The server embeds the same token into every state-changing HTML form as a hidden input field.
4.  **Submission:** When the user submits the form, the token is sent back as a parameter.
5.  **Validation:** The server performs the critical check:
    *   `if (form_token == session_token)` -> **Allow**
    *   `else` -> **Deny (CSRF Attack Detected!)**

### Code Review Analysis

**Client-Side Code to Look For (Sample 14.2):**
This is what you should see in the HTML source of a form.
```html
<form action="/transfer.do" method="post">
    <!-- The "Secret Handshake" -->
    <input type="hidden" 
           name="CSRFToken" 
           value="OWY4NmQwODE4ODRjN2Q2NTlhMmZlYWE..." />
    ...
</form>```

**Server-Side Logic to Verify (Java Pseudocode):**
```java
// On form submission
String formToken = request.getParameter("CSRFToken");
String sessionToken = (String) session.getAttribute("CSRFToken");

// The critical validation check
if (formToken != null && sessionToken != null && formToken.equals(sessionToken)) {
    // Process the transaction
} else {
    // SECURITY EVENT: Log and abort the request
    throw new SecurityException("Invalid CSRF Token");
}
```

### Per-Session vs. Per-Request Tokens
The guide (p. 143) mentions a way to further enhance security:
*   **Per-Session (Standard):** One token is generated and used for the entire user session.
*   **Per-Request (High Security):** A *new* token is generated for every single request. This is more secure but can break browser functionality like the "Back" button.

---

## 3. Double Submit Cookies
**Reference:** PDF Page 144.

This pattern is **stateless**, meaning the server does not need to store the CSRF token. This is useful for applications that use REST APIs or have multiple backend servers without shared session state.

### The "Two-Part Ticket" Analogy
1.  When you enter the theme park, the attendant gives you a ticket and rips it in half.
2.  They give you **Part A** as a wristband (the Cookie).
3.  They give you **Part B** as a paper stub (the Hidden Field).
4.  To get on a ride, you must show the operator both your wristband and your paper stub. They check if the two halves match.
5.  The key is that the operator **doesn't need to know** what the original ticket looked like. They just need to see if the two parts you present are identical.

### The Technical Logic Flow
1.  **Generation:** On login, the server generates a random token.
2.  **Transmission:** The server sends this token back to the browser in **two places**:
    *   As a **Cookie**.
    *   As a **Hidden Field** in the HTML form.
3.  **Submission:** When the form is submitted, the browser sends both the cookie and the form parameter automatically.
4.  **Validation:** The server compares the two values:
    *   `if (cookie_token == form_token)` -> **Allow**
    *   `else` -> **Deny**

### Why It Works: The Same-Origin Policy (SOP)
The entire security of this pattern rests on a browser security rule: An attacker on `evil.com` can make your browser send a request to `bank.com`, but they **cannot read the cookies** for `bank.com`.
*   The attacker can create a form with any hidden field they want, but they can't know the secret value inside the cookie to make them match.

### Code Review Checklist
*   **Stateless Check:** Verify that the server is **not** storing the token in the session.
*   **Comparison:** Find the code that reads the token from the request cookie and compares it to the token from the form body.
*   **Cookie Security:** The CSRF token cookie should be protected with `HttpOnly` to prevent JavaScript from reading it.

---

## 4. ASP.NET `ViewStateUserKey`
**Reference:** PDF Page 144 (Sample 14.3).

This is a specific defense for classic ASP.NET WebForms applications that use `ViewState`.

### Background: What is ViewState?
`ViewState` is a large, encrypted string that ASP.NET embeds in every form as a hidden field (`__VIEWSTATE`). It stores the state of all the controls on the page (e.g., the text in a textbox). When you submit the form, the `ViewState` is sent back so the server can reconstruct the page.

### The Vulnerability
By default, the `ViewState` is not tied to a specific user. It's theoretically possible for an attacker to trick a user into submitting a `ViewState` captured from another session, leading to a CSRF-like attack.

### The Defense: `ViewStateUserKey`
This property allows you to bind the `ViewState` to the user's current session.
*   **How it works:** When the `ViewState` is created, the framework mixes the value of `ViewStateUserKey` into the hashing algorithm.
*   **Best Practice:** Set this key to the user's `SessionID`.

### Secure Code Implementation (C# WebForms)
The guide (Sample 14.3) shows that this code must be placed in the `Page_Init` event. If it's in `Page_Load`, it is too late, and the protection will not work.

```csharp
// Must be in Page_Init, not Page_Load!
protected override void OnInit(EventArgs e) {
    base.OnInit(e);
    // Bind the ViewState to the unique Session ID.
    if (User.Identity.IsAuthenticated) {
        ViewStateUserKey = Session.SessionID;
    }
}
```

### Limitations
The guide notes a critical limitation:
> "ViewState MACs are only checked on POSTback, so any other application requests not using postbacks will happily allow CSRF."

This defense only works for WebForms PostBack events. It does not protect API endpoints or other parts of the application.

---

## 5. Summary Comparison for Reviewers

| Feature | Synchronizer Token | Double Submit Cookie | ViewStateUserKey |
| :--- | :--- | :--- | :--- |
| **Stateful?** | **Yes** (Server stores token) | **No** (Stateless) | **Yes** (Tied to Session state) |
| **Security** | **High** | **Medium-High** | **Medium** (Only for PostBacks) |
| **Complexity** | Medium (Requires session management) | Low (Easy to implement) | Low (One line of code) |
| **Where to Look** | Session logic, hidden fields | Cookie and form validation logic | `Page_Init` in ASP.NET code-behind |
| **Framework** | Universal | Universal | ASP.NET WebForms Only |

---
*Reference: OWASP Code Review Guide 2.0 (Pages 142-144)*
