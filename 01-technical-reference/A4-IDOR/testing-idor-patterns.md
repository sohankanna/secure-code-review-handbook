# Testing IDOR Patterns: SQL and HTTP Manipulation

## 1. Introduction: The "Reviewer's Mindset"
To find **Insecure Direct Object Reference (IDOR)**, you must think like a thief trying to open safety deposit boxes. 

In a secure bank, the guard checks your ID and only opens *your* box. In an IDOR-vulnerable bank, the guard asks, *"Which box number would you like to open?"* and opens whatever number you say, without checking if it belongs to you.

As a Code Reviewer, your job is to find the code that acts like the "bad guard." You are looking for locations where the application takes an ID from the user (the "Reference") and uses it to retrieve data (the "Object") without verifying ownership.

---

## 2. Pattern 1: The SQL Direct Reference (Sample 10.1)
This is the most common form of IDOR found in backend code. It occurs when a database query relies entirely on a parameter sent by the user.

### The Scenario
The application has a page called `ViewInvoice.jsp` or `ViewInvoice.aspx`. To know *which* invoice to show, it asks the browser for an account number.

### The Vulnerable Code Analysis
**Java (JDBC) Example:**
```java
// 1. The Source: The application asks the user "Which account?"
String accountID = request.getParameter("acct");

// 2. The Logic: Build a query to get that account.
String query = "SELECT * FROM accts WHERE account = ?";

// 3. The Sink: Execute the query.
PreparedStatement pstmt = connection.prepareStatement(query);
pstmt.setString(1, accountID);
ResultSet results = pstmt.executeQuery();
```

### Why this is IDOR (and NOT SQL Injection)
It is crucial to distinguish this from SQL Injection.
*   **SQL Injection:** The attacker changes the *structure* of the query (e.g., adding `OR 1=1`).
*   **IDOR:** The attacker changes the *target* of the query.

In the code above, the SQL syntax is safe (it uses `PreparedStatement`). However, the **Logic** is flawed.
*   If User A (who owns Account 100) changes `acct=100` to `acct=101`.
*   The Query becomes: `SELECT * FROM accts WHERE account = 101`.
*   The Database finds Account 101 and returns it.
*   **The Failure:** The code never asked: *"Does the currently logged-in user actually own Account 101?"*

### Reviewer Remediation Check
To fix this, the SQL query itself must enforce ownership. The query should look like this:
```sql
SELECT * FROM accts WHERE account = ? AND owner_id = ?
```
If the code does not include that `AND owner_id = ?` clause, it is a **High Severity IDOR Vulnerability**.

---

## 3. Pattern 2: The HTTP POST Pattern (The Yahoo Case Study)
The guide (p. 78) highlights a real-world vulnerability discovered by Ibrahim Raafat in Yahoo. This example is the perfect template for identifying IDOR in **state-changing operations** (Delete, Update, Edit).

### The Scenario
Yahoo had a suggestion forum. Users could post comments and delete *their own* comments.

### The Vulnerable Request
When a user clicked "Delete" on their comment, the browser sent a **POST** request to the server. The code reviewer would see a request that looks like this:

```http
POST /comments/delete HTTP/1.1
Host: suggestions.yahoo.com
Cookie: SessionId=...

cmd=delete_comment&fid=367443&cid=123654
```

### Breaking Down the Parameters
*   **`cmd=delete_comment`**: Tells the server what function to run.
*   **`fid=367443`**: The **Forum ID** (The topic being discussed).
*   **`cid=123654`**: The **Comment ID** (The specific item to delete).

### The "Star of the Show": The `cid` Parameter
In the Yahoo bug, the server logic worked like this:
1.  **Check Authentication:** Is the user logged in? (Yes).
2.  **Check Action:** Does the user want to delete? (Yes).
3.  **Execute:** Delete the record matching `cid=123654`.

**The Missing Step:** The server failed to check: *"Did the user who sent this request actually write comment #123654?"*

### The Attack (Pattern Recognition)
A code reviewer or tester simulates the attack by modifying the `cid`.
1.  Attacker writes a comment. ID is `1000`.
2.  Attacker intercepts the Delete request.
3.  Attacker changes `cid=1000` to `cid=999` (A comment written by the Admin).
4.  Attacker forwards the request.
5.  **Result:** The server deletes the Admin's comment.

### Impact Analysis
The guide notes that Ibrahim used this to delete roughly **1.5 million records**. This demonstrates that IDOR is not just a privacy risk (viewing data); it is a **Data Integrity Risk** (destroying data).

---

## 4. Pattern 3: URL Iteration (Predictability)
The guide (p. 79) discusses **Indirect Reference Maps**, but first describes the vulnerability they solve: **Predictable Resource Location**.

### The Vulnerable URL Pattern
Review the URL structure of the application. Look for simple integers.
`xyz.com/Customers/View/2148102445`

If the ID `2148102445` represents a customer, an attacker immediately asks:
1.  What happens if I subtract 1? (`2148102444`)
2.  What happens if I add 1? (`2148102446`)

### The "Enumeration" Attack
If the IDs are sequential (1, 2, 3, 4...), an attacker can write a simple script (using a tool like Burp Suite Intruder) to cycle from 1 to 1,000,000.
*   **Result:** The attacker downloads the entire customer database.

### Reviewer Red Flags
When reviewing code, look for **Auto-Incrementing Primary Keys** being exposed directly to the URL.
*   **Database Schema:** `id INT PRIMARY KEY AUTO_INCREMENT` -> Exposed as `user_id=15`.
*   **Mitigation:** The application should use **GUIDs** (Globally Unique Identifiers) like `a1b2-c3d4-e5f6` which are not guessable, OR implement strict ownership checks.

---

## 5. Pattern 4: Mass Assignment (Hidden IDOR)
This is detailed in **Section 13 (Page 80)** of the guide but relates directly to IDOR. This occurs when an attacker modifies fields they *shouldn't* have access to, effectively performing IDOR on the object's properties.

### The Scenario
A user has a profile page. They can edit their "Bio" and "Phone Number."
The internal User Object looks like this:
```csharp
class User {
    public int ID;
    public string Name;
    public bool IsAdmin; // <-- The target
}
```

### The Attack
The user submits the "Update Profile" form. The legitimate request looks like:
`POST /update?name=John&bio=Hello`

The attacker adds the hidden field to the request:
`POST /update?name=John&bio=Hello&IsAdmin=true`

### The Code Flaw
If the code uses automatic **Data Binding** (e.g., `UpdateModel(user)` in .NET MVC or Spring Binding), the framework sees the `IsAdmin` parameter and automatically updates that field in the database object. The user has "referenced" the `IsAdmin` object property directly and insecurely.

---

## 6. How to Test for IDOR: The "Two-User" Methodology
When performing a review (either code or manual testing), you cannot find IDOR with a single account. You need a specific testing workflow.

### Step 1: Establish Two Contexts
Create two distinct users in the application:
*   **User A (Attacker)**
*   **User B (Victim)**

### Step 2: Map the Objects
*   Log in as **User B (Victim)**.
*   Navigate to a private resource (e.g., "My Invoices").
*   Note the ID of the invoice (e.g., `invoice_id=555`).

### Step 3: Attempt the Swap (The Test)
*   Log in as **User A (Attacker)**.
*   Navigate to the same feature ("My Invoices").
*   Capture the request sent to the server.
*   **Change the ID:** Replace User A's invoice ID with User B's ID (`555`).
*   Send the request.

### Step 4: Analyze the Result
*   **Pass (Secure):** The server returns "403 Forbidden," "401 Unauthorized," or "Record Not Found."
*   **Fail (Vulnerable):** The server returns the details of Invoice 555.

---

## 7. Code Reviewer's "Grep" Sheet for IDOR Patterns
If you have the source code, searching for these patterns helps locate potential IDOR hotspots.

### 1. Request Parameters used in Database Calls
Look for variables taken from the request that go straight into a query without an intermediate check.
*   **Search:** `request.getParameter` near `SELECT`
*   **Search:** `Request.QueryString` near `WHERE`
*   **Search:** `FindById(id)` (Common ORM pattern that often misses ownership checks).

### 2. File Access via Parameters
Look for code that opens files based on input.
*   **Search:** `File.Open(Request["file"])`
*   **Search:** `FileInputStream(request.getParameter("path"))`
*   *Note: This checks for Path Traversal IDOR.*

### 3. Missing Authorization Attributes
In modern frameworks, look for controllers that handle IDs but lack permission checks.
*   **Search:** Controllers using `GET /user/{id}`.
*   **Check:** Is there a line like `@PreAuthorize` or `if (user.id == current_user.id)`? If not, flag it.

---
*Reference: OWASP Code Review Guide 2.0 (Pages 78-80)*
