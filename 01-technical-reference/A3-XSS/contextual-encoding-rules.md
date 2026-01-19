# Contextual Encoding Rules: The Reviewer's Guide

## 1. The Core Philosophy
XSS is not an input problem; it is an **Output Problem**.

You cannot "sanitize" data perfectly at the input stage because you don't know where that data will be used yet. For example, the character `'` (single quote) is perfectly valid in a name like *O'Reilly* (Input Validation passes). However, if that name is later output inside a JavaScript variable `var name = 'O'Reilly';`, it crashes the script or allows XSS.

**The Rule:** You must encode data **right before** it is rendered to the user, based on the **Context** (location) where it is being placed.

---

## 2. Context 1: HTML Body (The Simplest Context)
This occurs when data is placed between HTML tags.

**The Location:**
```html
<div> [UNTRUSTED DATA] </div>
<p> [UNTRUSTED DATA] </p>
<b> [UNTRUSTED DATA] </b>
```

### The Vulnerability
If the data contains tags like `<script>`, `<img>`, or `<iframe>`, the browser will render them as elements, not text.

### The Encoding Rule (HTML Entity Encoding)
You must convert meaningful HTML characters into their text-safe "Entity" equivalents.

| Character | Encoded Entity |
| :--- | :--- |
| `&` | `&amp;` |
| `<` | `&lt;` |
| `>` | `&gt;` |
| `"` | `&quot;` |
| `'` | `&#x27;` |
| `/` | `&#x2F;` |

### Code Review Check
**Java (Secure):**
```java
// Using OWASP Java Encoder
String safe = Encode.forHtml(request.getParameter("input"));
out.println(safe);
```

**.NET (Secure):**
```csharp
// AntiXSS Library or Standard Encode
var safe = HttpUtility.HtmlEncode(userInput);
// OR in Razor syntax, this happens automatically:
@userInput 
```

---

## 3. Context 2: HTML Attributes (The Tricky Context)
This occurs when data is placed *inside* an HTML tag's property. The PDF (p. 73) dedicates a significant section to this because it is often misunderstood.

**The Location:**
```html
<input type="text" name="fname" value="[UNTRUSTED DATA]">
```

### The Vulnerability
There are two scenarios here: **Quoted** and **Unquoted**.

**Scenario A: Breaking Out of Quotes**
If the developer writes: `<input value="[DATA]">` and the attacker sends `"> <script>...`, the HTML becomes:
`<input value=""> <script>...</script>">` (XSS Triggered).

**Scenario B: Unquoted Attributes (High Risk)**
The PDF (p. 73) explicitly warns:
> "The reason this rule is so broad is that developers frequently leave attributes unquoted."

If the code is: `<input value=[DATA]>` (No quotes around value), an attacker can break out using **Space, %, *, +, , -, /, ; <, =, >, ^, |**.
*   Attack: `[DATA]` = `x onclick=alert(1)`
*   Result: `<input value=x onclick=alert(1)>` (XSS Triggered).

### The Encoding Rule (Attribute Encoding)
You cannot just use HTML Body encoding here. You must be more aggressive.
*   **Rule:** Except for alphanumeric characters, escape **ALL** characters with ASCII values less than 256 using the `&#xHH;` format.

### Code Review Check
Look for the specific Attribute Encoder functions mentioned in the guide (p. 74):

**Java:**
```java
// Wrong
String safe = Encode.forHtml(data); // Might miss attribute-specific break-outs

// Correct
String safe = Encode.forHtmlAttribute(data);
```

**.NET:**
```csharp
// Correct
var safe = HttpUtility.HtmlAttributeEncode(data);
```

### ⚠️ Dangerous Attributes (Blacklist)
Some attributes are **Never Safe**, even if encoded. You must flag these in a review:
*   `href="javascript:..."`
*   `onload`, `onerror`, `onclick`, `onmouseover` (Event Handlers)
*   `style` (CSS context)

---

## 4. Context 3: JavaScript Data Values (The Hardest Context)
This occurs when the server dynamically writes data *inside* a `<script>` block.

**The Location:**
```html
<script>
    var username = '[UNTRUSTED DATA]';
</script>
```

### The Vulnerability
HTML Encoding does **NOT** work here.
If the input is `'; alert(1); //`, and you use HTML encoding (`&#x27;`), the browser sees:
`var username = '&#x27;; alert(1); //';`
The JavaScript engine does *not* understand HTML Entities inside code blocks. It might fail, or it might treat it as a literal string, but the logic is flawed.

However, if you perform **No Encoding**, the attacker breaks the string:
`var username = ''; alert(1); //';` (XSS Triggered).

### The Encoding Rule (JavaScript Escaping)
The guide (p. 74) specifies using **Unicode Escapes**.
*   Do not use backslash escapes (`\"`) alone, as they can sometimes be bypassed.
*   Use `\uXXXX` format.
*   Example: `'` becomes `\u0027`.

### Code Review Check
**Java (ESAPI/OWASP Encoder - PDF p. 75):**
```java
// Sample 9.1 Reference
String safe = ESAPI.encoder().encodeForJavaScript(request.getParameter("input"));
```

**JavaScript (Client-Side):**
Do not allow the server to write variables directly into script blocks. Instead, put the data in a hidden HTML element (HTML encoded) and read it using `textContent`.

---

## 5. Advanced Topic: Nested Contexts (PDF p. 76)
**This is a guaranteed exam question/advanced interview topic.**

A Nested Context is when data passes through **two** interpreters.

**The Location:**
```html
<div onclick="showError('[UNTRUSTED DATA]')">Click Me</div>
```

**The Logic Flow:**
1.  **Browser Parser (HTML):** Reads the `onclick` attribute. It decodes HTML entities.
2.  **JS Engine:** Executes the decoded content as JavaScript.

**The Failure of Single Encoding:**
If you only use **HTML Attribute Encoding**:
*   Input: `'); alert(1);//`
*   Encoded: `&#x27;); alert(1);//`
*   Result in Browser: `<div onclick="showError('&#x27;); alert(1);//')">`
*   **Execution:** The Browser HTML parser sees `&#x27;` and decodes it to `'`. It then hands the clean string `showError(''); alert(1);//')` to the JS engine. **XSS Triggered.**

### The Encoding Rule (Layered Encoding)
You must encode for the **Innermost** context first, then work your way out.

1.  **First:** JavaScript Encode the data.
    *   Input: `'); alert(1);//` -> `\u0027); alert(1);//`
2.  **Second:** HTML Attribute Encode the result.
    *   Input: `\u0027);...` -> `\u0027);...` (Alphanumerics usually stay safe, or become `&#x5C;u0027...`)

**Correct Code Review Pattern:**
```java
// Pseudocode for Nested Context
String jsEncoded = Encoder.forJavaScript(input);
String finalOutput = Encoder.forHtmlAttribute(jsEncoded);
%>
<div onclick="alert('<%= finalOutput %>')">
```

---

## 6. CSS & URL Contexts (Brief Overview)
While less detailed in the PDF, these contexts are standard knowledge for the A3 category.

### URL Context
**Location:** `<a href="/search?q=[DATA]">`
**Rule:** URL Encode (Percent Encoding).
*   Space becomes `%20`.
*   `/` becomes `%2F`.
*   **Reviewer Check:** Ensure `URLEncode` is used for parameter values, but **not** for the entire URL (or you break the protocol).

### CSS Context
**Location:** `<style> selector { property: [DATA]; } </style>`
**Rule:** CSS Hex Encoding (`\HH`).
**Reviewer Check:** Generally, allowing user input in CSS is dangerous (phishing/defacement) and should be avoided.

---

## 7. Secure Code Reviewer’s Cheat Sheet (Encoding)

| Context | Example Location | Encoding Strategy | Typical Function (Java/.NET) |
| :--- | :--- | :--- | :--- |
| **HTML Body** | `<div>...</div>` | HTML Entity Encode (`<` -> `&lt;`) | `Encode.forHtml` / `HtmlEncode` |
| **HTML Attribute** | `<input value="...">` | Attribute Encode (ASCII < 256 -> `&#xHH;`) | `Encode.forHtmlAttribute` / `HtmlAttributeEncode` |
| **JavaScript** | `<script>var x='...'` | Unicode Escape (`'` -> `\u0027`) | `Encode.forJavaScript` / `JavaScriptStringEncode` |
| **URL Parameter** | `href="?q=..."` | URL Encode (` ` -> `%20`) | `URLEncode` / `UrlEncode` |
| **Nested** | `onclick="..."` | **1.** JS Encode, **2.** HTML Attr Encode | **Double Encoding Required** |

---

## 8. Summary for the Exam
*   **Context is King:** The exact same data requires different encoding depending on whether it is in a `<div>`, an `href`, or a `<script>`.
*   **Never "Self-Write":** Do not accept custom regex functions to replace `<` with empty strings. Always use libraries (OWASP/Microsoft).
*   **Nested Contexts:** If data is in an event handler (`onclick`), it needs **two** layers of encoding.
*   **Unquoted Attributes:** Are a security nightmare. Always quote attributes (`value="x"`) *and* encode.

---
*Reference: OWASP Code Review Guide 2.0 (Pages 73-76)*
