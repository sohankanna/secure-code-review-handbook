# Prevention Strategies: Indirect References & Access Control

## 1. Executive Summary
IDOR is not a bug in the code's syntax; it is a bug in the application's **Business Logic**. Because of this, automated tools often fail to catch it. A scanner sees `SELECT * FROM invoices WHERE id=5` and thinks it is fine. It takes a human code reviewer to ask: *"Wait, does the user actually own invoice #5?"*

Prevention requires two distinct strategies:
1.  **Defense in Depth:** Hiding the real database keys from the user (**Indirect Reference Maps**).
2.  **The Cure:** Enforcing strict ownership checks every time data is accessed (**Access Control Logic**).

---

## 2. Strategy 1: Indirect Reference Maps (The "Coat Check" Method)
**Reference:** PDF Page 79 (Implicitly covered under "Indirect Reference Maps" concepts in IDOR mitigation).

The most effective way to stop an attacker from iterating through Database IDs (1, 2, 3, 4) is to ensure they **never see the real ID**.

### The Analogy: The Coat Check
*   **Direct Reference (Insecure):** You hand your coat to the attendant. They hang it on **Hook #501**. They write "501" on your hand. To get it back, you ask for "501". If you ask for "502", you get someone else's coat.
*   **Indirect Reference (Secure):** You hand your coat to the attendant. They hang it on **Hook #501**. They give you a random plastic token that says **"Blue-Zebra"**.
    *   You ask for "Blue-Zebra". The attendant looks at a list: "Blue-Zebra = Hook 501". You get your coat.
    *   You try to guess "Red-Lion". The attendant says, "That token doesn't exist," or "That token belongs to someone else."
    *   Even if you guess "Blue-Zebra-2", it won't map to Hook 502 mathematically.

### How to Implement in Code
Instead of exposing the Database Primary Key (Integer) to the client, we create a temporary, random map in the User's **Session**.

#### 1. The Setup (Server Side)
When the user logs in or requests a list of items, the server populates a map.

```java
// Java Implementation Logic
// 1. Create a Map for the Session
Map<String, Integer> indirectMap = new HashMap<String, Integer>();

// 2. Get Real Data from DB
List<File> userFiles = db.getFilesForUser(currentUserId);

// 3. Populate the Map
for (File f : userFiles) {
    String randomToken = generateRandomString(); // e.g., "x7z9q"
    indirectMap.put(randomToken, f.getId()); // Map "x7z9q" -> Database ID 101
}

// 4. Save to Session
session.setAttribute("AccessMap", indirectMap);
```

#### 2. The Presentation (Client Side)
The HTML rendered to the user uses the **Random Token**, not the real ID.

```html
<!-- Secure HTML -->
<a href="/download?file_key=x7z9q">Report.pdf</a>
```
*   **Attacker View:** They see `file_key=x7z9q`. They cannot guess the next key. If they try `x7z9r`, it likely doesn't exist in the map.

#### 3. The Lookup (Server Action)
When the user clicks the link, the server translates the token back to the real ID.

```java
// On Request Processing
String userToken = request.getParameter("file_key");
Map<String, Integer> map = (Map)session.getAttribute("AccessMap");

if (map.containsKey(userToken)) {
    int realDbId = map.get(userToken);
    // Proceed to load data for ID 101
    loadFile(realDbId);
} else {
    // SECURITY ALERT: User tried to access a key not in their map!
    logSecurityEvent("IDOR Attempt Detected");
}
```

### Reviewer Checklist for Indirect Maps
*   **Session Bound:** Is the map stored in the user's specific session? (It must not be a global static map, or User A could guess User B's tokens).
*   **Randomness:** Are the tokens unguessable (GUIDs or Cryptographic Hashes)?
*   **Performance:** Note that this consumes server memory. It is excellent for high security (banking) but might scale poorly for massive datasets.

---

## 3. Strategy 2: Access Control Logic (The "Ownership Check")
**Reference:** PDF Page 80 ("What the Code Reviewer needs to do").

While Indirect Maps hide the key, they are complex to implement. The most robust fix is **Access Control Checks** at the Data Layer. This forces the application to verify ownership *before* returning data.

### The "Syntactic" vs. "Semantic" Validation
*   **Syntactic Validation (A1 - Input Validation):** "Is the ID a number?" (e.g., `500`).
    *   *Reviewer:* This prevents SQL Injection, but **allows** IDOR.
*   **Semantic Validation (A4 - Access Control):** "Is ID 500 owned by User A?"
    *   *Reviewer:* This **prevents** IDOR.

### The Logic Pattern: "Trust but Verify"
Every time a "Direct Reference" (ID) comes from the client, the code must perform a check.

#### Vulnerable Pattern (No Verification)
```csharp
// Controller
public ActionResult ViewOrder(int orderId) {
    // VULNERABLE: The database just gets the order by ID.
    // It doesn't care who is logged in.
    Order order = db.Orders.Find(orderId); 
    return View(order);
}
```

#### Secure Pattern 1: The "And Clause" (Best Performance)
The most efficient fix is to add the User ID to the SQL `WHERE` clause.

```csharp
// Controller
public ActionResult ViewOrder(int orderId) {
    int currentUserId = Session["UserId"];
    
    // SECURE: The query asks for the Order ID AND the Owner ID.
    // If the order exists but belongs to someone else, this returns null.
    Order order = db.Orders.FirstOrDefault(o => o.ID == orderId && o.OwnerID == currentUserId);
    
    if (order == null) {
        return Error("Order not found or access denied.");
    }
    return View(order);
}
```

#### Secure Pattern 2: The "Post-Fetch Check" (Logic Check)
Sometimes complex business logic requires loading the object first.

```java
// 1. Load the Object
Invoice inv = db.getInvoice(invoiceId);

// 2. Perform the Security Check (Gatekeeper)
if (inv.getOwnerId() != currentUser.getId()) {
    // 3. Hard Fail
    throw new AccessDeniedException("User attempted to view unauthorized invoice.");
}

// 4. Return Data
return inv;
```

---

## 4. Secure Design Recommendations (PDF p. 80)
The OWASP guide emphasizes that security should be "Designed In," not "Bolted On."

### A. Centralized Access Control
Do not scatter `if (user.id == owner.id)` checks inside every single function. Developers will forget one.
**Reviewer Recommendation:** Look for a **Centralized Service Layer**.
*   All database calls should go through a `DataService`.
*   The `DataService` should automatically require the `CurrentContext` (User) to be passed in.

### B. Deny by Default
If the user's permissions cannot be explicitly verified (e.g., the database lookup failed, or the session is null), the code must default to **Access Denied**.
*   **Bad Logic:** `if (isRestricted == true) { deny(); }` (If the dev forgets to set the restricted flag, it is open).
*   **Good Logic:** `if (isAuthorized == true) { allow(); } else { deny(); }` (If the dev forgets to set the flag, it is closed).

---

## 5. Reviewer’s Checklist: Prevention Strategies

### Testing the Logic
When you are reading the code, ask these questions for every database call:

1.  **The Source:** Where did this ID come from? (URL? Hidden Field? Cookie?).
2.  **The Verification:** Is there an `if` statement checking ownership?
3.  **The Query:** Does the SQL `WHERE` clause include the `UserID`?
4.  **The Association:** If the object doesn't belong to a user (e.g., a shared document), is there an Access Control List (ACL) table being checked?
    *   *Example:* `SELECT * FROM doc_permissions WHERE doc_id = ? AND user_id = ?`

### Testing Indirect Maps
If the application claims to use Indirect References (Random Tokens):
1.  **Scope:** Are the tokens unique per user? (If User A sends User B's token, does it work?).
2.  **Entropy:** Are the tokens predictable? (e.g., `token_1`, `token_2`).
3.  **Exposure:** Is the real database ID ever leaked in the HTML source code or JSON response? (If the map hides the ID in the URL, but the JSON body contains `{"id": 101}`, the protection is useless).

---
*Reference: OWASP Code Review Guide 2.0 (Pages 79-80)*
