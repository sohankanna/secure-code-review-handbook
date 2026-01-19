# A1: Injection — Technical Reference & Theory Overview

## 1. Executive Summary
According to the **OWASP Code Review Guide v2.0**, Injection vulnerabilities are among the most common and damaging flaws in modern software. They occur when an application sends untrusted data to an **interpreter** (SQL, LDAP, OS Shell, etc.) as part of a command or query. 

The core of the problem is a failure in logic: the interpreter is unable to distinguish between the **developer's intended commands** and the **attacker's supplied data**.

---

## 2. The Mechanics of an Injection Attack
### The Interpreter's Blind Spot
The PDF (p. 44) emphasizes that "the SQL parser is not able to distinguish between code and data." In a secure scenario, commands are fixed and data is variable. In an injection scenario, the data becomes part of the command string.

**Visualizing the Failure (Sample 7.2):**
Imagine this Java code used to query a database:
```java
String custQuery = "SELECT custName FROM cust_table WHERE custID = '" + request.GetParameter("id") + "'";
```

| Component | Content | Role |
| :--- | :--- | :--- |
| **Developer's Code** | `SELECT custName FROM cust_table WHERE custID = '` | **COMMAND** |
| **Attacker's Data** | `101' OR '1'='1` | **INJECTED COMMAND** |
| **Closing Syntax** | `'` | **COMMAND** |

Because the developer used **String Concatenation**, the parser receives one long string and executes the `OR '1'='1'` as if the developer wrote it. This results in the database returning every record in the table.

---

## 3. Scope of Injection
While SQL Injection (SQLi) is the most prominent, the "A1" category is an umbrella for several interpreter-based attacks (PDF p. 44):
*   **SQL Injection:** Targeting relational databases.
*   **HQL/ORM Injection:** Targeting abstraction layers like Hibernate (Java) or Entity Framework (.NET).
*   **LDAP Injection:** Targeting directory services.
*   **XPath Injection:** Targeting XML query logic.
*   **OS Command Injection:** Injecting commands into the host operating system (e.g., via `system()` or `exec()` calls).
*   **JSON Injection:** Exploiting insecure parsing logic in JavaScript environments.

---

## 4. Impact Analysis (The "Four Pillars of Risk")
The guide defines four primary consequences of a successful injection attack:

1.  **Confidentiality Breach (Disclosure):** Leakage of sensitive information, such as PII (Personally Identifiable Information), passwords, or trade secrets.
2.  **Loss of Data Integrity:** Attackers may not just read data; they can modify, add, or delete it (e.g., changing a balance in a banking app).
3.  **Privilege Escalation:** An attacker may gain administrative access to the database or the application logic by bypassing authentication queries.
4.  **Network Access:** In severe cases, gaining access to the backend database server allows an attacker to pivot and attack other systems on the internal backend network.

---

## 5. Advanced Theoretical Concepts
### Blind SQL Injection (PDF p. 45)
In many modern applications, the database does not return raw errors or data directly to the screen. However, attackers can still "inject" by asking the database **True/False questions**.
*   **Boolean-based:** The page loads normally if the query is true and shows an error/different UI if false.
*   **Time-based:** The attacker tells the database to "Wait 10 seconds if the first letter of the admin password is 'A'." If the page takes 10 seconds to load, the attacker has confirmed the data.

### The "Logic Trap" in Concatenation
The guide warns that even if you aren't "injecting" malicious code, dynamic query building is a **Code Review Red Flag**.
*   **Scenario:** If a developer builds a search query where `WHERE` clauses are only added if input is present, but fails to handle the case where *all* inputs are null, the application may inadvertently return the entire database (Sample 7.3).

---

## 6. The Defense-in-Depth Framework (The Big 5)
To pass a Secure Code Review, an application should ideally use all five of these strategies in combination:

1.  **Parameterization:** Using Prepared Statements to ensure the parser treats input as literal data only.
2.  **Input Validation (Whitelisting):** Validating input for type, length, format, and range (The "Golden Rule").
3.  **Output Encoding:** HTML Encoding user input before rendering it to prevent script injection in the UI.
4.  **Canonicalization:** Reducing input to its simplest form before validation to prevent bypasses via encoding (e.g., Hex or Unicode).
5.  **SAST & Tooling:** Using Static Analysis to find common patterns like `EXECUTE` or `mysql_query` with concatenated variables.

---

## 7. Reviewer’s "Source-to-Sink" Methodology
When reviewing code for A1 vulnerabilities, follow this mental model:

1.  **Identify the Source:** Locate every point where the application receives data from the outside world (`GET`, `POST`, `Cookies`, `Headers`, `File Uploads`).
2.  **Trace the Flow:** Follow that variable through the code. Does it get modified? Is it sanitized?
3.  **Analyze the Sink:** Look at the function where the data is finally used. Is it a database execution? A file system call? An OS command?
4.  **Check the Boundary:** If the data crosses from the application to an interpreter (The Sink) without being **parameterized** or **validated**, you have found a vulnerability.

---

## 🔗 Sub-Note Navigation
*   [**SQL Injection Deep Dive**](./sql-injection-deep-dive.md): Language-specific (Java, .NET, PHP) vulnerable patterns and fixes.
*   [**ORM & HQL Injection**](./orm-hql-injection.md): Why Hibernate and Linq aren't always safe.
*   [**Input Validation & Canonicalization**](./input-validation-canonicalization.md): The technical mechanics of whitelisting and path resolution.

---
