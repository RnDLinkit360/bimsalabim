---
name: refactoring-based-quality-gate
description: Use when user asks to refactor code to meet quality gates, fix all quality findings, improve code quality to pass thresholds, or resolve duplication/coverage/vulnerability/hotspot issues. 
Triggers on: "refactor quality gate", "fix quality issues", "meet quality thresholds", "quality gate refactoring", "resolve all findings", "fix duplication coverage vulnerabilities hotspots".
---

# Refactoring Based Quality Gate

## Overview

Run all four quality checks, refactor code to resolve every finding, re-run checks to verify thresholds met, then produce a before/after summary report.

## Quality Gate Thresholds

| Metric | Threshold | Pass Condition |
|--------|-----------|----------------|
| Code Duplication | < 1% | duplication percentage below 1% |
| Code Coverage | > 80% | total coverage above 80% |
| Vulnerabilities | 0 | zero open vulnerabilities (any severity) |
| Security Hotspots | 100% reviewed | all hotspots have Fix, Accept, or False Positive decision |

## Phase 1 — Baseline (BEFORE)

Run all four checks and record results. Do NOT skip any — all four are required.

### Step 1.1 — Code Duplication Baseline
Use skill `linkit360-checking-code-duplication`. Record:
- Duplication percentage
- All duplicate blocks with file locations

### Step 1.2 — Code Coverage Baseline
Use skill `linkit360-checking-code-coverage`. Record:
- Total coverage percentage
- Per-package/file breakdown (files below 80% are targets)

### Step 1.3 — Vulnerability Baseline
Use skill `linkit360-checking-vulnerabilities`. Record:
- Total count by severity (Critical / High / Medium / Low)
- Each finding with file:line and category

### Step 1.4 — Security Hotspot Baseline
Use skill `linkit360-checking-security-hotspot`. Record:
- Total hotspot count by sensitivity
- Each hotspot with file:line and pending decision

## Phase 2 — Refactoring

Address findings in priority order: Vulnerabilities → Security Hotspots → Duplication → Coverage.

### 2.1 — Fix Vulnerabilities (target: 0)

For each vulnerability finding:
1. Locate the vulnerable code at the reported file:line
2. Apply the recommended fix from the vulnerability report
3. Verify fix removes the pattern (re-read the file)
4. Do not suppress/ignore — fix root cause

Common fixes:
- SQL injection → parameterized queries
- Hardcoded secrets → environment variables
- Weak crypto → modern algorithms (bcrypt, argon2, AES-256)
- Shell injection → sanitized inputs or safe APIs
- XSS → proper escaping/encoding

### 2.2 — Resolve Security Hotspots (target: 100% reviewed)

For each hotspot, make an explicit decision:

| Decision | When | Action |
|----------|------|--------|
| **Fix** | Code confirmed security-sensitive and protection missing | Apply fix, add code comment explaining the change |
| **Accept** | Risk acceptable given context (infra handles it, non-security use) | Add inline comment: `// SECURITY-REVIEWED: <reason>` |
| **False Positive** | Pattern triggered scan but no security relevance | Add inline comment: `// SECURITY-FP: <reason>` |

Every hotspot must have one of these three decisions — no "Pending" allowed.

### 2.3 — Reduce Code Duplication (target: < 1%)

For each duplicate block:
1. Identify all locations where block appears
2. Extract into a shared function/module/utility
3. Replace all occurrences with the extracted abstraction
4. Ensure extracted code is properly named and located

Extraction strategies:
- Repeated utility logic → shared helper file
- Repeated class methods → base class or mixin
- Repeated configuration → constants file
- Repeated test setup → test fixtures/factories

Do NOT extract if:
- Block is trivially short (< 3 meaningful lines)
- Contexts differ enough that extraction would require many params
- Duplication is in test files with distinct intent

### 2.4 — Increase Code Coverage (target: > 80%)

Identify files/packages below 80% from baseline report. For each:

1. Read the uncovered file
2. Identify untested functions, branches, edge cases
3. Write unit tests targeting the gaps
4. Use language-appropriate test framework (Jest, pytest, Go test, JUnit)
5. Run coverage again per-file to confirm improvement

Test writing rules:
- Test each public function with at least one happy path + one error path
- Test boundary conditions and null/empty inputs
- Mock external dependencies (DB, HTTP, file I/O) — test logic, not infrastructure
- Do not write trivial tests just to inflate numbers — cover real logic paths

## Phase 3 — Verification (AFTER)

Re-run all four checks using the same skills as Phase 1.

### Step 3.1 — Re-check Duplication
Use skill `linkit360-checking-code-duplication`. Record new percentage.

### Step 3.2 — Re-check Coverage
Use skill `linkit360-checking-code-coverage`. Record new percentage.

### Step 3.3 — Re-check Vulnerabilities
Use skill `linkit360-checking-vulnerabilities`. Record remaining count.

### Step 3.4 — Re-check Security Hotspots
Use skill `linkit360-checking-security-hotspot`. Confirm all hotspots have decisions.

If any threshold still not met → repeat Phase 2 for that metric only, then re-verify.

## Phase 4 — Summary Report

Output this report after Phase 3 passes all thresholds.

```
## Quality Gate Report

**Date:** YYYY-MM-DD
**Project:** <project name>

---

## Results Summary

| Metric | Before | After | Threshold | Status |
|--------|--------|-------|-----------|--------|
| Code Duplication | X.X% | X.X% | < 1% | PASS / FAIL |
| Code Coverage | X.X% | X.X% | > 80% | PASS / FAIL |
| Vulnerabilities | X | 0 | 0 | PASS / FAIL |
| Security Hotspots | X pending | X reviewed | 100% reviewed | PASS / FAIL |

**Overall:** PASS / FAIL

---

## Duplication Changes

**Before:** X.X% — Y duplicate lines / Z total
**After:** X.X% — Y duplicate lines / Z total

Extractions made:
- `<function/module name>` extracted from: file1.ts:12-24, file2.ts:88-100
- ...

---

## Coverage Changes

**Before:** X.X%
**After:** X.X%

Tests added:
- `<test file>` — covers `<module/function>` (added N test cases)
- ...

Files still below 80% (if any):
- `<file>`: X.X% → X.X% (needs further work)

---

## Vulnerabilities Fixed

**Before:** X total (Critical: X | High: X | Medium: X | Low: X)
**After:** 0

Fixed:
- [VULN-001] SQL Injection at userRepo.ts:42 → parameterized query
- ...

---

## Security Hotspot Decisions

**Before:** X hotspots pending review
**After:** X reviewed (Fix: X | Accept: X | False Positive: X)

Decisions:
- [HS-001] Weak hash at userService.ts:88 → Fix (replaced MD5 with bcrypt)
- [HS-002] Cookie flag at session.ts:34 → Accept (HTTPS enforced at infra)
- ...

---

## Quality Gate: PASS ✓ / FAIL ✗
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Skip baseline — jump straight to refactor | Always run Phase 1 first — need before numbers for report |
| Fix vulnerability symptom not root cause | Trace the input source, fix the pattern not just the instance |
| Extract duplicate code blindly | Check if contexts differ — forced extraction creates wrong abstractions |
| Write tests just to hit 80% | Cover real logic paths — trivial tests rot and give false confidence |
| Mark hotspot "Accept" without comment | Always add `// SECURITY-REVIEWED: <reason>` inline |
| Report PASS before re-running checks | Phase 3 is mandatory — refactoring can introduce new issues |
| Run checks in wrong order | Priority: Vulns → Hotspots → Duplication → Coverage |
