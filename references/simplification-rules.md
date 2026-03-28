# Code Simplification & Readability Rules

## Language-Agnostic Patterns

These apply across all languages — flag as 🔵 Suggestion unless they mask a bug (then 🟡 Warning).

| Issue | Violation | Fix |
|---|---|---|
| Redundant boolean return | `if cond: return True` / `else: return False` | `return cond` |
| Redundant boolean comparison | `if x == True:` / `if x == False:` | `if x:` / `if not x:` |
| Unnecessary variable before return | `result = compute(); return result` | `return compute()` |
| Nested `if` instead of guard clause | Deep nesting for happy path | Invert condition and return/raise early |
| Unnecessary `else` after `return`/`raise` | `if cond: return x; else: return y` | `if cond: return x` / `return y` |
| Magic number / magic string | `if status == 3:` / `time.sleep(86400)` | Named constant: `PENDING = 3` / `SECONDS_PER_DAY = 86400` |
| Too many function parameters | `def f(a, b, c, d, e, f)` (>4 params) | Group into a config dataclass/struct/object |
| Function violating SRP | One function fetches, transforms, validates, and saves | Split into focused single-purpose functions |
| Double negation | `if not not x:` / `if not (a != b):` | `if x:` / `if a == b:` |
| Comparing to `None` with `==` | `if x == None:` | `if x is None:` (identity check, not equality) |

---

## Python

| Issue | Violation | Fix |
|---|---|---|
| `len()` check for emptiness | `if len(items) == 0:` / `if len(items) > 0:` | `if not items:` / `if items:` |
| `range(len())` loop | `for i in range(len(items)): use(items[i])` | `for item in items:` or `for i, item in enumerate(items):` |
| `lambda` in `map`/`filter` | `map(lambda x: x.name, items)` | `[item.name for item in items]` (comprehension) |
| Loop building a list | `result = []; for x in xs: result.append(f(x))` | `result = [f(x) for x in xs]` |
| Loop building a dict | `d = {}; for k, v in pairs: d[k] = v` | `d = {k: v for k, v in pairs}` |
| Manual `any`/`all` loop | `for x in xs: if pred(x): return True; return False` | `return any(pred(x) for x in xs)` |
| Old-style string formatting | `"Hello %s" % name` / `"Hello {}".format(name)` | `f"Hello {name}"` |
| `os.path` instead of `pathlib` | `os.path.join(base, "sub", "file.txt")` | `Path(base) / "sub" / "file.txt"` |
| `dict()` / `list()` literal constructors | `d = dict()` / `l = list()` | `d = {}` / `l = []` |
| Manual parallel index loop | `for i in range(len(a)): use(a[i], b[i])` | `for x, y in zip(a, b):` |
| `try/except/else` missing `else` | Logic after `try` block still inside `try` | Move non-exception-raising code to `else` clause |
| `dict.get()` with manual default | `val = d[k] if k in d else default` | `val = d.get(k, default)` |
| `collections.defaultdict` opportunity | Repeated `if key not in d: d[key] = []` | `d = defaultdict(list)` |
| Unpacking opportunity | `x = pair[0]; y = pair[1]` | `x, y = pair` |

---

## JavaScript / TypeScript

| Issue | Violation | Fix |
|---|---|---|
| Manual null/undefined chain | `a && a.b && a.b.c` | `a?.b?.c` (optional chaining) |
| Manual nullish default | `val !== null && val !== undefined ? val : def` | `val ?? def` (nullish coalescing) |
| `filter` + `map` two passes | `arr.filter(pred).map(fn)` | Single `reduce` or `flatMap` when the two passes are heavy |
| Unnecessary array spread | `[...arr].map(fn)` (no mutation needed) | `arr.map(fn)` |
| `!!x` for boolean cast | `const flag = !!value` | `const flag = Boolean(value)` (explicit intent) |
| String concatenation | `"Hello " + name + "!"` | `` `Hello ${name}!` `` |
| `Object.assign({}, obj)` shallow clone | `const copy = Object.assign({}, obj)` | `const copy = { ...obj }` |
| Callback-style promise | `.then(res => { ... }).then(val => { ... })` | `async/await` for readability |
| `Array.prototype.indexOf` existence check | `arr.indexOf(x) !== -1` | `arr.includes(x)` |
| Manual object iteration | `Object.keys(obj).forEach(k => use(k, obj[k]))` | `Object.entries(obj).forEach(([k, v]) => use(k, v))` |
| `var` in module scope | `var x = ...` | `const` / `let` |
| Redundant `return` in arrow function | `const f = x => { return x * 2; }` | `const f = x => x * 2` |
| TypeScript: unnecessary type assertion | `x as string` when type is already `string` | Remove assertion |
| TypeScript: `interface` with single implementor | Overly abstract interface used only once | Inline the type or use a `type` alias |

---

## Go

| Issue | Violation | Fix |
|---|---|---|
| String comparison for error type | `if err.Error() == "not found"` | `if errors.Is(err, ErrNotFound)` |
| Unwrapped error | `return fmt.Errorf("failed: %s", err)` | `return fmt.Errorf("failed: %w", err)` (wrap for `errors.Is`) |
| String concatenation in loop | `s += part` in loop | `var sb strings.Builder; sb.WriteString(part)` |
| `len(slice) == 0` check | `if len(items) == 0` | `if len(items) == 0` is idiomatic in Go — flag only if `items == nil` check is also needed |
| Repeated test structure | Near-identical test functions | Table-driven test: `tests := []struct{input, want}{...}` |
| Named return used inconsistently | Named returns declared but not used in all paths | Either use consistently or remove |
| `interface{}` / `any` overuse | `func f(v interface{})` for a known type | Use the concrete type or a constrained generic |
| Unnecessary pointer to small value | `*bool`, `*int` for optional → nil check | Use a wrapper struct or zero-value sentinel if semantics allow |

---

## Ruby

| Issue | Violation | Fix |
|---|---|---|
| `each` + `push` instead of `map` | `result = []; items.each { \|i\| result << f(i) }` | `result = items.map { \|i\| f(i) }` |
| `select` + `first` instead of `find` | `items.select { \|i\| pred(i) }.first` | `items.find { \|i\| pred(i) }` |
| `map` + `compact` instead of `filter_map` | `items.map { \|i\| maybe(i) }.compact` | `items.filter_map { \|i\| maybe(i) }` (Ruby 2.7+) |
| Manual presence check | `if str && !str.empty?` | `if str.present?` (Rails) or `if str&.then { \|s\| !s.empty? }` |
| Ternary with block | `condition ? do_a : do_b` where branches are multi-line | Full `if/else` for readability |
| String interpolation in non-interpolating string | `'Hello #{name}'` (single quotes — interpolation won't work) | `"Hello #{name}"` |

---

## SQL

| Issue | Violation | Fix |
|---|---|---|
| Subquery where `JOIN` suffices | `SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE ...)` | Rewrite as `JOIN` — often better optimised |
| `HAVING` without `GROUP BY` | `SELECT ... HAVING COUNT(*) > 1` with no `GROUP BY` | Add `GROUP BY` or use `WHERE` if no aggregation needed |
| Redundant `DISTINCT` | `SELECT DISTINCT` when result set is already unique by key | Remove `DISTINCT` — masks missing `JOIN` conditions |
| `OR` on indexed column | `WHERE status = 'a' OR status = 'b'` | `WHERE status IN ('a', 'b')` — cleaner and often better planned |
| Correlated subquery in `SELECT` | `SELECT (SELECT name FROM ...) FROM ...` per-row | Rewrite as `LEFT JOIN` |
