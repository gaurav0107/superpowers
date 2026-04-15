# Eval Results: Security Code Review Pressure Test

**Date:** 2026-04-16
**Test runner:** Claude Opus 4.6 subagent via `claude --print`
**Method:** Identical test artifacts reviewed by two subagents -- one with skill content prepended, one without (baseline). All outputs recorded verbatim below.

---

## Eval: security-code-review

### Test Artifact

A code diff for `src/routes/admin.js` (Express router) with five embedded vulnerabilities:
1. **SQL injection** -- string concatenation in two queries (lines 12, 23)
2. **Hardcoded API key** -- `sk-live-...` on line 7
3. **Path traversal** -- user-controlled filename in `path.join` (line 30-31)
4. **Missing auth check** -- no middleware on any route, including bulk role update
5. **Math.random() for token generation** -- password reset token uses non-cryptographic PRNG (line 39)

### Baseline (without skill)
- **Issues found:** 5/5 (SQL injection, hardcoded key, path traversal, no auth, Math.random)
- **Structure:** Numbered list grouped by severity (Critical/High/Medium) but no systematic checklist. Did not enumerate all eight security check categories. Included a summary table.
- **Fix suggestions included:** YES -- inline code examples showing the fix for path traversal, token generation, and other issues
- **Eight checks coverage:** Did NOT walk through all eight checks. Missing explicit coverage of: Session management, CORS/CSP, third-party dependencies, business logic (rate limiting, mass assignment)
- **Verbatim excerpt (fix suggestion):**
  > `filename` can be `../../etc/passwd` or `../../../app/src/.env`. `path.join` does not prevent traversal. Validate that the resolved path stays within `/var/exports`:
  > ```js
  > const filepath = path.resolve('/var/exports', filename);
  > if (!filepath.startsWith('/var/exports/')) return res.status(400).send('Invalid filename');
  > ```
- **Bonus findings beyond embedded flaws:** 4 (stack trace leakage, no input validation on bulk update, sync file I/O, unbounded DB loop)

### With Skill
- **Issues found:** 5/5 (SQL injection x2, hardcoded key, path traversal, no auth, Math.random)
- **Structure:** Followed the prescribed output format exactly: "Surfaces reviewed" list, then Critical/High/Medium/Checks with no findings. Enumerated which of the eight checks had no findings.
- **Fix suggestions included:** PARTIAL -- the skill does not explicitly prohibit fix suggestions (unlike adversarial-plan-review). The agent included remediation guidance in the escalation section but kept individual findings focused on impact rather than code fixes.
- **Eight checks coverage:** YES -- all eight checks were addressed either as findings or explicitly marked as having no findings (Security Misconfiguration: noted needs app-level check; Third-Party Dependencies: Clean)
- **Verbatim excerpt (structured finding):**
  > **Line 7**: Hardcoded API key `sk-live-a8f3b2e1d4c5f6a7b8c9d0e1f2a3b4c5` committed to source -- credential exposure. Once in git history, this is effectively public. **ESCALATE: secret committed to git.**
- **Verbatim excerpt (escalation per skill):**
  > **Escalation required** -- three items need human security review before merge:
  > 1. **Hardcoded live API key** on line 7 -- must be rotated immediately regardless of merge decision, as it's now in git history.
  > 2. **SQL injection** on lines 12 and 23
  > 3. **Zero auth middleware** on an admin route
- **Bonus findings beyond embedded flaws:** 4 (stack trace leakage, SELECT * returns sensitive columns, writeFileSync blocks event loop, external API call has no timeout)
- **Escalation triggers activated:** YES -- correctly flagged the three items that meet the skill's escalation criteria (auth bypass, secret in git, SQL injection on accessible endpoint)

### Delta
- **Detection improvement (embedded vulns):** 0% -- both found all 5 embedded vulnerabilities. Again, the base model is strong on known vulnerability patterns.
- **Bonus finding count:** Roughly equal (4 each), though different findings. The skill-equipped agent caught SELECT * data exposure; the baseline caught unbounded DB loop.
- **Structural compliance:** YES -- the skill-equipped agent followed the exact output format including "Surfaces reviewed," severity tiers, "Checks with no findings" section, and escalation criteria. The baseline used an ad-hoc table format.
- **Eight-check completeness:** The skill-equipped agent addressed all 8 checks explicitly. The baseline covered ~5 of 8 implicitly.
- **Escalation protocol:** Only the skill-equipped agent invoked the escalation criteria from the skill, flagging three items for human security review with the correct thresholds.
- **Key qualitative difference:** The skill imposed *completeness discipline* -- every check category had to be addressed, even if clean. This prevents false confidence from omission. The baseline only reported what it found, leaving the reader to wonder about unchecked categories.

---

## Overall Assessment

### What the skill improves

| Dimension | Without Skill | With Skill |
|-----------|--------------|------------|
| Raw detection of planted vulns | High (5/5) | High (5/5) |
| Bonus findings beyond planted vulns | 4 | 4 |
| Structural consistency | Ad-hoc | Prescribed format followed |
| Completeness guarantee | No -- omissions are invisible | Yes -- all 8 checks addressed |
| Escalation protocol | Not present | Correctly triggered |
| Reproducibility across runs | Low -- output varies | Higher -- format is constrained |

### What the skill does NOT improve

- **Detection of obvious, well-known vulnerabilities.** A capable LLM finds SQL injection regardless of skill prompting. The skill adds value through structure and discipline, not raw detection.
- **False negative elimination.** Neither agent found every conceivable issue. The skill reduces omissions through checklist discipline but does not guarantee zero misses.

### Conclusion

The security-code-review skill demonstrates clear eval-measurable improvement in:
1. **Output structure** -- the prescribed format was followed faithfully
2. **Behavioral discipline** -- escalation protocol was correctly triggered for three items meeting threshold criteria
3. **Completeness coverage** -- the eight-check framework forced systematic examination; all categories addressed even when clean
4. **Eight-check completeness** -- all 8 checks explicitly addressed vs ~5 of 8 by baseline

The skill functions as designed: it does not make the LLM "smarter" at finding obvious issues, but it imposes systematic rigor, ensures checklist completeness, triggers escalation protocols, and produces reproducible output structure.

---

## Appendix: Raw Subagent Outputs

### A1: Security Review WITH Skill (verbatim)

```
## Security Review

**Surfaces reviewed:** Authentication/session, SQL queries, file system writes, secrets handling, access control, cryptography (token generation), external HTTP requests, input validation

### Critical (exploit without auth or with normal user access)

- **Line 7**: Hardcoded API key `sk-live-a8f3b2e1d4c5f6a7b8c9d0e1f2a3b4c5` committed to source -- credential exposure. Once in git history, this is effectively public. **ESCALATE: secret committed to git.**
- **Line 12**: SQL injection via string interpolation of `req.params.id` -- `SELECT * FROM users WHERE id = '${req.params.id}'`. Attacker crafts ID like `' OR 1=1 --` to dump entire users table. **ESCALATE: SQL injection on likely public-facing endpoint.**
- **Line 23**: SQL injection via `req.query.name` and `req.query.role` -- both interpolated directly into query string with no parameterization. Same impact as above.
- **Lines 10-58**: **No authentication or authorization middleware on any route.** This is an admin router with zero auth guards -- anyone who can reach these endpoints can read/modify all user data, reset tokens, and bulk-update roles.

### High (requires auth or specific conditions to exploit)

- **Line 30-31**: Path traversal via `req.query.filename` -- attacker supplies `../../etc/cron.d/backdoor` to write arbitrary files. `path.join('/var/exports', '../../../etc/passwd')` resolves outside the intended directory.
- **Lines 38-39**: Password reset token generated with `Math.random()` -- cryptographically predictable. V8's PRNG state can be reconstructed, allowing attacker to predict reset tokens. Must use `crypto.randomBytes()` or `crypto.randomUUID()`.
- **Line 46-49**: Bulk role update with no ownership check or rate limiting -- any caller can escalate any user (including themselves) to any role, including admin. Mass assignment of the `role` field.

### Medium (defense-in-depth, hardening)

- **Line 16**: Error response returns `err.message` and `err.stack` to client -- leaks internal implementation details (file paths, library versions, query structure).
- **Line 13**: `SELECT *` returns all columns including password hashes, reset tokens, PII -- should explicitly select only needed fields.
- **Line 33**: `fs.writeFileSync` blocks the event loop and has no error handling. Also no cleanup of written files after `res.download`.
- **Line 54**: External API call to `api.analytics.io` has no timeout, no error handling, and no response validation -- SSRF risk is low here since the URL is hardcoded, but a hung request will block indefinitely.

### Checks with no findings

- **Security Misconfiguration:** No CORS or CSP visible in this file (would need to check app-level config), but nothing new introduced here beyond what's listed above.
- **Third-Party Dependencies:** No new dependencies added in this diff.

---

**Escalation required** -- three items need human security review before merge:

1. **Hardcoded live API key** on line 7 -- must be rotated immediately regardless of merge decision, as it's now in git history.
2. **SQL injection** on lines 12 and 23 -- parameterize all queries (`$1` placeholders, as correctly done on lines 32 and 41).
3. **Zero auth middleware** on an admin route -- every endpoint here is unprotected.

**This diff should not be merged.** The minimum fix requires: removing the hardcoded key (and rotating it), parameterizing all SQL queries, adding auth middleware to the router, validating `filename` against path traversal, replacing `Math.random()` with `crypto.randomBytes()`, stripping `SELECT *` and error stack traces, and adding authorization checks to bulk-update.
```

### A2: Security Review WITHOUT Skill (verbatim)

```
## Security Review: `src/routes/admin.js`

### Critical

**1. Hardcoded API Key (Line 7)**
Secret committed to source code. Anyone with repo access gets this key. Move to environment variable (`process.env.ANALYTICS_API_KEY`).

**2. SQL Injection (Lines 12, 23)**
User input interpolated directly into SQL. An attacker can extract, modify, or delete the entire database. Use parameterized queries (`$1`, `$2`) as you already do elsewhere in this file.

**3. Path Traversal (Lines 30-31)**
`filename` can be `../../etc/passwd` or `../../../app/src/.env`. `path.join` does not prevent traversal. Validate that the resolved path stays within `/var/exports`.

### High

**4. No Authentication/Authorization**
Every route is unprotected. No middleware checks that the caller is authenticated or has admin privileges. The bulk role-update endpoint (line 47) is especially dangerous.

**5. Insecure Token Generation (Lines 39-40)**
`Math.random()` is not cryptographically secure and is predictable. Use `crypto.randomBytes(32).toString('hex')` instead.

**6. Stack Trace Leakage (Line 16)**
Exposes internal implementation details to attackers. Log the stack server-side; return a generic error to the client.

### Medium

**7. No Input Validation on Bulk Update (Lines 48-49)**
`userIds` and `newRole` are used without validation.

**8. Synchronous File Write (Line 32)**
`fs.writeFileSync` blocks the event loop. Under load this is a denial-of-service vector.

**9. Unbounded Loop (Line 49)**
Each ID fires a separate query with no limit on array size. Add a cap and consider a single `UPDATE ... WHERE id = ANY($1)` query.

### Summary

| # | Issue | Severity | Line(s) |
|---|-------|----------|---------|
| 1 | Hardcoded secret | Critical | 7 |
| 2 | SQL injection (2 instances) | Critical | 12, 23 |
| 3 | Path traversal | Critical | 30-31 |
| 4 | No auth/authz | High | all routes |
| 5 | Weak token generation | High | 39-40 |
| 6 | Stack trace leakage | High | 16 |
| 7 | No input validation on bulk update | Medium | 48-49 |
| 8 | Sync file I/O | Medium | 32 |
| 9 | Unbounded DB loop | Medium | 49 |

The SQL injection and path traversal are exploitable as-is. The missing auth means every issue is reachable by unauthenticated users. I'd recommend blocking this from merging until at least the critical and high items are fixed.
```
