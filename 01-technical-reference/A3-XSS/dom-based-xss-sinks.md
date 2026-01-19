# DOM-Based XSS Sinks: The Enemy Within

## 1. Executive Summary: "The Call is Coming from Inside the House"
In Reflected and Stored XSS, the malicious script comes from the server. In **DOM-Based XSS**, the server is often innocent. The vulnerability happens because the JavaScript code running in the user's browser takes data from a **Source** and hands it to a dangerous **Sink** without checking it.

### The Mechanics
1.  **Source:** A JavaScript property that the attacker can control (e.g., the URL).
2.  **Sink:** A function that executes code or renders HTML (e.g., `document.write`).
3.  **The Flow:** Data moves from **Source -> Sink** without validation.

**Why Reviewers Miss This:**
Standard server-side tools (like checking Java or C# code) often miss these bugs because the server never sees the malicious payload (especially if it's after the `#` hash in the URL). You **must** review the `.js` files.

---

## 2. The Dangerous Sinks (Execution Points)
A "Sink" is a function or DOM object that allows the execution of JavaScript or the rendering of HTML. If tainted data reaches these sinks, XSS occurs.

### A. HTML Execution Sinks
These sinks take a string and turn it into visual HTML elements. If the string contains `<script>`, the browser executes it.

#### 1. `document.write()` (Sample 25.1)
This is an older method but highly dangerous. It writes raw HTML directly into the page stream.

**Vulnerable Code (PDF p. 189):**
```javascript
// Source: Reading the URL
var pos = document.URL.indexOf("name=") + 5;
var name = document.URL.substring(pos, document.URL.length);

// Sink: Writing directly to the page
document.write(name);
```
*   **The Attack:** `http://site.com?name=<script>stealCookies()</script>`
*   **Result:** The script tag is written to the DOM and executes immediately.

#### 2. `innerHTML` (The Modern Threat)
This is the most common sink in modern web apps. It replaces the content of an element.

**Vulnerable Code:**
```javascript
var searchParams = document.location.search; // Source
var element = document.getElementById("results");

// Sink: innerHTML parses the string as HTML
element.innerHTML = "You searched for: " + searchParams;
```
*   **Note:** In HTML5, `innerHTML` generally won't execute `<script>` tags inserted this way, but it **will** execute event handlers like `<img src=x onerror=alert(1)>`.

#### 3. jQuery Sinks (`.html()`)
The guide (p. 188) notes that frameworks like jQuery can aggravate the problem.
*   **Vulnerable:** `$('#div').html(userInput);`
*   **Safe:** `$('#div').text(userInput);`

---

### B. Execution Sinks (The "Eval" Family)
These sinks take a string and run it as pure JavaScript code. These are extremely dangerous.

#### 1. `eval()` (PDF p. 190)
The guide explicitly states: **"eval() is prone to security threats, and thus not recommended to be used."**

**Vulnerable Code (Processing JSON):**
```javascript
var userData = request.responseText; // Untrusted JSON from server
var json = eval("(" + userData + ")"); // Sink
```
*   **The Attack:** If `userData` contains `alert(1);`, it executes.
*   **The Fix:** Use `JSON.parse(userData)`.

#### 2. `setTimeout` and `setInterval`
These functions usually take a function callback, but they can also accept a **String**. If they accept a string, they behave exactly like `eval()`.

**Vulnerable Code:**
```javascript
var callback = location.hash.substring(1); // Source: #myFunction
// Sink: setTimeout treating string as code
window.setTimeout("customCallback('" + callback + "')", 1000);
```
*   **The Attack:** `#'); alert('Hacked'); //`

---

### C. Navigation Sinks (Open Redirects)
These sinks change the page location. While often used for Phishing (Open Redirects), they can cause XSS if the attacker uses the `javascript:` protocol.

**Sinks:**
*   `document.location`
*   `window.location.href`
*   `window.navigate`

**Vulnerable Code (Sample 25.2):**
```javascript
var url = document.location.url;
// ... logic to extract login path ...
document.location.url = url; // Sink
```
*   **The Attack:** If the attacker sets the URL to `javascript:alert(1)`, the browser will execute the code instead of navigating to a page.

---

## 3. The Tainted Sources (Where Data Comes From)
As a reviewer, when you grep for Sinks, you must trace the variable back to see if it came from a "Tainted Source." The guide (p. 189) lists these critical sources:

| Source | Description | Risk Level |
| :--- | :--- | :--- |
| `document.URL` | The full URL of the page. | **High** |
| `document.location` | Similar to URL, includes Hash/Fragment. | **High** |
| `document.referrer` | The URL of the *previous* page. | **High** (Attacker controls the previous page) |
| `window.name` | A property that persists across page loads. | **High** (Often used to smuggle payloads) |
| `location.search` | The Query String (`?id=1`). | **High** |
| `location.hash` | The Fragment (`#section`). | **High** (Not sent to server, pure Client-Side) |
| `localStorage` | HTML5 Local Storage. | **Medium** (If attacker polluted it previously) |

---

## 4. Remediation Strategies

### A. Use Safe Sinks
Replace dangerous sinks with safe alternatives that treat input as text, not code.

| Dangerous Sink | Safe Alternative | Why it's Safe |
| :--- | :--- | :--- |
| `innerHTML` | `innerText` / `textContent` | Renders `<script>` as visible text, doesn't run it. |
| `document.write` | (None) / DOM Manipulation | Use `document.createElement()` and `appendChild()`. |
| `eval()` | `JSON.parse()` | Parses data structure only, doesn't execute logic. |
| `setTimeout("string")` | `setTimeout(function)` | Passes a code reference, not a compilable string. |

### B. JavaScript Encoding
If you *must* put untrusted data into a dangerous sink (rare), you must encode it.
*   **Hex Encoding:** Convert characters to `\xHH`.
*   **Unicode Encoding:** Convert characters to `\uXXXX`.

---

## 5. Reviewer’s Checklist: Finding DOM XSS (Grep Sheet)
Use this list to search `.js` and `.html` files (Derived from Appendix C, p. 215).

### Search for Sinks (Red Flags)
- [ ] `document.write` / `document.writeln`
- [ ] `.innerHTML` / `.outerHTML`
- [ ] `eval(`
- [ ] `setTimeout(` / `setInterval(` (Check if arguments are strings)
- [ ] `execScript`
- [ ] `.src` (Check for `javascript:` protocol injection)
- [ ] `location.href` / `location.replace`

### Search for Sources (Taint Tracing)
- [ ] `location.search`
- [ ] `location.hash`
- [ ] `document.referrer`
- [ ] `window.name`

### Search for Framework Specifics
- [ ] **jQuery:** `.html()`, `.append()`, `.prepend()`, `.wrap()`
- [ ] **Angular:** `ng-bind-html` (TrustAsHtml)
- [ ] **React:** `dangerouslySetInnerHTML` (The name says it all!)

---

## 6. Summary for the Exam
*   **DOM XSS** happens in the browser.
*   **Source:** Where the bad data comes from (`location.hash`).
*   **Sink:** Where the bad data executes (`innerHTML`).
*   **Reviewer Job:** Trace the path from Source to Sink. If there is no validation/encoding in between -> **Vulnerable**.
*   **Primary Fix:** Use `textContent` instead of `innerHTML`.

---
*Reference: OWASP Code Review Guide 2.0 (Sections 17, 25, Appendix C)*
