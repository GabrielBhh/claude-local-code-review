---
name: claude-code-audit
description: |
  Deep local code audit — runs the project's real analyzers (ruff, eslint, gosec,
  cargo clippy, semgrep, bandit, etc.), scans dependencies for CVEs via OSV.dev,
  detects secrets in git history with gitleaks, analyses git-history hotspots
  (churn × complexity, ownership, temporal coupling), cross-checks findings
  against project conventions (CLAUDE.md, ADRs), and layers Claude's reasoning
  on top of tool output. Respects `.codereview-baseline.json` so only new
  regressions surface. Trigger when the user asks to "audit my code", "review my
  changes", "check for security issues", "run a deep review", "run the tools",
  or mentions /claude-code-audit. Position: the gate before commit/PR — deeper
  than Claude's built-in review. Languages: Python, JS/TS, Go, Java, Kotlin,
  Ruby, Rust, C, C++, PHP, C#, Dart/Flutter, SQL, Shell, Swift, Dockerfile,
  YAML, Terraform.
---

# claude-code-audit

A deep, **tool-orchestrating** code audit. Runs the real static analyzers, CVE scanners, secret history scanners, and git-intelligence commands on your local repo, then merges their output with Claude's reasoning into one structured report.

**Positioning**: this is the **gate** — the deep pass you run before commit/PR. It's intentionally different from Claude's built-in review (which is the fast in-flow first-pass). This skill adds what raw prompting cannot: real tool execution, CVE data from OSV.dev, secrets in git history, and git-archaeology.

---

## Pipeline overview

Every invocation runs these steps in order:

| Step | What it does |
|---|---|
| 1 | Resolve target + load config (baseline, ignore, conventions) |
| 2 | Claude reasoning pass (baked-in first-layer review) |
| 3 | Static analysis orchestration (per-language tools) |
| 4 | CVE + license scanning (OSV.dev + CLI tools) |
| 5 | Secrets in git history (gitleaks/trufflehog) |
| 6 | Git intelligence (hotspots, ownership, coupling) |
| 7 | Convention compliance (CLAUDE.md, AGENTS.md, ADRs) |
| 8 | Merge + dedupe + apply baseline + ignore |
| 9 | Write structured report |

Each tool step is **opportunistic** — use what's installed, fall back to Claude reasoning if nothing is available, and note which tools would have added value.

---

## Step 1 — Resolve target and load config

### Resolve what to audit

| User input | How to gather the code |
|---|---|
| (nothing / "my changes") | `git diff HEAD` |
| `staged` | `git diff --cached` |
| `unstaged` | `git diff` |
| `all` / "whole codebase" / "everything" | `git ls-files` |
| `<directory-path>` (e.g. `src/`) | `git ls-files <dir>` |
| `<file-path>` | Read the file directly |
| `<commit-hash>` | `git show <hash>` |
| `<branch>` | `git diff main..<branch>` |
| `<hash1>..<hash2>` | `git diff <hash1>..<hash2>` |
| `baseline update` | Re-snapshot current findings as accepted baseline |

### Load project config

Before analysing, read these if present:

- `.codereview-baseline.json` — baseline of accepted findings (see `references/baseline-and-state.md`)
- `.codereview-ignore` — patterns and file-specific ignores
- Project conventions: `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, `ARCHITECTURE.md`, `docs/adr/*.md`, `.editorconfig`, `.prettierrc`, language-specific configs (`ruff.toml`, `.eslintrc`, etc.)

### Large-codebase strategy

For `all` or directory reviews of >50 files: summarise scope up front, then work through by language group. For >150 files: ask whether to narrow scope before starting.

---

## Step 2 — Claude reasoning pass (baked-in first layer)

Apply Claude's native reasoning before running any tools. This is the skill's equivalent of Claude's built-in review — deliberately included here so users don't need to invoke two separate skills.

Focus on:

- **Bugs & logic errors** — null/undefined dereferences, off-by-one, race conditions, unhandled edge cases, wrong conditionals, crash paths
- **Security patterns** — apply `references/security-rules.md` (OWASP Top 10, SSTI, XXE, ReDoS, JWT pitfalls, open redirect, CSRF, timing attacks, secret detection regex)
- **Non-obvious language patterns** — apply `references/language-rules.md` and `references/simplification-rules.md` (only the patterns in these files — do NOT additionally list generic idioms Claude would catch by default, like `var → const` or `== vs ===`)
- **API design** — apply `references/api-design-rules.md` for REST/GraphQL/gRPC surfaces
- **Swift/SwiftUI** — apply `references/swift-apple-rules.md` for `.swift` files

**Important**: the reference files are intentionally curated. Do not invent generic "best practice" findings beyond what tools or references explicitly call out. The point of this skill is to add what tools + repo-wide analysis surface, not to repeat what Claude would say by default.

---

## Step 3 — Static analysis orchestration

Load `references/tool-orchestration-rules.md`. For each detected language:

1. Detect which tools are installed (`command -v <tool>`)
2. Run the installed tools (may include network calls for updated rules)
3. Parse output, map findings to severity tiers
4. Note which tools were NOT installed and would have added value

Tools covered per language: `ruff`, `mypy`, `bandit`, `semgrep`, `vulture`, `eslint`, `tsc --noEmit`, `biome`, `ts-prune`, `go vet`, `staticcheck`, `golangci-lint`, `gosec`, `cargo clippy`, `cargo-udeps`, `rubocop`, `brakeman`, `phpstan`, `psalm`, `shellcheck`, `tfsec`, `checkov`, `tflint`, `hadolint`, `trivy`, `detekt`, `spotbugs`, and more.

**Fallback pattern**: if no tools are installed for an ecosystem, rely on Claude's reasoning (already done in Step 2) and emit a single 🔵 Suggestion noting which tools would help.

---

## Step 4 — CVE + license scanning

Load `references/tool-orchestration-rules.md` (CVE section). For each detected dependency file:

1. Try the ecosystem CLI tool first (`pip-audit`, `npm audit`, `cargo audit`, `govulncheck`, `bundle audit`, `composer audit`, `dotnet list package --vulnerable`, `dart pub outdated`, `trivy fs .`)
2. If the tool is not installed, POST to `https://api.osv.dev/v1/querybatch` with extracted `(name, version)` pairs
3. Map CVSS scores to severity: ≥7.0 → 🔴, 4.0–6.9 → 🟡, <4.0 → 🔵
4. **License compliance**: try `pip-licenses`, `license-checker`, `cargo-license` — flag GPL/AGPL/copyleft in transitive deps if the project licence is MIT/Apache/BSD

---

## Step 5 — Secrets in git history

If `gitleaks` is available:
```
gitleaks detect --source . --report-format json
```

If not, try `trufflehog git file://. --json`. If neither is available, emit a 🔵 Suggestion with the install command. Do not scan history manually (too slow, too many false positives) — this is where tools clearly win.

Finding severity: **always 🔴 Critical** when a secret is detected, even if later deleted from current HEAD — the commit SHA is public the moment it's pushed.

---

## Step 6 — Git intelligence

Load `references/git-intelligence-rules.md`. Run:

- **Churn × complexity hotspots**: combine `git log --since="6 months ago" --name-only` counts with cyclomatic complexity per file — flag files that are both frequently changed AND complex (🟡 Warning)
- **Ownership / bus factor**: `git log --format='%an' <file>` — flag files only one author has ever touched in the changed set (🔵 Suggestion unless the file is core infra, then 🟡)
- **Temporal coupling**: files changed together in the same commits more than 70% of the time, but in different modules → hidden coupling (🔵)
- **Dead file candidates**: tracked files with zero commits in 2+ years and zero inbound imports → possible dead code (🔵)

For a `staged` scope, only report hotspot metrics for files in the diff — don't derail into whole-repo archaeology.

---

## Step 7 — Convention compliance

Load `references/convention-compliance-rules.md`. Cross-check findings against:

- Rules stated in `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`
- Architectural decisions in `docs/adr/*.md` (only those with status `Accepted`)
- Declared conventions in `.editorconfig`, linter configs, `package.json` scripts

When flagging a violation: **cite the source document and line**, e.g. `violates ADR-012 (docs/adr/0012-use-postgres.md:L14)`. This turns generic findings into team-specific guardrails.

---

## Step 8 — Merge, dedupe, apply baseline

1. **Dedupe**: the same file:line flagged by multiple tools → single finding with `detected-by: [tool1, tool2, claude]` for confidence scoring
2. **Apply baseline**: if `.codereview-baseline.json` exists, suppress findings that match. Only surface regressions and new findings.
3. **Apply ignore**: respect `.codereview-ignore` patterns (glob-style file patterns + per-line `# codereview: ignore` comments)
4. **Confidence scoring**:
   - Flagged by 2+ tools + Claude = **high confidence** (bump severity up a tier if borderline)
   - Flagged by 1 tool only = **medium**
   - Claude reasoning only = **low** (but can still be 🔴 if clearly a bug)

---

## Step 9 — Write the report

```
# Code Audit Report
**Target**: <what was audited>
**Date**: <today's date>
**Language(s)**: <detected>
**Scope**: <N files / N lines>
**Tools run**: <list>
**Tools missing**: <list>
**Baseline**: <in-use / not present>

## Summary

| Severity | Count (new) | Count (baseline) |
|---|---|---|
| 🔴 Critical | N | M |
| 🟡 Warning | N | M |
| 🔵 Suggestion | N | M |

## Findings

### 🔴 CRITICAL — [Category]
**`path/to/file.ext:42`**  •  detected-by: `gosec` + Claude (high confidence)
One-sentence description.
**Fix**: concrete snippet or short explanation.

### 🟡 WARNING — [Category]
**`path/to/file.ext:17`**  •  detected-by: `ruff` (medium confidence)
...

### 🔵 SUGGESTION — [Category]
**`path/to/file.ext:8`**  •  detected-by: Claude reasoning (low confidence)
...

## Convention violations
List any findings tied to a specific project doc with a link.

## Hotspot notes
Brief git-intelligence callouts for changed files (if any).

## Tooling suggestions
Tools that weren't installed but would have added depth. One-line each, optional.

## What looks good
2–4 bullets of positive patterns worth preserving.

## Recommended next steps
Ordered action list, most critical first.
```

### Severity guide

| Level | Use for |
|---|---|
| 🔴 Critical | Security vulnerabilities, credential leaks, known CVEs ≥7.0, crash bugs, data loss, secrets in git history |
| 🟡 Warning | Best-practice violations confirmed by tools, performance issues, hotspot files, moderate CVEs |
| 🔵 Suggestion | Style, minor improvements, low-confidence signals, tool installation hints |

---

## Tips

- **Cite `file:line`** on every finding. Without it, findings aren't actionable.
- **Cite tool provenance**: include `detected-by` so users know which tool flagged what.
- **Be opportunistic, not demanding** — never fail if a tool is missing; fall back to reasoning and suggest the install.
- **Do not repeat Claude's baseline** — the reference files are the curated list. Don't invent generic "best practice" findings that aren't in the refs. If a user wants that, they can ask Claude directly.
- **Respect the baseline** — a clean report against a baseline is success, not failure. Users adopt this skill on mature codebases; reporting legacy issues as if they were new creates noise.
- **Network calls permitted** for: OSV.dev (CVE scanning), package registries (license lookup), and tool auto-update. Not permitted: remote git hosts, PR systems, unrelated APIs.
