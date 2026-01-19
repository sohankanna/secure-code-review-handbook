# XSS Types Deep Dive: Reflected, Stored, and DOM

## 1. Introduction: The Three Vectors
Cross-Site Scripting (XSS) is not a single vulnerability; it is a class of attacks. While the end result is often the same (JavaScript execution in the victim's browser), the **path** the malicious data takes to get there differs significantly.

As a Secure Code Reviewer, you must be able to classify the XSS type because the **remediation strategy** changes depending on the vector.

| Type | Persistency | Location of Flaw | The "Source" | The "Sink" |
| :--- | :--- | :--- | :--- | :--- |
| **Reflected** | Non-Persistent | Server-Side | HTTP Request (URL/Form) | HTTP Response Body |
| **Stored** | Persistent | Server-Side | Database / File System | HTTP Response Body |
| **DOM-Based** | Non-Persistent | Client-Side | DOM Objects (`document.url`) | DOM Execution (`innerHTML`) |

---

## 2. Reflected XSS (Non-Persistent)
This is the most common and "classic" form of XSS. The attack is **immediate**. The malicious script is part of the request sent by the victim and is "reflected" back by the web server in the response.

### The Mechanism: "The Mirror"
1.  **Attacker** creates a malicious link: `http://vulnerable-site.com/search?q=<script>alert('Hacked')</script>`
2.  **Victim** clicks the link.
3.  **Server** receives the parameter `q`.
4.  **Server** logic says: *"You searched for: [Input]"*.
5.  **Browser** renders the response. Because the server didn't encode the input, the browser executes the alert script.

### Code Review Analysis: Locating Reflected XSS

#### Java (JSP) Vulnerability
In Java Web Applications, look for `request.getParameter` flowing directly into `out.print`.

```java
// VULNERABLE CODE
<% 
    String query = request.getParameter("search_term"); 
%>
<!-- The sink is the expression tag <%= %> which prints raw HTML -->
<h1>Search Results for: <%= query %> </h1>
```
*   **Reviewer Logic:** The variable `query` is tainted. It goes straight from the Source (`getParameter`) to the Sink (`<%=`). There is no encoding step (like `ESAPI.encoder().encodeForHTML()`) in between.

#### .NET (ASPX) Vulnerability
In .NET, look for `Response.Write`.

```csharp
// VULNERABLE CODE
protected void Page_Load(object sender, EventArgs e)
{
    string username = Request.QueryString["user"];
    // The sink is Response.Write
    Response.Write("Welcome back, " + username); 
}
```
*   **Reviewer Logic:** The `username` is taken from the URL and written directly to the page. If the input contains `<script>`, it executes.

### Key Characteristics for the Exam:
*   Requires **Social Engineering**: The attacker must trick the user into clicking a link.
*   **Scope:** It usually affects only the user who clicked the link.

---

## 3. Stored XSS (Persistent)
Stored XSS is more dangerous than Reflected XSS because it is a **Time Bomb**. The attack does not require a malicious link; the payload is waiting inside the application itself.

### The Mechanism: "The Graffiti"
1.  **Attacker** logs into a forum or comment section.
2.  **Attacker** posts a comment: `Nice post! <script>stealCookies()</script>`.
3.  **Server** saves this comment into the **Database**.
4.  **Victim** (Admin or User) views the forum thread hours or days later.
5.  **Server** pulls the comment from the Database and serves it to the Victim.
6.  **Browser** executes the script.

### Code Review Analysis: The "Trusted Source" Trap
The most common mistake developers make here is trusting the database. They assume, *"If data is coming from my database, it must be safe."* **This is false.**

#### PHP Vulnerability (Guestbook Example)
```php
// VULNERABLE CODE
// 1. Reading from Database (The Source)
$sql = "SELECT comment FROM guestbook";
$result = mysql_query($sql);

while($row = mysql_fetch_array($result)) {
    // 2. Printing to Screen (The Sink)
    // The developer forgot that $row['comment'] might contain scripts saved by a hacker.
    echo "<div>" . $row['comment'] . "</div>"; 
}
```

#### Java Vulnerability (Profile Page)
```java
// VULNERABLE CODE
User profile = database.getUser(id);
// Getting data from the object, which came from the DB
String bio = profile.getBio(); 
%>
<!-- Rendering the bio -->
<div class="user-bio">
    <%= bio %>
</div>
```

### Key Characteristics for the Exam:
*   **No Phishing Required:** The victim just visits a normal page on the site.
*   **High Impact:** Can affect thousands of users (e.g., a worm spreading on a social network).
*   **Reviewer Focus:** You must check the code that **Reads** from the database and **Displays** to the user. Input validation upon saving the data is good, but **Output Encoding upon display** is the only true fix.

---

## 4. DOM-Based XSS (Client-Side)
This type is distinct because the server (Java, PHP, .NET) might **never see the payload**. The vulnerability exists entirely in the JavaScript code running in the user's browser.

### The Mechanism: "The Internal Sabotage"
1.  **Attacker** creates a link: `http://site.com/page.html#<script>...`
2.  **Server** sends the generic `page.html` (It ignores everything after the `#`).
3.  **Browser** loads the page and starts executing the legitimate JavaScript.
4.  **JavaScript** reads the URL (Source) and updates the page HTML (Sink).
5.  **Payload** executes.

### Code Review Analysis: JavaScript Sources & Sinks (PDF p. 188-190)
According to the OWASP Code Review Guide (Section "Client Side JavaScript"), you must look for specific sources and sinks in `.js` files.

#### Common DOM Sources (Where data comes from)
*   `document.URL`
*   `document.location`
*   `document.referrer`
*   `window.name`
*   `location.search` (Query parameters)
*   `location.hash` (The fragment after `#`)

#### Common DOM Sinks (Where data executes)
*   `document.write()`
*   `innerHTML`
*   `outerHTML`
*   `eval()`
*   `setTimeout()` (if passing strings)

#### Vulnerable Example 1: `document.write` (Sample 25.1 in PDF)
```html
<script>
    // 1. Source: Reading the URL
    var pos = document.URL.indexOf("name=") + 5;
    
    // 2. Processing: Extracting the name
    var name = document.URL.substring(pos, document.URL.length);
    
    // 3. Sink: Writing it back to the page
    // If URL is http://site.com?name=<script>alert(1)</script>
    // The browser writes the script tag into the DOM and executes it.
    document.write(name);
</script>
```

#### Vulnerable Example 2: `innerHTML` (Sample 25.2 Concept)
```javascript
// VULNERABLE CODE
var searchParams = document.location.search; // Source
var element = document.getElementById("results");

// Sink: assigning to innerHTML parses the string as HTML code
element.innerHTML = "You searched for: " + searchParams;
```

### Remediation for DOM XSS
You cannot use server-side encoding here. You must use safe JavaScript practices.
*   **Use `textContent` instead of `innerHTML`:** This forces the browser to treat the input as text, not HTML.
    ```javascript
    // SAFE CODE
    element.textContent = "You searched for: " + searchParams;
    ```
*   **Use `JSON.parse` instead of `eval`.**

---

## 5. Advanced Topic: The "Nested Context" Trap
Section 9.1 (PDF p. 76) describes a complex scenario called "Nested Contexts." This is where data is placed inside an HTML attribute that *also* executes JavaScript.

**The Scenario:**
```html
<div onclick="showError('[UNTRUSTED DATA]')">Click me</div>
```

**The Reviewer's Nightmare:**
1.  The browser reads HTML. It sees `onclick`.
2.  It passes the content to the JavaScript engine.
3.  The JavaScript engine runs `showError('...')`.

**Why standard encoding fails:**
If you just use **HTML Encoding**, the single quote `'` becomes `&#39;`.
*   Result: `showError('&#39;); ...')` -> The JavaScript engine decodes the HTML entity `&#39;` back to `'` *before* execution.
*   The attacker breaks out of the string and executes code.

**The Fix (Layered Encoding):**
You must encode for JavaScript **first**, and then encode for HTML attributes **second**. This is rare but critical for high-security reviews.

---

## 6. Summary Comparison Table

| Feature | Reflected XSS | Stored XSS | DOM XSS |
| :--- | :--- | :--- | :--- |
| **Persistence** | Low (Single Request) | High (Database) | Low (Client Session) |
| **Server Logs** | Attack visible in Access Logs (URL) | Attack visible in POST logs | **Invisible** (Fragment `#` is not sent to server) |
| **Review Focus** | Controllers, Views, JSP, ASPX | Database Access Objects (DAOs), Views | **JavaScript files**, jQuery code |
| **Primary Fix** | Output Encoding (Contextual) | Output Encoding (Contextual) | Use `textContent` / Avoid `eval` |

---
*Reference: OWASP Code Review Guide 2.0 (Sections 9.1, 25.1, and Appendix C)*
