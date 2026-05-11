---
name: linkit360-checking-vulnerabilities
description: Use when user asks to check, scan, analyze, audit, or report security vulnerabilities in code. Triggers on: "check vulnerabilities", "security audit", "find vulnerabilities", "security scan", "vulnerable code", "security issues", "OWASP", "CVE", "injection", "XSS", "SQL injection", "security report".
---

# Checking Vulnerabilities

## Overview

Analyze code for security vulnerabilities, classify by severity, and produce a structured report with remediation recommendations for each finding.

## Steps

1. **Detect stack** — language, framework, dependencies
2. **Run automated scan** — use language-appropriate tool
3. **Manual review** — check OWASP Top 10 patterns in source
4. **Classify findings** — severity + CWE/OWASP category
5. **Generate report** — structured output with recommendations

## Automated Scan Commands

### Node.js / TypeScript
```bash
# Dependency vulnerabilities
npm audit --json

# Static analysis
npx snyk test
# or
npx semgrep --config=p/owasp-top-ten .
```

### Python
```bash
# Dependency vulnerabilities
pip install safety && safety check

# Static analysis
pip install bandit && bandit -r . -f json
```

### Go
```bash
# Dependency vulnerabilities
govulncheck ./...

# Static analysis
go install github.com/securego/gosec/v2/cmd/gosec@latest && gosec ./...
```

### Java (Maven)
```bash
mvn org.owasp:dependency-check-maven:check
```

### Java (Gradle)
```bash
./gradlew dependencyCheckAnalyze
```

### Docker / Container
```bash
trivy image <image-name>
trivy fs .
```

### Generic (Semgrep — works any language)
```bash
npx semgrep --config=p/owasp-top-ten --config=p/security-audit .
```

## OWASP Top 10 Manual Review Checklist

Scan source code for these patterns:

| # | Category | Look For |
|---|----------|----------|
| A01 | Broken Access Control | Missing auth checks, path traversal, IDOR |
| A02 | Cryptographic Failures | MD5/SHA1, hardcoded keys, plain-text secrets, HTTP not HTTPS |
| A03 | Injection | Raw SQL string concat, unsanitized shell exec, eval(), unescaped HTML |
| A04 | Insecure Design | No rate limiting, missing input validation, business logic flaws |
| A05 | Security Misconfiguration | Debug mode on, default creds, verbose error messages, open CORS |
| A06 | Vulnerable Components | Outdated packages, known CVEs in dependencies |
| A07 | Auth Failures | Weak passwords, no MFA, insecure session tokens, broken JWT |
| A08 | Data Integrity Failures | Unsigned updates, deserialization of untrusted data |
| A09 | Logging Failures | No audit logs, logging sensitive data (passwords, tokens) |
| A10 | SSRF | User-supplied URLs fetched server-side without validation |

## Severity Classification

| Severity | Description |
|----------|-------------|
| **Critical** | Remote code execution, auth bypass, data breach risk |
| **High** | Injection, privilege escalation, sensitive data exposure |
| **Medium** | CSRF, insecure defaults, information disclosure |
| **Low** | Verbose errors, weak config, minor info leaks |
| **Informational** | Best-practice deviations, no direct exploit path |

## Output Format

```
## Security Vulnerability Report

**Scan Date:** YYYY-MM-DD
**Project:** <project name>
**Total Findings:** X (Critical: X | High: X | Medium: X | Low: X)

---

### [VULN-001] SQL Injection — Critical
**File:** src/database/userRepo.ts:42
**Category:** OWASP A03 – Injection / CWE-89
**Description:** User input concatenated directly into SQL query. Attacker can manipulate query logic or extract data.
**Vulnerable Code:**
```sql
const query = `SELECT * FROM users WHERE id = ${userId}`;
```
**Recommendation:** Use parameterized queries / prepared statements:
```typescript
const query = 'SELECT * FROM users WHERE id = $1';
await db.query(query, [userId]);
```

---

### [VULN-002] Hardcoded Secret — High
**File:** config/app.ts:15
**Category:** OWASP A02 – Cryptographic Failures / CWE-798
**Description:** API key hardcoded in source. Exposed via version control.
**Vulnerable Code:**
```typescript
const apiKey = "sk-prod-abc123xyz";
```
**Recommendation:** Move to environment variable:
```typescript
const apiKey = process.env.API_KEY;
if (!apiKey) throw new Error('API_KEY not set');
```

---

## Summary Table

| ID | Severity | Category | File | Status |
|----|----------|----------|------|--------|
| VULN-001 | Critical | SQL Injection | userRepo.ts:42 | Open |
| VULN-002 | High | Hardcoded Secret | app.ts:15 | Open |

## Remediation Priority

1. Fix all Critical findings before any deployment
2. High findings: fix within current sprint
3. Medium: schedule in next sprint
4. Low/Info: track in backlog
```

## Common Vulnerability Patterns by Language

### Node.js / TypeScript
- `eval()` or `new Function()` with user input → Code injection
- `child_process.exec()` with template literals → Shell injection
- `res.send(req.query.x)` without sanitize → XSS
- `JWT` verified without algorithm check → JWT confusion attack
- `cors({ origin: '*' })` on auth endpoints → CORS misconfiguration

### Python
- `subprocess.call(shell=True)` with user input → Shell injection
- `pickle.loads()` on untrusted data → Deserialization RCE
- `yaml.load()` instead of `yaml.safe_load()` → Code execution
- `os.system(user_input)` → Command injection
- SQLAlchemy raw `text()` with f-strings → SQL injection

### Go
- `fmt.Sprintf` in SQL queries → SQL injection
- `os/exec.Command` with user data → Shell injection
- `http.Get(userInput)` without validation → SSRF
- `math/rand` for tokens → Weak randomness (use `crypto/rand`)

### Java
- `Runtime.exec()` with user input → Command injection
- `ObjectInputStream.readObject()` → Deserialization
- `String.format()` in SQL → SQL injection
- `MD5` / `SHA1` for passwords → Weak hashing

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Only run dep scan, skip static analysis | Both required — dep scan misses code-level vulns |
| Ignore Medium/Low findings | Medium can chain into Critical exploits |
| Fix symptom not root cause | Understand why vuln exists, fix the pattern not the instance |
| Report without file:line | Always include exact location for developer to act |
| No code example in recommendation | Always show vulnerable code AND fixed code side-by-side |
