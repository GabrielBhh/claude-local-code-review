# Tool Orchestration Rules

This skill's unique value comes from running **real tools** on the local repo, not just reasoning about code. This file documents:

1. Static analysis tools per language (SAST, linters, type checkers, dead code)
2. CVE / dependency vulnerability scanning
3. Secrets in git history
4. License compliance
5. Graceful fallback when tools are missing

---

## The Universal Fallback Pattern

Every tool invocation follows this pattern:

```
1. Check if tool is installed:   command -v <tool>
2. If installed:                  run it, parse output, merge findings
3. If not installed:              try alternative (if listed)
4. If nothing available:          fall back to Claude reasoning
                                  and note in report that the tool would help
```

**Never fail the audit because a tool is missing.** Missing tools are a 🔵 Suggestion ("install X for deeper results"), not a failure.

---

## Section 1 — Static Analysis Per Language

### Python

| Tool | Command | What it catches |
|---|---|---|
| `ruff` | `ruff check .` | 800+ rules: style, bugs, security subset |
| `mypy` | `mypy .` | Type errors |
| `pyright` | `pyright` | Type errors (alternative to mypy) |
| `bandit` | `bandit -r .` | Security issues (hardcoded passwords, insecure crypto, `subprocess` with `shell=True`) |
| `semgrep` | `semgrep --config auto .` | Semantic pattern match — extensive security ruleset |
| `vulture` | `vulture .` | Dead code |
| `pylint` | `pylint src/` | Broader style + bug checks |

**Preferred invocation order**: `ruff` → `mypy` → `bandit` → `semgrep` → `vulture`. Each adds a distinct layer.

### JavaScript / TypeScript

| Tool | Command | What it catches |
|---|---|---|
| `tsc --noEmit` | `npx tsc --noEmit` | Type errors |
| `eslint` | `npx eslint .` | Style, bugs, React hooks violations |
| `biome` | `npx @biomejs/biome check .` | Faster ESLint alternative |
| `semgrep` | `semgrep --config auto .` | Security patterns |
| `ts-prune` | `npx ts-prune` | Dead exports |
| `knip` | `npx knip` | Dead files + dead exports + unused deps |

**Use project's own script first**: if `package.json` has `"lint"`, run `npm run lint` — matches what CI runs.

### Go

| Tool | Command | What it catches |
|---|---|---|
| `go vet` | `go vet ./...` | Suspicious constructs (built-in) |
| `staticcheck` | `staticcheck ./...` | Deeper analysis than `go vet` |
| `golangci-lint` | `golangci-lint run` | Runs many linters at once |
| `gosec` | `gosec ./...` | Security patterns |
| `ineffassign` | `ineffassign ./...` | Ineffective assignments |
| `errcheck` | `errcheck ./...` | Unchecked errors |

### Rust

| Tool | Command | What it catches |
|---|---|---|
| `cargo check` | `cargo check --all-targets` | Compile errors (fast, no codegen) |
| `cargo clippy` | `cargo clippy --all-targets` | Lints, idiomatic suggestions |
| `cargo-udeps` | `cargo +nightly udeps` | Unused dependencies |
| `cargo-deny` | `cargo deny check` | Licensing + vulnerability policies |

### Ruby

| Tool | Command | What it catches |
|---|---|---|
| `rubocop` | `bundle exec rubocop` | Style, bugs, security cops |
| `brakeman` | `brakeman -A` | Rails security scanner |
| `bundler-audit` | `bundle audit` | Gem vulnerabilities |

### PHP

| Tool | Command | What it catches |
|---|---|---|
| `phpstan` | `vendor/bin/phpstan analyse` | Static analysis |
| `psalm` | `vendor/bin/psalm` | Alternative static analyser |
| `phan` | `vendor/bin/phan` | Another option |

### Java / Kotlin

| Tool | Command | What it catches |
|---|---|---|
| `spotbugs` | `mvn spotbugs:check` / `gradle spotbugsMain` | Bug patterns |
| `pmd` | `mvn pmd:check` / `gradle pmdMain` | Code style + bugs |
| `detekt` (Kotlin) | `gradle detekt` | Kotlin lints |
| `ktlint` (Kotlin) | `ktlint` | Kotlin formatting |
| `checkstyle` | `mvn checkstyle:check` | Style |

### C / C++

| Tool | Command | What it catches |
|---|---|---|
| `clang-tidy` | `clang-tidy src/*.cpp --` | Modernizations, bugs |
| `cppcheck` | `cppcheck --enable=all .` | Static analysis |
| `scan-build` | `scan-build make` | Clang's analyzer |

### Shell

| Tool | Command | What it catches |
|---|---|---|
| `shellcheck` | `shellcheck *.sh` | Nearly everything shell-specific |

### Terraform / IaC

| Tool | Command | What it catches |
|---|---|---|
| `tflint` | `tflint` | Terraform lints |
| `tfsec` | `tfsec .` | Terraform security |
| `checkov` | `checkov -d .` | Multi-IaC security (Terraform, K8s, Dockerfile, CloudFormation) |
| `kics` | `kics scan -p .` | Alternative to checkov |
| `terrascan` | `terrascan scan` | Another option |

### Docker

| Tool | Command | What it catches |
|---|---|---|
| `hadolint` | `hadolint Dockerfile` | Dockerfile best practices |
| `trivy` | `trivy fs .` | Image + filesystem CVEs, secrets, misconfigs |
| `docker scout` | `docker scout cves` | Docker's own scanner |

### Running the project's own scripts first

Before detecting tools globally, check the project's own configuration:

- **JS/TS**: look at `package.json` `scripts` — run `npm run lint`, `npm run typecheck` if defined
- **Python**: look at `pyproject.toml` / `Makefile` — run `make lint` if defined
- **Makefiles**: `make lint`, `make test`, `make check`
- **justfiles**: `just lint`

This ensures the audit uses the **same rules the team's CI uses**.

---

## Section 2 — CVE & Dependency Scanning

### Ecosystem map

| Language / Platform | Lockfiles | CLI tool (try first) | OSV.dev ecosystem |
|---|---|---|---|
| Python | `requirements.txt`, `poetry.lock`, `Pipfile.lock`, `uv.lock` | `pip-audit` | `PyPI` |
| JS/TS | `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml` | `npm audit --json` / `yarn audit` / `pnpm audit` | `npm` |
| Go | `go.mod` + `go.sum` | `govulncheck ./...` | `Go` |
| Java/Kotlin | `pom.xml`, `build.gradle`, `gradle.lockfile` | `mvn dependency-check:check` | `Maven` |
| Ruby | `Gemfile.lock` | `bundle audit` | `RubyGems` |
| Rust | `Cargo.lock` | `cargo audit` | `crates.io` |
| PHP | `composer.lock` | `composer audit` | `Packagist` |
| C#/.NET | `packages.lock.json`, `*.csproj` | `dotnet list package --vulnerable` | `NuGet` |
| Dart/Flutter | `pubspec.lock` | `dart pub outdated` | `Pub` |
| Swift | `Package.resolved` | _(none — use OSV.dev)_ | `SwiftURL` |
| Terraform | `.terraform.lock.hcl` | `trivy fs .` | `Go` (modules) |
| Dockerfile | `Dockerfile`, `docker-compose.yml` | `trivy fs .` / `docker scout cves` | _(image scanning)_ |

### Process

1. Detect dependency files in the repo
2. For each, try the CLI tool first (richer output)
3. If the tool isn't installed, POST to `https://api.osv.dev/v1/querybatch` with extracted `(name, version)` pairs
4. Parse vulnerabilities, map CVSS → severity (≥7.0 → 🔴, 4.0–6.9 → 🟡, <4.0 → 🔵)

### OSV.dev batch query

**Endpoint**: `POST https://api.osv.dev/v1/querybatch`

```json
{
  "queries": [
    { "package": { "name": "requests", "ecosystem": "PyPI" }, "version": "2.25.0" },
    { "package": { "name": "lodash",   "ecosystem": "npm"  }, "version": "4.17.20" }
  ]
}
```

Response is ordered-parallel to input. Batch up to 1,000 per request.

### Lockfile parsing quick reference

| Lockfile | Format | Extract |
|---|---|---|
| `requirements.txt` | Plain | `name==version` |
| `poetry.lock` / `uv.lock` | TOML | `[[package]]` → `name`, `version` |
| `Pipfile.lock` | JSON | `default` + `develop` → strip `==` |
| `package-lock.json` (v2/v3) | JSON | `packages` → key + `version` |
| `yarn.lock` | Custom | Header `"name@..."` + `version:` line |
| `pnpm-lock.yaml` | YAML | `/name/version:` keys |
| `go.sum` | Plain | `module version hash` (strip `/go.mod` suffix) |
| `Cargo.lock` | TOML | `[[package]]` → `name`, `version` |
| `Gemfile.lock` | Custom | Under `GEM > specs:` → `  name (version)` |
| `pom.xml` | XML | `<dependency>` → `<groupId>:<artifactId>` + `<version>` |
| `composer.lock` | JSON | `packages` + `packages-dev` |
| `packages.lock.json` | JSON | `dependencies` → name + `resolved.version` |
| `pubspec.lock` | YAML | `packages` map → `version` |
| `Package.resolved` | JSON | `pins` → `identity` + `state.version` |

---

## Section 3 — Secrets in Git History

Current-file scanning misses secrets committed then deleted. Git history retains them.

### Tools

| Tool | Command | Notes |
|---|---|---|
| `gitleaks` | `gitleaks detect --source . --report-format json --no-banner` | Fast, actively maintained |
| `trufflehog` | `trufflehog git file://. --json` | Verifies secrets against real services |
| `git-secrets` | `git-secrets --scan-history` | AWS-focused, older |

### Severity

**Always 🔴 Critical** when a secret is detected in history — even if deleted from HEAD — because:
- The commit SHA is public the moment it was pushed
- Rotating the credential is the only remediation
- `git filter-repo` cleanup is required to remove it from the repo's history

### Fallback

If no history scanner is installed, emit one 🔵 Suggestion: "install `gitleaks` (`brew install gitleaks`) to scan git history for previously-committed secrets."

Do **not** attempt to manually scan history via `git log -p` — too slow, too noisy.

---

## Section 4 — License Compliance

### Tools

| Ecosystem | Tool | Command |
|---|---|---|
| Python | `pip-licenses` | `pip-licenses --format=json` |
| JS/TS | `license-checker` | `npx license-checker --json` |
| Rust | `cargo-license` | `cargo license` |
| Go | `go-licenses` | `go-licenses report ./...` |
| Ruby | `license_finder` | `license_finder` |
| Java | `mvn license:add-third-party` | Maven |
| Multi | `scancode-toolkit` | `scancode-toolkit -ilp --json-pp out.json .` |

### What to flag

1. **Project license detection**: read `LICENSE` or `LICENSE.md` from repo root
2. **Transitive dep licenses**: run the tool above
3. **Conflicts**:
   - Project is MIT/Apache-2.0/BSD + any transitive dep is **GPL/AGPL/LGPL** → 🔴 Critical (copyleft contamination)
   - Project is MIT + transitive is **GPL** used as library → 🔴 (distribution triggers GPL requirements)
   - Any dep with `UNKNOWN` / `Custom` / no identifiable license → 🟡 Warning
   - Any dep with explicit "for non-commercial use only" clause → 🟡 Warning

### If no tool is installed

Emit one 🔵 Suggestion with install command.

---

## Section 5 — Mutation Testing (optional, rich signal)

Coverage percentage is a weak signal. Mutation testing actually measures whether tests **catch bugs** by mutating the source and seeing if any test fails.

| Language | Tool | Command |
|---|---|---|
| Python | `mutmut` | `mutmut run` |
| JS/TS | `stryker` | `npx stryker run` |
| Rust | `cargo-mutants` | `cargo mutants` |
| Ruby | `mutant` | `mutant run` |
| Java | `pitest` | `mvn org.pitest:pitest-maven:mutationCoverage` |

**Only run on demand** — mutation testing is slow (minutes to hours). Suggest it in the report when:
- Code coverage is high (>80%) but the user asked for a deep review
- User explicitly opts in

---

## Section 6 — Handling Missing Tools

Add a single "Tooling suggestions" section at the end of the report. One line per missing tool, grouped by ecosystem. Never nag — just inform.

Example:
```
## Tooling suggestions

The following tools would have added depth. Optional — install any if useful:

- `ruff` (Python) — install with `pip install ruff` — 800+ static checks
- `bandit` (Python) — `pip install bandit` — security scanner
- `gitleaks` — `brew install gitleaks` — secrets in git history
- `license-checker` (npm) — `npm install -g license-checker` — transitive license audit
```

---

## Reporting format for tool findings

Every tool-sourced finding includes a `detected-by` tag for confidence scoring:

```
### 🔴 CRITICAL — SQL Injection
**`src/db/users.py:42`**  •  detected-by: `bandit` + `semgrep` + Claude (high confidence)
User input concatenated directly into SQL query string.
**Fix**: Use parameterized query: `cursor.execute("SELECT * WHERE id = %s", (user_id,))`
```

Multiple detectors = higher confidence. Claude-only detection = medium-low confidence (still valid, but more likely a false positive than a multi-tool consensus).
