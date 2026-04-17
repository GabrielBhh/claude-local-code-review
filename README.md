# claude-code-audit

> A Claude Code skill that **orchestrates real tools** — static analyzers, CVE scanners, secret history scanners, git archaeology — and layers Claude's reasoning on top. Runs locally. Designed as the **gate before commit or PR**.

This is intentionally different from Claude's built-in code review. Built-in review is fast, reasoning-only, and great in-flow. `claude-code-audit` is the deep pass: it actually runs your project's linters, queries CVE databases, scans git history for secrets, and analyses churn/complexity hotspots — things raw prompting can't do.

---

## What makes this different

| | Claude's built-in review | `claude-code-audit` |
|---|---|---|
| Speed | Seconds | 1–3 minutes |
| Reasoning | ✅ | ✅ (baked in) |
| Real tool execution | ❌ | ✅ (ruff, eslint, gosec, semgrep, bandit, etc.) |
| CVE scanning | ❌ | ✅ (OSV.dev + `pip-audit`/`npm audit`/etc.) |
| Secrets in git history | ❌ | ✅ (gitleaks/trufflehog) |
| Git hotspot analysis | ❌ | ✅ (churn × complexity, ownership) |
| Convention compliance | ❌ | ✅ (enforces CLAUDE.md, AGENTS.md, ADRs) |
| Stateful across reviews | ❌ | ✅ (`.codereview-baseline.json`) |
| License compliance | ❌ | ✅ (copyleft detection) |

---

## The pipeline

Every invocation runs this 9-step pipeline:

```
1. Resolve target + load config (baseline, ignore, conventions)
        ↓
2. Claude reasoning pass (bugs, security, patterns)
        ↓
3. Static analysis — run project's real linters/analyzers
        ↓
4. CVE + license scanning (CLI tools + OSV.dev fallback)
        ↓
5. Secrets in git history (gitleaks/trufflehog)
        ↓
6. Git intelligence (hotspots, ownership, temporal coupling)
        ↓
7. Convention compliance (CLAUDE.md, AGENTS.md, ADRs)
        ↓
8. Merge + dedupe + apply baseline + ignore
        ↓
9. Structured report with severity, confidence, tool provenance
```

**Every tool step is opportunistic** — uses what's installed, falls back to reasoning if nothing is available, and notes which tools would have added depth.

---

## Installation

### Option A — Download from Releases (easiest)

1. Go to the [Releases](../../releases) page and download `claude-code-audit.skill`.
2. Open **Claude Code**.
3. **Customize/Settings → Skills → Upload skill** → select the file.

### Option B — Build from source

```bash
git clone https://github.com/GabrielBhh/claude-code-audit.git
cd ..
zip -r claude-code-audit.skill claude-code-audit/ --exclude "*.git*" --exclude "*.skill"
```

Upload via **Customize/Settings → Skills → Upload skill**.

---

## Usage

Type `/claude-code-audit` followed by a target. All commands run locally (except CVE/license lookups which query public APIs).

### By scope

| Command | What gets audited |
|---|---|
| `/claude-code-audit` | All uncommitted changes (`git diff HEAD`) |
| `/claude-code-audit staged` | Only staged changes |
| `/claude-code-audit unstaged` | Only unstaged changes |
| `/claude-code-audit all` | Entire codebase |

### By path

| Command | What gets audited |
|---|---|
| `/claude-code-audit src/` | All files in `src/` |
| `/claude-code-audit src/auth.py` | Single file |

### By commit or branch

| Command | What gets audited |
|---|---|
| `/claude-code-audit last commit` | Most recent commit |
| `/claude-code-audit a3f92c1` | Specific commit by hash |
| `/claude-code-audit my-feature-branch` | Branch vs main |
| `/claude-code-audit a3f92c1..HEAD` | Range of commits |

### Baseline management

| Command | Effect |
|---|---|
| `/claude-code-audit baseline update` | Re-snapshot current findings as accepted baseline |

---

## When to use

**This is a gate, not an in-flow helper.**

| Situation | Use |
|---|---|
| Writing code, asking questions | Claude directly (no skill) |
| Quick sanity check | `review my staged changes` (Claude's built-in) |
| Final pass before commit | `/claude-code-audit staged` |
| Before opening a PR | `/claude-code-audit <branch>` |
| Weekly drift check | `/claude-code-audit all` |
| Initial adoption on mature codebase | `/claude-code-audit all` → `/claude-code-audit baseline update` |

Claude's built-in review is your **rough-draft editor**. This skill is the **fact-checking pass** before shipping.

---

## Tool orchestration — what it runs

The skill tries the following tools per language, uses whatever is installed, and gracefully skips what isn't. You don't have to install any of them — but the more you have, the richer the audit.

| Language | Tools the skill will run if available |
|---|---|
| Python | `ruff`, `mypy`, `bandit`, `semgrep`, `vulture`, `pip-audit` |
| JavaScript / TypeScript | `eslint`, `tsc --noEmit`, `biome`, `semgrep`, `ts-prune`, `npm audit` |
| Go | `go vet`, `staticcheck`, `golangci-lint`, `gosec`, `govulncheck` |
| Rust | `cargo check`, `cargo clippy`, `cargo audit`, `cargo-udeps` |
| Ruby | `rubocop`, `brakeman`, `bundle audit` |
| PHP | `phpstan`, `psalm`, `composer audit` |
| Java / Kotlin | `spotbugs`, `detekt`, `ktlint` |
| Shell | `shellcheck` |
| Terraform | `tflint`, `tfsec`, `checkov` |
| Docker | `hadolint`, `trivy`, `docker scout` |
| Git history | `gitleaks`, `trufflehog` |
| Licenses | `pip-licenses`, `license-checker`, `cargo-license` |

Missing tools appear in a "Tooling suggestions" section with the one-line install command — take them or leave them.

---

## Baseline mode

Most teams adopting a code audit tool drown in pre-existing findings. Baseline mode fixes this.

1. Run `/claude-code-audit all` on a fresh repo
2. Run `/claude-code-audit baseline update` — snapshots all current findings as accepted
3. Future audits only report **new regressions**, not legacy findings

The baseline file (`.codereview-baseline.json`) should be **committed to the repo** so the whole team shares it.

Each accepted finding has an `acknowledged` field with a human explanation ("Legacy code. Migration in JIRA-1234"). Empty explanations are allowed but flagged for review.

See `references/baseline-and-state.md` for the full schema.

---

## Per-line ignores

`.codereview-ignore` at repo root supports glob-style exclusions:

```
# Ignore files
generated/**
*_pb.go

# Ignore specific rules in specific files
src/legacy/db.py:bandit:B608
```

Inline suppression comments require a reason:

```python
password = "hunter2"  # codereview: ignore bandit:B105  (test fixture)
```

No reason = still flagged. This prevents silent muting.

---

## Convention compliance

The skill reads the following if present and **cites them** when a finding matches a team decision:

- `CLAUDE.md`, `AGENTS.md`, `.claude/instructions.md`
- `CONTRIBUTING.md`, `ARCHITECTURE.md`, `STYLE.md`
- `docs/adr/*.md` (only those with `Status: Accepted`)
- Tool configs: `.editorconfig`, `.eslintrc`, `ruff.toml`, `.golangci.yml`, etc.

Findings that violate an ADR get bumped in severity and **cite the source document and line** — turning generic style suggestions into "violates ADR-012."

---

## Supported languages

Python · JavaScript · TypeScript · Go · Java · Kotlin · Ruby · Rust · C · C++ · PHP · C# · Dart/Flutter · SQL · Shell/Bash · Swift/SwiftUI · Dockerfile · YAML · Terraform

---

## File structure

```
claude-code-audit/
├── SKILL.md                               # Skill definition (loaded by Claude Code)
├── README.md                              # This file
└── references/
    ├── security-rules.md                  # OWASP Top 10, SSTI, XXE, ReDoS, secret regex
    ├── language-rules.md                  # Non-obvious language patterns (pruned, ~10 rules/lang)
    ├── simplification-rules.md            # Code simplification patterns (60+)
    ├── swift-apple-rules.md               # Swift/SwiftUI + Apple Liquid Glass (iOS 26)
    ├── api-design-rules.md                # REST, GraphQL, gRPC design
    ├── tool-orchestration-rules.md        # SAST tools + CVE scanning + git history secrets + licenses
    ├── git-intelligence-rules.md          # Hotspots, ownership, temporal coupling
    ├── convention-compliance-rules.md     # CLAUDE.md / ADR enforcement
    └── baseline-and-state.md              # Baseline + ignore file schemas
```

---

## Requirements

- [Claude Code](https://claude.ai/code)
- A git repository
- Network access (for CVE/license database lookups)
- **Optional** tools — whatever you install adds depth. Nothing is required.

---

## Roadmap

Things being considered for v2.1+:

- Framework-specific reference files: `react-rules.md`, `django-rules.md`, `rails-rules.md`, `fastapi-rules.md`, `nextjs-rules.md`
- Domain-specific modes: financial (money as `Decimal`), ML (data leakage), blockchain (reentrancy)
- SARIF output format for CI integration (GitHub Code Scanning, GitLab)
- Pre-commit hook installer: `/claude-code-audit install-hook`
- Mutation testing integration for deep test-quality review

Opinions and PRs welcome — see Contributing below.

---

## Contributing

Contributions are welcome. Especially:

- New tool integrations in `references/tool-orchestration-rules.md`
- Framework-specific reference files
- Git-intelligence heuristics
- Convention parsers for more project docs

### To propose a change:

1. Fork this repository
2. Edit the relevant `references/*.md` file or `SKILL.md`
3. Repackage (from the parent directory):
   ```bash
   zip -r claude-code-audit.skill claude-code-audit/ --exclude "*.git*" --exclude "*.skill"
   ```
4. Open a PR with a short description of what you added and why

**When adding a new review dimension**: think carefully about what raw Claude reasoning *can't* do. The skill's value is specifically in things that require execution, state, or data from external sources. Generic "here's another best practice" entries should live in Claude's default reasoning, not here.

---

## License

MIT

---

## History

Previously named `claude-local-code-review` (v1.0–1.1). Renamed and repositioned in v2.0 to reflect the tool-orchestration focus.
