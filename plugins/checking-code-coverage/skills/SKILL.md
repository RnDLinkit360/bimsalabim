---
name: linkit360-checking-code-coverage
description: Use when user asks to check, measure, run, or report unit test code coverage for a project or file. Triggers on: "check coverage", "run tests with coverage", "code coverage report", "test coverage", "coverage percentage", "how much code is covered".
---

# Checking Code Coverage

## Overview

Run unit tests with coverage enabled and produce a coverage report showing percentage of code covered by tests.

## Steps

1. **Detect language** — check for `go.mod`, `package.json`, `pyproject.toml`, `pom.xml`, `build.gradle`
2. **Run tests with coverage** — use language-appropriate command
3. **Parse output** — extract total coverage percentage
4. **Report** — structured output (see format below)

## Commands by Language

### Go
```bash
go test ./... -coverprofile=coverage.out -covermode=atomic
go tool cover -func=coverage.out
```
Summary only:
```bash
go test ./... -cover
```

### Node.js / TypeScript (Jest)
```bash
npx jest --coverage --coverageReporters=text
```

### Node.js / TypeScript (Vitest)
```bash
npx vitest run --coverage
```

### Python (pytest-cov)
```bash
pytest --cov=. --cov-report=term-missing
```

### Java (Maven + JaCoCo)
```bash
mvn test jacoco:report
cat target/site/jacoco/index.html | grep -o 'Total[^%]*%'
```

### Java (Gradle)
```bash
./gradlew test jacocoTestReport
cat build/reports/jacoco/test/html/index.html | grep -o 'Total[^%]*%'
```

## Output Format

```
## Code Coverage Report

**Coverage: XX.X%**

| Package / File | Coverage |
|----------------|----------|
| domain/entities | 85.0% |
| application/usecases | 72.3% |
| infrastructure/api | 61.0% |

**Total: XX.X%**
```

## Severity Scale

| Coverage | Status |
|----------|--------|
| ≥80% | Good — production ready |
| 60–79% | Acceptable — improve critical paths |
| 40–59% | Low — add tests for core logic |
| <40% | Critical — significant test gap |

## Go Example (Full Flow)

```bash
# Run and capture
go test ./... -coverprofile=coverage.out -covermode=atomic 2>&1

# Per-function breakdown
go tool cover -func=coverage.out

# HTML report (optional, opens browser)
go tool cover -html=coverage.out
```

Sample output:
```
domain/entities/user.go:     NewUser    100.0%
application/usecases/create.go: Execute  75.0%
...
total:                       (statements) 78.4%
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Coverage runs but no `coverage.out` found | Add `-coverprofile=coverage.out` flag |
| 0% on all packages | Run from project root, not subdirectory |
| Missing packages in report | Use `./...` not `.` for recursive |
| Go: vendor dir inflating results | Add `-mod=vendor` or exclude with `grep -v vendor` |
| Tests pass but coverage low | Integration tests don't count — need unit tests per package |
