# A9: Using Components with Known Vulnerabilities — Technical Reference

## 1. Executive Summary
Modern software is not written; it is **assembled**. Applications are built on top of countless third-party components, frameworks, and libraries (e.g., Log4j, Struts, jQuery, Newtonsoft.Json).

**The Vulnerability (A9):**
This vulnerability occurs when an application uses a component with a publicly known security flaw (a **CVE - Common Vulnerabilities and Exposures**).

According to the **OWASP Code Review Guide 2.0 (p. 147)**, if a vulnerable component is exploited, the attack often facilitates serious data loss or server takeover because these components almost always run with the full privileges of the application.

### The "Car Recall" Analogy
*   You build a car. You are an expert at making engines and transmissions.
*   To save time, you buy pre-made airbags from a third-party company.
*   One year later, the airbag company announces a major recall: "Our airbags from last year's model might explode."
*   If you don't know that you used those specific airbags in your car, you can't fix the problem. Your car is now a ticking time bomb.

In software, the "airbag" is a library like `log4j-2.14.0.jar`. The "recall" is a CVE like **Log4Shell**. If you don't know you are using that version, you cannot protect your application.

---

## 2. The Reviewer's Role: Beyond Source Code
The guide (p. 147) explicitly states:
> "There is really no code to review for this topic... However code review can be used within the larger company-wide tracking or audit mechanisms."

A traditional code review (looking for SQLi, XSS) will **completely miss** A9 vulnerabilities. You are not looking for bugs in the custom code; you are auditing the **"Bill of Materials."**

Your job is to answer three questions:
1.  **What components are we using?** (Inventory)
2.  **Are any of these components known to be vulnerable?** (Analysis)
3.  **Do we have a process to update them?** (Remediation)

---

## 3. Identifying the Components: The Bill of Materials
To find the components, you must review the **Package Manager Configuration Files**.

| Language / Ecosystem | Configuration File | Example Libraries |
| :--- | :--- | :--- |
| **Java** | `pom.xml` (Maven) / `build.gradle` (Gradle) | `log4j`, `spring-core`, `struts2-core` |
| **.NET** | `packages.config` / `.csproj` (NuGet) | `Newtonsoft.Json`, `log4net`, `EntityFramework` |
| **JavaScript** | `package.json` | `express`, `lodash`, `jquery` |
| **Python** | `requirements.txt` | `Django`, `Flask`, `requests` |

**Example `pom.xml` (Java/Maven):**
```xml
<dependencies>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <!-- REVIEWER: Check this version! -->
        <version>2.14.0</version> 
    </dependency>
</dependencies>
```
*   **Reviewer Action:** You see `log4j-core` version `2.14.0`. You google "log4j 2.14.0 vulnerability." You immediately find **Log4Shell (CVE-2021-44228)**. This is a **Critical Finding**.

---

## 4. Automation: Software Composition Analysis (SCA)
Manually checking every version of every library is impossible. The industry standard is to use **Software Composition Analysis (SCA)** tools.

The guide (p. 148) recommends the **OWASP Dependency-Check** project.
*   **How it Works:** The tool scans your package manager files, builds a list of all dependencies (including transitive dependencies), and checks them against the **National Vulnerability Database (NVD)** for known CVEs.
*   **Reviewer Action:** As part of the code review process, you should run (or ensure the CI/CD pipeline runs) an SCA tool. The report from this tool is a primary artifact for the A9 review.

---

## 5. Controlling the Ingredients (PDF p. 147)
Large organizations cannot allow developers to pull in any random library from the internet.

### A. The "Approved List" Policy
*   **The Strategy:** The company maintains a central repository (like Artifactory or Nexus) with a curated list of approved and vetted libraries.
*   **Reviewer Check:** If you see a dependency in a `pom.xml` that is not on the approved list, the code review should be **rejected**.

### B. The "Sledgehammer to Crack a Nut" Problem
**Reference:** PDF Page 148.

Developers often include a massive library to use one tiny function.
*   **Example:** Including a 10MB PDF generation library just to use its `isEmailValid()` function.
*   **The Risk:** You have now inherited the entire attack surface of that 10MB library (its file parsers, its network connections) for a feature you don't even need.

**Reviewer Action:**
*   **Question the Dependency:** "Is this entire library necessary, or can we write this simple function ourselves?"
*   **Minimalism:** If the library is required, can unused features be disabled? Can unused modules be excluded from the build? This reduces the attack surface.

---

## 6. The "Transitive Dependency" Trap
You might only have 5 direct dependencies in your `package.json`. But those 5 libraries might each have 10 dependencies of their own. This is called a **Transitive Dependency Chain**.

**Example (JavaScript):**
Your App -> `express.js` -> `accepts` -> `mime-types` -> `mime-db`

**The Risk:** A vulnerability in `mime-db` (a library you've never heard of) makes your entire application vulnerable.
**Reviewer Action:** You **must** use an SCA tool. Only an automated tool can build the full dependency tree and check for these hidden vulnerabilities.

---

## 7. Secure Code Reviewer’s Checklist (A9)

- [ ] **Inventory:** Is there a clear, machine-readable list of all third-party components? (`pom.xml`, `package.json`).
- [ ] **SCA Tool:** Is an SCA tool (like OWASP Dependency-Check or Snyk) integrated into the build pipeline?
- [ ] **Review Reports:** Are the results of the SCA scan reviewed for high-severity CVEs?
- [ ] **Patching Policy:** Is there a policy in place to update libraries when a new vulnerability is announced? (e.g., "All Critical CVEs must be patched within 72 hours").
- [ ] **Unused Dependencies:** Are there libraries in the project that are commented out or no longer used? (They should be removed).
- [ ] **Minimalism:** Is a large, multi-function library being used for a single, simple purpose? (The "Sledgehammer" problem).
- [ ] **License Risk:** (Beyond security) Does the SCA tool also check for license conflicts? (e.g., using a GPL library in a commercial product).

---
*Reference: OWASP Code Review Guide 2.0 (Pages 146-148)*
