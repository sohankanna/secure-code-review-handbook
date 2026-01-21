# Transport Layer Security: TLS, HSTS, and Certificates

## 1. Executive Summary
**Transport Layer Security (TLS)** (formerly SSL) provides two essential services:
1.  **Confidentiality:** Encrypting data so eavesdroppers (Man-in-the-Middle) cannot read it.
2.  **Integrity:** Ensuring data is not modified in transit.
3.  **Authentication:** Verifying that the server you are talking to is actually who they claim to be (via Certificates).

**The Reviewer's Mandate:**
You must ensure that TLS is not just "turned on," but that it is **configured correctly**, used **everywhere**, and that the application code validates the connection properly.

---

## 2. Scope of Protection (The "All or Nothing" Rule)
In the past, websites only encrypted the Login Page and the Credit Card page. **This is no longer acceptable.**

### The "Mixed Mode" Vulnerability (PDF p. 120)
If an application switches between HTTP and HTTPS, it is vulnerable.
*   **Scenario:** User logs in on `https://site.com/login`.
*   **The Switch:** The site redirects them to `http://site.com/home` (HTTP).
*   **The Leak:** The browser sends the **Session Cookie** over the unencrypted HTTP connection.
*   **The Result:** An attacker on the same Wi-Fi network sniffs the cookie and hijacks the session.

**Reviewer Requirement:**
> "Prefer all interfaces (or pages) being accessible only over HTTPS... All networks, both external and internal, which transmit sensitive data must utilize TLS." (p. 120).

---

## 3. HTTP Strict Transport Security (HSTS)
**Reference:** PDF Page 53 (Table 11) & Page 121.

HSTS is a security header that solves the "Downgrade Attack" problem. It tells the browser: *"Never talk to me over HTTP again. Even if the user types 'http://', you must automatically switch to 'https://' before sending a single packet."*

### Reviewing HSTS Configuration
Check `web.config`, `.htaccess`, or the application code for this header.

**Secure Header Syntax:**
```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

**Reviewer Checklist for HSTS:**
1.  **Max-Age:** Is it long enough? (Recommended: 31536000 seconds = 1 Year). If it is set to `0`, HSTS is disabled.
2.  **SubDomains:** Is `includeSubDomains` present? If not, the `admin.site.com` subdomain might still be vulnerable to HTTP interception.
3.  **Preload:** (Advanced) Is the `preload` directive used? (Ensures protection on the very first visit).

---

## 4. Certificate Validation Logic (The "Trust" Trap)
This is a critical area for code review when the application **acts as a client** (e.g., a Java backend server talking to a Payment Gateway API).

### The Vulnerability: "Trust All Certs"
Developers often encounter SSL errors in development (due to self-signed certs) and write code to disable validation. **If this code reaches production, MITM attacks are trivial.**

#### 🚩 Red Flag Code (Java)
Grep for `TrustManager`, `X509TrustManager`, or `checkServerTrusted`.

```java
// VULNERABLE: The "Blind Trust" Manager
TrustManager[] trustAllCerts = new TrustManager[] { 
    new X509TrustManager() {
        public java.security.cert.X509Certificate[] getAcceptedIssuers() { 
            return null; 
        }
        // Empty method = No Check performed!
        public void checkClientTrusted(X509Certificate[] certs, String authType) { }
        public void checkServerTrusted(X509Certificate[] certs, String authType) { }
    } 
};

// Installing the vulnerable manager
SSLContext sc = SSLContext.getInstance("SSL");
sc.init(null, trustAllCerts, new java.security.SecureRandom());
```

#### 🚩 Red Flag Code (.NET)
Grep for `ServerCertificateValidationCallback`.

```csharp
// VULNERABLE: Always returns true, regardless of certificate errors
ServicePointManager.ServerCertificateValidationCallback = 
    delegate { return true; };
```

**Reviewer Action:** If you see any code overriding the default Trust Manager or Validation Callback, it is a **Critical Finding**. The application must rely on the OS/Java Trust Store, not custom bypass logic.

---

## 5. Certificate Quality & Chain of Trust (PDF p. 121)
When reviewing infrastructure-as-code or server configs, check the certificate parameters.

### A. Key Length & Algorithm
*   **Private Key:** Must be at least **2048 bits**. (1024 bits is considered broken/weak).
*   **Signature Algorithm:** Must be **SHA-256** (or better). SHA-1 is deprecated and marked as insecure by all modern browsers.

### B. Wildcards & RFC 1918
The guide warns against using Wildcard Certificates (`*.example.com`) carelessly.
*   **Risk:** If `internal.example.com` is compromised, the key can be used to impersonate `bank.example.com`.
*   **RFC 1918:** Certificates should never be issued for internal IP addresses (e.g., `192.168.1.1`). They should use Fully Qualified Domain Names (FQDN).

### C. The Certificate Chain
> "Always provide all certificates in the chain." (p. 121).

**The Issue:** A server sends its own certificate but forgets the "Intermediate" certificate. Desktop browsers might fix this (via AIA fetching), but **Mobile Apps** and **Server-to-Server** calls will often crash or fail.
**Reviewer Check:** Check the SSL configuration to ensure the "Bundle" or "Chain" file is included, not just the Leaf Certificate.

---

## 6. Information Leakage via URL (PDF p. 120)
TLS encrypts the *path* and *query string* of a request, but it does **not** hide them from:
1.  **Browser History**
2.  **Server Access Logs**
3.  **Proxy Logs**

**The Vulnerability:** Passing sensitive data in the URL (GET request).
`https://bank.com/login?password=SuperSecret`

Even though the traffic is encrypted on the wire, the URL `?password=SuperSecret` is stored in plaintext in the user's browser history and the server's `access.log`.

**Reviewer Action:**
*   Identify all sensitive data (passwords, SSN, credit cards, tokens).
*   Ensure they are transmitted via **HTTP POST** (Body), never HTTP GET (URL).

---

## 7. Caching Sensitive Data (PDF p. 121)
> "The TLS protocol provides confidentiality only for data in transit but it does not help with potential data leakage issues at the client."

If a user downloads a sensitive PDF bank statement over HTTPS, the browser might cache that file on the disk.

**Reviewer Check:**
Look for HTTP Response Headers on pages containing sensitive data.
*   **Secure:** `Cache-Control: no-store, no-cache`
*   **Vulnerable:** Missing Cache-Control headers allow the browser to save the file to the hard drive.

---

## 8. Secure Code Reviewer’s Checklist (Transport Layer)

### Configuration Checks
- [ ] **HSTS:** Is the `Strict-Transport-Security` header present with a long `max-age`?
- [ ] **Protocols:** Are SSLv2, SSLv3, and TLS 1.0/1.1 disabled? (Only TLS 1.2/1.3 allowed).
- [ ] **Weak Ciphers:** Are `RC4`, `DES`, and `NULL` ciphers disabled?
- [ ] **Cookies:** Do all cookies have the `Secure` flag? (See A2).

### Code Logic Checks
- [ ] **Trust Logic:** Search for `TrustManager` (Java) or `ServerCertificateValidationCallback` (.NET). Ensure they do NOT blindly return `true`.
- [ ] **Hostname Verifier:** Ensure the code verifies that the certificate Subject CN matches the Hostname being connected to.
- [ ] **Redirects:** Does the application force a redirect from HTTP to HTTPS immediately?
- [ ] **Mixed Content:** Ensure no scripts, images, or CSS are loaded over HTTP on an HTTPS page.

### Data Handling
- [ ] **GET vs POST:** Ensure passwords and tokens are never in the URL parameters.
- [ ] **Caching:** Ensure sensitive pages send `Cache-Control: no-store`.

---
*Reference: OWASP Code Review Guide 2.0 (Pages 53, 119-122)*
