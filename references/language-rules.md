# Language-Specific Rules Reference

These are **curated non-obvious rules** — patterns that linters often miss, that require context (framework, timezone, concurrency), or that are security-adjacent. Generic idioms (bare except, `var` → `const`, `== vs ===`, `print()` in production) are intentionally NOT listed — Claude's baseline reasoning and standard linters handle those.

Use this file to add depth *beyond* what `ruff`, `eslint`, `clippy`, `rubocop`, etc. already catch.

---

## Python

| Issue | Violation | Fix |
|---|---|---|
| Mutable default argument | `def f(items=[])` | `def f(items=None): ...` — shared state across calls is a common bug |
| `yaml.load()` without `Loader=` | `yaml.load(f)` | `yaml.safe_load(f)` — arbitrary code execution via crafted YAML |
| `pickle.loads` on untrusted data | `pickle.loads(user_data)` | Use JSON or a signed/validated serializer — RCE risk |
| Hardcoded env var fallback | `os.getenv("SECRET_KEY", "dev-secret")` | `os.environ["SECRET_KEY"]` — fail loudly rather than silently use a weak default |
| Non-constant-time secret comparison | `if token == expected:` | `hmac.compare_digest(token, expected)` — prevents timing attacks |
| File read before size check | `data = await file.read(); if len(data) > MAX:` | Check `Content-Length` first, or `file.read(MAX + 1)` — OOM/DoS risk |
| Unconstrained enum-like field (Pydantic) | `status: str` | `status: Literal["active", "inactive"]` or `Enum` |
| Privileged route missing role dep (FastAPI) | `@router.put("/settings/api-keys")` with only auth | `current_user: User = Depends(require_admin)` |
| Known unmaintained package | `python-jose`, `pycrypto` imported | Replace with `authlib`/`PyJWT`; `pycrypto` → `cryptography` |
| SSTI via `render_template_string` | `render_template_string(user_input)` | Use `render_template("file.html", value=user_input)` |
| JWT algorithm not pinned | `jwt.decode(token, key)` | `jwt.decode(token, key, algorithms=["HS256"])` — prevents `"alg": "none"` confusion |

---

## JavaScript / TypeScript

| Issue | Violation | Fix |
|---|---|---|
| `innerHTML` assignment from user input | `el.innerHTML = input` | `el.textContent = input`, or `DOMPurify.sanitize(input)` if HTML needed |
| N+1 awaits in a loop | `for (u of users) await fetch(u.id)` | `await Promise.all(users.map(u => fetch(u.id)))` |
| `eval()` on external input | `eval(userInput)` | Never; refactor to a parser or lookup table |
| `dangerouslySetInnerHTML` without sanitization (React) | `<div dangerouslySetInnerHTML={{__html: content}}>` | Sanitize `content` with DOMPurify or render as text |
| `useEffect` missing cleanup | `useEffect(() => { subscribe() }, [])` with no return | Return the unsubscribe fn: `return () => unsubscribe()` |
| Stale closure in `useEffect` / `useCallback` | deps array stale; referenced var doesn't re-trigger | Add to deps, or use `useRef`/functional `setState` |

---

## Go

| Issue | Violation | Fix |
|---|---|---|
| `fmt.Sprintf` in SQL | `fmt.Sprintf("SELECT ... %s", val)` | `db.Query("SELECT ... $1", val)` — parameterize |
| Goroutine closure capture in loop | `for _, i := range xs { go func() { use(i) }() }` | `go func(i int) { use(i) }(i)` |
| Concurrent map write | unsynchronized writes to `map` from multiple goroutines | `sync.RWMutex` or `sync.Map` |
| `defer` inside loop | `for ... { defer f.Close() }` | Move close outside loop or extract a helper |
| Goroutine leak | goroutine with no cancel/stop mechanism | Use `context.Context` with cancellation |

---

## Java / Kotlin

| Issue | Violation | Fix |
|---|---|---|
| Java: `equals`/`hashCode` contract broken | Overriding one without the other | Always override both (IDE generator or `Objects.hash(...)`) |
| Java: `Collections.unmodifiableList` still mutable at source | `return Collections.unmodifiableList(srcHeld)` | `return List.copyOf(src)` (Java 10+) for true immutability |
| Kotlin: `GlobalScope.launch` leak | `GlobalScope.launch { ... }` without cancellation | `viewModelScope.launch` / `lifecycleScope.launch` |
| Kotlin: cold `Flow` for UI state | `Flow<List<T>>` collected in UI resubscribes on recomposition | `.stateIn(scope, SharingStarted.WhileSubscribed(), default)` |
| Kotlin: `runBlocking` on main thread | `runBlocking { fetchData() }` on Android main | `lifecycleScope.launch(Dispatchers.IO) { fetchData() }` |

---

## Ruby

| Issue | Violation | Fix |
|---|---|---|
| Mass assignment (Rails) | `User.new(params)` | Strong parameters: `params.require(:user).permit(:name, :email)` |
| SQL injection via string | `User.where("name = '#{name}'")` | `User.where(name: name)` |
| Missing `dependent:` on `has_many` | `has_many :posts` | `has_many :posts, dependent: :destroy` (or `:nullify`) — orphan records otherwise |
| `Time.now` ignores Rails timezone | `Time.now` in a Rails app | `Time.current` or `Time.zone.now` |
| `update_attribute` skips validations | `user.update_attribute(:role, :admin)` | `user.update(role: :admin)` — or document why bypass is intentional |

---

## SQL

| Issue | Violation | Fix |
|---|---|---|
| String interpolation | `"SELECT ... WHERE id = " + id` | Parameterized/prepared statements |
| Missing index on FK | FK column with no index | `CREATE INDEX idx_orders_user_id ON orders(user_id)` |
| Leading wildcard defeats index | `WHERE name LIKE '%smith'` | Full-text search or reversed-string suffix index |
| `NOT IN` with nullable subquery | `WHERE id NOT IN (SELECT user_id FROM orders)` when `user_id` is nullable | `WHERE NOT EXISTS (SELECT 1 FROM orders ...)` |
| Multi-step write without transaction | Sequential `INSERT`/`UPDATE` with no `BEGIN`/`COMMIT` | Wrap in a transaction for atomicity |
| Composite index column order | Index `(status, id)` used in query filtering only on `id` | Put most-queried / most-selective column first |

---

## Shell / Bash

| Issue | Violation | Fix |
|---|---|---|
| Command injection via `eval` | `` eval `cmd $input` `` | Avoid `eval`; use arrays + `"${arr[@]}"` |
| Insecure temp file | `tmp=/tmp/myfile` (predictable, race-prone) | `tmp=$(mktemp)` |
| `curl \| sh` pattern | `curl https://... \| bash` | Download to file, verify checksum, then execute |
| Missing `--` before variable filenames | `rm $file` (fails if `$file` starts with `-`) | `rm -- "$file"` |

---

## Rust

| Issue | Violation | Fix |
|---|---|---|
| `unwrap()` / `expect()` in non-test paths | `value.unwrap()` | Propagate with `?` or handle explicitly |
| `unsafe` block without `// SAFETY:` comment | `unsafe { ... }` | Document the invariants being upheld |
| Integer overflow in release mode | Unchecked `+`/`*` on `usize`/`u32` | `checked_add`, `saturating_add`, or `wrapping_add` |
| Unhandled `Mutex` poison | `lock.unwrap()` after potential panic | Match on the `PoisonError`, decide recovery strategy |

---

## C / C++

| Issue | Violation | Fix |
|---|---|---|
| Unbounded string copy | `strcpy(buf, src)` / `gets(buf)` | `strncpy(buf, src, sizeof(buf)-1)` / `fgets` |
| Format string injection | `printf(user_input)` | `printf("%s", user_input)` |
| Use-after-free | Pointer used after `delete` / `free` | Smart pointers; null out pointer after free |
| Double-free | `delete ptr` called twice | `unique_ptr` / set to `nullptr` after free |
| `sprintf` without bounds | `sprintf(buf, fmt, val)` | `snprintf(buf, sizeof(buf), fmt, val)` |
| Signed integer overflow (UB) | `INT_MAX + 1` | `__builtin_add_overflow` or `<limits>` guards |

---

## PHP

| Issue | Violation | Fix |
|---|---|---|
| SQL injection via concatenation | `"SELECT ... WHERE id = " . $_GET['id']` | PDO prepared statements |
| XSS via direct output | `echo $_GET['name']` | `echo htmlspecialchars($_GET['name'], ENT_QUOTES, 'UTF-8')` |
| `eval()` on user input | `eval($_POST['code'])` | Never |
| Weak password hashing | `md5($password)` / `sha1($password)` | `password_hash($password, PASSWORD_BCRYPT)` |
| SSRF via user-supplied URL | `file_get_contents($_GET['url'])` | Validate against an allowlist of domains |
| Type juggling with `==` | `if ($token == 0)` allows `"0abc" == 0` → true | `if ($token === 0)` |
| Variable injection | `extract($_REQUEST)` | Never on user input |
| Unvalidated file upload | `move_uploaded_file(...)` without checks | Validate MIME, extension, store outside webroot |

---

## C# / .NET

| Issue | Violation | Fix |
|---|---|---|
| `async void` method (non-event handler) | `async void HandleClick(...)` | `async Task HandleClick(...)` — `async void` swallows exceptions |
| Missing `ConfigureAwait(false)` in library code | `await SomeAsync()` | `await SomeAsync().ConfigureAwait(false)` |
| `First()` NRE risk | `list.First(x => x.Id == id)` | `list.FirstOrDefault(...)` with null check |
| `DateTime.Now` stored as a timestamp | `DateTime.Now` written to DB | `DateTime.UtcNow` — timezone-independent |
| `Thread.Sleep` in async context | `Thread.Sleep(1000)` inside `async` method | `await Task.Delay(1000)` |

---

## Dart / Flutter

| Issue | Violation | Fix |
|---|---|---|
| `setState` after `dispose` (crash) | `setState(() { ... })` in async callback | `if (mounted) setState(...)` |
| Async work in `build()` | `Future.delayed(...)` inside `build` | Move to `initState` / controller |
| `BuildContext` across async gap | `context.push(...)` after `await` without check | `if (context.mounted) context.push(...)` |
| `initState` calling `async` directly without catch | `initState() { fetchData(); }` | `.then((v) { if (mounted) setState(...); }).catchError(...)` |

---

## Dockerfile / YAML / Terraform

| Issue | Violation | Fix |
|---|---|---|
| Secrets baked into image | `ENV SECRET=abc123` | Build args + secrets mount; never bake |
| `latest` tag | `FROM node:latest` | Pin to digest or explicit version |
| Running as root | No `USER` directive | `USER node` (or any non-root) |
| Overly permissive IAM | `"Action": "*"`, `"Resource": "*"` | Least-privilege policy |
| `privileged: true` in pod spec | K8s/Docker privileged container | Remove unless absolutely required |
| Hardcoded credentials in YAML | `password: hunter2` | K8s Secrets / Vault references |

---

## What was intentionally removed

Generic style and obvious-idiom rules were pruned because linters and Claude's baseline reasoning already catch them. Examples of removed rules:

- Python: bare `except:`, missing type hints, `print()` in production, wildcard imports
- JS/TS: `var` → `const`, `== vs ===`, missing `await`, `any` type, missing React `key`
- Go: ignored error return, `panic` vs error (go vet catches)
- Java/Kotlin: swallowed exception, `!!` operator, missing `data class`
- Ruby: N+1 ActiveRecord (brakeman catches), `rescue` without type, `Symbol#to_proc`, `||=` on falsy
- SQL: `SELECT *`, missing `LIMIT`
- Shell: unquoted variables, missing `set -euo pipefail`, backticks (shellcheck catches)
- Dart: missing `const`, uncaught `Future`, deep widget nesting

If your project's linters aren't configured to catch these, add them in `.ruff.toml` / `.eslintrc` / `rubocop.yml` etc. — that's the right layer for style enforcement.
