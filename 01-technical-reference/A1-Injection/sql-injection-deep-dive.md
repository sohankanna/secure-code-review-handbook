# SQL Injection Deep Dive: From Root Cause to Remediation

## 1. Introduction: The "Order Form" Analogy
Imagine you are at a restaurant. There is an order form where you write down what you want.

*   **Normal Order:** You write "Burger." The waiter (the application) takes the note to the chef (the database). The chef sees the order and makes a burger.
*   **The Injection Attack:** Instead of just a food item, you write: **"Burger... AND also give me all the money in the cash register."**

If the chef is a "dumb" robot who just reads the whole line as a command, he will make the burger AND empty the register. This is precisely what happens in an SQL Injection attack. The database (the chef) cannot tell where the "food order" ends and the "illegal command" begins because they are mixed together in the same sentence.

### According to the Guide (Page 44):
> "SQL commands are not protected from the untrusted input. The SQL parser is not able to distinguish between code and data."

---

## 2. The Root Cause: String Concatenation
The primary reason SQL Injection exists is a coding habit called **String Concatenation**. This is when a developer builds a database query by "gluing" parts of a sentence together with user input.

### The Anatomy of a Vulnerable Query
Look at this basic logic:
`Final Sentence` = `"Show me the profile for user ID: "` + `[User Input]`

If the user enters `101`, the database sees:
`Show me the profile for user ID: 101` (Safe)

If the user enters `101' OR '1'='1`, the database sees:
`Show me the profile for user ID: 101' OR '1'='1`
Because `'1'='1'` is always true, the database stops looking for just ID 101 and instead shows **every single user profile** in the system.

---

## 3. Language-Specific Vulnerabilities: Reviewing the Code

As a Secure Code Reviewer, you must look for specific "Red Flags" in different programming languages. The following samples are taken directly from the OWASP guide.

### A. PHP: The Legacy Trap (Sample 7.5)
PHP is often the target of SQLi because of older coding styles.

**Vulnerable Code:**
```php
1 <?php 
2 $pass = $_GET["pass"]; 
3 $con = mysql_connect('localhost', 'owasp', 'abc123'); 
4 $sql = "SELECT card FROM users WHERE password = '" . $pass . "'"; 
5 $result = mysql_query($sql); 
6 ?> 
```

**Reviewer’s Technical Analysis:**
*   **Line 2 (The Source):** The variable `$pass` is taken directly from the URL (`$_GET`). This is "Tainted" data.
*   **Line 4 (The Injection Point):** The `.` symbol in PHP is used to glue strings. Here, the password is glued into the command.
*   **Line 5 (The Sink):** The `mysql_query()` function sends this mixed-up string to the database.
*   **The Flaw:** If an attacker enters `' OR 1=1 --`, they bypass the password check entirely.

---

### B. .NET: The UI Control Flaw (Sample 7.8)
Even in modern frameworks like .NET, if a developer grabs text directly from a "textbox" on a screen and glues it into a query, it is vulnerable.

**Vulnerable Code:**
```csharp
1 SqlDataAdapter thisCommand = new SqlDataAdapter(
2 "SELECT name, lastname FROM employees WHERE ei_id = '" + idNumber.Text + "'", thisConnection);
```

**Reviewer’s Technical Analysis:**
*   **Red Flag:** The `+` sign on Line 2.
*   **Source:** `idNumber.Text` is whatever the user typed into the box on their screen.
*   **The Attack:** An attacker types: `123'; DROP TABLE employees; --`.
*   **Result:** The semicolon `;` tells the database the first command is over. It then executes the second command: `DROP TABLE employees`, which deletes the entire employee list.

---

### C. Java & The ORM Myth (Sample 7.7)
Many developers believe that using an ORM (Object-Relational Mapper) like Hibernate makes them immune to SQLi. **This is a dangerous misconception.**

**Vulnerable Code:**
```java
1 String userName = request.getParameter("name"); 
2 String query = "SELECT * FROM Users WHERE name = '" + userName + "'"; 
3 con.execute(query);
```

**Reviewer’s Technical Analysis:**
*   Even if using a database "Connection" object, if the `query` string is built using `+ userName +`, the protection of the framework is bypassed. 
*   **ORM Specifics:** If you see `session.createQuery("... " + input)` in Java Hibernate, it is an **HQL Injection**. It works exactly like SQLi but targets the Hibernate abstraction layer.

---

## 4. The "Safe" Concatenation Trap (Logic Leaks)
The OWASP guide (p. 46) warns about a very subtle bug that many reviewers miss. Sometimes, code is "safe" from a hacker running commands, but it still leaks data because of bad logic.

### Sample 7.3: The Search Leak
```java
1 String query = "select id, firstname, lastname FROM authors";
2 if (firstname != null) {
3    query += " WHERE forename = ?";
4 } else if (lastname != null) {
5    query += " WHERE surname = ?";
6 }
```

**Why this fails a Secure Code Review:**
1.  If a user provides a `firstname`, the query is parameterized (Safe).
2.  If a user provides a `lastname`, the query is parameterized (Safe).
3.  **HOWEVER**, if the user (or an attacker) sends a request with **neither** field, the `if` and `else if` both fail.
4.  The final query becomes: `select id, firstname, lastname FROM authors;`.
5.  **The Result:** The application displays every author in the database to the user. This is a **Data Disclosure Vulnerability** caused by using concatenation to build logic.

---

## 5. Advanced Injection: Blind SQL Injection
Sometimes, an attacker cannot see the data directly. The application might just say "Login Successful" or "Login Failed." This is where **Blind SQL Injection** (p. 45) comes in.

### Two Types of Blind Attacks:
1.  **Boolean-Based:** The attacker asks: "Is the first letter of the admin password 'A'?" 
    *   If the page loads normally, the answer is **Yes**. 
    *   If the page shows an error, the answer is **No**. 
    *   The attacker repeats this for every letter until they have the whole password.
2.  **Time-Based:** The attacker tells the database: "If the admin password starts with 'A', wait 10 seconds before responding."
    *   The attacker uses a stopwatch. If the site hangs for 10 seconds, they know the letter is 'A'.

**Reviewer Action:** Even if the code doesn't "output" data to the screen, any concatenated input used in a `WHERE` clause is still a critical risk.

---

## 6. The "Gold Standard" Fix: Parameterized Queries
How do we stop this? We use **Parameterized Queries** (also called Prepared Statements). 

Going back to our restaurant analogy: Instead of an open order form, the waiter gives the chef a form with **pre-labeled boxes**.
*   Box 1: [Burger]
*   Box 2: [Extra Cheese]

The chef **only** looks inside the boxes for data. If the user writes "and blow up the kitchen" inside the "Extra Cheese" box, the chef just tries to find a cheese named "and blow up the kitchen." He doesn't execute it as an instruction.

### Secure Java Example (Sample 7.1)
```java
1 String query = "SELECT id FROM authors WHERE forename = ? and surname = ?"; 
2 PreparedStatement pstmt = connection.prepareStatement(query); 
3 pstmt.setString(1, firstname); 
4 pstmt.setString(2, lastname);
```
*   **The `?` Placeholder:** These are the "security boxes." The database is told the structure of the command *before* the data is ever added.

### Secure .NET Example (Sample 7.10)
```csharp
1 sqlAdapter.SelectCommand.Parameters.Add("@usrId", SqlDbType.VarChar, 15); 
2 sqlAdapter.SelectCommand.Parameters["@usrId"].Value = UID.Text;
```
*   **Type Checking:** Notice `SqlDbType.VarChar, 15`. The code now forces the input to be a string and limited to 15 characters. If an attacker tries to send a 500-character SQL command, the system rejects it automatically.

---

## 7. Secondary Defense: Stored Procedures
Stored Procedures can be secure, but the guide (p. 45) offers a warning:
> "Stored Procedures can be used to build dynamic SQL statements... causing it to become vulnerable to injection."

**Reviewer Tip:** If you see a call to a Stored Procedure, you must also check the **database code** itself. If the stored procedure uses `exec()` or `sp_executeSql` with concatenated strings *inside* the database, the vulnerability hasn't been fixed—it's just been moved.

---

## 8. Secure Code Reviewer’s Checklist for A1
Use this checklist during your exam and your real-world reviews:

*   [ ] **Search for Concatenation:** Grep the codebase for `+`, `&`, or `.` inside database strings.
*   [ ] **Trace the Source:** Is data coming from `HTTP Headers`, `Cookies`, or `URL Parameters`? (All are untrusted).
*   [ ] **Verify Sinks:** Are functions like `mysql_query`, `execute`, `createQuery`, or `SqlCommand` being used?
*   [ ] **Check ORM Usage:** Even if using Hibernate/Entity Framework, ensure they aren't building queries with raw strings.
*   [ ] **Validate "The Big Three":** Is input checked for **Type** (is it a number?), **Length** (is it too long?), and **Format** (does it contain illegal characters like `;` or `'`)?
*   [ ] **Check Logic Paths:** Does the code handle cases where user input is missing to avoid "Select All" leaks?
*   [ ] **Evaluate Stored Procs:** Ensure they use parameters and not internal dynamic string building.

---

## 9. Glossary for Non-Coders
*   **Interpreter:** A program (like a SQL Parser) that reads text and turns it into action.
*   **Concatenation:** The act of joining two pieces of text together.
*   **Sanitization:** Cleaning user input by removing dangerous characters.
*   **Whitelisting:** A list of "allowed" things. (Much safer than a "Blacklist" of forbidden things).
*   **Payload:** The malicious part of the input that performs the attack.

