# Bimsalabim

Claude Code skills library by [LinkIT360](https://linkit360.com) — reusable skills for code quality checks, security analysis, and clean architecture scaffolding.

## Skills

| Skill | Description |
|---|---|
| `checking-code-coverage` | Measure unit test coverage (Go, Node.js, Python, Java) |
| `checking-code-duplication` | Detect duplicate code blocks and report duplication % |
| `checking-vulnerabilities` | Scan for security vulnerabilities (OWASP Top 10) |
| `checking-security-hotspot` | Identify security-sensitive code requiring manual review |
| `refactoring-based-quality-gate` | Run all four checks, refactor to meet thresholds, report before/after |
| `scaffolding-backend-clean-architecture` | Generate clean architecture folder structure for backend services |

## Quality Gate Thresholds

| Metric | Threshold |
|---|---|
| Code Duplication | < 1% |
| Code Coverage | > 80% |
| Vulnerabilities | 0 (any severity) |
| Security Hotspots | 100% reviewed |

## Installation

Add this plugin to your Claude Code project:

```bash
claude plugin add https://github.com/RnDLinkit360/bimsalabim
```

## Usage

Invoke skills by describing what you want in Claude Code:

- `"Check code coverage for this project"`
- `"Find duplicate code"`
- `"Scan for security vulnerabilities"`
- `"Review security hotspots"`
- `"Refactor to meet quality gates"`
- `"Scaffold clean architecture for a new service"`

## Plugin Structure

```
plugins/<name>/
├── .claude-plugin/
│   └── plugin.json     # metadata
└── skills/
    └── SKILL.md        # skill definition
```

## License

MIT — Ghoffar Setiawan, LinkIT360
