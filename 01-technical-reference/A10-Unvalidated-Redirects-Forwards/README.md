# A10: Unvalidated Redirects and Forwards — Technical Reference

## 1. Executive Summary
**Unvalidated Redirects and Forwards** occur when an application accepts untrusted input that determines the destination of a user's navigation.

According to the **OWASP Code Review Guide 2.0 (p. 150)**, attackers can exploit this to:
1.  **Redirect** victims to phishing or malware sites.
2.  Use **Forwards** to bypass access control and access unauthorized pages within the application.

### The "GPS Tampering" Analogy
*   Imagine your web application is a GPS. A user types in a destination, and you give them directions.
*   **The Flaw:** The application takes a "return URL" from a parameter without checking if it is a safe destination.
*   **The Attack:** An attacker sends a user a link like `http://YourBank.com/login?returnUrl=http://EvilPhishingSite.com`.
    *   The user sees a legitimate `YourBank.com` link and clicks it.
    *   After they log in, your application's GPS says, "Okay, now I'll send you to `http://EvilPhishingSite.com`."
    *   The user is redirected to a perfect clone of your bank, and their credentials are stolen.

**The Core Flaw:** The application trusts user-supplied data to control navigation.

---

## 2. Redirects vs. Forwards: A Critical Distinction
The guide (p. 150) separates these two because they have different attack vectors and impacts.

### A. Redirects (Client-Side Navigation)
*   **Mechanism:** The server sends an `HTTP 302 Found` response with a `Location` header. The **user's browser** sees this and makes a new request to the new URL.
*   **Impact:** **Phishing.** The attacker can send the user to any domain on the internet.
*   **Code to Look For (Sample 16.1 - 16.3):**
    *   Java: `response.sendRedirect(url)`
    *   .NET: `Response.Redirect(url)`
    *   PHP: `header("Location: " . $url)`

### B. Forwards (Server-Side Navigation)
*   **Mechanism:** The server **internally** fetches the content of another page and presents it to the user. The browser's URL **does not change**.
*   **Impact:** **Authorization Bypass.** The attacker can only forward to pages *on the same server*, but this can be used to access administrative pages.
*   **Code to Look For:**
    *   Java: `RequestDispatcher.forward()`
    *   .NET: `Server.Transfer()`

---

## 3. The Vulnerability in Action

### A. The "Open Redirect" (Phishing)
This is the most common form of this vulnerability.

**Vulnerable Code (C#):**
```csharp
// The URL parameter is untrusted input from the user
string url = request.QueryString["url"];

// The application blindly redirects to whatever the user provided
Response.Redirect(url);
```

**The Attack:**
1.  Attacker crafts a URL: `http://trusted-site.com/redirect?url=http://malicious-site.com`
2.  Attacker sends this link to a victim in a phishing email.
3.  Victim sees `trusted-site.com` and clicks it.
4.  The application redirects them to the malicious site.

### B. The "Forward" Bypass (Authorization)
This attack is more subtle and targets the application's internal logic.

**Vulnerable Logic (Java):**
```java
// On the "Accept Payment" page, there's a parameter for what to do next
String forwardUrl = request.getParameter("FWD");

if (forwardUrl.equals("purchase")) {
    // Forward to the purchase confirmation page
    dispatcher = context.getRequestDispatcher("/purchase.do");
} else if (forwardUrl.equals("cancel")) {
    // Forward to the cancellation page
    dispatcher = context.getRequestDispatcher("/cancelled.do");
}
dispatcher.forward(request, response);
```

**The Attack:**
*   The developer assumes `FWD` will only ever be "purchase" or "cancel".
*   An attacker crafts a request: `http://site.com/acceptPayment?FWD=/admin/viewAllUsers.do`
*   **The Flaw:** If the application has security checks on `/acceptPayment` but **not** on `/admin/viewAllUsers.do`, the forward bypasses the security check.

---

## 4. Remediation Strategies
The guide (p. 151) recommends two primary defenses.

### A. Whitelisting (The "Safe List" Approach)
Do not accept a full URL as a parameter. Instead, use an **indirect reference**.

**Secure Logic:**
```java
String destination = request.getParameter("dest");

String targetUrl;
switch (destination) {
    case "profile":
        targetUrl = "/user/profile.jsp";
        break;
    case "dashboard":
        targetUrl = "/dashboard.jsp";
        break;
    default:
        // Default to a safe page if the input is unknown
        targetUrl = "/home.jsp";
        break;
}
response.sendRedirect(targetUrl);
```
*   **Why it's Secure:** The attacker can only provide short keys ("profile", "dashboard"). The actual URL is hardcoded on the server.

### B. Input Validation (If a URL is Required)
If you *must* accept a URL, you must strictly validate it.

**Secure Validation Logic:**
1.  **Check for "http":** Ensure the URL starts with `http://` or `https://`. (Prevents `javascript:` XSS attacks).
2.  **Parse the URL:** Break the URL into its components (protocol, hostname, path).
3.  **Whitelist the Hostname:** Check if the hostname is on an approved list of safe domains.
    *   **Vulnerable:** `if (url.contains("trusted-site.com"))` -> Attacker uses `trusted-site.com.evil.com`.
    *   **Secure:** `if (hostname.equals("trusted-site.com") || hostname.endsWith(".trusted-site.com"))`.

---

## 5. Secure Code Reviewer’s Checklist (A10)

- [ ] **Grep for Sinks:** Search the codebase for `sendRedirect`, `Response.Redirect`, `header("Location:")`, `forward()`, and `Server.Transfer()`.
- [ ] **Trace the Source:** Is the destination URL or path derived from any user-controllable input (`getParameter`, `QueryString`)?
- [ ] **Whitelisting:** Is there a whitelist of allowed redirect targets?
- [ ] **Validation:** If a full URL is used, is it parsed and the hostname validated against a strict allow-list?
- [ ] **Relative Paths:** If the redirect is internal, does the code enforce relative paths (e.g., `/user/home`) instead of absolute URLs?
- [ ] **Referer Check:** The guide (p. 152) mentions that checking the `Referer` header can be a defense, but it is weak and can be bypassed. It should not be the only defense.

---
*Reference: OWASP Code Review Guide 2.0 (Pages 149-153)*
