---
name: local-code-review
description: |
  Thorough local code review — no remote connection needed. Trigger when the user asks
  to "review my code", "check for bugs", "audit for security", "look at my changes",
  "review this file/codebase/everything", or mentions /local-code-review, code smells,
  crashes, memory leaks, security issues, or wants a diff review. Covers: bugs, security
  (OWASP Top 10), secrets/credential leaks, best practices, complexity, dead code,
  duplication, test gaps, dependency risks, performance, docs, accessibility, API design
  (REST/GraphQL/gRPC), i18n, observability, concurrency, SDLC hygiene, and Apple Liquid
  Glass/iOS 26 for Swift. Languages: Python, JS/TS, Go, Java, Kotlin, Ruby, Rust, C,
  C++, PHP, C#, Dart/Flutter, SQL, Shell, Swift, Dockerfile, YAML, Terraform.
---

# local-code-review

A comprehensive local code review skill. Reviews staged changes, unstaged diffs, specific
commits, branches, individual files, directories, or the entire codebase — entirely offline.

---

## Step 1 — Resolve the review target

| User input | How to gather the code |
|---|---|
| (nothing / "my changes") | `git diff HEAD` |
| `staged` | `git diff --cached` |
| `unstaged` | `git diff` |
| `all` / "whole codebase" / "everything" | `git ls-files` → read each file (see full-codebase strategy below) |
| `<directory-path>` (e.g. `src/`, `app/`) | `git ls-files <dir>` → read each file in that directory |
| `<file-path>` | Read the file directly |
| `<commit-hash>` | `git show <hash>` |
| `<branch>` | `git diff main..<branch>` |
| `<hash1>..<hash2>` | `git diff <hash1>..<hash2>` |

If the diff is empty, say so clearly and ask the user what they'd like to review.

### Full-codebase / directory review strategy

When reviewing `all` or a directory, the codebase may be large. Work through it systematically:

1. Run `git ls-files [<dir>]` to get the full file list.
2. Skip files that can't contain reviewable code: lock files (`package-lock.json`, `Podfile.lock`, `*.lock`), generated files (`*.pb.go`, `*_generated.*`), vendored dependencies (`vendor/`, `node_modules/`, `.venv/`), binary assets, and minified files (`*.min.js`).
3. Group remaining files by language/type. Read and review each group in turn.
4. If the codebase is large (>50 files), tell the user up front: summarise which directories you'll cover, then work through them group by group, producing one unified report at the end.
5. For very large codebases (>150 files), ask the user if they'd like to narrow the scope to a specific directory or file type, or confirm they want the full sweep — it will be thorough but long.

---

## Step 2 — Detect languages and load reference files

Scan file extensions in the diff/files. Then read the relevant reference files:

- Any language present → always read `references/security-rules.md`
- Python / JS / TS / Go / Java / Kotlin / Ruby / SQL / Shell / Dockerfile / YAML / Terraform / Rust / C / C++ / PHP / C# / Dart → read `references/language-rules.md`
- `.swift` files → read `references/swift-apple-rules.md`
- REST controllers, GraphQL schemas (`.graphql`, `.gql`), or `.proto` files detected → read `references/api-design-rules.md`

---

## Step 3 — Analyse across all dimensions

Work through every dimension below. Don't skip any — a clean result is still worth noting.

1. **Bugs & Logic Errors** — null dereferences, off-by-one, wrong conditionals, race conditions, unhandled edge cases
2. **Security (OWASP Top 10)** — injection (SQL, shell, XSS), broken auth, IDOR, misconfigurations, cryptography misuse. Apply patterns from `references/security-rules.md`
3. **Secrets / Credential Leaks** — API keys, tokens, passwords, private keys in source. Use regex patterns from `references/security-rules.md`
4. **Language Best Practices** — idioms, DRY, SOLID, naming. Apply rules from `references/language-rules.md` (or `references/swift-apple-rules.md` for Swift)
5. **Complexity** — functions >20 lines or nesting >3 levels → warn
6. **Dead Code** — unused imports, variables, unreachable branches
7. **Code Duplication** — visually similar blocks that could be extracted
8. **Test Coverage Gaps** — new public functions/classes with no corresponding test
9. **Dependency Risk** — newly added packages: unknown, unmaintained, or license-risky
10. **Performance Anti-patterns** — N+1 queries, sync I/O in async context, unnecessary re-renders, blocking the main thread
11. **Documentation Gaps** — public APIs or exported functions missing docstrings/JSDoc
12. **Accessibility (Frontend)** — missing alt text, aria labels, keyboard navigation
13. **SDLC Hygiene** — missing error handling at system boundaries, hardcoded environment assumptions, no availability guards for new platform APIs
14. **API Design** — apply rules from `references/api-design-rules.md` when REST endpoints, GraphQL schemas, or proto files are present: wrong HTTP verbs, incorrect status codes, inconsistent error envelopes, missing pagination, IDOR, unauthenticated sensitive endpoints, GraphQL N+1 (missing DataLoader), unbounded query depth, gRPC missing deadlines
15. **Internationalisation (i18n)** — hardcoded user-facing strings that should be in a strings/translation file, locale-sensitive number or date formatting done with raw string operations, pluralisation logic baked into code rather than delegated to an i18n library
16. **Observability** — raw `print`/`fmt.Println`/`console.log` in request-handling paths (use structured logger), missing trace/correlation ID propagation across service calls, silent `catch`/`except` blocks that swallow errors without logging, metric names missing units or using unbounded label cardinality
17. **Concurrency & Thread Safety** — shared mutable state accessed without synchronisation (beyond Go-specific rules), lock ordering inconsistency (potential deadlock), `sleep`-based polling instead of signals/channels/condition variables, blocking I/O on an async/event-loop thread causing starvation

---

## Step 4 — Write the report

Use this exact structure. Every finding must include a `file:line` reference and a concrete Fix.

```
# Code Review Report
**Target**: <what was reviewed>
**Date**: <today's date>
**Language(s)**: <detected>
**Scope**: <N files reviewed / N lines added in diff>

## Summary

| Severity | Count |
|---|---|
| 🔴 Critical | N |
| 🟡 Warning | N |
| 🔵 Suggestion | N |

## Findings

### 🔴 CRITICAL — [Short category label]
**`path/to/file.ext:42`**
One-sentence description of the problem.
**Fix**: `corrected_code_snippet` or short explanation

### 🟡 WARNING — [Short category label]
**`path/to/file.ext:17`**
...

### 🔵 SUGGESTION — [Short category label]
**`path/to/file.ext:8`**
...

## What Looks Good
- Callout positive patterns already present (keep this brief — 2-4 bullets)

## Recommended Next Steps
Ordered action list from most to least critical.
```

### Severity guide

| Level | Use for |
|---|---|
| 🔴 Critical | Security vulnerabilities, credential leaks, crash bugs, data loss, memory leaks |
| 🟡 Warning | Best practice violations, potential bugs, performance issues, missing tests for critical paths |
| 🔵 Suggestion | Style, documentation, minor improvements, refactoring opportunities |

---

## Tips

- Always cite `file:line` — without it, findings are hard to act on.
- For each finding, lean toward a concrete fix snippet over a vague recommendation.
- If the diff is large (>400 lines), note it at the top: large diffs are harder to review thoroughly.
- Do not make network calls. Everything must be derivable from the local repository.
