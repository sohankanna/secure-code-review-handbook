# Secure Code Review Handbook (OWASP v2.0 Study Guide)

This repository contains comprehensive notes, code examples, and checklists for mastering **Secure Code Review (SCR)**. It is based primarily on the **OWASP Code Review Guide 2.0**.

## 🛡️ The "Golden Rule" of SCR
> **"All external input, no matter what it is, will be examined and validated."**

## 📖 Methodology Overview
Secure Code Review is a "white-box" approach. Unlike penetration testing (black-box), SCR identifies the **root cause** of vulnerabilities within the logic and flow of the code.

### The CMM Maturity Levels for Review
1. **Level 1 (Ad-hoc):** Unstable, inconsistent reviews.
2. **Level 2 (Repeatable):** Defined process for high-risk modules.
3. **Level 3 (Defined):** Security standards are at the forefront of the organization.

### Detection Effectiveness
Based on industry surveys, SCR is the **most effective** method for finding:
*   **Business Logic Bugs** (where scanners fail)
*   **Privacy Issues**
*   **Compliance Flaws** (PCI-DSS, HIPAA)

---

## 📂 Repository Index

### Section 3: Technical Reference (The OWASP Top 10)
*   [**A1: Injection**](./01-technical-reference/A1-Injection/README.md) - SQL, HQL, JSON, and Command Injection.
*   [**A2: Broken Auth & Session Management**](./01-technical-reference/A2-Broken-Auth/README.md)
*   [**A3: Cross-Site Scripting (XSS)**](./01-technical-reference/A3-XSS/README.md)
*   [**A4: Insecure Direct Object Reference (IDOR)**](./01-technical-reference/A4-IDOR/README.md)
*   *...[Link other folders as you create them]...*

### Section 4: Security Topics
*   [HTML5 Security](./02-security-topics/HTML5.md)
*   [Same Origin Policy](./02-security-topics/SOP.md)
*   [Reviewing Logging & Error Handling](./02-security-topics/Logging-Errors.md)

---
*Created for Exam Preparation - Based on OWASP Release 2.0*
