---
name: checking-code-duplication
description: Use when user asks to check, analyze, detect, or report code duplication, duplicate lines, repeated code, or copy-paste issues in a project or file. Triggers on: "check duplication", "find duplicate code", "code duplication report", "duplicate lines", "copy paste detection".
---

# Checking Code Duplication

## Overview

Analyze source files for duplicate code blocks. Produce a **duplication percentage** and a **line-by-line duplicate report**. Target: exact line duplicates and repeated multi-line blocks (2+ consecutive identical lines shared across ≥2 locations).

## Algorithm

### Step 1 — Collect files

Use `Glob` to list all source files in scope. Exclude: `node_modules`, `dist`, `build`, `.git`, lock files, generated files.

Default extensions: `.ts`, `.tsx`, `.js`, `.jsx`, `.py`, `.go`, `.java`, `.cs`, `.php`, `.rb`

### Step 2 — Extract non-trivial lines

Read each file. For each line, normalize it:
- Trim leading/trailing whitespace
- Skip blank lines
- Skip single-symbol lines: `{`, `}`, `(`, `)`, `;`, `//`, `#`, `*`
- Skip import/require statements
- Skip lines shorter than 10 characters after trim

Store as: `Map<normalizedLine, Array<{file, lineNumber, original}>>`.

### Step 3 — Find duplicates

A line is **duplicate** if it appears in the map with **count ≥ 2** (across same or different files).

For block detection: find sequences of 3+ consecutive normalized lines that repeat as a block elsewhere. Flag the entire block.

### Step 4 — Compute percentage

```
totalNonTrivialLines = count of all lines that passed Step 2 filter
duplicateLines       = count of lines that are duplicates (each occurrence counted)
duplicationPct       = (duplicateLines / totalNonTrivialLines) * 100
```

### Step 5 — Report

Output structured report (see format below).

## Output Format

```
## Code Duplication Report

**Duplication: XX.X%** (N duplicate lines / M total non-trivial lines)

### Duplicate Blocks

#### Block 1 — X occurrences
Duplicate content:
```
<lines>
```
Found at:
- `path/to/file.ts` lines 12–14
- `path/to/other.ts` lines 88–90

#### Block 2 — X occurrences
...

### Single-Line Duplicates (top 10 by frequency)

| Line | Content | Occurrences | Locations |
|------|---------|-------------|-----------|
| — | `someCode()` | 5 | file.ts:10, file.ts:44, other.ts:7, ... |
```

## Severity Scale

| Percentage | Severity |
|------------|----------|
| 0–5% | Low — acceptable |
| 5–15% | Medium — review candidates for extraction |
| 15–30% | High — refactor recommended |
| >30% | Critical — significant DRY violations |

## Implementation Notes

- Use `Bash` with PowerShell or grep to count lines efficiently on large codebases
- For large projects (>200 files), process per-directory and aggregate
- When running on single file: compare against rest of codebase
- Block duplicates take priority over single-line duplicates in the report

## Example Invocation

User: "check code duplication in src/"

1. Glob `src/**/*.ts` (and other extensions)
2. Read each file, build normalized line map
3. Identify duplicates
4. Compute percentage
5. Output report with blocks first, then single-line top-10
