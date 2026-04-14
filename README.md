# local-code-review

A Claude Code skill that performs a thorough, multi-dimensional code review entirely on your local repository — no pull request, no remote connection needed.

---

## What it does

Invoke it and Claude will review your code across 18 dimensions:

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
| 9 | Dependency risk & CVE scanning |
| 10 | Performance anti-patterns |
| 11 | Documentation gaps |
| 12 | Accessibility (frontend) |
| 13 | SDLC hygiene |
| 14 | API design (REST / GraphQL / gRPC) |
| 15 | Internationalisation (i18n) |
| 16 | Observability (logging, tracing, metrics) |
| 17 | Concurrency & thread safety |
| 18 | Code simplification & readability |

**Languages supported**: Python · JavaScript · TypeScript · Go · Java · Kotlin · Ruby · Rust · C · C++ · PHP · C# · Dart/Flutter · SQL · Shell/Bash · Swift/SwiftUI · Dockerfile · YAML · Terraform

---

## Installation

### Option A — Download from Releases (easiest)

1. Go to the [Releases](../../releases) page and download `claude-local-code-review.skill`.
2. Open **Claude Code**.
3. Go to **Customize/Settings → Skills → Upload skill** and select the downloaded file.

### Option B — Build from source

```bash
git clone https://github.com/GabrielBhh/claude-local-code-review.git
cd ..
zip -r claude-local-code-review.skill claude-local-code-review/ --exclude "*.git*" --exclude "*.skill"
```

Then upload `claude-local-code-review.skill` via **Customize/Settings → Skills → Upload skill**.

That's it — the skill is available immediately in any project.

---

## Usage

Type `/local-code-review` followed by a target in any Claude Code conversation. All commands run entirely offline — no PR or remote connection needed.

### By scope

| Command | What gets reviewed |
|---|---|
| `/local-code-review` | All uncommitted changes (`git diff HEAD`) |
| `/local-code-review uncommitted` | All uncommitted changes (`git diff HEAD`) |
| `/local-code-review staged` | Only staged changes (`git diff --cached`) |
| `/local-code-review unstaged` | Only unstaged changes (`git diff`) |
| `/local-code-review all` | Entire codebase (every tracked file) |

### By path

| Command | What gets reviewed |
|---|---|
| `/local-code-review src/` | All files in the `src/` directory |
| `/local-code-review app/auth/` | All files in a subdirectory |
| `/local-code-review src/auth.py` | A single file |

### By commit or branch

| Command | What gets reviewed |
|---|---|
| `/local-code-review last commit` | The most recent commit (`git show HEAD`) |
| `/local-code-review a3f92c1` | A specific commit by hash |
| `/local-code-review my-feature-branch` | Everything on a branch vs `main` |
| `/local-code-review a3f92c1..HEAD` | A range of commits |

### Natural language also works

```
review my staged changes
check src/auth.py for security issues
audit the whole codebase
review the last commit
```

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
claude-local-code-review/
├── SKILL.md                          # Skill definition (loaded by Claude Code)
├── README.md                         # This file
└── references/
    ├── security-rules.md             # OWASP Top 10, secret detection, SSTI, XXE, ReDoS
    ├── language-rules.md             # Per-language best practice rules (19 languages)
    ├── simplification-rules.md       # Code simplification & readability patterns
    ├── swift-apple-rules.md          # Swift/SwiftUI + Apple Liquid Glass (iOS 26)
    ├── api-design-rules.md           # REST, GraphQL, and gRPC design rules
    └── cve-scanning-rules.md         # CVE scanning via CLI tools + OSV.dev API
```

---

## CVE Scanning

Dimension #9 performs live CVE scanning against your dependencies — no separate tool invocation needed.

**How it works:**

1. Detects lockfiles in the repository (`package-lock.json`, `requirements.txt`, `Cargo.lock`, `go.sum`, `Gemfile.lock`, `composer.lock`, `pubspec.lock`, `Package.resolved`, `pom.xml`, `.terraform.lock.hcl`, etc.)
2. Runs the ecosystem's CLI audit tool if installed:

| Ecosystem | CLI tool |
|---|---|
| Python | `pip-audit` |
| JavaScript / TypeScript | `npm audit`, `yarn audit`, `pnpm audit` |
| Go | `govulncheck ./...` |
| Java / Kotlin | `mvn dependency-check:check` |
| Ruby | `bundle audit` |
| Rust | `cargo audit` |
| PHP | `composer audit` |
| C# / .NET | `dotnet list package --vulnerable` |
| Dart / Flutter | `dart pub outdated` |
| Terraform / Dockerfile | `trivy fs .` |

3. If the CLI tool is not installed, falls back to querying the **[OSV.dev](https://osv.dev) batch API** — a free, open vulnerability database covering all major ecosystems.
4. Reports findings with CVSS severity, CVE ID, affected version, and the fix version.

> CVE scanning requires internet access to reach OSV.dev. The rest of the review runs fully offline.

---

## Requirements

- [Claude Code](https://claude.ai/code) (CLI, desktop app, or IDE extension)
- A git repository (for diff-based reviews; file reviews work without git)

---

## Contributing

Contributions are welcome — new language rules, additional review dimensions, or fixes to existing patterns.

1. Fork this repository.
2. Edit the relevant file under `references/` or `SKILL.md`.
3. Repackage from the **parent directory**:
   ```bash
   zip -r claude-local-code-review.skill claude-local-code-review/ --exclude "*.git*" --exclude "*.skill"
   ```
4. Open a pull request with a short description of what you added and why.

**Adding a new language**: Add a section to `references/language-rules.md` using the existing table format (Issue / Violation / Fix), then add the language name to the detection list in `SKILL.md` Step 2.

**Adding a new review dimension**: Add a numbered entry to Step 3 in `SKILL.md`. If it needs a reference file, add that under `references/` and wire it up in Step 2.

---

## License

MIT
