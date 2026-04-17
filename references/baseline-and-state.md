# Baseline & State Rules

Most teams adopting a code review tool drown in findings from **pre-existing** code. The skill must support:

1. **Baseline mode** — snapshot accepted findings, only surface new regressions
2. **Per-line ignores** — mark specific lines as intentional

This turns the skill from a one-shot audit into a **continuous drift monitor**.

---

## `.codereview-baseline.json`

Lives at the repo root. Contains a snapshot of findings that have been reviewed and accepted.

### Schema

```json
{
  "version": 1,
  "created_at": "2026-04-17T10:30:00Z",
  "updated_at": "2026-04-17T10:30:00Z",
  "commit_sha": "a3f92c1b...",
  "skill_version": "2.0.0",
  "findings": [
    {
      "id": "a1b2c3d4",
      "file": "src/legacy/db.py",
      "line": 42,
      "severity": "critical",
      "category": "sql-injection",
      "detected_by": ["bandit", "claude"],
      "rule": "bandit:B608",
      "content_hash": "sha256-of-surrounding-5-lines",
      "acknowledged": "Legacy code. Migration tracked in JIRA-1234. Target: Q3 2026.",
      "acknowledged_by": "gabriel.bh@example.com",
      "acknowledged_at": "2026-04-17T10:30:00Z"
    }
  ]
}
```

### Field semantics

| Field | Purpose |
|---|---|
| `id` | Short deterministic hash of `{file, line, category, rule}` — survives reformats |
| `file` + `line` | Location at time of snapshot |
| `severity` | Matches skill's tier (critical/warning/suggestion) |
| `category` | Finding category (e.g., `sql-injection`, `hotspot`, `cve`) |
| `detected_by` | Tools that flagged this |
| `rule` | Tool-specific rule ID if applicable |
| `content_hash` | Hash of surrounding ~5 lines — used to detect if the flagged code moved but is still present |
| `acknowledged` | Human explanation — required; empty means auto-accepted (flag for review) |
| `acknowledged_by` / `acknowledged_at` | Audit trail |

### Matching logic

When a new audit runs, match each finding against the baseline using this priority:

1. **Exact match**: same `(file, line, category, rule)` → suppress (was in baseline)
2. **Content-hash match**: same `(file, category, rule)` + same `content_hash` but different line → suppress (code moved, still same finding)
3. **Fuzzy file match**: renamed file (detect via git rename detection) → ask to update baseline
4. **No match**: this is a **new** finding → surface it

### Surfacing diff

Report shows:

```
## Summary

| Severity | New | In baseline | Resolved |
|---|---|---|---|
| 🔴 Critical | 2 | 5 | 1 |
| 🟡 Warning | 4 | 18 | 3 |
| 🔵 Suggestion | 1 | 42 | 8 |
```

`New` = findings not in baseline (focus of this report)
`In baseline` = acknowledged/accepted (mentioned in summary only, not detailed)
`Resolved` = in baseline but no longer present (good signal — team is reducing debt)

---

## `/code-audit baseline update`

Command: when the user types this, re-snapshot the current set of findings as the new baseline.

### Process

1. Run a full audit (all dimensions, all tools)
2. Serialize findings into the schema above
3. Write to `.codereview-baseline.json`
4. Ask the user to fill in `acknowledged` fields (or accept with an empty default)
5. Advise committing the baseline file to the repo

### Initial baseline

For a fresh repo, the first baseline typically includes everything currently in the codebase. That's fine — the skill's goal is to **prevent regressions**, not to force an immediate cleanup.

### Updating the baseline

Should happen when:
- A finding is legitimately resolved (drop it from the baseline naturally on next `baseline update`)
- A new intentional pattern is introduced that would otherwise be flagged
- The skill is upgraded and emits new finding categories

---

## `.codereview-ignore`

For per-file or per-pattern exclusions. Uses glob-style patterns similar to `.gitignore`.

### Format

```
# Ignore entire files
generated/**
vendor/**
*_pb.go
*.min.js

# Ignore specific rules in specific files
src/legacy/db.py:bandit:B608
src/legacy/db.py:sql-injection

# Ignore a specific rule everywhere
*:stylelint:color-no-hex
```

### Syntax per line

```
<glob-file-pattern>[:<tool>:<rule-id>]
```

- No colon: ignore all findings in matching files
- One colon: ignore a specific category/rule in matching files
- Glob rules: `**` for recursive, `*` for single-segment

### Per-line ignore comments

Users can add inline suppression comments:

| Language | Syntax |
|---|---|
| Python | `# codereview: ignore` at end of line |
| JS/TS | `// codereview: ignore` |
| Go | `// codereview: ignore` |
| Rust | `// codereview: ignore` |
| Ruby | `# codereview: ignore` |
| Shell | `# codereview: ignore` |

Specific rule suppression:

```python
password = "hunter2"  # codereview: ignore bandit:B105  (test fixture)
```

**Always require a reason** after the rule — no reason = still flag. This prevents silent muting.

---

## Interaction with tools

When running tools, pass through their own ignore mechanisms:

- ESLint: respect `.eslintrc` `overrides` + `// eslint-disable-next-line`
- ruff: respect `[tool.ruff.per-file-ignores]` + `# noqa`
- Semgrep: respect `# nosemgrep` + `semgrepignore` files
- Bandit: respect `# nosec` + `.bandit` config

This means the tool's own suppressions are **already applied** before the skill sees the output. The skill's `.codereview-ignore` is an additional layer for suppressions **across tools** (or for Claude-reasoning findings that don't have a tool-specific suppression).

---

## Suggested `.gitignore` entries

`.codereview-baseline.json` **should be committed** (team shares the baseline).
`.codereview-ignore` **should be committed**.

Add to `.gitignore`:
```
# Not the baseline file — that should be committed
# But temp state goes here:
.codereview-cache/
.codereview-last-run.json
```

---

## State cache (optional)

For large codebases, Tool output can be cached in `.codereview-cache/` keyed by content hash:

```
.codereview-cache/
├── ruff/
│   └── <content-hash>.json
├── semgrep/
│   └── <content-hash>.json
└── osv/
    └── <lockfile-hash>.json
```

Cache entries expire after 24 hours. This is an optimization, not required — if the cache directory doesn't exist, the skill just runs tools fresh every time.
