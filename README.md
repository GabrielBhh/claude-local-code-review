# local-code-review

A Claude Code skill that performs a thorough, multi-dimensional code review entirely on your local repository — no pull request, no remote connection needed.

---

## What it does

Invoke it and Claude will review your code across 17 dimensions:

| # | Dimension |
|---|---|
| 1 | Bugs & logic errors |
| 2 | Security — OWASP Top 10 |
| 3 | Secrets & credential leaks |
| 4 | Language best practices |
| 5 | Complexity (functions >20 lines, nesting >3) |
| 6 | Dead code |
| 7 | Code duplication |
| 8 | Test coverage gaps |
| 9 | Dependency risk |
| 10 | Performance anti-patterns |
| 11 | Documentation gaps |
| 12 | Accessibility (frontend) |
| 13 | SDLC hygiene |
| 14 | API design (REST / GraphQL / gRPC) |
| 15 | Internationalisation (i18n) |
| 16 | Observability (logging, tracing, metrics) |
| 17 | Concurrency & thread safety |

**Languages supported**: Python · JavaScript · TypeScript · Go · Java · Kotlin · Ruby · Rust · C · C++ · PHP · C# · Dart/Flutter · SQL · Shell/Bash · Swift/SwiftUI · Dockerfile · YAML · Terraform

---

## Installation

1. Download `local-code-review.skill` from this repository (or clone it).
2. Open **Claude Code**.
3. Go to **Customize/Settings → Skills → Upload skill** and select `local-code-review.skill`.

That's it — the skill is available immediately in any project.

---

## Usage

Once installed, trigger the skill naturally in any Claude Code conversation:

```
/local-code-review
```

Or just describe what you want:

```
review my staged changes
check this file for security issues: src/auth.py
review the whole codebase
audit app/ for bugs
review the last commit
```

### Review targets

| What you say | What gets reviewed |
|---|---|
| (nothing / "my changes") | `git diff HEAD` |
| "staged" | `git diff --cached` |
| "unstaged" | `git diff` |
| "all" / "whole codebase" | Every tracked file via `git ls-files` |
| A directory path (e.g. `src/`) | All files in that directory |
| A file path | That file directly |
| A commit hash | `git show <hash>` |
| A branch name | `git diff main..<branch>` |
| `<hash1>..<hash2>` | `git diff <hash1>..<hash2>` |

### Report format

Every finding is tagged by severity and includes a `file:line` reference plus a concrete fix:

```
🔴 CRITICAL   — security vulnerabilities, credential leaks, crash bugs, data loss
🟡 WARNING    — best practice violations, potential bugs, performance issues
🔵 SUGGESTION — style, documentation, minor improvements
```

---

## File structure

```
local-code-review/
├── SKILL.md                          # Skill definition (loaded by Claude Code)
├── README.md                         # This file
└── references/
    ├── security-rules.md             # OWASP Top 10 + secret detection patterns
    ├── language-rules.md             # Per-language best practice rules
    ├── swift-apple-rules.md          # Swift/SwiftUI + Apple Liquid Glass (iOS 26)
    └── api-design-rules.md           # REST, GraphQL, and gRPC design rules
```

---

## Requirements

- [Claude Code](https://claude.ai/code) (CLI, desktop app, or IDE extension)
- A git repository (for diff-based reviews; file reviews work without git)

---

## Contributing

Contributions are welcome — new language rules, additional review dimensions, or fixes to existing patterns.

1. Fork this repository.
2. Edit the relevant file under `references/` or `SKILL.md`.
3. Repackage: `zip -r local-code-review.skill local-code-review/`
4. Open a pull request with a short description of what you added and why.

**Adding a new language**: Add a section to `references/language-rules.md` using the existing table format (Issue / Violation / Fix), then add the language name to the detection list in `SKILL.md` Step 2.

**Adding a new review dimension**: Add a numbered entry to Step 3 in `SKILL.md`. If it needs a reference file, add that under `references/` and wire it up in Step 2.

---

## License

MIT
