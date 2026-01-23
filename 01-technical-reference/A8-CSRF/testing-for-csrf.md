# Testing for CSRF: A Practical Guide for Reviewers

## 1. Executive Summary
A Secure Code Review can identify if anti-CSRF defenses are *present* in the code, but only **testing** can confirm if they are *working*. The goal of testing is to prove that an attacker can trick a victim's browser into submitting a state-changing request without the victim's knowledge or intent.

This guide provides a step-by-step methodology for manually testing the most common CSRF implementation patterns.

### The Tester's Mindset
Your goal is to answer one question: **"Does the application trust my browser too much?"** You will attempt to remove or forge the "proof of intent" (the anti-CSRF token) and see if the server still accepts the request.

### Prerequisites: The Tester's Toolkit
To perform these tests, you need an **intercepting proxy**.
*   **OWASP ZAP** (Free, Open Source)
*   **Burp Suite** (Community Edition is free)

You also need two user accounts to simulate the attack:
*   **User A (The Attacker)**
*   **User B (The Victim)**

---

## 2. Test Case 1: State-Changing GET Requests
This is the lowest-hanging fruit and the easiest CSRF vulnerability to exploit. It tests if the application performs sensitive actions using a simple URL.

### The Vulnerability
If an application allows a user to change data using a GET request (i.e., just by visiting a URL), it is vulnerable.

**Vulnerable URL Patterns to Look For:**
```
http://site.com/user/delete?id=123
http://site.com/cart/add?item=45&quantity=1
http://site.com/password/change?new=password123
```

### How to Test
1.  **Log In** to the application as **User B (The Victim)**.
2.  **Perform an Action** that changes data (e.g., add an item to your cart, delete a post, change your email).
3.  **Check Your Proxy History.** Look at the request that was sent. Is it a `GET` request? Does it have parameters in the URL?
4.  **Craft the Proof of Concept (PoC).** If the vulnerable URL is `http://site.com/user/delete?id=123`, create a simple HTML file with an `<img>` tag.
    ```html
    <!-- csrf_poc.html -->
    <h1>You've won a prize!</h1>
    <p>This page is totally innocent.</p>
    
    <!-- This hidden image will make the victim's browser send the GET request -->
    <img src="http://site.com/user/delete?id=123" width="1" height="1" />
    ```
5.  **Execute the Test.** While still logged in as the victim, open `csrf_poc.html` in your browser.
6.  **Verify the Result.** Check the application. Was user 123 deleted?

### Expected Results
*   **FAIL (Vulnerable):** The action is completed (the user is deleted).
*   **PASS (Secure):** The application either ignores the request or returns an error message. Best practice is for all state-changing actions to require `POST`.

---

## 3. Test Case 2: Absence of an Anti-CSRF Token
This is the most common test for modern web applications that use POST requests. It checks if the "Synchronizer Token Pattern" is even implemented.

### The Vulnerability
The application uses POST for sensitive actions but does not include a unique, secret token to verify the user's intent.

### How to Test
1.  **Log In** as the victim.
2.  **Navigate** to a form that performs a sensitive action (e.g., the "Update Profile" page).
3.  **Intercept the Request.** Turn on your proxy (ZAP or Burp) and submit the form.
4.  **Analyze the Parameters.** Look at the body of the POST request. Is there a parameter with a name like `CSRFToken`, `__RequestVerificationToken`, `csrf`, or `nonce`?

    **Example Request with a Token (Likely Secure):**
    ```http
    POST /profile/update HTTP/1.1
    
    name=John&email=john@test.com&__RequestVerificationToken=aBcDeFg123...
    ```5.  **Perform the Test.** Forward the request to your proxy's "Repeater" tool. Now, **delete the entire `__RequestVerificationToken` parameter** and send the request.

### Expected Results
*   **FAIL (Vulnerable):** The server responds with `HTTP 200 OK` or `302 Redirect`, and the profile is successfully updated. This proves the token is not being checked.
*   **PASS (Secure):** The server responds with an error, typically `HTTP 400 Bad Request`, `HTTP 403 Forbidden`, or a custom error page saying "Invalid Request."

---

## 4. Test Case 3: Weak Token Validation
The token exists, but is the server actually checking its *value*?

### The Vulnerability
The server checks that a CSRF token parameter is *present* but does not validate that its value matches the one in the user's session.

### How to Test
1.  Follow the steps from Test Case 2 to intercept a valid request with a token.
2.  **Perform the Test.** In your proxy, modify the value of the token.
    *   **Tamper with it:** Change the last few characters (e.g., `...123` -> `...456`).
    *   **Send an empty token:** Leave the parameter name, but make the value blank (e.g., `__RequestVerificationToken=`).
3.  Send the modified request.

### Expected Results
*   **FAIL (Vulnerable):** The server accepts the request with the tampered or empty token.
*   **PASS (Secure):** The server rejects the request with an error.

---

## 5. Test Case 4: Token Reuse and Session Binding
This tests two more advanced flaws.

### A. Is the token usable more than once?
A high-security application might issue a new token for every request to prevent replay attacks.

**How to Test:**
1.  Intercept a valid request with a valid token.
2.  Send the request to the server (it should succeed).
3.  **Send the exact same request again.**

**Expected Results:**
*   **FAIL (Medium Risk):** The second request also succeeds. This means the token is valid for the entire session. This is common but less secure.
*   **PASS (High Security):** The second request fails. This proves the token was a "one-time use" nonce.

### B. Is the token tied to the user's session?
The Synchronizer Token Pattern requires the token to be validated against the user's specific session.

**How to Test:**
1.  Log in as **User A (Attacker)**. Go to a form and get a valid CSRF token (`token_A`).
2.  Log in as **User B (Victim)** in a separate browser.
3.  As the victim, intercept a request to perform an action (e.g., send a message). Note the victim's token (`token_B`).
4.  In your proxy, replace the victim's token (`token_B`) with the attacker's token (`token_A`).
5.  Send the request.

**Expected Results:**
*   **FAIL (Vulnerable):** The server accepts the request. This means the server is not binding the token to the session, or the validation is flawed.
*   **PASS (Secure):** The server rejects the request because `token_A` does not match the token stored in Victim B's session.

---

## 6. Test Case 5: Bypassing Header-Based Defenses
If the application does not use tokens but relies on `Referer` or `Origin` headers, test them directly.

### How to Test
1.  Intercept a valid `POST` request.
2.  Look for the `Referer` or `Origin` header.
    *   **Example:** `Origin: https://bank.com`
3.  **Perform the Test:**
    *   **Remove the header** completely and send the request.
    *   **Change the header** to a malicious domain (e.g., `Origin: https://evil.com`).

### Expected Results
*   **FAIL (Vulnerable):** The request succeeds even with a missing or incorrect header.
*   **PASS (Secure):** The request is blocked with an error.

---
*Reference: OWASP Code Review Guide 2.0 (Pages 141-145)*
