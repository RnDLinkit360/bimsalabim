# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

**Bimsalabim** is a Claude Code plugin marketplace — a collection of reusable skills for LinkIT360's development workflows. No runtime code, no build step, no tests. Everything is Markdown skill definitions + JSON plugin manifests.

Repository: https://github.com/linkit360/bimsalabim

## Plugin Structure

Every plugin lives under `plugins/<name>/` and follows this layout:

```
plugins/<name>/
├── .claude-plugin/
│   └── plugin.json     # plugin metadata (name, description, version, author)
└── skills/
    └── SKILL.md        # full skill definition with frontmatter + instructions
```

Root `.claude-plugin/` holds the umbrella plugin manifest:
- `plugin.json` — root plugin metadata
- `marketplace.json` — lists all plugins with their `source` paths for dev marketplace registration

## Skills Inventory

| Skill name (frontmatter) | Trigger keywords |
|---|---|
| `linkit360-checking-code-coverage` | "check coverage", "run tests with coverage", "coverage report" |
| `linkit360-checking-code-duplication` | "check duplication", "find duplicate code", "copy paste detection" |
| `linkit360-checking-vulnerabilities` | "check vulnerabilities", "security audit", "OWASP", "SQL injection" |
| `linkit360-checking-security-hotspot` | "security hotspot", "sensitive code review", "hotspot analysis" |
| `linkit360-refactoring-based-quality-gate` | "refactor quality gate", "fix quality issues", "meet quality thresholds" |
| `scaffolding-backend-clean-architecture` | "clean architecture", "scaffold project", "create folder structure" |

## Quality Gate Thresholds

The `refactoring-based-quality-gate` skill enforces these across all four checks:

| Metric | Threshold |
|---|---|
| Code Duplication | < 1% |
| Code Coverage | > 80% |
| Vulnerabilities | 0 (any severity) |
| Security Hotspots | 100% reviewed |

Refactoring priority order: **Vulnerabilities → Security Hotspots → Duplication → Coverage**

## Clean Architecture Pattern

The scaffolding skill generates three layers with inward dependency flow only:

- **`domain/`** — entities + rules; zero framework dependencies
- **`application/`** — use cases + service interfaces (ports); depends on domain only
- **`infrastructure/`** — all DB, HTTP, cache, messaging adapters; depends on application

Supported infrastructure: PostgreSQL, FerretDB, Redis, YugabyteDB, Elasticsearch, AerospikeDB, message publishers/subscribers, Temporal workflows.

## Adding or Editing a Skill

1. Create `plugins/<name>/.claude-plugin/plugin.json` with `name`, `description`, `version`, `author`.
2. Create `plugins/<name>/skills/SKILL.md` with YAML frontmatter (`name`, `description`) followed by skill instructions.
3. Add entry to `.claude-plugin/marketplace.json` under `plugins[]` with `source` pointing to `./plugins/<name>`.

Skill frontmatter `description` field is used by Claude Code to match the skill — make it specific and include trigger phrases.
