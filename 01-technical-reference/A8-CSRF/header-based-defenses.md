# Header-Based Defenses: SameSite, Referer, and Origin

## 1. Executive Summary
Instead of checking for a token inside the request body, header-based defenses look at the **metadata** of the HTTP request itself.
They ask two main questions:

1.  **Where did this request come from?** (Referer/Origin Check).
2.  **Should the browser have sent the cookie at all?** (SameSite Attribute).

These defenses are often easier to implement globally than token-based systems, but they are not a complete solution and can be
bypassed.

---

## 2. The `SameSite` Cookie Attribute (The Modern Defense)
While not explicitly detailed in the OWASP Code Review Guide 2.0 (which predates its widespread adoption), the `SameSite`
attribute is now a **primary defense** against CSRF and should be part of any modern code review.

### The Concept
The `SameSite` attribute is a flag on a cookie that tells the browser when it should (and should not) send the cookie with a
request. This attacks the root cause of CSRF: the browser's willingness to automatically attach cookies to cross-site requests.

### The Three Modes

1.  **`SameSite=Strict`** (Most Secure)
    *   **Rule:** The browser will **only** send the cookie if the request originates from the **exact same domain**.
    *   **Impact:** If a user clicks a link from `gmail.com` to `bank.com`, the browser will **not** send the `bank.com`
        session cookie. The user will appear to be logged out. This breaks many common user workflows.
    *   **Use Case:** Ideal for high-security, state-changing actions that should never be initiated from an external link (e.g.,
        "Change Password").

2.  **`SameSite=Lax`** (The Default in Modern Browsers)
    *   **Rule:** The browser sends the cookie for "top-level navigations" (e.g., clicking a link) but **blocks** it on
        sub-requests like `<img>`, `<iframe>`, and `POST` forms from other domains.
    *   **Impact:** This stops the classic `<img>` and hidden `POST` form CSRF attacks, while still allowing users to stay logged
        in when they click a link from an email.

3.  **`SameSite=None`**
    *   **Rule:** The old, insecure behavior. The browser sends the cookie on all requests.
    *   **Requirement:** To use `None`, you **must** also specify the `Secure` attribute (`SameSite=None; Secure`).

### Code Review Checklist for `SameSite`
*   **Presence:** Look at the code that sets the session cookie. Is `SameSite` being set?
*   **Value:**
    *   If `Lax` or `Strict`, this is a **Pass**.
    *   If `None`, this is a **Warning**. Verify that this is intentional (e.g., for a legitimate cross-site API).
    *   If **Missing**, this is a **Finding**. The application is relying on browser defaults, which can change.

**Java (Servlet API 4.0+) Example:**
```java
// Secure by default
response.setHeader("Set-Cookie", "session_id=...; SameSite=Lax");
```

**.NET (`web.config`) Example:**
```xml
<system.web>
    <!-- Sets SameSite=Lax for all cookies issued by ASP.NET -->
    <sessionState cookieSameSite="Lax" />
</system.web>
```

---

## 3. The `Referer` Header Check
**Reference:** PDF Page 145.

The `Referer` header tells the server the URL of the page that **initiated** the request.

### The Defense Logic
The server checks the `Referer` header. If the request is a `POST` to `/transfer`, the server checks if the `Referer` is a
page from its own domain.
*   **Legitimate:** `Referer: https://bank.com/transfer-form` -> **Allow**.
*   **Forged:** `Referer: https://evil.com/cat-pictures.html` -> **Deny**.

### Why It Is a **Weak** Defense
The guide explicitly states that this is a "weaker form of CSRF protection."

1.  **Privacy Stripping:** Many browsers, proxies, and security software are configured to **strip** the `Referer` header for
    privacy. In this case, the header is missing, and the server must decide whether to block or allow the request.
2.  **Downgrade Attack:** If a user goes from an `HTTPS` site to an `HTTP` site, the `Referer` is stripped.
3.  **Spoofing:** While an attacker cannot spoof the `Referer` in the victim's browser, they can find other bugs to make it look
    legitimate.
4.  **Open Redirect Vulnerabilities:** If `bank.com` has an open redirect flaw, the attacker can craft a URL like
    `https://bank.com/redirect?url=evil.com`. The request will originate from `bank.com`, making the `Referer` look valid.

### Code Review Checklist
*   **Null Check:** If the `Referer` is missing, does the code **deny** the request? (It should).
*   **String Matching:** Is the validation too loose?
    *   **Vulnerable:** `if (referer.contains("site.com"))` -> Attacker uses `site.com.evil.com`.
    *   **Secure:** The code must check that the **hostname** *exactly* matches the trusted domain.

---

## 4. The `Origin` Header Check
**Reference:** PDF Page 145.

The `Origin` header is a newer, more secure alternative to the `Referer`. It is specifically designed for security checks.

### Key Differences
*   **Content:** The `Origin` header only contains the **Scheme, Host, and Port**. It does **not** include the path or query
    string.
    *   `Referer`: `https://bank.com/user/123/profile`
    *   `Origin`: `https://bank.com`
*   **Reliability:** The browser sends the `Origin` header on all cross-site `POST`, `PUT`, and `DELETE` requests. It is more
    consistent than `Referer`.
*   **HTTPS:** Unlike `Referer`, the `Origin` is sent on requests from an `HTTPS` origin.

### Defense Logic
The server should check that the `Origin` header exactly matches its own origin.

**Secure Java Pseudocode:**
```java
String originHeader = request.getHeader("Origin");
String expectedOrigin = "https://www.myapplication.com";

// On a state-changing POST request
if (expectedOrigin.equals(originHeader)) {
    // Process the request
} else {
    // CSRF Attempt! Log and deny.
}
```

### Reviewer Insight
*   While better than `Referer`, a strict `Origin` check can still be brittle and might block legitimate requests from subdomains
    if not configured carefully.
*   The `Origin` header is the primary defense used by **CORS (Cross-Origin Resource Sharing)**.

---

## 5. The Challenge-Response Pattern
**Reference:** PDF Page 145.

This is a very strong defense but impacts the user experience. It should be reserved for high-risk actions. Instead of a
silent token, the user is actively challenged.

### Implementation Examples
1.  **Re-Authentication:** The user clicks "Transfer Funds." The server responds with a page asking them to **re-enter their
    password**.
2.  **CAPTCHA:** The user must solve a CAPTCHA before the action is committed.
3.  **One-Time Token (OTP):** The user receives a code via SMS or an Authenticator App and must enter it to proceed.

### Reviewer Checklist
*   **Is it necessary?** Is this being used for a truly critical action (like changing an email address or transferring
    $10,000)? Using it for "posting a comment" is overkill.
*   **Is it secure?** The challenge itself must be secure (e.g., the OTP must be rate-limited, the CAPTCHA must not be
    replayable).

---

## 6. Summary: Defense in Depth

No single header-based defense is a silver bullet. The best approach is **Defense in Depth**.

**The Ideal Hierarchy:**
1.  **Primary Defense:** Use the **Synchronizer Token Pattern** or **Double Submit Cookies**.
2.  **Browser Defense:** Set the **`SameSite=Lax` or `Strict`** attribute on the session cookie.
3.  **Integrity Check:** For `POST`/`PUT`/`DELETE` requests, validate the **`Origin`** header.
4.  **High-Risk Actions:** Use a **Challenge-Response** (re-authentication).

A code reviewer should look for the presence of the Synchronizer Token first. If it's missing, the application is likely
vulnerable, even if it checks the `Referer`.

---
*Reference: OWASP Code Review Guide 2.0 (Page 145)*
