# Git Intelligence Rules

Reading source code tells you what's *there*. Reading git history tells you what's **risky**. This file documents how to extract repo-health signals that no single-file inspection can surface.

All analysis uses local `git` commands — no remote API calls.

---

## When to run these checks

- **On `staged` / branch reviews**: run a narrow version — analyse only files in the diff. Don't derail into whole-repo archaeology on a small commit.
- **On `all` reviews**: run the full analysis — this is where the repo-wide signal is most useful.
- **On large codebases (>200 files)**: cap the analysis to the last 6 months of history (`--since="6 months ago"`) to keep runtime reasonable.

---

## 1. Churn × Complexity Hotspots

Files that are **both frequently changed AND complex** are technical-debt hotspots — the most valuable refactor targets.

### Churn signal

```bash
git log --since="6 months ago" --format=format: --name-only \
  | grep -v '^$' \
  | sort | uniq -c | sort -rn | head -20
```

Output: each file's change count over the last 6 months.

### Complexity signal

Use whichever tool is available (match to the language):

| Language | Tool | Command |
|---|---|---|
| Python | `radon` | `radon cc -s -a path/` |
| JS/TS | `eslint` (complexity rule) | ESLint with `complexity: ["error", 10]` |
| Go | `gocyclo` | `gocyclo -top 20 .` |
| Java | `lizard` (language-agnostic) | `lizard path/` |
| Any | `lizard` | `lizard --CCN 10 .` |

### Combine

Rank files by `churn × cyclomatic_complexity`. Top results are the **hotspots**.

### Severity mapping

| File is in diff? | Rank in hotspot list | Severity |
|---|---|---|
| Yes | Top 3 | 🟡 Warning — "Changing a high-churn-high-complexity file; consider splitting" |
| Yes | Top 10 | 🔵 Suggestion |
| No | (informational only, omit unless `all` review) | — |

Include a note: "This file has been changed N times in the last 6 months and has cyclomatic complexity X. Consider refactoring before adding more."

---

## 2. Ownership / Bus Factor

Files that only one person has ever touched are **knowledge silos** — risky when that person is unavailable.

### Command

```bash
git log --format='%an' -- path/to/file.py | sort -u
```

Output: unique list of authors who've touched the file.

### Severity mapping

| Authors count | Severity |
|---|---|
| 1 author, file is >100 lines and <1 year old | 🔵 Suggestion — "Single author; consider knowledge-sharing via code review or docs" |
| 1 author, file is core infrastructure (`src/core/`, `src/auth/`, `src/billing/`) | 🟡 Warning — "Critical module with single author; bus-factor risk" |
| 2+ authors | (no finding) |

Only flag files in the reviewed diff. Don't flag every single-author file in the repo.

---

## 3. Temporal Coupling

Files that **change together** but live in different modules suggest hidden coupling — a refactor target.

### Command

For a given file, find files changed in the same commits:

```bash
git log --format='%H' -- src/foo.py \
  | xargs -I {} git show --format= --name-only {} \
  | sort | uniq -c | sort -rn | head -10
```

Interpret: if `src/foo.py` and `src/bar.py` change together in >70% of `src/foo.py`'s commits, they're coupled.

### Severity mapping

| Ratio | Location | Severity |
|---|---|---|
| >70% | Same module/package | (normal — related code) |
| >70% | Different modules | 🔵 Suggestion — "Files change together despite being in different modules; consider consolidating" |
| >90% | Different modules | 🟡 Warning — "Strong hidden coupling" |

---

## 4. Dead File Candidates

Files with zero commits in 2+ years **and** zero inbound imports are likely dead code.

### Command

```bash
# Files not touched in 2 years
git log --all --pretty=format: --name-only --since="2 years ago" \
  | sort -u > /tmp/recent_files.txt

git ls-files > /tmp/all_files.txt
comm -23 /tmp/all_files.txt /tmp/recent_files.txt
```

### Inbound imports check

For each candidate dead file, grep for imports:

```bash
# Python
grep -rn "from <module>" --include="*.py" .
grep -rn "import <module>" --include="*.py" .

# JS/TS
grep -rn "from ['\"].*<name>" --include="*.{ts,tsx,js,jsx}" .

# Go
grep -rn "\"<module_path>\"" --include="*.go" .
```

If no inbound references → 🔵 Suggestion: "Possible dead file: last touched N years ago, no inbound imports found."

**Do not auto-delete** — these findings require human verification. Tests and dynamic imports can hide references.

---

## 5. Commit Hygiene (advisory, lightweight)

Only surface findings from Step 1 (the reviewed diff / commit range):

### Check for

- **WIP / fixup commits left in history**: commits with messages like `WIP`, `fixup!`, `squash!`, `tmp`, `asdf`
- **Zero-message commits**: empty or single-word messages
- **Large commits**: single commits touching >500 lines (suggests they should be split)

### Command

```bash
git log --oneline <range> | grep -iE '^\S+ (wip|fixup|squash|tmp|asdf|debug|test)$'
git log --format='%H %s' <range> | awk '$2 == ""'
git log --stat --format='%H' <range> | ...
```

### Severity

- Advisory only — 🔵 Suggestion. Users may prefer to squash before PR.

---

## 6. Scope-Aware Execution

Always scope git-intelligence to what the user asked for:

| Review target | Git intelligence scope |
|---|---|
| `staged` / `unstaged` | Only files in diff; hotspot rank matters only for those files |
| `<branch>` | Files touched in branch vs main; hotspot/coupling only for those files |
| `<hash>..<hash>` | Files in range |
| Single file | Just that file's history + coupling |
| `all` / directory | Full repo archaeology (most useful case for this section) |

Do not output hotspot / coupling findings for files **not in the review target** — users find that noisy and confusing.

---

## 7. Performance Guardrails

Git archaeology on large repos is slow. Protect against:

- Repos with >10k commits: use `--since="6 months ago"` universally
- Repos with >1000 files: sample rather than enumerate (top 20 by churn, not all)
- Timeout per command: 30 seconds. If exceeded, skip that check and note it in the report.

---

## Reporting format

Include a dedicated "Hotspot notes" section in the report:

```
## Hotspot notes

- **`src/billing/payment.py`** — hotspot: 14 changes in 6 months × complexity 32.
  Consider splitting. (If you're changing it now, aim to reduce complexity rather than add to it.)

- **`src/core/auth.py`** — bus-factor risk: only one author in 2 years of history.
  Worth getting a second reviewer familiar with this module.

- **`src/api/users.py` ↔ `src/api/orders.py`** — temporal coupling (82% co-change).
  Despite being in separate endpoints, they always change together. Possible refactor target.
```

Keep it short — 3-5 bullets maximum unless the user asked for a deep architecture review.
