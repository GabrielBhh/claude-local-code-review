# Language-Specific Rules Reference

## Python

| Issue | Violation | Fix |
|---|---|---|
| Bare except | `except:` | `except Exception as e:` |
| Mutable default argument | `def f(items=[])` | `def f(items=None): if items is None: items = []` |
| `print()` in production code | `print("debug")` | `logger.debug("debug")` |
| Missing type hints on public API | `def process(data):` | `def process(data: list[str]) -> dict:` |
| `except Exception` without re-raise | catch-and-ignore pattern | Log and re-raise, or return typed error |
| String interpolation in SQL | `f"SELECT ... {val}"` | Parameterized queries |
| `yaml.load()` without Loader | `yaml.load(f)` | `yaml.safe_load(f)` |
| `pickle` of untrusted data | `pickle.loads(user_data)` | Use JSON or signed/validated serialization |
| Wildcard imports | `from module import *` | Explicit imports |
| Global mutable state | module-level `list` or `dict` mutated by functions | Encapsulate in class or pass as argument |

**Complexity thresholds**: Functions >20 lines → warn. Nesting >3 levels → warn.

---

## JavaScript / TypeScript

| Issue | Violation | Fix |
|---|---|---|
| `var` declaration | `var x = 1` | `const x = 1` / `let x = 1` |
| Loose equality | `x == null` | `x === null` |
| Missing `await` | `const r = fetchData()` | `const r = await fetchData()` |
| `any` type | `function f(x: any)` | Proper interface/type |
| `innerHTML` assignment | `el.innerHTML = input` | `el.textContent = input` or DOMPurify |
| N+1 in loop | `for (u of users) await fetch(u.id)` | `await Promise.all(users.map(...))` |
| `console.log` left in | `console.log("debug")` | Remove or use proper logger |
| Unhandled promise rejection | `.then(cb)` without `.catch` | `.then(cb).catch(err => ...)` |
| `eval()` usage | `eval(userInput)` | Never use eval with external input |
| Missing `key` prop in React | `<Item>` in list | `<Item key={item.id}>` |

**TypeScript specifics**: Avoid `as any` casts; prefer `unknown` with a type guard when the shape is truly unknown.

---

## Go

| Issue | Violation | Fix |
|---|---|---|
| Ignored error return | `f, _ := os.Open(...)` | `f, err := os.Open(...); if err != nil { ... }` |
| `fmt.Sprintf` in SQL | `fmt.Sprintf("SELECT ... %s", val)` | `db.Query("SELECT ... $1", val)` |
| Goroutine closure capture | `go func() { use(i) }()` in loop | `go func(i int) { use(i) }(i)` |
| Concurrent map write | unsynchronized `map` writes from goroutines | `sync.RWMutex` or `sync.Map` |
| `defer` inside loop | `defer f.Close()` inside `for` loop | Move close outside loop or use helper func |
| Goroutine leak | goroutine with no cancel/stop mechanism | Use `context.Context` with cancellation |
| Panic instead of error return | `panic("something went wrong")` | Return `error` to caller |
| Naked `return` in long function | `return` in 20+ line named-return function | Explicit return values |

---

## Java / Kotlin

| Issue | Violation | Fix |
|---|---|---|
| Swallowed checked exception | `catch (Exception e) {}` | Log and rethrow, or handle meaningfully |
| `NullPointerException` risk (Java) | No null check before dereference | Use `Optional<T>` or null checks |
| Resource leak | `new FileInputStream(...)` without try-with-resources | `try (var fis = new FileInputStream(...)) { }` |
| `System.out.println` in production | Debug prints | Use SLF4J / Logback |
| Kotlin: `!!` operator | `user!!.name` | Safe call `user?.name` or elvis `user?.name ?: ""` |
| Kotlin: missing `data class` | Plain class for value objects | `data class Point(val x: Int, val y: Int)` |
| Java: broken `equals`/`hashCode` contract | Overriding `equals` without `hashCode` (or vice versa) | Always override both — IDE generators or `Objects.hash(...)` / Lombok `@EqualsAndHashCode` |
| Java: `Collections.unmodifiableList` still mutable at source | `List<T> src = new ArrayList<>(); return Collections.unmodifiableList(src);` (caller holds `src`) | `return List.copyOf(src)` (Java 10+) for a truly immutable snapshot |
| Kotlin: `GlobalScope.launch` coroutine leak | `GlobalScope.launch { ... }` with no cancellation | Use `viewModelScope`, `lifecycleScope`, or a structured `CoroutineScope` tied to component lifecycle |
| Kotlin: cold `Flow` for UI state | `val items: Flow<List<T>>` collected in UI — resubscribes on every recomposition | Convert with `.stateIn(scope, SharingStarted.WhileSubscribed(), emptyList())` → `StateFlow` |
| Kotlin: `runBlocking` on main/UI thread | `runBlocking { fetchData() }` on Android main thread | Use `lifecycleScope.launch` or `viewModelScope.launch` with `Dispatchers.IO` |

---

## Ruby

| Issue | Violation | Fix |
|---|---|---|
| N+1 ActiveRecord | `User.all.map { |u| u.posts }` | `User.includes(:posts).all` |
| Mass assignment | `User.new(params)` | Strong parameters: `params.require(:user).permit(:name, :email)` |
| SQL injection via string | `User.where("name = '#{name}'")` | `User.where(name: name)` |
| `rescue` without type | `rescue => e` (too broad) | `rescue ActiveRecord::RecordNotFound => e` |
| Missing `frozen_string_literal` | No magic comment at top | `# frozen_string_literal: true` |
| Verbose block where `Symbol#to_proc` suffices | `users.map { \|u\| u.name }` | `users.map(&:name)` |
| `\|\|=` memoization on falsy value | `@flag \|\|= false` (re-evaluates every call) | `@flag = compute if @flag.nil?` |
| Missing `dependent:` on `has_many` | `has_many :posts` (no dependent option) | `has_many :posts, dependent: :destroy` (or `:nullify`) |
| `Time.now` ignores Rails timezone | `Time.now` in Rails app | `Time.current` or `Time.zone.now` |
| `update_attribute` skips validations | `user.update_attribute(:role, :admin)` | `user.update(role: :admin)` — runs validations; add comment if bypass is intentional |

---

## SQL

| Issue | Violation | Fix |
|---|---|---|
| String interpolation | `"SELECT ... WHERE id = " + id` | Parameterized / prepared statements |
| `SELECT *` | `SELECT * FROM users` | Select only needed columns |
| Missing index on FK | FK column with no index | `CREATE INDEX idx_orders_user_id ON orders(user_id)` |
| No `LIMIT` on user queries | Unbounded `SELECT` | Add `LIMIT N` |
| Implicit type coercion | Comparing int column to string | Match types explicitly |
| Leading wildcard defeats index | `WHERE name LIKE '%smith'` | Use full-text search (`MATCH`/`@@`) or store reversed string for suffix search |
| `NOT IN` with nullable subquery | `WHERE id NOT IN (SELECT user_id FROM orders)` (user_id nullable) | `WHERE NOT EXISTS (SELECT 1 FROM orders WHERE orders.user_id = users.id)` |
| Multi-step write without transaction | Sequential `INSERT`/`UPDATE` with no `BEGIN`/`COMMIT` | Wrap in a transaction so partial failure doesn't leave inconsistent state |
| `COUNT(col)` vs `COUNT(*)` confusion | `COUNT(nullable_col)` when intent is row count | Use `COUNT(*)` for row count; `COUNT(col)` intentionally skips NULLs — be explicit |
| Composite index column order wrong | Index `(status, id)` used in query filtering only on `id` | Put the most-queried / most-selective column first: `(id, status)` |

---

## Shell / Bash

| Issue | Violation | Fix |
|---|---|---|
| Unquoted variable | `rm $file` | `rm -- "$file"` |
| Missing `set -euo pipefail` | Script continues after error or unbound variable | Add `set -euo pipefail` at top |
| Pipefail not set separately | `false \| true` exits 0 without `pipefail` | `set -o pipefail` catches failures in pipelines |
| Command injection via `eval` | `` eval `cmd $input` `` | Avoid `eval`; use arrays and `"${arr[@]}"` expansion |
| Insecure temp file | `tmp=/tmp/myfile` (predictable, race-prone) | `tmp=$(mktemp)` |
| `curl \| sh` pattern | `curl https://... \| bash` | Download to file, verify checksum, then execute |
| Backtick command substitution | `` result=`cmd` `` | `result=$(cmd)` — nestable and more readable |
| `[` instead of `[[` in bash | `[ $var = "x" ]` — word-splits on spaces | `[[ $var = "x" ]]` in bash scripts |
| `IFS` changed globally | `IFS=:` at script level breaks subsequent word splitting | Scope the change: `(IFS=:; read -ra parts <<< "$str")` |
| Missing `--` before variable filenames | `rm $file` silently fails if `$file` starts with `-` | `rm -- "$file"` |

---

## Rust

| Issue | Violation | Fix |
|---|---|---|
| `unwrap()` / `expect()` in non-test code | `value.unwrap()` | `value.unwrap_or_else(\|e\| ...)` or propagate with `?` |
| `unsafe` block without comment | `unsafe { ... }` (no explanation) | Add a `// SAFETY:` comment justifying invariants |
| `clone()` on large types in hot path | `big_vec.clone()` in loop | Use references or `Cow<T>` |
| Missing `?` propagation | Manual `match` on `Result` where `?` suffices | Use `?` operator |
| `todo!()` / `unimplemented!()` in production path | `todo!("handle this")` | Implement or return a typed error |
| Unhandled `Mutex` poison | `lock.unwrap()` after potential panic | Match on `lock.lock()` and handle `PoisonError` |
| Integer overflow in release mode | Unchecked arithmetic on `usize`/`u32` | Use `checked_add`, `saturating_add`, or `wrapping_add` explicitly |
| `std::mem::forget` / manual `ManuallyDrop` | Bypassing drop without clear safety justification | Document invariants or use RAII guards |

---

## C / C++

| Issue | Violation | Fix |
|---|---|---|
| Unbounded string copy | `strcpy(buf, src)` / `gets(buf)` | `strncpy(buf, src, sizeof(buf)-1)` / `fgets` |
| Format string injection | `printf(user_input)` | `printf("%s", user_input)` |
| Memory leak | `malloc` / `new` without corresponding `free` / `delete` | RAII wrappers (`std::unique_ptr`, `std::vector`) |
| Use-after-free | Pointer used after `delete` / `free` | Null pointer after free; prefer smart pointers |
| Double-free | `delete ptr` called twice | Set `ptr = nullptr` after free; use `unique_ptr` |
| Null pointer dereference | Pointer used without null check | Check before dereference or use references |
| Signed integer overflow (UB) | `INT_MAX + 1` | Use `unsigned`, `__builtin_add_overflow`, or `<limits>` guards |
| Missing `const` on pointer params | `void f(char* s)` (read-only) | `void f(const char* s)` |
| `sprintf` without bounds | `sprintf(buf, fmt, val)` | `snprintf(buf, sizeof(buf), fmt, val)` |
| C++: raw owning pointer in new code | `Foo* p = new Foo()` | `auto p = std::make_unique<Foo>()` |

---

## PHP

| Issue | Violation | Fix |
|---|---|---|
| SQL injection via concatenation | `"SELECT ... WHERE id = " . $_GET['id']` | PDO prepared statements: `$stmt->execute([$_GET['id']])` |
| XSS via direct output | `echo $_GET['name']` | `echo htmlspecialchars($_GET['name'], ENT_QUOTES, 'UTF-8')` |
| `eval()` on user input | `eval($_POST['code'])` | Never eval user input |
| Weak password hashing | `md5($password)` / `sha1($password)` | `password_hash($password, PASSWORD_BCRYPT)` |
| SSRF via user-supplied URL | `file_get_contents($_GET['url'])` | Validate against allowlist of domains |
| Type juggling with `==` | `if ($token == 0)` (loose) | `if ($token === 0)` (strict) |
| Variable injection | `extract($_REQUEST)` | Never use `extract` on user input |
| Unvalidated file upload | `move_uploaded_file($_FILES['f']['tmp_name'], $dest)` | Validate MIME type, extension, and store outside webroot |
| `$_GET`/`$_POST` used without validation | Direct use in business logic | Sanitize and validate all superglobals |
| Error display in production | `display_errors = On` in `php.ini` | `display_errors = Off`; log to file only |

---

## C# / .NET

| Issue | Violation | Fix |
|---|---|---|
| `async void` method | `async void HandleClick(...)` | `async Task HandleClick(...)` — `async void` swallows exceptions |
| Missing `ConfigureAwait(false)` in library | `await SomeAsync()` in library code | `await SomeAsync().ConfigureAwait(false)` |
| `IDisposable` not in `using` | `var conn = new SqlConnection(...); conn.Open()` | `using var conn = new SqlConnection(...)` |
| String concat in loop | `result += item` in `foreach` | `var sb = new StringBuilder(); sb.Append(item)` |
| `First()` NRE risk | `list.First(x => x.Id == id)` | `list.FirstOrDefault(...)` with null check |
| Hardcoded connection string | `"Server=prod;Password=secret"` in source | Use `IConfiguration` / environment variables / secrets manager |
| `DateTime.Now` instead of UTC | `DateTime.Now` stored in DB | `DateTime.UtcNow` |
| `Thread.Sleep` in async context | `Thread.Sleep(1000)` inside `async` method | `await Task.Delay(1000)` |
| Catching `Exception` at top level silently | `catch (Exception) { }` | Log and rethrow, or handle specific exception types |
| SQL string interpolation | `$"SELECT ... WHERE id = {id}"` | `SqlCommand` with parameters |

---

## Dart / Flutter

| Issue | Violation | Fix |
|---|---|---|
| `setState` after `dispose` | `setState(() { ... })` in async callback | Guard with `if (mounted) setState(...)` |
| Async work in `build()` | `Future.delayed(...)` called inside `build` | Move to `initState`, `didChangeDependencies`, or `.then` on a controller |
| Missing `const` constructor | `Text("Hello")` in stateless widget | `const Text("Hello")` — enables widget caching |
| `print()` in production code | `print("debug value")` | `debugPrint(...)` or a proper logging package |
| Uncaught `Future` | `someAsync()` (no `await`, no `.catchError`) | `await someAsync()` or `.catchError((e) => ...)` |
| `BuildContext` across async gap | `context.push(...)` after `await` without mounted check | `if (context.mounted) context.push(...)` |
| Deeply nested widget tree | Widget tree >5 levels deep in one `build` | Extract sub-widgets into named `Widget` methods or classes |
| `initState` calling `async` directly | `initState() { fetchData(); }` without handling errors | Use `.then((v) { if (mounted) setState(...); }).catchError(...)` |

---

## Dockerfile / YAML / Terraform

| Issue | Violation | Fix |
|---|---|---|
| Secrets in image | `ENV SECRET=abc123` | Use build args + secrets mount, never bake in |
| `latest` tag | `FROM node:latest` | Pin to digest or specific version: `node:20.11-alpine` |
| Running as root | No `USER` instruction | `USER node` (non-root user) |
| Overly permissive IAM | `"Action": "*"`, `"Resource": "*"` | Least-privilege policy |
| `privileged: true` in pod spec | K8s/Docker privileged container | Remove unless absolutely required |
| Hardcoded credentials in YAML | `password: hunter2` | Use Kubernetes Secrets or Vault references |
