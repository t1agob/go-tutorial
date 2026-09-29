# 02 — Domain: the `Todo` type and its rules

## Goal

Build the heart of the application: a `Todo` type in `internal/todo`, a constructor that enforces the title rules, and a `Complete` method. It's pure Go, with no HTTP, no database, and no logging, and it's fully tested.

Time: about 1 hour.

## Concepts

### Structs and constructors

Go has no classes or constructors in the language. You define a struct, then write an ordinary function, conventionally called `NewXxx`, that builds a valid one:

```go
type Money struct {
    Cents    int64
    Currency string
}

func NewMoney(cents int64, currency string) (Money, error) {
    if currency == "" {
        return Money{}, ErrNoCurrency
    }
    return Money{Cents: cents, Currency: currency}, nil
}
```

Notes:

- `Money{Cents: ..., Currency: ...}` is a **composite literal**. Always name the fields; it survives new fields being added.
- On error, return the **zero value** (`Money{}`) along with the error. By convention, callers don't use the first value when `err != nil`.
- Returning a value (`Money`) rather than a pointer (`*Money`) is fine for small structs. Our `Todo` will be returned by value.

### Errors are values

Go has no exceptions. A function that can fail returns an `error` as its **last** result, and the caller checks it straight away:

```go
m, err := NewMoney(100, "")
if err != nil {
    return err // or handle it
}
```

`error` is just an interface with one method, `Error() string`.

### Sentinel errors and `errors.Is`

A **sentinel error** is a package-level variable that callers can compare against:

```go
var ErrNoCurrency = errors.New("currency must not be empty")
```

Callers check for it with `errors.Is`:

```go
if errors.Is(err, money.ErrNoCurrency) { ... }
```

Why `errors.Is(err, X)` instead of `err == X`? Errors are often **wrapped** with extra context as they travel up the stack:

```go
return fmt.Errorf("parsing invoice %s: %w", id, err)   // %w wraps err
```

`==` fails on a wrapped error, but `errors.Is` looks through the wrapping. Make `errors.Is` your habit from day one. You'll wrap errors for real in stage 7.

> **If you remember old Go…** comparing `err.Error()` strings, or using `github.com/pkg/errors` for wrapping, has been replaced by the standard library's `%w`, `errors.Is`, and `errors.As` (since Go 1.13).

In tests, check errors the same way:

```go
if !errors.Is(err, tt.wantErr) {
    t.Errorf("NewMoney(...) error = %v, want %v", err, tt.wantErr)
}
```

This also works when `tt.wantErr` is `nil`: `errors.Is(nil, nil)` is `true`.

### Methods, and value vs pointer receivers

A method is a function with a **receiver**:

```go
func (m Money) IsZero() bool   { return m.Cents == 0 }  // value receiver: works on a copy
func (m *Money) Add(c int64)   { m.Cents += c }         // pointer receiver: can modify the original
```

Rules of thumb:

- If the method **changes** the struct, use a pointer receiver.
- If the struct is large, or holds something that mustn't be copied (such as a mutex), use a pointer receiver.
- Otherwise a value receiver is fine. Many teams pick one style per type and stick to it.

You can call a pointer method on an addressable value, `m.Add(5)`, and Go takes `&m` for you.

### Don't call `time.Now()` in your domain

If `NewTodo` called `time.Now()` itself, your tests couldn't know what `CreatedAt` should be. Instead, **pass the time in**:

```go
func NewTodo(id, title string, now time.Time) (Todo, error)
```

Tests pass a fixed time, and production code passes `time.Now()`. This is **dependency injection** in its simplest form: a plain parameter. In stage 3, the service will hold a `now func() time.Time` so that callers don't have to pass it on every call.

For the same reason, the ID is passed in too. Generating IDs is the service's job, in stage 3.

Useful `time` pieces:

```go
t0 := time.Date(2026, 9, 29, 10, 0, 0, 0, time.UTC)  // a fixed instant
t1 := t0.Add(time.Hour)
t1.Equal(t0)                                         // compare instants: use Equal, not ==
```

`==` on `time.Time` compares internal fields (including the location and the monotonic clock reading). Two values that represent the *same instant* can differ in those fields. **Always use `Equal`.**

### Optional values: `*time.Time` vs zero `time.Time`

`CompletedAt` only makes sense once a todo is done. There are two common ways to model that:

| Option | "Not completed" looks like | Pros | Cons |
| --- | --- | --- | --- |
| `CompletedAt *time.Time` | `nil` | "Absent" is explicit; maps cleanly to JSON `null` or an omitted field, and to SQL `NULL` | Pointers need `nil` checks |
| `CompletedAt time.Time` | `time.Time{}` (check with `.IsZero()`) | No pointers | "Zero" can be mistaken for a real value; JSON `omitempty` doesn't omit it |

**This guide recommends `*time.Time`**, and later stages assume it. If you choose the other option, later hints will need small adjustments. Tell Claude in your review and the guides will be adapted.

Taking the address of a parameter is safe in Go, because the value escapes to the heap automatically:

```go
func (e *Event) Close(at time.Time) { e.ClosedAt = &at }
```

### Strings are bytes; characters are runes

`len(s)` counts **bytes**, not characters. `"é"` is 2 bytes in UTF-8. If the rule is "at most 200 characters", count **runes**:

```go
len("héllo")                      // 6
utf8.RuneCountInString("héllo")   // 5
```

`strings.TrimSpace` removes leading and trailing whitespace, including tabs and newlines.

## Steps

What you're building, in `internal/todo/todo.go` (`package todo`):

```go
type Todo struct {
    ID          string
    Title       string
    Completed   bool
    CreatedAt   time.Time
    CompletedAt *time.Time
}

const MaxTitleLength = 200

var ErrEmptyTitle, ErrTitleTooLong, ErrNotFound // sentinel errors (write them out properly)

func NewTodo(id, title string, now time.Time) (Todo, error)
func (t *Todo) Complete(now time.Time)
```

The rules:

1. The title is trimmed of surrounding whitespace before it's stored.
2. An empty title after trimming returns `ErrEmptyTitle`.
3. A title of more than `MaxTitleLength` **characters** (runes) returns `ErrTitleTooLong`.
4. A new todo is not completed, has `CreatedAt == now`, and has a `nil` `CompletedAt`.
5. `Complete(now)` sets `Completed = true` and `CompletedAt = &now`.
6. Calling `Complete` again on a completed todo does nothing. In particular, `CompletedAt` keeps its **original** time. (This is called *idempotent*.)

`ErrNotFound` isn't used until stage 3, but it belongs to the domain, so declare it here with the others.

Put your tests in `internal/todo/todo_test.go`. Declare a fixed time once at the top of the test file, for example `var t0 = time.Date(2026, 9, 29, 10, 0, 0, 0, time.UTC)`, and reuse it.

### Step 1 — Skeleton so tests can compile

Create `todo.go` with the struct, the constant, the three error variables, and **stub** versions of `NewTodo` (returns `Todo{}, nil`) and `Complete` (does nothing). Give each exported name a doc comment.

Pick clear error messages. Lower-case with no trailing punctuation is the Go convention, for example `"title must not be empty"`.

### Step 2 — A valid title (red → green)

**Test:** calling `NewTodo("id-1", "buy milk", t0)` returns no error and a `Todo` with the right `ID`, `Title`, `CreatedAt` (use `Equal`), `Completed == false`, and `CompletedAt == nil`.

Use `t.Fatalf` if `err != nil`, because checking the fields makes no sense then.

**Expect red.** A useful trick for the failure message is `%+v`, which prints a struct with field names:

```text
--- FAIL: TestNewTodo_Valid (0.00s)
    todo_test.go:18: NewTodo = {ID: Title: Completed:false CreatedAt:0001-01-01 00:00:00 +0000 UTC CompletedAt:<nil>}
```

**Make it green** with the simplest possible implementation. Don't validate anything yet.

### Step 3 — The title is trimmed (red → green)

**Test:** `NewTodo("id", "  buy milk  ", t0)` stores `Title == "buy milk"`. You can make this another case alongside Step 2, or change Step 2's input.

### Step 4 — Invalid titles (red → green)

**Test:** a table-driven test with a `wantErr error` field. Cases:

| name | title | wantErr |
| --- | --- | --- |
| `empty` | `""` | `ErrEmptyTitle` |
| `whitespace only` | `"  \t\n"` | `ErrEmptyTitle` |
| `too long` | 201 × `"a"` | `ErrTitleTooLong` |

Build long strings with `strings.Repeat("a", 201)`, or better, `strings.Repeat("a", MaxTitleLength+1)`. Check them with `errors.Is`.

Expected red, before you validate:

```text
--- FAIL: TestNewTodo_Errors (0.00s)
    --- FAIL: TestNewTodo_Errors/empty (0.00s)
        todo_test.go:36: NewTodo("") error = <nil>, want title must not be empty
    --- FAIL: TestNewTodo_Errors/whitespace_only (0.00s)
        ...
```

`%q` in the message prints the input with quotes and escapes, so whitespace-only input shows up as `"  \t\n"` instead of looking blank.

<details><summary>Hint 1</summary>

Trim first, then check for emptiness, then check the length. The order matters: `"   "` has a length of 3 but should be reported as empty.

</details>

<details><summary>Hint 2</summary>

Each check is an `if` that returns `Todo{}, ErrSomething` early. The happy path goes at the bottom, not indented. Go style avoids `else` after a `return`.

</details>

### Step 5 — Exactly at the limit, and multi-byte characters (red → green)

**Test:** both of these titles are **accepted**:

- 200 × `"a"`: exactly at the limit
- 200 × `"é"`: 200 characters but **400 bytes**

If you used `len(title)` in Step 4, the second case fails:

```text
--- FAIL: TestNewTodo_MaxLength (0.00s)
    todo_test.go:45: len(bytes)=400: unexpected error title is too long
```

This is why we write the edge case down as a test. Fix the implementation.

<details><summary>Hint</summary>

The package `unicode/utf8` has a function that counts runes in a string.

</details>

### Step 6 — `Complete` (red → green)

**Test:** create a todo, call `Complete(t1)` where `t1 := t0.Add(time.Hour)`, and check that:

- `Completed` is `true`
- `CompletedAt` is not `nil` (use `t.Fatalf` if it is, or the next line will panic)
- `CompletedAt.Equal(t1)`

Note that `td` must be a variable (addressable) for `td.Complete(...)` to change it. That's the pointer receiver at work.

### Step 7 — Completing twice is idempotent (red → green)

**Test:** complete at `t1`, then complete again at `t2 := t1.Add(time.Hour)`. `CompletedAt` must still equal `t1`.

If your Step 6 implementation always overwrites the field, this goes red. Fix it.

### Step 8 — Refactor

With everything green, look over the code:

- Do the tests share repeated setup that could become a small helper? If you write one, call `t.Helper()` inside it.
- Are the test names descriptive? Go usually uses `TestNewTodo_EmptyTitle` or subtests with readable names.
- Does every exported identifier have a doc comment?

## Checkpoint

```sh
go test -v ./...
go vet ./...
gofmt -l .
go list -deps -f '{{if not .Standard}}{{.ImportPath}}{{end}}' ./internal/todo
```

- [ ] All tests pass. `vet` and `gofmt -l` print nothing.
- [ ] The `go list` command prints only `<module>/internal/todo`. This is the dependency rule from the overview: the core uses only the standard library.
- [ ] Your tree looks like this (no `greet` left over):

```text
.
├── README.md
├── docs/guide/...
├── go.mod
└── internal/
    └── todo/
        ├── todo.go
        └── todo_test.go
```

## Commit

```sh
git add internal/todo
git commit -m "feat(todo): add Todo entity with title validation and completion"
```

Then tell Claude **"done with stage 2"** for a review.

## Stretch goals

- **Fuzz testing.** Go has built-in fuzzing. Write `func FuzzNewTodo(f *testing.F)` that seeds a few inputs with `f.Add("buy milk")` and so on, then in `f.Fuzz(func(t *testing.T, title string) { ... })` checks these **properties**: `NewTodo` never panics, and whenever it succeeds, the stored title is non-empty, already trimmed, and at most 200 runes. Run it with:

  ```sh
  go test -run XXX -fuzz FuzzNewTodo -fuzztime 10s ./internal/todo
  ```

  (`-run XXX` skips the ordinary tests.) If the fuzzer finds a failing input, it saves it under `testdata/fuzz/`, and it becomes a permanent regression test.
- **A `String()` method.** Add `func (t Todo) String() string` so that `fmt.Println(td)` prints `[x] buy milk`. Any type with a `String()` method satisfies `fmt.Stringer`, which is your first taste of implicit interfaces (coming in stage 3).
- **Wrapped errors.** Make the too-long error say how long the title was, for example `fmt.Errorf("%w: got %d characters", ErrTitleTooLong, n)`. Your `errors.Is` tests keep passing without any change, which shows why we used `errors.Is`.

## Further reading

- [Effective Go: Errors](https://go.dev/doc/effective_go#errors)
- [Go blog: Working with errors in Go 1.13](https://go.dev/blog/go1.13-errors)
- [Go blog: Strings, bytes, runes and characters](https://go.dev/blog/strings)
- [Package `time`](https://pkg.go.dev/time): see "Monotonic Clocks" and `Time.Equal`
- [Go FAQ: Should I define methods on values or pointers?](https://go.dev/doc/faq#methods_on_values_or_pointers)
- [Tutorial: Getting started with fuzzing](https://go.dev/doc/tutorial/fuzz)

Next: [03 — Ports and service](03-ports-and-service.md)
