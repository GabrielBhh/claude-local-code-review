# Convention Compliance Rules

Generic "best practices" are one thing. **Team decisions** are another. This file documents how to detect and cross-check the project's own stated conventions so the audit enforces team guardrails, not just Claude's defaults.

When a finding violates a project convention, **cite the source document and line** — this turns a generic style suggestion into "violates `docs/adr/0012-use-postgres.md`," which carries real authority.

---

## Files to detect and load

Read these files at the start of every audit (Step 1 of the SKILL.md pipeline):

### Agent / AI guidance
- `CLAUDE.md` — Claude-specific instructions (what to prefer, avoid, format)
- `AGENTS.md` — generic AI-agent instructions
- `.claude/instructions.md` — per-project Claude instructions

### Human-readable conventions
- `CONTRIBUTING.md` — how to contribute, code style, PR process
- `ARCHITECTURE.md` — how the system is structured
- `STYLE.md` / `STYLE_GUIDE.md` — style decisions
- `README.md` — project overview (scan for stated conventions)

### Architecture Decision Records (ADRs)
- `docs/adr/*.md` — numbered architectural decisions
- `doc/adr/*.md` — alternate location
- `adr/*.md` — repo-root location

### Tool configs (reveal enforced conventions)
- `.editorconfig` — whitespace, line endings
- `.prettierrc` / `.prettierrc.json` — JS/TS formatting
- `.eslintrc.*` — JS/TS linting rules (pay attention to custom rules)
- `ruff.toml` / `pyproject.toml` `[tool.ruff]` — Python linting
- `rubocop.yml` — Ruby linting
- `rustfmt.toml` — Rust formatting
- `gofmt`-related (implicit)
- `.golangci.yml` — Go linting configuration
- `stylelint.config.*` — CSS
- `biome.json` — Biome config

### CI / workflow files
- `.github/workflows/*.yml` — what CI enforces (tests, lint commands)
- `.gitlab-ci.yml`, `.circleci/config.yml`, `Jenkinsfile`
- `Makefile` / `justfile` / `package.json` scripts

---

## ADR parsing

ADRs typically follow a template with a status field:

```markdown
# ADR-012: Use PostgreSQL for primary datastore

**Status**: Accepted
**Date**: 2024-03-15

## Context
...

## Decision
We will use PostgreSQL 15+ for all primary data storage. MySQL is deprecated.

## Consequences
...
```

**Only enforce ADRs with `Status: Accepted`.** Skip:
- `Proposed` — not yet adopted
- `Deprecated` / `Superseded` — no longer binding (follow the superseding ADR instead)
- `Rejected` — explicitly decided against

### Extracting the rule

For each Accepted ADR, extract:
1. **What's required** (from the Decision section)
2. **What's forbidden** (if stated)
3. **The source path** (file + line) for citation

---

## Cross-referencing findings

When the audit surfaces a finding, check:

1. Does any loaded convention document mention the pattern?
2. If yes → tag the finding with the source: `violates ADR-012 (docs/adr/0012-use-postgres.md:L14)`
3. If the finding *contradicts* a convention (e.g., ADR says "use MySQL" but code uses PostgreSQL) → 🔴 Critical: "contradicts ADR-012"
4. If the finding *enforces* a convention that's being violated → severity stays but add the ADR cite

### Example mapping

| Convention rule | Finding type | Severity with convention cite |
|---|---|---|
| `CONTRIBUTING.md` says "no `print()` in production" | `print("debug")` found | 🟡 Warning (was 🔵) |
| `CLAUDE.md` says "prefer Decimal for money" | `float` used for `price` | 🔴 Critical (was 🟡) |
| ADR says "use `tokio` not `async-std`" | `async-std` imported in new file | 🔴 Critical — contradicts ADR |
| ADR says "all services use Prometheus metrics" | Service emits logs only, no metrics | 🟡 Warning |

**Rule of thumb**: a convention cite **bumps severity up one tier** from what Claude's default would assign, because team-sanctioned rules have a stronger stake than generic best practices.

---

## Inferring conventions from config files

Even without explicit prose, config files carry rules:

### `.eslintrc` says `"complexity": ["error", 10]`
→ Enforce cyclomatic complexity ≤ 10 for JS/TS.

### `ruff.toml` has `select = ["E", "F", "B", "S"]`
→ Run ruff with those rule selections. If ruff finds something, cite the project's own selection.

### `.editorconfig` says `indent_size = 2`
→ Flag inconsistent indentation (though formatters should auto-fix).

### `package.json` `"engines": { "node": ">=20" }`
→ Flag Node-specific APIs below 20 that are used but no version check exists.

---

## Reporting format

Add a dedicated "Convention violations" section in the report (only if any were found):

```
## Convention violations

### 🔴 Contradicts ADR-012 — Database choice
**`src/db/client.rs:5`**
ADR-012 requires PostgreSQL; this file imports `mysql_async`.
Source: `docs/adr/0012-use-postgres.md:L14`
**Fix**: use `sqlx-postgres` instead.

### 🟡 Violates CONTRIBUTING.md — Logging
**`src/api/handler.py:88`**
CONTRIBUTING.md section "Production code" forbids `print()`.
Source: `CONTRIBUTING.md:L42`
**Fix**: use `logger.debug(...)` (logger is already imported in this module).
```

If no convention violations were found, omit the section entirely. Don't write "no violations found" — it's noise.

---

## What to explicitly NOT do

- Do **not** treat README prose as a strict ruleset unless it's in a clearly marked section like "Conventions" or "Style".
- Do **not** enforce ADRs with `Status: Proposed` or `Status: Superseded`.
- Do **not** invent conventions that aren't clearly stated. If unclear, skip rather than misattribute.
- Do **not** flag things that Claude's baseline would already flag unless the project doc explicitly confirms the rule — the value here is the **citation**, not adding findings.

---

## Fallback for projects with no convention docs

If none of the above files exist, skip this step entirely — do not synthesize conventions. Emit **no** suggestion to add them (that's prescriptive and not the skill's job). Just move to the next step.
