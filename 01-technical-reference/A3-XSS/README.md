# A3: Cross-Site Scripting (XSS) — Technical Reference

## 1. Executive Summary
Cross-Site Scripting (XSS) is one of the most prevalent vulnerabilities in web applications. According to the **OWASP Code Review Guide 2.0 (p. 71)**, XSS allows an attacker to inject malicious scripts (usually JavaScript) into web pages viewed by other users.

Unlike SQL Injection (which attacks the database), XSS attacks the **Users** of the application.
*   **The Goal:** Execute scripts in the victim's browser.
*   **The Impact:** Session hijacking (stealing cookies), defacement, redirecting users to malware, or performing actions on behalf of the user (CSRF).

### The Root Cause: Lack of Contextual Encoding
The core failure in XSS is the application taking untrusted data and sending it to a web browser **without defining what that data is**. The browser blindly executes it as code (HTML/JS) instead of displaying it as text.

---

## 2. The Three Types of XSS (PDF p. 71)
As a reviewer, you must classify findings into one of these three buckets, as the remediation strategies differ slightly.

### A. Reflected XSS (Non-Persistent)
The malicious script is reflected off the web server, such as in an error message or search result.
*   **Source:** The request URL or form data (e.g., `http://site.com?search=<script>...`).
*   **Sink:** The HTML response body.
*   **Reviewer Check:** Look for places where `request.getParameter()` is immediately echoed back to the screen (e.g., "You searched for: [Input]").

### B. Stored XSS (Persistent)
The malicious script is stored permanently on the target server (database, forum post, comment field).
*   **Source:** The Database or File System.
*   **Sink:** The HTML response body (viewed by many users).
*   **Reviewer Check:** This is dangerous because the "Source" (the database) is often considered "Trusted." **Never trust the database.**

### C. DOM-Based XSS
The vulnerability exists entirely in the Client-Side code (JavaScript). The server might not even see the payload.
*   **Source:** `document.location`, `document.referrer`, `window.name`.
*   **Sink:** `document.write()`, `innerHTML`, `eval()`.
*   **Reviewer Check:** You must review the `.js` files, not just the server code.

---

## 3. Reviewing Code for XSS: The "Sinks"
The guide provides specific keywords (Code Crawling, p. 205) to grep for to find XSS vulnerabilities.

### Java (JSP/Servlet) Sinks
```java
// VULNERABLE: Printing raw input to the browser
out.println(request.getParameter("username")); 
```
**Search Terms:** `out.print`, `out.println`, `<%=`, `c:out` (check for `escapeXml="false"`).

### .NET (ASPX/MVC) Sinks
```csharp
// VULNERABLE: Writing raw input
Response.Write(Request.QueryString["id"]);
```
**Search Terms:** `Response.Write`, `<%=`, `Html.Raw` (MVC), `Label.Text` (WebForms).

### JavaScript (Client-Side) Sinks
```javascript
// VULNERABLE: Writing to the DOM
var name = document.location.search;
document.getElementById('profile').innerHTML = name;
```
**Search Terms:** `innerHTML`, `outerHTML`, `document.write`, `document.writeln`.

---

## 4. The Defense: Contextual Output Encoding
**Input Validation is NOT enough.** You cannot validate against every possible XSS vector because legal characters (like `'` or `>`) are often valid in names (e.g., O'Reilly).

The solution is **Output Encoding** based on **Where** the data is placed (The Context).

### A. HTML Body Context
If data is going between `<div>...</div>`:
*   **Rule:** Encode `&`, `<`, `>`, `"`, `'`, `/`.
*   **Transformation:** `<script>` becomes `&lt;script&gt;`.
*   **Result:** The browser displays the text "<script>" but does not execute it.

### B. HTML Attribute Context (PDF p. 73)
If data is going inside an attribute: `<input value="[DATA]">`.
*   **The Risk:** An attacker can close the attribute quotes (`"`) and start a new event handler (`onload=...`).
*   **Rule:** Attribute Encode.
*   **Vulnerable:** `<input type="text" name="fname" value="<%= request.getParameter("name") %>">`
*   **Attack:** `"><script>alert(1)</script>`

### C. JavaScript Context (PDF p. 74)
If data is going inside a script block: `<script>var name = '[DATA]';</script>`.
*   **The Risk:** HTML encoding helps, but isn't enough. The attacker can break out of the string using `'`.
*   **Rule:** JavaScript Escape (Unicode Escapes).
*   **Example:** `'; alert(1); //`

---

## 5. Dangerous Framework Features (Red Flags)

### .NET Request Validation (PDF p. 72)
The guide explicitly warns:
> "You can never use request validation for securing your application against cross-site scripting attacks."

If you see `<pages validateRequest="false" />` in `web.config` or `<%@ Page ... ValidateRequest="false" %>` in a file, this is a **High Severity Finding**. The developer has turned off the basic safety net.

### Anti-XSS Libraries
Developers should not write their own encoders. Look for standard libraries:
*   **Java:** OWASP Java Encoder Project.
*   **Microsoft:** AntiXSS Library (`Encoder.HtmlEncode`).
*   **JavaScript:** DOMPurify (for sanitizing HTML).

---

## 6. Advanced Concept: Nested Contexts (PDF p. 76)
The guide highlights situations where data passes through multiple parsers.

**Example:**
```html
<div onclick="showError('[DATA]')">...</div>
```
Here, the data is:
1.  Inside an HTML Attribute (`onclick`).
2.  Inside a JavaScript function (`showError`).

**Reviewer Action:** The data must be **JavaScript Encoded** FIRST, and then **HTML Attribute Encoded** SECOND. If the developer only does one, the application is likely vulnerable.

---

## 7. Secure Code Reviewer’s Checklist (A3)

- [ ] **Context Identification:** For every variable printed to the screen, identify the context (HTML Body, Attribute, Script, CSS, URL).
- [ ] **Matching Encoder:** Is the correct encoder used? (e.g., don't use URL Encoding for HTML Body).
- [ ] **Raw Output:** Search for "Unsafe" methods like `Html.Raw()` (.NET) or `utext` (Struts).
- [ ] **DOM XSS:** Search .js files for `innerHTML`. Is the data sanitized before assignment?
- [ ] **Configuration:** Is `.NET Request Validation` enabled? (It should be, even though it's not a complete fix).
- [ ] **Content Security Policy (CSP):** Does the application send a `Content-Security-Policy` header? (PDF p. 52).
- [ ] **HTTP Headers:** Is `X-XSS-Protection: 1; mode=block` enabled?

---

## 8. Summary for the Exam
*   **XSS** happens when the browser executes user data as code.
*   **Reflected** comes from the request; **Stored** comes from the DB; **DOM** stays in the browser.
*   **Encoding** must match the **Context**.
*   **Input Validation** is a defense in depth, but **Output Encoding** is the cure.
*   **Blacklisting** scripts (filtering `<script>`) always fails.

---
*Reference: OWASP Code Review Guide 2.0 (Pages 70-76, 205)*
