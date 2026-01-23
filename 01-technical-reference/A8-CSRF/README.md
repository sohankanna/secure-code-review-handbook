# A8: Cross-Site Request Forgery (CSRF) — Technical Reference

## 1. Executive Summary
**Cross-Site Request Forgery (CSRF)** is an attack that tricks a victim's web browser into executing an unwanted action on a web application to which they are currently authenticated.

According to the **OWASP Code Review Guide 2.0 (p. 140)**, the attacker forces a logged-on victim's browser to send a forged HTTP request, which automatically includes the victim's session cookie and any other authentication tokens. The vulnerable application thinks the request is legitimate because it came from an authenticated user.

### The "Forged Check" Analogy
Imagine you are logged into your online bank. This is like having your checkbook open on the table, pre-signed.
*   An attacker sends you an email with a "cute cat picture."
*   You click on the picture.
*   The picture is actually a hidden command. The `<img>` tag in the email is secretly a "check" made out to the attacker for $1,000.
*   Your browser, trying to load the "picture," sends the request to the bank.
*   Because you are logged in, your browser automatically attaches your "signature" (the session cookie).
*   The bank sees a valid, signed check and transfers the money. **You never intended to do this.**

**The Core Flaw:** The server validates **who** sent the request (Authentication) but not **if** they intended to send it (Intent).

---

## 2. CSRF vs. XSS: The Critical Distinction (PDF p. 140)
This is a fundamental concept for your exam.
*   **XSS (Cross-Site Scripting):** The attacker injects malicious script **into** the vulnerable website. The trust is between the User and the Vulnerable Site.
*   **CSRF (Cross-Site Request Forgery):** The attacker, on an **external site** (`evil.com`), forges a request that is sent **to** the vulnerable website (`bank.com`). The trust is between the Vulnerable Site and the User's Browser.

---

## 3. The Attack in Action: How it Works
The attack relies on the fact that browsers automatically include cookies for a domain with every request to that domain.

### A. The GET Request Attack (The `<img>` Trick)
If a website allows state-changing actions via GET requests, it is trivially vulnerable.

**Vulnerable URL:**
`http://bank.com/transfer?to=bob&amount=1000`

**Attacker's Email Code:**
```html
<p>Check out this cute cat!</p>
<!-- The browser will make a GET request to the 'src' URL -->
<img src="http://bank.com/transfer?to=attacker&amount=10000" width="1" height="1" />
```*   **Reviewer Red Flag:** Any URL that uses `GET` to change data (`delete`, `update`, `transfer`, `buy`) is a **High Severity Finding**.

### B. The POST Request Attack (The Hidden Form)
The guide (p. 141) explicitly states that only accepting POST requests is **NOT** a fix. The attacker simply creates a hidden form on their own website.

**Attacker's Website (`evil.com`):**```html
<body onload="document.forms[0].submit()">
  <form action="http://vulnerable-site.com/change_password" method="POST">
    <input type="hidden" name="new_password" value="hacked123" />
    <input type="hidden" name="confirm_password" value="hacked123" />
  </form>
</body>
```
*   **The Flow:** The victim visits `evil.com`. The JavaScript automatically submits the form. The browser sends a `POST` request to `vulnerable-site.com` and includes the session cookie. The password is changed.

---

## 4. The "Golden Standard" Defense: Synchronizer Token Pattern
**Reference:** PDF Page 142.

This is the most robust and widely used defense against CSRF.

### The Logic
1.  When the user visits a form, the server generates a **large, random, secret token** (the "Anti-CSRF Token").
2.  The server stores this token in the user's **session**.
3.  The server embeds this token in the HTML form as a **hidden field**.
4.  When the user submits the form, the token is sent back.
5.  The server **compares** the token from the form with the token in the session.
    *   **If they match:** The request is valid.
    *   **If they DO NOT match (or the token is missing):** The request is forged. Reject it.

### Secure Code Example (Sample 14.2)
```html
<form action="/transfer.do" method="post">
    <!-- 
        This token is unique to this user's session.
        An attacker on another site has NO WAY of knowing what it is.
    -->
    <input type="hidden" name="CSRFToken" value="OWY4NmQwODE4ODRjN2Q2NTlhMmZlYWE..." />
    ...
</form>
```

---

## 5. Other Defense Mechanisms

### A. Double Submit Cookies
A stateless alternative to the Synchronizer Token.
1.  Server generates a random token and sends it as **both** a Cookie and a Hidden Form Field.
2.  The server does **not** store the token.
3.  On submission, the server checks if the token from the cookie **matches** the token from the form field.
*   **Why it works:** An attacker on another domain cannot read the cookie to copy its value into the form field (Same-Origin Policy).

### B. SameSite Cookie Attribute
A modern browser defense. You set an attribute on the session cookie.
`Set-Cookie: SessionID=...; SameSite=Strict`
*   `Strict`: The browser will **not** send the cookie on any cross-site request.
*   `Lax`: The cookie is sent on top-level GET requests (e.g., clicking a link), but not POSTs or `<img>` requests.
*   **Reviewer Action:** Check for `SameSite=Strict` or `SameSite=Lax` on session cookies.

### C. Referer / Origin Header Check
The server can check the `Referer` or `Origin` header to see where the request came from. If it's not from its own domain, it can reject it.
*   **Weakness (PDF p. 145):** The `Referer` header can be spoofed or stripped for privacy reasons. Open Redirect vulnerabilities can be used to make the Referer look legitimate.

---

## 6. Reviewer's Checklist for CSRF
- [ ] **State-Changing GET:** Grep the codebase for actions like `delete`, `update`, `transfer` that are mapped to `GET` requests.
- [ ] **Token Presence:** For every `POST` form that performs a sensitive action, is there a hidden field for an Anti-CSRF token?
- [ ] **Token Validation:** Find the server-side code that processes the form. Does it compare the submitted token against a value stored in the user's session?
- [ ] **Token Uniqueness:** Is the token unique per session? (It should not be a static, hardcoded value).
- [ ] **SameSite Cookies:** Check the cookie configuration. Is `SameSite=Strict` or `Lax` used?
- [ ] **Stateless CSRF:** If Double Submit Cookies are used, does the server validate that both the cookie token and form token are identical?

---
*Reference: OWASP Code Review Guide 2.0 (Pages 139-146)*
