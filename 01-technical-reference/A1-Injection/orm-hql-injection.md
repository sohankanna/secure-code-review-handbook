# ORM & HQL Injection: Framework Specific Vulnerabilities

## 1. Introduction: The "Translator" Analogy
To understand an **ORM (Object-Relational Mapper)**, imagine you are trying to communicate with someone who speaks a different language.

*   **The Developer** speaks "Java" or "C#."
*   **The Database** speaks "SQL."
*   **The ORM (Hibernate or Entity Framework)** is the **Translator**.

The Developer says to the Translator: *"Find me the user with ID 5."* The Translator turns that into a SQL command: `SELECT * FROM users WHERE id = 5`.

**The Myth:** Developers often think, *"I'm using a Translator (ORM), so I never have to speak SQL directly. Since I'm not speaking SQL, I can't have SQL Injection!"*

**The Reality (PDF p. 50):** 
> "It is a very common misconception that ORM solutions, like Hibernate, are SQL Injection proof."

If the Developer gives the Translator a "dirty" sentence, the Translator will faithfully translate those "dirty" parts into the final SQL command, resulting in an attack.

---

## 2. What is HQL Injection? (Java/Hibernate)
**HQL** stands for **Hibernate Query Language**. It looks almost exactly like SQL, but instead of talking about database tables, it talks about Java objects.

### 🚩 The Vulnerable Pattern: String Concatenation in HQL
Just like in "Classic SQL," the danger arises when a developer uses the `+` sign to glue user input into an HQL string.

**Vulnerable Java Code (Sample 7.7/12.3):**
```java
// THE RED FLAG: Look for the "+" sign inside createQuery
String userSuppliedId = request.getParameter("id");
List results = session.createQuery("from Items as item where item.id = " + userSuppliedId).list();
```

**Why it is a vulnerability:**
Even though this isn't "Raw SQL," Hibernate's engine will take this string and convert it into SQL. 
*   If a user enters `5`, it becomes: `where item.id = 5`.
*   If a user enters `5 OR 1=1`, it becomes: `where item.id = 5 OR 1=1`.

The **HQL Parser** gets confused just like a SQL Parser. It sees the injected `OR 1=1` and executes it. This allows an attacker to bypass security checks, see all items in the database, or even delete data depending on the permissions.

---

## 3. The .NET Side: Linq and Entity Framework
In the Microsoft world (.NET), we use **Entity Framework** and a language called **Linq**.

### The Safety of Linq (PDF p. 52)
The guide notes that: 
> "Linq is not SQL and because of that is not prone to SQL injection." 

When a developer uses standard Linq "Method Syntax" like `.Where(x => x.Id == id)`, it is **inherently safe** because the framework handles the data and code separately by default.

### 🚩 The .NET Red Flag: `ExecuteQuery`
The danger happens when a developer decides to "break out" of the safe Linq box to write something custom.

**Vulnerable .NET Code (Sample 7.10/PDF p. 53):**
```csharp
string userName = ctx.GetAuthenticatedUserName();
// DANGER: Building a raw string for an ORM query
string query = "SELECT * FROM Items WHERE owner = '" + userName + "' AND itemname = '" + ItemName.Text + "'";
List items = sess.CreateSQLQuery(query).List();
```

**Why this fails a Secure Code Review:**
The developer is using `CreateSQLQuery`. This tells the ORM: *"Don't use your safe logic; just take this string and run it as raw SQL."* Because the developer used the `+` sign to add `ItemName.Text`, an attacker can now perform a standard SQL Injection attack.

---

## 4. Why do developers make these mistakes?
As a reviewer, it helps to understand why this code exists in the first place:
1.  **Laziness:** It is faster to type `+ input` than to set up a parameter object.
2.  **Complexity:** Some complex searches are hard to write in "Safe" Linq or HQL, so developers revert to string building.
3.  **Legacy Habits:** Older developers who are used to "Classic ASP" or "Legacy PHP" often bring their string-gluing habits into modern frameworks.

---

## 5. The "Native SQL" Backdoor (The "Double Sink")
Most ORMs have a feature called **Native SQL**. This allows a developer to bypass the ORM entirely and talk to the database in its native tongue.

**The Logic (PDF p. 52):**
> "Hibernate allows the use of 'native SQL'... the former is prone to SQL Injection and the later is prone to HQL (or ORM) injection."

As a reviewer, if you see keywords like:
*   `createSQLQuery()` (Java)
*   `ExecuteSqlCommand()` (.NET)
*   `SqlQuery()` (.NET)

...you should immediately assume the code is **High Risk**. You must check every single variable being passed into these functions.

---

## 6. How to Identify this during a Code Review (Grep Guide)
When you are prepping for your exam, remember these "Search Terms" to find ORM vulnerabilities in a massive codebase:

| Framework | Search for these "Sinks" | What to look for |
| :--- | :--- | :--- |
| **Hibernate (Java)** | `createQuery` | Look for `+` or `String.format` inside. |
| **Hibernate (Java)** | `createSQLQuery` | This is raw SQL; it's almost always a red flag. |
| **Entity Framework (.NET)** | `ExecuteQuery` | Check if variables are concatenated. |
| **Entity Framework (.NET)** | `FromSqlRaw` | This is a modern .NET sink that is dangerous. |
| **Linq (.NET)** | `.Where("...")` | Some versions of Linq allow string-based queries. |

---

## 7. The Remediation: How to Fix it Properly
The OWASP guide (p. 53) provides the clear solution for both Java and .NET.

### The Java Fix: Parameterized HQL
Instead of gluing the string, we use "Named Parameters."
```java
// SECURE: Notice the ":" symbol. That is a placeholder.
Query query = session.createQuery("from Items where id = :itemId");
query.setParameter("itemId", currentItem.getId()); // The ORM now treats this as data only.
```

### The .NET Fix: Parameter Collections (Sample 7.10)
```csharp
// SECURE: Use Parameters.Add instead of concatenation
sqlAdapter.SelectCommand.Parameters.Add("@usrId", SqlDbType.VarChar, 15);
sqlAdapter.SelectCommand.Parameters["@usrId"].Value = UID.Text;
```
**Why this is safe:**
1.  **Literal Value:** The input is treated as a literal value (just text), never as code.
2.  **Type Checking:** If the database expects a Number and the attacker sends a String, the system will throw an error before the query even runs.
3.  **Length Validation:** The `15` in the code above forces the input to be short, preventing long "buffer-heavy" attacks.

---

## 8. Secure Code Reviewer's "Mental Checklist" for ORMs
1.  **Is there a `+` sign?** If I see a string being built with a plus sign inside a database-related function, it is **vulnerable** until proven otherwise.
2.  **Is it a Native Query?** If the code says `createSQLQuery`, the developer has discarded all framework protections. I must review it like 1990s-style SQL.
3.  **Is Input Missing?** (The Logic Trap). If the user doesn't provide input, does the ORM generate a query that returns everything? (Check `if/else` logic).
4.  **Are placeholders used?** Look for `:` (Hibernate) or `@` (.NET). If these are missing, the code is likely insecure.

---

## 9. Summary for the Exam
*   **HQL Injection** is real.
*   **Linq** is usually safe, but **Raw SQL functions** in .NET break that safety.
*   **Concatenation** is the enemy, regardless of the framework.
*   **Named Parameters** (`:name`) are the required fix.

---
*End of ORM & HQL Injection Notes*
*Reference: OWASP Code Review Guide 2.0 (Pages 50-53)*
