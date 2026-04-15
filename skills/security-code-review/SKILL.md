---
name: security-code-review
description: Use before merging any feature that touches auth, input handling, APIs, file system, database queries, secrets, or permissions.
---

# Security Code Review

Systematic vulnerability assessment for code changes. Standalone from `requesting-code-review` because security review requires adversarial thinking about how code can be abused, not whether it meets requirements. Run independently — this skill has no ordering dependency on requesting-code-review.

**REQUIRED BACKGROUND:** Use `superpowers:requesting-code-review` for general quality review. This skill is for security-specific review only.

## Trigger Surfaces

Always run this review when the diff touches any of:

- Authentication or session management
- Input parsing, validation, or sanitization
- SQL queries or ORM calls
- File system reads/writes
- External HTTP requests or webhook handlers
- Environment variables, secrets, or credentials
- Permission checks or authorization logic
- Cryptography (hashing, signing, encryption)
- Dependency additions or version changes

**When NOT to use:** pure refactoring with no security surface changes, documentation-only changes, or test file changes (unless testing security features).

## The Eight Checks

Work through each in order. Skip none. If a check has no findings, write "Clean" -- don't omit it. See [checks-and-patterns.md](./checks-and-patterns.md) for detailed sub-checks per category.

### 1. Injection
SQL, command, template, and path traversal with user-controlled input.

### 2. Authentication & Session
Hardcoded credentials, insecure token storage, weak comparison, JWT validation gaps.

### 3. Access Control
Endpoint auth middleware, object-level ownership, function-level guards, IDOR.

### 4. Sensitive Data Exposure
Logged secrets/PII, verbose error responses, unnecessary fields in API output, HTTPS enforcement.

### 5. Security Misconfiguration
CORS wildcards on auth endpoints, missing CSP, debug mode in production, dependency CVEs.

### 6. Cryptography
Weak hashes for security, proper password hashing (bcrypt/argon2/scrypt), `secrets.token_hex(16)` not `Math.random()`, IV/nonce reuse.

### 7. Third-Party Dependencies
Maintenance status, version pinning, postinstall scripts.

### 8. Business Logic
Step-skipping via direct API calls, rate limiting on sensitive endpoints, mass assignment.

## Output Format

```
## Security Review

**Surfaces reviewed:** [list what was touched]

### Critical (exploit without auth or with normal user access)
- [location]: [vulnerability] -- [impact]

### High (requires auth or specific conditions to exploit)
- [location]: [vulnerability] -- [impact]

### Medium (defense-in-depth, hardening)
- [location]: [finding]

### Checks with no findings
- Injection: Clean
- [etc.]
```

Findings should state impact. Remediation guidance is optional — include it only in the escalation section for Critical items.

## Common Mistakes

- Reviewing only the diff without reading surrounding unchanged code for context
- Assuming framework defaults are secure without verifying the active configuration
- Missing logic bugs because you only pattern-matched known vulnerability types
- False confidence from passing automated tools -- they miss business logic flaws

## When to Escalate

Flag for human security review (not just report in findings) when:
- Auth bypass is possible with no credentials
- Secrets are committed to git history
- SQL injection exists on a publicly accessible endpoint
- A dependency has a CVE with a CVSS score >= 8.0
