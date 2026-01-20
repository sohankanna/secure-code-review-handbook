# A4: Insecure Direct Object Reference (IDOR) — Technical Reference

## 1. Executive Summary
**Insecure Direct Object Reference (IDOR)** occurs when an application exposes a reference to an internal implementation object (such as a file, directory, database record, or key) to the user.

According to the **OWASP Code Review Guide 2.0 (p. 78)**, the vulnerability arises when an application **fails to verify that the user is authorized to access the specific object** referenced by that key.

### The Analogy: The Hotel Key Card
*   **Secure System:** You have a key card. It only opens Room 101.
*   **IDOR System:** You have a key card. It has the number "101" written on it in magic marker. You wipe off "101", write "102", and the system lets you into Room 102.

In software, the "Magic Marker" is the URL parameter or Form Field (e.g., `id=101`). If the server trusts the user input without checking ownership, IDOR occurs.

---

## 2. The Core Mechanics
IDOR is not a bug in the code syntax (like SQLi); it is a bug in the **Business Logic**. The application functions exactly as programmed, but the design fails to enforce access control.

### Scenario A: SQL Data Access (Sample 10.1)
The application retrieves data based on a user-supplied parameter without checking if the user *owns* that data.

**Vulnerable Code (Java):**
```java
// Source: request.getParameter("acct") -> The user controls this!
String query = "SELECT * FROM accts WHERE account = ?";
PreparedStatement pstmt = connection.prepareStatement(query);
pstmt.setString(1, request.getParameter("acct"));
ResultSet results = pstmt.executeQuery();
```
**The Flaw:**
If I am User A (Account 100), I can change `acct` to 200. The query becomes `SELECT * FROM accts WHERE account = 200`. The database happily returns User B's account because the code **never asked**: *"Does User A have permission to see Account 200?"*

### Scenario B: HTTP POST Manipulation (The Yahoo Example)
The guide (p. 78) references a real-world vulnerability found in Yahoo.
*   **The Request:** `POST ... cmd=delete_comment&fid=367443&cid=123`
*   **The Logic:** `fid` was the Forum ID, `cid` was the Comment ID.
*   **The Attack:** By changing the `cid` to a random number, the attacker could delete **any comment on the entire platform**, regardless of who posted it.

---

## 3. URL & Path Manipulation (Indirect References)
Attackers often guess URLs based on predictable patterns.

**Direct Reference (Vulnerable):**
`xyz.com/Customers/View/2148102445`
*   **Risk:** The ID is exposed. An attacker can iterate (`2148102446`, `2148102447`) to scrape the entire customer database.

**Indirect Reference (Secure):**
The guide (p. 79) suggests using **Indirect Reference Maps**.
Instead of the database key (`102445`), the user sees a temporary, random key (`abc-123`) that is mapped to their session. If they change it to `abc-124`, the map lookup fails because `abc-124` doesn't belong to their session.

---

## 4. Modern IDOR: Data Binding / Mass Assignment
This is a critical sub-category for modern frameworks (Spring, .NET MVC, Rails).

**The Concept (p. 79):**
Frameworks often automatically bind HTTP parameters to internal code objects to save time.
*   **The Risk:** If the object has a field called `isAdmin`, and the attacker adds `&isAdmin=true` to the request, the framework might automatically overwrite that field in the database.
*   *(This topic is covered in detail in `mass-assignment-vulnerability.md`)*.

---

## 5. Reviewer’s "Source-to-Object" Methodology
When reviewing for IDOR, you cannot rely on automated scanners (SAST), because scanners don't understand your business rules (i.e., they don't know that User A shouldn't see Record B). You must review manually.

### Step 1: Map the "Direct References"
Look for code that uses input to access data.
*   `request.getParameter("id")` used in a `SELECT` statement.
*   `request.QueryString["file"]` used in `File.Open()`.
*   URLs that look like `/account/view/12345`.

### Step 2: Verify the Access Control
For every Direct Reference found in Step 1, ask:
**"Is there a check BEFORE the data is retrieved?"**

**Vulnerable Flow:**
1.  Get Input (`id=5`).
2.  Query DB (`SELECT * FROM items WHERE id=5`).
3.  Return Data.

**Secure Flow:**
1.  Get Input (`id=5`).
2.  Get Current User (`user_id = session.getID()`).
3.  Query DB (`SELECT * FROM items WHERE id=5 AND owner_id = user_id`).
4.  Return Data.

---

## 6. Secure Code Reviewer’s Checklist (A4)

- [ ] **Database Access:** Does every `SELECT` query using a user parameter *also* filter by the currently logged-in `owner_id`?
- [ ] **File Access:** If a filename is passed (`report=123.pdf`), does the code verify the user owns that file?
- [ ] **Predictability:** Are IDs sequential (1, 2, 3)? If so, is there a strict Access Control List (ACL) check?
- [ ] **Mass Assignment:** Are "Bind" or "Allowlist" attributes used on form submissions to prevent overwriting hidden fields (like `role` or `price`)?
- [ ] **Authorization:** Is the authorization check performed **on the server**, not just by hiding the button in the UI?

---
*Reference: OWASP Code Review Guide 2.0 (Pages 77-81)*
