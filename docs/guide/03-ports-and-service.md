# 03 — Ports and service: interfaces, fakes, and the in-memory adapter

## Goal

Give the core a **port**, a `Repository` interface that says "I need somewhere to keep todos", and a **service** that carries out the use cases (create, get, list, complete, delete). Test the service with a hand-written **fake** repository. Then build the first real **adapter**: a thread-safe, in-memory repository.

After this stage the application works end to end in code, just without HTTP yet.

Time: about 1.5–2 hours.

## Concepts

### Interfaces are satisfied implicitly

In Go, a type satisfies an interface simply by having the right methods. There's no `implements` keyword:

```go
type Notifier interface {
    Notify(msg string) error
}

type EmailNotifier struct{ /* ... */ }

func (e *EmailNotifier) Notify(msg string) error { /* ... */ return nil }

// *EmailNotifier is now a Notifier. Nothing else to declare.
```

This has a big design consequence: **the package that *uses* an interface can define it**, and implementations elsewhere satisfy it without importing anything special. That's exactly what hexagonal architecture wants. The core declares the port (`todo.Repository`), and the adapters (`memory`, later `sqlite`) import `todo` and implement it. The core never imports the adapters.

Keep interfaces **small**, containing only what the consumer actually needs. The standard library's most-used interfaces have one method: `io.Reader`, `io.Writer`, `fmt.Stringer`.

### "Accept interfaces, return structs"

A common Go maxim:

- Functions **take** interfaces as parameters, so callers can pass anything suitable, including fakes in tests.
- Functions **return** concrete types, so callers get the full type and can decide for themselves which interface to use it as.

So `NewService(repo Repository, ...)` takes an interface and returns `*Service`, a concrete struct.

### Compile-time interface check

The compiler checks interface satisfaction only when a value is *used* as the interface. To get an error straight away in the adapter's own package, add this line:

```go
var _ todo.Repository = (*Repository)(nil)
```

It declares a variable that is thrown away (`_`), of type `todo.Repository`, set to a `nil` `*Repository`. If a method is missing or has the wrong signature, this line won't compile. It costs nothing at run time.

### Fakes, not mocks

A **fake** is a small, working, simplified implementation of an interface. For a repository, that's a struct wrapping a map. You write it once in a test file and use it in many tests:

```go
type fakeClock struct{ now time.Time }

func (f *fakeClock) Now() time.Time { return f.now }
```

Compared with mocking libraries that record expected calls, fakes:

- test **behaviour** ("after Create, I can Get it") rather than **interactions** ("Save was called once with these arguments")
- don't break when you refactor how the service uses the repository internally
- are plain Go, with nothing to learn

You can make a fake return errors when a test needs it, for example with a `saveErr error` field that, when set, makes `Save` return it.

### `context.Context`

`context.Context` carries **cancellation, deadlines, and request-scoped values** across API boundaries. Conventions:

- It's the **first parameter**, named `ctx`: `func (s *Service) Get(ctx context.Context, id string)`.
- Pass it down. Don't store it in a struct.
- In tests, use **`t.Context()`**. It gives each test its own context, which is cancelled when the test finishes.
- Outside tests, when there's no parent context, use `context.Background()`.

The memory adapter will ignore `ctx`. Name that parameter `_` to show you're ignoring it on purpose. The SQLite adapter in stage 7 will pass it to the database driver, so a cancelled HTTP request stops its query. Adding `ctx` now means you won't have to change every signature later.

> **If you remember old Go…** `t.Context()` is new (Go 1.24). Older code creates `context.Background()` in each test.

### Maps are not safe for concurrent use

An HTTP server handles each request in its own goroutine, so two requests can call `Save` at the same time. If they both write to a plain `map`, the program crashes:

```text
fatal error: concurrent map writes
```

Protect shared state with a mutex:

```go
type Counter struct {
    mu sync.RWMutex
    n  map[string]int
}

func (c *Counter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n[key]++
}

func (c *Counter) Get(key string) int {
    c.mu.RLock()          // many readers may hold RLock at once
    defer c.mu.RUnlock()
    return c.n[key]
}
```

- `defer` runs when the function returns, so the unlock can't be forgotten on an early return.
- A struct containing a mutex **must not be copied**. Use pointer receivers, and return `*Repository` from the constructor. `go vet` warns you if you copy one.
- The **race detector**, `go test -race ./...`, finds unsynchronised access even when it happens not to crash. Make `-race` a habit for any package with goroutines or shared state.

### Don't leak internal state

If your repository stores a `Todo` and hands the caller a value whose `CompletedAt` pointer points at the *same* `time.Time` stored inside the map, the caller could change the stored data without going through `Save`. Return **copies**: copy the struct, and give it a *new* pointer for `CompletedAt`.

### Sorting

Maps have **no order**; iterating over the same map twice can give different orders. If `List` should return todos oldest-first, sort explicitly:

```go
slices.SortFunc(people, func(a, b Person) int {
    if c := cmp.Compare(a.Age, b.Age); c != 0 {
        return c
    }
    return cmp.Compare(a.Name, b.Name) // tie-breaker for a stable, predictable order
})
```

`time.Time` has its own `Compare` method: `a.CreatedAt.Compare(b.CreatedAt)`.

> **If you remember old Go…** `sort.Slice(xs, func(i, j int) bool {...})` still works, but the generic `slices` and `cmp` packages (Go 1.21) are clearer and type-safe.

### Nil slices vs empty slices

```go
var a []Todo          // nil: len 0, and a == nil
b := []Todo{}         // empty: len 0, and b != nil
```

Both behave the same with `len`, `range`, and `append`. **But they encode to JSON differently**: `nil` becomes `null`, and empty becomes `[]`. The API promises `[]` for an empty list, so decide now that `Service.List` **never returns `nil`**. It's easier to guarantee at the source than in every caller.

### Generating IDs

```go
crypto/rand.Read(b)          // fills b with secure random bytes (never fails since Go 1.24)
encoding/hex.EncodeToString  // bytes → "3f9a..."
```

Sixteen random bytes give a 32-character hex ID. That's the same amount of randomness as a UUID, with no dependency. (Go 1.24 also added `crypto/rand.Text()`, which returns a random base32 string. Either is fine.)

The service will take the ID generator as a **function value**, `newID func() string`. Tests pass `func() string { return "id-1" }`, and production passes `todo.NewID`. It's the same injection idea as `now func() time.Time`.

## Steps

What you're building:

```go
// internal/todo/ports.go
type Repository interface {
    Save(ctx context.Context, t Todo) error            // insert or replace (by ID)
    Get(ctx context.Context, id string) (Todo, error)  // ErrNotFound if missing
    List(ctx context.Context) ([]Todo, error)          // oldest first (CreatedAt, then ID)
    Delete(ctx context.Context, id string) error       // ErrNotFound if missing
}

// internal/todo/id.go
func NewID() string

// internal/todo/service.go
func NewService(repo Repository, now func() time.Time, newID func() string) *Service
func (s *Service) Create(ctx context.Context, title string) (Todo, error)
func (s *Service) Get(ctx context.Context, id string) (Todo, error)
func (s *Service) List(ctx context.Context) ([]Todo, error)          // never nil
func (s *Service) Complete(ctx context.Context, id string) (Todo, error)
func (s *Service) Delete(ctx context.Context, id string) error

// internal/adapters/memory/memory.go
func New() *Repository   // *memory.Repository satisfies todo.Repository
```

### Part A — the service, tested with a fake

#### Step 1 — Declare the port

Write `ports.go` with the `Repository` interface above. Document each method's contract in its comment, including which methods return `ErrNotFound`. That comment is the specification that every adapter must follow.

#### Step 2 — Write the fake

Create `internal/todo/service_test.go`. Use **`package todo_test`**, the external test package, so your tests use the package exactly as outside code would, through exported names only. You'll import `<module>/internal/todo` and write `todo.NewService`, `todo.ErrNotFound`, and so on.

In that file, write a `fakeRepo` struct backed by a `map[string]todo.Todo` that implements all four methods, returning `todo.ErrNotFound` where the contract says so. Give it a `saveErr error` field: when it's non-nil, `Save` returns it.

It doesn't need a mutex, because the service tests don't use goroutines. It doesn't need to sort either, because none of the service tests depend on order.

<details><summary>Hint</summary>

Write a `newFakeRepo()` constructor that initialises the map. Writing to a `nil` map panics, although reading from one is fine.

</details>

Also write a small test helper, for example `newTestService(repo todo.Repository) *todo.Service`, that passes a fixed clock (`func() time.Time { return t0 }`) and a fixed ID (`func() string { return "id-1" }`).

#### Step 3 — `Create` (red → green)

Write a stub `service.go` first (the struct, `NewService`, and `Create` returning zero values) so the test compiles. Then add these tests:

- **Happy path:** `Create(ctx, "buy milk")` returns a todo with `ID == "id-1"`, `Title == "buy milk"`, and `CreatedAt` equal to `t0`. Afterwards, the fake contains it under `"id-1"`.
- **Validation error is propagated:** `Create(ctx, "  ")` returns an error that `errors.Is(err, todo.ErrEmptyTitle)`, and **nothing is saved**.
- **Repository error is propagated:** set `repo.saveErr = errors.New("boom")`, and `Create` returns an error that `errors.Is(err, boom)`.

<details><summary>Hint 1</summary>

`Create` is three steps: build the entity with `NewTodo`, save it, and return it. Use `s.newID()` and `s.now()` for the parameters.

</details>

<details><summary>Hint 2</summary>

Return each error straight away with `if err != nil { return todo.Todo{}, err }`. Don't wrap them yet; stage 7 talks about when wrapping is worth it.

</details>

#### Step 4 — `Get` and `Delete` (red → green)

- `Get` on an unknown ID returns `ErrNotFound`.
- `Get` after `Create` returns the created todo.
- `Delete` on an unknown ID returns `ErrNotFound`.
- After `Delete`, `Get` returns `ErrNotFound`.

These are thin pass-throughs today. They still get tests, because the tests pin down the contract that the HTTP layer will rely on.

#### Step 5 — `Complete` (red → green)

- `Complete` on an unknown ID returns `ErrNotFound`.
- `Complete` on an existing todo returns it with `Completed == true`, **and the change is saved**: a following `Get` shows it completed.

To test the "saved" part meaningfully, give the service a clock that moves. For example, a `now` function that returns `t0` on the first call and `t0.Add(time.Hour)` after that, or a variable you change between calls. Then check that `CompletedAt` equals the *later* time.

<details><summary>Hint</summary>

Load the todo, call its `Complete(s.now())` method (the domain rule from stage 2), then save it again.

</details>

#### Step 6 — `List` never returns nil (red → green)

- On an empty repository, `List` returns a slice that is **non-nil** and has length 0. Test `got == nil` explicitly. Printing it with `%#v` shows the difference: `[]todo.Todo(nil)` vs `[]todo.Todo{}`.
- After two `Create`s, `List` returns two todos. With a fixed ID generator both would get `"id-1"`, so use a counter-based `newID` in this test.

Your fake's `List` probably builds its result with `var out []todo.Todo` plus `append`, which returns `nil` when there's nothing to add. Good: that's what makes the first test go red. Fix it in the **service**, not the fake. The service's promise shouldn't depend on how each adapter happens to behave.

#### Step 7 — `NewID`

In `internal/todo/id_test.go`:

- `NewID()` returns a 32-character string made only of `0-9a-f`.
- Two calls return different values.

Then implement it with `crypto/rand` and `encoding/hex`.

### Part B — the in-memory adapter

#### Step 8 — Skeleton and compile-time check

Create `internal/adapters/memory/memory.go` (`package memory`) with a `Repository` struct (a `sync.RWMutex` plus a `map[string]todo.Todo`), a `New()` constructor, and **stub** methods. Add the `var _ todo.Repository = (*Repository)(nil)` line.

Try deleting one method (say `Delete`) and running `go build ./...`. The build fails, and the error names the missing method:

```text
internal/adapters/memory/memory.go:12:25: cannot use (*Repository)(nil) (value of type *Repository) as todo.Repository value in variable declaration: *Repository does not implement todo.Repository (missing method Delete)
```

Put it back.

#### Step 9 — Adapter behaviour (red → green, one at a time)

In `internal/adapters/memory/memory_test.go` (`package memory_test`), write one test per behaviour and make each one pass before writing the next:

1. **Save then Get** returns an equal todo (compare fields; use `Equal` for times).
2. **Get of a missing ID** returns `todo.ErrNotFound`.
3. **Save with an existing ID replaces** the stored todo (for example, save it again with `Completed: true` and check that `Get` shows it).
4. **List on empty** returns length 0.
5. **List order**: save three todos with `CreatedAt` values `t0+2h`, `t0`, `t0+1h` (deliberately out of order). `List` returns them oldest-first. Add a fourth with the *same* `CreatedAt` as another and check that the tie is broken by `ID`.
6. **Delete** removes the todo; a following `Get` returns `ErrNotFound`.
7. **Delete of a missing ID** returns `todo.ErrNotFound`.
8. **No leaking internal state**: save a completed todo, `Get` it, change the returned `*CompletedAt` (for example, `*got.CompletedAt = time.Time{}`), then `Get` again. The stored value must be unchanged.

Keep these adapter tests tidy and focused on the **`todo.Repository` contract**, not on memory-specific details. In stage 7 you'll move them into a shared suite that runs against SQLite too.

<details><summary>Hint for item 8</summary>

Write a small unexported `clone(t todo.Todo) todo.Todo` that copies the struct and, if `CompletedAt` isn't nil, points it at a fresh copy of the time. Use it on the way in (`Save`) and on the way out (`Get`, `List`).

</details>

#### Step 10 — Concurrency under the race detector

Write a test that starts 50 goroutines that each `Save` a todo with a different ID, waits for them all, then checks that `List` returns 50:

```go
var wg sync.WaitGroup
for i := range 50 {
    wg.Go(func() {
        // ... use i ...
    })
}
wg.Wait()
```

(`wg.Go` is new in Go 1.25. It combines `wg.Add(1)`, `go func(){...}()`, and `defer wg.Done()`.)

Run it with `go test -race ./internal/adapters/memory`. To see why the mutex matters, temporarily comment out your `Lock`/`Unlock` calls and run it again. Without `-race`, you'll probably get `fatal error: concurrent map writes`. With `-race`, you get a report starting with:

```text
==================
WARNING: DATA RACE
Write at 0x00c00007c5d0 by goroutine 23:
  ...
  example.com/todo/internal/adapters/memory.(*Repository).Save()
      .../internal/adapters/memory/memory.go:22 +0x1d0
```

It points at the exact line. Put the locks back.

#### Step 11 — Refactor

- Is there repeated setup in the adapter tests? A helper that saves a todo and fails the test on error, using `t.Helper()`, tidies things up.
- Use `RLock` in the read methods and `Lock` in the write methods.

## Checkpoint

```sh
go test -race ./...
go vet ./...
gofmt -l .
go list -deps -f '{{if not .Standard}}{{.ImportPath}}{{end}}' ./internal/todo
```

- [ ] All tests pass **with `-race`**. `vet` and `gofmt -l` print nothing.
- [ ] The `go list` command still prints only `<module>/internal/todo`. The core doesn't know `memory` exists.
- [ ] The adapter imports the core, never the reverse:

  ```sh
  go list -deps -f '{{if not .Standard}}{{.ImportPath}}{{end}}' ./internal/adapters/memory
  # <module>/internal/todo
  # <module>/internal/adapters/memory
  ```

## Commit

```sh
git add internal/
git commit -m "feat(todo): add Repository port, Service use cases and in-memory adapter"
```

Then tell Claude **"done with stage 3"**.

## Stretch goals

- **A `Clock` interface instead of a function.** Replace `now func() time.Time` with a one-method interface. Which reads better? Many Go codebases prefer function values for single-method dependencies.
- **Test coverage.** Run `go test -cover ./...`. Is anything in the service untested? Is that deliberate?
- **Context cancellation.** Make the memory adapter return `ctx.Err()` if the context is already cancelled (check `ctx.Err() != nil` at the top of each method). Test it by cancelling a context before calling. It's a small taste of what the SQLite driver will do for you in stage 7.

## Further reading

- [Effective Go: Interfaces](https://go.dev/doc/effective_go#interfaces)
- [Go Code Review Comments: Interfaces](https://go.dev/wiki/CodeReviewComments#interfaces): why interfaces belong to the consumer
- [Go blog: Go Concurrency Patterns: Context](https://go.dev/blog/context)
- [Go blog: Introducing the Go Race Detector](https://go.dev/blog/race-detector)
- [Package `sync`](https://pkg.go.dev/sync), [package `slices`](https://pkg.go.dev/slices), [package `cmp`](https://pkg.go.dev/cmp)
- [Go FAQ: Why are map operations not defined to be atomic?](https://go.dev/doc/faq#atomic_maps)

Next: [04 — HTTP adapter](04-http.md)
