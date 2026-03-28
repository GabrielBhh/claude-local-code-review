# Security Rules Reference

## OWASP Top 10 Detection Guide

### A01 — Broken Access Control
- Direct object references using user-supplied IDs without authorization checks
- Missing role/permission checks before sensitive operations
- CORS misconfiguration: `Access-Control-Allow-Origin: *` on authenticated endpoints
- Path traversal: `../` in file paths derived from user input

### A02 — Cryptographic Failures
- HTTP URLs for sensitive data transmission
- Weak algorithms: MD5, SHA1, DES, RC4 for passwords or tokens
- Hardcoded encryption keys
- Missing TLS certificate validation (`verify=False`, `InsecureSkipVerify`)
- Storing passwords in plaintext or with reversible encoding

### A03 — Injection

**SQL Injection patterns:**
```python
# BAD
query = f"SELECT * FROM users WHERE email = '{email}'"
cursor.execute("SELECT * FROM users WHERE id = " + user_id)

# GOOD
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

**Shell Injection patterns:**
```python
# BAD
os.system(f"convert {filename} output.png")
subprocess.run(f"grep {search_term} logs.txt", shell=True)

# GOOD
subprocess.run(["convert", filename, "output.png"])
```

**XSS patterns (JavaScript):**
```javascript
// BAD
element.innerHTML = userInput
document.write(userInput)

// GOOD
element.textContent = userInput
// or: DOMPurify.sanitize(userInput) if HTML is required
```

**PHP injection patterns:**
```php
// BAD — SQL injection
$result = mysqli_query($conn, "SELECT * FROM users WHERE id = " . $_GET['id']);

// GOOD — prepared statement
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$_GET['id']]);

// BAD — XSS
echo $_GET['name'];

// GOOD
echo htmlspecialchars($_GET['name'], ENT_QUOTES, 'UTF-8');

// BAD — type juggling (loose equality allows "0abc" == 0 → true)
if ($_POST['role'] == 0) { grant_admin(); }

// GOOD
if ($_POST['role'] === 0) { grant_admin(); }
```

### A04 — Insecure Design
- Business logic that can be abused (e.g., negative quantities, skipping steps)
- Missing rate limiting on authentication or sensitive endpoints

### A05 — Security Misconfiguration
- Debug mode enabled in production (`DEBUG=True`, `app.run(debug=True)`)
- Default credentials not changed
- Verbose error messages exposing stack traces to end users
- Unnecessary open ports or services

### A06 — Vulnerable and Outdated Components
- Newly added dependencies with known CVEs
- Pinning to a specific old version with known vulnerabilities

### A07 — Identification and Authentication Failures
- Weak password policies
- Missing account lockout after failed attempts
- JWT without expiry (`exp` claim missing)
- Sessions not invalidated on logout

### A08 — Software and Data Integrity Failures
- Deserialization of untrusted data (`pickle.loads`, `yaml.load` without `Loader=`)
- Missing integrity checks on downloaded artifacts

### A09 — Security Logging and Monitoring Failures
- Authentication failures not logged
- Sensitive operations (admin actions, privilege changes) not audited

### A10 — Server-Side Request Forgery (SSRF)
- User-controlled URLs passed to HTTP clients without validation
- Internal endpoints reachable via redirect

---

## Secret / Credential Detection Patterns

Scan changed lines for these patterns. Any match is a 🔴 Critical finding.

| Secret type | Regex pattern |
|---|---|
| AWS Access Key | `AKIA[0-9A-Z]{16}` |
| AWS Secret Key | `(?i)aws.{0,20}secret.{0,20}['\"][0-9a-zA-Z/+]{40}` |
| GitHub Token | `ghp_[a-zA-Z0-9]{36}` or `github_pat_[a-zA-Z0-9_]{82}` |
| Stripe Live Key | `sk_live_[0-9a-zA-Z]{24,}` |
| Stripe Test Key | `sk_test_[0-9a-zA-Z]{24,}` |
| SendGrid API Key | `SG\.[a-zA-Z0-9_-]{22,}\.[a-zA-Z0-9_-]{43,}` |
| Slack Token | `xox[baprs]-[0-9a-zA-Z-]{10,}` |
| Private Key Block | `-----BEGIN (RSA\|EC\|OPENSSH\|PGP) PRIVATE KEY` |
| Generic Password | `(?i)(password\|passwd\|pwd)\s*=\s*['"][^'"]{4,}['"]` |
| DB Connection String | `(?i)(postgres\|mysql\|mongodb):\/\/[^:]+:[^@]+@` |
| JWT | `eyJ[a-zA-Z0-9_-]+\.eyJ[a-zA-Z0-9_-]+\.[a-zA-Z0-9_-]+` |

**Recommended fix for all secrets**: Replace with `os.environ["VAR_NAME"]` (or equivalent) and rotate the exposed credential immediately.
