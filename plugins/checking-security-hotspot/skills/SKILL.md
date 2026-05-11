---
name: checking-security-hotspot
description: Use when user asks to check, identify, or report security hotspots in code — security-sensitive areas that need developer review to determine if a fix is required. Triggers on: "security hotspot", "security-sensitive code", "review security", "hotspot analysis", "SonarQube hotspot", "RSPEC hotspot", "security review needed", "sensitive code review".
---

# Checking Security Hotspots

## Overview

Security hotspots are security-sensitive pieces of code that **require developer review** before deciding if a fix is needed. Unlike vulnerabilities (must fix immediately), hotspots may or may not need remediation depending on context, design intent, and threat model.

**Key distinction:**

| | Hotspot | Vulnerability |
|-|---------|---------------|
| **Definition** | Security-sensitive code needing review | Confirmed security problem |
| **Impact** | May or may not affect security | Directly impacts security |
| **Action** | Review → decide → fix or accept | Fix immediately |
| **Example** | Cookie without `Secure` flag | SQL injection with user input |

## Steps

1. **Scan code** — identify security-sensitive patterns
2. **Categorize** — assign hotspot category and risk level
3. **Provide context** — explain why it's sensitive and what questions to ask
4. **Developer decides** — fix, accept with justification, or mark false positive

## Hotspot Categories

### Cryptography
- Weak algorithms: `MD5`, `SHA1`, `DES`, `RC4`
- Hard-coded IV or salt
- Small key sizes (RSA < 2048, AES < 128)
- `Math.random()` / `rand()` for security-sensitive values
- **Review question:** Is strong crypto required here, or is this non-security use?

### Authentication & Sessions
- Session tokens not using `crypto/rand` or equivalent
- Cookies missing `Secure`, `HttpOnly`, or `SameSite` flags
- Long-lived tokens without expiry
- Password hashing with fast algorithms (MD5, SHA256 without salt)
- **Review question:** What is the threat model? Is HTTPS enforced at infrastructure level?

### Authorization & Access Control
- Role checks done client-side only
- Hardcoded user IDs or roles in logic
- `isAdmin` flag from user-controlled input
- **Review question:** Is server-side validation also present?

### Input Handling
- `eval()`, `new Function()`, `exec()` with variables (not user input confirmed)
- Dynamic regex from config or DB
- `innerHTML`, `document.write()` with non-user data
- **Review question:** Can this value ever reach user input?

### Data Exposure
- Logging objects that may contain PII or secrets
- Verbose error messages returned to client
- Debug endpoints left in production code
- Sensitive fields not excluded from serialization
- **Review question:** What environments does this run in? Is data classified?

### Network & Transport
- HTTP URLs in config (not HTTPS)
- TLS certificate validation disabled
- `CORS` wildcard `*` on non-public endpoints
- **Review question:** Is this internal-only? Is TLS terminated at load balancer?

### File & Path Operations
- Path constructed from variables (not user input confirmed)
- File permissions set broadly (`0777`, `chmod 777`)
- Temp files in world-writable directories
- **Review question:** Can the path value ever be influenced externally?

### Deserialization
- `JSON.parse`, `pickle.loads`, `ObjectInputStream` on external data
- YAML loaded with full loader
- **Review question:** Is the source trusted? Is schema validation applied?

### Dependency & Configuration
- Outdated dependency version (no known CVE, but old)
- Debug/development config flags present
- Commented-out security checks
- **Review question:** Is this intentional for dev environment only?

## Output Format

```
## Security Hotspot Report

**Date:** YYYY-MM-DD
**Project:** <project name>
**Total Hotspots:** X (High Sensitivity: X | Medium: X | Low: X)
**Status:** Requires developer review — NOT confirmed vulnerabilities

---

### [HS-001] Weak Hashing Algorithm — High Sensitivity
**File:** src/auth/userService.ts:88
**Category:** Cryptography / CWE-327
**Code:**
```typescript
const hash = crypto.createHash('md5').update(password).digest('hex');
```
**Why sensitive:** MD5 is cryptographically broken and unsuitable for password hashing.
**Review questions:**
- Is this hash used for security (passwords, tokens) or non-security (deduplication, caching)?
- Is bcrypt/argon2/scrypt available in this stack?

**If security use → Fix:** Replace with `bcrypt` or `argon2`:
```typescript
import bcrypt from 'bcrypt';
const hash = await bcrypt.hash(password, 12);
```
**If non-security use → Accept:** Document why MD5 is acceptable here.

---

### [HS-002] Cookie Missing Secure Flag — Medium Sensitivity
**File:** src/middleware/session.ts:34
**Category:** Authentication / RSPEC-2092
**Code:**
```typescript
res.cookie('session', token, { httpOnly: true });
```
**Why sensitive:** Without `Secure` flag, cookie may be sent over non-HTTPS connections.
**Review questions:**
- Is HTTPS enforced at all network entry points (load balancer, CDN)?
- Is this cookie used on non-HTTPS subdomains intentionally (tracking, analytics)?

**If HTTPS not guaranteed → Fix:** Add `secure: true`:
```typescript
res.cookie('session', token, { httpOnly: true, secure: true, sameSite: 'strict' });
```
**If HTTPS enforced at infra level → Accept:** Document the infra-level enforcement.

---

## Hotspot Summary Table

| ID | Sensitivity | Category | File | Decision |
|----|-------------|----------|------|----------|
| HS-001 | High | Cryptography | userService.ts:88 | Pending Review |
| HS-002 | Medium | Authentication | session.ts:34 | Pending Review |

## Decision Guide

| Decision | When to use |
|----------|-------------|
| **Fix** | Code is confirmed security-sensitive and protection is missing |
| **Accept** | Context makes the risk acceptable (infra handles it, non-security use) |
| **False Positive** | Code pattern triggered scan but has no security relevance |
```

## Sensitivity Classification

| Sensitivity | Meaning |
|-------------|---------|
| **High** | Directly handles auth, secrets, crypto, or external input in sensitive context |
| **Medium** | Adjacent to security concerns, context determines risk |
| **Low** | Peripheral patterns, best-practice deviations with minimal direct risk |

## Hotspot vs Vulnerability Decision Tree

```
Is there confirmed user-controlled input reaching the sensitive operation?
├── YES → Vulnerability (fix immediately)
└── NO → Is the operation inherently security-critical?
    ├── YES → Hotspot (review required)
    └── NO → Informational (document, low priority)
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Treating all hotspots as vulnerabilities | Review context first — hotspots need judgment, not auto-fix |
| Accepting hotspots without justification | Always document the reason for acceptance |
| Missing the "review questions" step | Questions guide the developer to the right decision |
| Fixing without understanding intent | Some patterns are intentional (tracking cookies, debug builds) |
| No follow-up after acceptance | Schedule periodic re-review — context changes over time |
