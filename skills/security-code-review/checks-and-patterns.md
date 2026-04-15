# Security Code Review — Detailed Checks and Patterns

Full check details and language-specific patterns. Referenced from [SKILL.md](./SKILL.md).

> **OWASP Coverage Note:** SSRF is partially covered under "External HTTP requests" in Trigger Surfaces rather than as a standalone check, since it typically surfaces during HTTP/webhook review. Other OWASP categories (e.g., logging failures, integrity verification) are intentionally deferred to infrastructure-level tooling.

## The Eight Checks — Full Details

### 1. Injection
- SQL: raw string concatenation with user input? Parameterized queries used everywhere?
- Command injection: `exec`, `spawn`, `eval` with user-controlled input?
- Template injection: user input rendered in template strings?
- Path traversal: file paths constructed from user input without canonicalization?

### 2. Authentication & Session
- Hardcoded credentials or API keys in source?
- Session tokens stored in localStorage (should be httpOnly cookies)?
- Password comparison using `==` instead of constant-time comparison?
- JWT `alg: none` accepted? Secret validated on every request?
- Auth checks applied to every protected route, including new ones added in this diff?

### 3. Access Control
- New endpoints: are they behind the same auth middleware as existing ones?
- Object-level access: does the code verify the requesting user owns the resource?
- Function-level access: admin-only operations guarded at the function, not just UI?
- IDOR: IDs in URLs/params — are they validated against the authenticated user's scope?

### 4. Sensitive Data Exposure
- Secrets, tokens, or PII logged to stdout/files?
- Error responses include stack traces or internal paths?
- Sensitive fields returned in API responses that shouldn't be (passwords, internal IDs)?
- HTTPS enforced for all external calls in this diff?

### 5. Security Misconfiguration
- CORS: `Access-Control-Allow-Origin: *` on authenticated endpoints?
- CSP headers set where HTML is served?
- Debug mode or verbose errors enabled in production paths?
- New dependencies: any with known CVEs? (check with `npm audit` / `pip-audit` / `trivy`)

### 6. Cryptography
- MD5 or SHA1 used for security purposes (not just checksums)?
- Passwords hashed with bcrypt/argon2/scrypt — not SHA-256 or similar?
- Random tokens generated with `crypto.randomBytes` / `secrets.token_hex(16)` — not `Math.random()`?
- IV/nonce reused across encryptions?

### 7. Third-Party Dependencies
- New packages added — are they maintained, widely used, not typosquatted?
- Pinned to exact versions in lock file?
- Any `postinstall` scripts in new packages?

### 8. Business Logic
- Can a user skip a required step (payment, verification) by directly calling a later API?
- Rate limiting applied to sensitive endpoints (login, password reset, OTP)?
- Mass assignment: does the code bind request body directly to model without allowlist?

## Node.js Red Flags

```js
// BAD: SQL injection
db.query(`SELECT * FROM users WHERE id = ${req.params.id}`)
// GOOD:
db.query('SELECT * FROM users WHERE id = ?', [req.params.id])

// BAD: path traversal
fs.readFile(path.join(__dirname, req.query.file))
// GOOD:
const safe = path.resolve(baseDir, req.query.file)
if (!safe.startsWith(baseDir)) throw new Error('Invalid path')
```

## Python Red Flags

```python
# BAD: command injection
subprocess.run(f"convert {user_filename}", shell=True)
# GOOD:
subprocess.run(["convert", user_filename])

# BAD: weak random (predictable PRNG, not cryptographically secure)
import random; token = ''.join(random.choices('0123456789abcdef', k=32))
# GOOD:
import secrets; token = secrets.token_hex(16)
```
