# Security Rules Reference

## OWASP Top 10 Detection Guide

### A01 — Broken Access Control
- Direct object references using user-supplied IDs without authorization checks
- Missing role/permission checks before sensitive operations
- CORS misconfiguration: `Access-Control-Allow-Origin: *` on authenticated endpoints
- Path traversal: `../` in file paths derived from user input
- **Resource ownership not verified after auth** — JWT validated but resource ownership not checked (e.g. `GET /projects/{id}` returns data without verifying `project.owner_id == current_user.id`)
- **Privileged endpoint missing admin/role guard** — settings, API-key management, or admin routes with only authentication but no role check
- **Open redirect** — `redirect(request.args.get("next"))` without validating the URL is on an allowlisted domain

### A02 — Cryptographic Failures
- HTTP URLs for sensitive data transmission
- Weak algorithms: MD5, SHA1, DES, RC4 for passwords or tokens
- Hardcoded encryption keys
- Missing TLS certificate validation (`verify=False`, `InsecureSkipVerify`)
- Storing passwords in plaintext or with reversible encoding
- **JWT passed as URL query parameter** — `?token=<jwt>` gets recorded in server logs, proxy logs, and browser history; pass in `Authorization: Bearer` header instead
- **Token stored in `localStorage`** — accessible to any JavaScript on the page (XSS risk); use `httpOnly` + `Secure` + `SameSite=Strict` cookies instead
- **Non-constant-time secret comparison** — `token == expected` is vulnerable to timing attacks; use `hmac.compare_digest(token, expected)` (Python) or equivalent

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

**Server-Side Template Injection (SSTI):**
```python
# BAD — Jinja2 renders user input as a template
from jinja2 import Environment
env = Environment()
template = env.from_string(user_input)  # user can inject {{ config }} or RCE payloads

# GOOD — treat user input as data, never as template source
template = env.get_template("fixed_template.html")
template.render(user_value=user_input)
```

**XML External Entity (XXE):**
```python
# BAD — lxml resolves external entities by default
from lxml import etree
tree = etree.parse(user_file)

# GOOD — disable entity resolution
parser = etree.XMLParser(resolve_entities=False, no_network=True)
tree = etree.parse(user_file, parser)
```

**ReDoS — catastrophic backtracking:**
```python
# BAD — nested quantifiers on overlapping character classes
import re
re.match(r"(a+)+$", user_input)   # exponential on "aaa...X"
re.match(r"(\w+\s?)+$", user_input)

# GOOD — use possessive quantifiers, atomic groups, or set a timeout
# Or validate input length before applying complex regex
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
- **File content read fully into memory before size check** — `data = file.read()` then `if len(data) > limit` — an attacker sends a huge payload causing OOM before the check runs; check `Content-Length` header or stream with a size cap
- **Unconstrained string field where an enum is required** — schema accepts `status: str` but only `"active"` / `"inactive"` are valid; use `Literal["active", "inactive"]` or an `Enum` type

### A05 — Security Misconfiguration
- Debug mode enabled in production (`DEBUG=True`, `app.run(debug=True)`)
- Default credentials not changed
- Verbose error messages exposing stack traces to end users
- Unnecessary open ports or services
- **Missing security headers** — responses should include:
  - `Strict-Transport-Security: max-age=63072000; includeSubDomains`
  - `Content-Security-Policy: default-src 'self'`
  - `X-Frame-Options: DENY` (or `SAMEORIGIN`)
  - `X-Content-Type-Options: nosniff`
  - `Referrer-Policy: strict-origin-when-cross-origin`
- **CSRF protection absent** — state-changing endpoints (POST/PUT/DELETE) on cookie-authenticated apps must verify a CSRF token or use `SameSite=Strict` cookies

### A06 — Vulnerable and Outdated Components
- Newly added dependencies with known CVEs
- Pinning to a specific old version with known vulnerabilities
- **Known unmaintained packages** (flag on sight):
  - Python: `python-jose` (no releases since 2021) → use `authlib` or `PyJWT`
  - Python: `itsdangerous<2.0`, `pycrypto` (abandoned) → use `cryptography`
  - JS: `node-uuid` → use `uuid`, `request` (deprecated) → use `axios` or `got`
  - Any package with 0 commits in 2+ years + open security issues

### A07 — Identification and Authentication Failures
- Weak password policies
- Missing account lockout after failed attempts
- JWT without expiry (`exp` claim missing)
- Sessions not invalidated on logout
- **Hardcoded fallback secret** — `os.getenv("SECRET_KEY", "change-me-in-prod")` — if the env var is missing in production, tokens are signed with a known string; fail loudly instead: `os.environ["SECRET_KEY"]` (raises `KeyError` rather than silently using a weak default)
- **JWT algorithm not pinned** — `jwt.decode(token, key)` without specifying `algorithms=["HS256"]` allows algorithm confusion attacks (`"alg": "none"`)
- **Refresh token not rotated** — issuing new access tokens without invalidating the old refresh token enables token replay after compromise

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
| Anthropic API Key | `sk-ant-api03-[a-zA-Z0-9\-_]{93}` |
| OpenAI API Key | `sk-[a-zA-Z0-9]{48}` |
| Google API Key | `AIza[0-9A-Za-z\-_]{35}` |
| Twilio Auth Token | `(?i)twilio.{0,20}['\"][0-9a-f]{32}['\"]` |
| Generic Secret (env fallback) | `os\.getenv\(['\"][^'\"]+['\"],\s*['\"][^'\"]{4,}['\"]` — second arg is a hardcoded fallback |

**Recommended fix for all secrets**: Replace with `os.environ["VAR_NAME"]` (or equivalent) and rotate the exposed credential immediately.
