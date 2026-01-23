# Dependency Management & SCA: The Modern Code Review

## 1. Executive Summary
In the past, code reviews focused on the code written by the developer. In a modern application, **over 80%** of the final codebase often comes from third-party, open-source libraries.

This means the most significant security risk is not a bug in the 1,000 lines of custom code, but a known vulnerability (CVE) in the **1,000,000 lines of library code** the application depends on.

**Software Composition Analysis (SCA)** is the automated process of:
1.  Identifying all third-party components in a project.
2.  Checking those components against databases of known vulnerabilities.

As a reviewer, your primary job for A9 is to ensure this process exists, is automated, and that its findings are acted upon.

---

## 2. The "Bill of Materials": Where to Look
The first step in any SCA process is to identify the **Software Bill of Materials (SBOM)**. This is not a separate document; it is the **package manager's configuration file**.

As a reviewer, these files are your primary target.

### A. Java: `pom.xml` (Maven)
Maven is the most common build tool for Java. The `pom.xml` file lists all direct dependencies.

**Example `pom.xml`:**
```xml
<project>
    ...
    <dependencies>
        <!-- REVIEW THIS SECTION -->
        <dependency>
            <groupId>org.apache.struts</groupId>
            <artifactId>struts2-core</artifactId>
            <version>2.5.20</version> <!-- Is this version vulnerable? -->
        </dependency>
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version> <!-- Is this version vulnerable? -->
        </dependency>
    </dependencies>
</project>
```**Reviewer Action:**
1.  Open the `pom.xml`.
2.  For each `<dependency>`, take the `version` and google it (e.g., "struts 2.5.20 vulnerability").
3.  *Note:* This manual process is slow and only finds direct dependencies. It demonstrates the need for an automated tool.

### B. JavaScript: `package.json` (NPM/Yarn)
The `package.json` file lists dependencies for Node.js and front-end projects.

**Example `package.json`:**
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.17.1",
    "lodash": "4.17.15",
    "jquery": "3.4.1"
  }
}
```
**Reviewer Action:**
*   Check the versions. The `^` (caret) in `^4.17.1` means "accept any minor update," which can introduce risk if not managed.
*   Run `npm audit` in the command line. This is NPM's built-in SCA tool.

### C. .NET: `.csproj` / `packages.config` (NuGet)
In .NET, dependencies are managed by NuGet.

**Example `.csproj`:**
```xml
<ItemGroup>
    <PackageReference Include="Newtonsoft.Json" Version="12.0.3" />
    <PackageReference Include="EntityFramework" Version="6.2.0" />
</ItemGroup>
```
**Reviewer Action:**
*   Check the versions against known CVEs.
*   Modern Visual Studio and tools like `dotnet list package --vulnerable` can perform this check automatically.

---

## 3. The Transitive Dependency Nightmare
This is the most critical concept in SCA. You are responsible not only for the libraries you choose, but for the libraries **they** choose.

### The Dependency Tree
Your App -> **Library A** -> **Library B** -> **Library C (Vulnerable)**

You have never heard of Library C, but it is now part of your application.

**How SCA Tools Solve This:**
An SCA tool builds a **full dependency graph**. It traverses the entire tree and identifies every component, direct or transitive.

**Example SCA Tool Output:**
```
[INFO] Scanning Project...
[WARNING] Found CVE-2022-12345 in jackson-databind-2.13.0.jar
    Description: Remote Code Execution vulnerability.
    Severity: CRITICAL (9.8)
    Dependency Path:
        com.my.app:myapp:jar:1.0
        -> com.example:data-importer:jar:2.1
           -> com.fasterxml.jackson.core:jackson-databind:jar:2.13.0
```
**Reviewer Analysis:**
*   The report clearly shows that the vulnerability is in `jackson-databind`.
*   You didn't include it directly. It was brought in by the `data-importer` library.
*   **Actionable Finding:** "The `data-importer` library has a critical transitive dependency. Update `data-importer` to a version that uses a patched version of `jackson-databind`."

---

## 4. OWASP Dependency-Check (The Reviewer's Tool)
The guide (p. 148) explicitly recommends this free, open-source tool.

### How to Integrate it into a Review
1.  **Run the Scan:** It can be run as a command-line tool, a Maven plugin, a Gradle plugin, etc.
    ```bash
    # Example for a Java/Maven project
    mvn org.owasp:dependency-check-maven:check
    ```
2.  **Analyze the Report:** The tool generates an HTML report listing every dependency and any associated CVEs.
3.  **The Review Finding:**
    *   **If no SCA tool is used:** "The project lacks an automated process for identifying known vulnerabilities in its dependencies. Implement OWASP Dependency-Check in the CI/CD pipeline."
    *   **If the tool finds vulnerabilities:** "The SCA scan has identified Critical vulnerability CVE-XXXX-XXXX in library Y. This must be remediated by updating the library to version Z."

---

## 5. Reviewing the Remediation Strategy
Finding vulnerabilities is only half the battle. The reviewer must also check the team's process for fixing them.

### A. Version Pinning and Ranges
**Vulnerable `package.json`:**
```json
"dependencies": {
  "some-library": "*"  // DANGER: Pulls the latest version, which could be anything.
}
```

**Secure Practice:**
*   **Pinning:** Specify an exact version (`"1.2.3"`). This is reproducible but requires manual updates.
*   **Semantic Versioning:** Use `~1.2.3` (patch versions) or `^1.2.3` (minor versions). This allows for security patches but requires trust in the library author.

### B. Stale Dependencies
*   **Reviewer Check:** Look at the "Last Modified" date on the package manager files. If dependencies haven't been updated in 2-3 years, the project is likely riddled with unfixed vulnerabilities. This is a sign of poor "Security Hygiene."

### C. The "Internal Repository" Policy
The guide (p. 147) recommends that large companies control which libraries can be used.
*   **The Architecture:** The company hosts an internal repository (Artifactory, Nexus).
*   **The Policy:** The build server is configured to **only** download from this internal repo.
*   **The Review:** Check the build configuration (`settings.xml` in Maven). Does it point to the public internet (`repo.maven.apache.org`) or the internal repository (`artifactory.mycompany.com`)?

---

## 6. Secure Code Reviewer’s Checklist (SCA)

- [ ] **SCA Tool:** Is an SCA tool integrated into the project's build process?
- [ ] **CI/CD Integration:** Does the build **fail** if a new dependency with a Critical vulnerability is introduced?
- [ ] **Vulnerability Triage:** Is there a documented process for handling the SCA report? Who is responsible for updating libraries?
- [ ] **Package Files:**
    *   Are versions pinned or using safe ranges?
    *   Are there any dependencies loaded from untrusted or non-standard repositories?
- [ ] **Dependency Hygiene:** Are there any commented-out or unused dependencies that should be removed?
- [ ] **License Compliance:** Does the SCA tool also scan for license risks (e.g., GPL in a proprietary product)?

---
*Reference: OWASP Code Review Guide 2.0 (Pages 147-148)*
