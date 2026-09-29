# 01 — Tooling: modules, packages, and your first tests

## Goal

Set up a Go module, write a tiny throwaway package test-first, and get comfortable with `go test`, `go vet`, and `gofmt`. At the end of this stage you'll delete the throwaway package. Its job is to warm you up, not to stay in the project.

Time: about 30–45 minutes.

## Concepts

### Modules (goodbye `GOPATH`)

A **module** is a folder with a `go.mod` file at its root. It names your project and lists its dependencies. You can put a module anywhere on disk; there's no need for `~/go/src/github.com/...` any more.

```sh
go mod init example.com/todo
```

The argument is the **module path**. It becomes the prefix of every import path in your project. For a project you'll never publish, any domain-like name works. `example.com/todo` is used throughout this guide; wherever you see `<module>`, substitute your own.

The file it creates looks like this:

```text
module example.com/todo

go 1.27.1
```

The `go` line records the minimum Go version the module needs. Later, when you `go get` a dependency (stage 7), `go.mod` gains `require` lines, and a `go.sum` file appears with checksums for them. Commit both files.

> **If you remember old Go…** `dep`, `glide`, `vendor/` folders and `GOPATH` layouts are all obsolete for new code. Modules are the one standard way to manage dependencies.

### Packages and directories

- **One directory is one package.** Every `.go` file in a directory starts with the same `package name` line.
- The package name is usually the **last element of the directory path**: code in `internal/greet` starts with `package greet`.
- You import a package by its full path: `import "example.com/todo/internal/greet"`, then use it as `greet.Hello(...)`.

### Exported vs unexported

Go has no `public` or `private` keywords. **Capitalisation decides:**

```go
func Parse(s string) {}   // exported: other packages can call it
func parse(s string) {}   // unexported: only this package can call it
```

The same goes for types, struct fields, constants and variables.

### `internal/`

Any package under a directory named `internal` can only be imported by code in the tree rooted at `internal`'s parent directory. For us, that means only code inside this module. It's how Go says "not a public API".

### Test files

- Tests live next to the code, in files ending in `_test.go`. `go build` ignores these files; `go test` compiles them.
- A test is a function named `TestXxx` that takes `t *testing.T`.
- Test files in the same package (`package greet`) can see unexported names. There's also an "external test package" form (`package greet_test`) that sees only exported names. We'll use that in a later stage.

The `*testing.T` methods you'll use most:

| Method | Effect |
| --- | --- |
| `t.Errorf(format, args...)` | Record a failure and **keep going** |
| `t.Fatalf(format, args...)` | Record a failure and **stop this test now** |
| `t.Run(name, func(t *testing.T))` | Run a named **subtest** |
| `t.Helper()` | Mark a function as a helper, so failure lines point at the caller |

Use `Errorf` when later checks still make sense after a failure. Use `Fatalf` when they don't; for example, if a value you need is `nil`.

Go has no built-in `assert`. You compare values and call `t.Errorf` yourself. The standard message shape is **`got X, want Y`**, and you'll see it throughout Go code:

```go
if got != want {
    t.Errorf("Double(%d) = %d, want %d", in, got, want)
}
```

### Table-driven tests

When you're testing one function with many inputs, Go's standard idiom is a **table**: a slice of test cases, looped over with a subtest for each one. Here's the shape, using an unrelated `Abs` function:

```go
func TestAbs(t *testing.T) {
    tests := []struct {
        name string
        in   int
        want int
    }{
        {name: "positive", in: 3, want: 3},
        {name: "negative", in: -3, want: 3},
        {name: "zero", in: 0, want: 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if got := Abs(tt.in); got != tt.want {
                t.Errorf("Abs(%d) = %d, want %d", tt.in, got, tt.want)
            }
        })
    }
}
```

Why this is worth it:

- Adding a case takes one line.
- Each case appears by name in the output (`TestAbs/negative`).
- You can run a single case with `-run 'TestAbs/negative'`.

> **If you remember old Go…** you may have written `tt := tt` inside the loop before starting a goroutine or a parallel subtest. Since Go 1.22, each iteration gets a fresh `tt`, so that line is no longer needed.

### The everyday tools

| Command | What it does |
| --- | --- |
| `go test ./...` | Run all tests in all packages (`./...` means "this directory and everything below it") |
| `go test -v ./...` | Verbose: show every test and subtest as it runs |
| `go test -run 'TestHello/empty' ./internal/greet` | Run only tests matching a regex (`/` separates subtest names) |
| `go test -cover ./...` | Show the percentage of statements covered |
| `go vet ./...` | Static checks for likely bugs (such as wrong `Printf` verbs) |
| `gofmt -l .` | List files that aren't formatted; **no output means all good** |
| `gofmt -w .` | Rewrite files in the standard format |

Your editor (with `gopls`) will usually format on save, so `gofmt -l .` is mostly a safety net. Formatting isn't a matter of taste in Go: everyone uses `gofmt`.

## Steps

You're going to build `greet.Hello(name string) string` test-first:

- `Hello("Ada")` returns `"Hello, Ada!"`
- an empty or whitespace-only name returns `"Hello, world!"`

### Step 1 — Create the module

In the repo root (the folder containing this `README.md`):

```sh
go mod init example.com/todo
```

Check that `go.mod` looks like the example above.

### Step 2 — Write the first test (red: compile error)

Create `internal/greet/greet_test.go`. This is the **only complete test the guide gives you**. From stage 2 on, the guide describes each test and you write it.

```go
package greet

import "testing"

func TestHello(t *testing.T) {
	got := Hello("Ada")
	want := "Hello, Ada!"
	if got != want {
		t.Errorf("Hello(%q) = %q, want %q", "Ada", got, want)
	}
}
```

Run it:

```sh
go test ./...
```

Expected output (the timing and exact paths may differ):

```text
# example.com/todo/internal/greet [example.com/todo/internal/greet.test]
internal/greet/greet_test.go:6:9: undefined: Hello
FAIL	example.com/todo/internal/greet [build failed]
FAIL
```

That's the first kind of red: the test refers to something that doesn't exist yet.

### Step 3 — Make it compile (red: assertion failure)

Create `internal/greet/greet.go` with `package greet` and a `Hello` function that has the right signature but **returns an empty string**. Run `go test ./...` again. Now you should see the red you actually want:

```text
--- FAIL: TestHello (0.00s)
    greet_test.go:9: Hello("Ada") = "", want "Hello, Ada!"
FAIL
FAIL	example.com/todo/internal/greet	0.975s
FAIL
```

This proves the test really checks the behaviour.

### Step 4 — Make it pass (green)

Implement `Hello` so the test passes. Run `go test ./...` and look for:

```text
ok  	example.com/todo/internal/greet	0.681s
```

<details><summary>Hint 1</summary>

The `fmt` package has a function that builds a string from a format, the same way `Printf` does but without printing.

</details>

<details><summary>Hint 2</summary>

`fmt.Sprintf("Hello, %s!", name)`

</details>

Add a doc comment above `Hello`. By convention it starts with the function's name: `// Hello returns ...`. Try `go doc ./internal/greet` to see it.

### Step 5 — Convert to a table and add the new cases (red)

Rewrite `TestHello` as a table-driven test (use the `TestAbs` shape above) with three cases:

| name | input | want |
| --- | --- | --- |
| `with a name` | `"Ada"` | `"Hello, Ada!"` |
| `empty name` | `""` | `"Hello, world!"` |
| `whitespace only` | `"   "` | `"Hello, world!"` |

Run it. The first case passes and the other two fail, each reported by name:

```text
--- FAIL: TestHello (0.00s)
    --- FAIL: TestHello/empty_name (0.00s)
        greet_test.go:20: Hello("") = "Hello, !", want "Hello, world!"
    --- FAIL: TestHello/whitespace_only (0.00s)
        greet_test.go:20: Hello("   ") = "Hello,    !", want "Hello, world!"
FAIL
```

Notice that spaces in subtest names become underscores.

Try running just one case:

```sh
go test -v -run 'TestHello/empty' ./internal/greet
```

### Step 6 — Make it pass (green)

Update `Hello` so all three cases pass.

<details><summary>Hint 1</summary>

The `strings` package has a function that removes leading and trailing whitespace.

</details>

<details><summary>Hint 2</summary>

Trim the name with `strings.TrimSpace`. If the result is `""`, use `"world"` instead.

</details>

### Step 7 — Refactor and explore the tools

Your tests are green, so it's safe to tidy up. Is there any duplication or an unclear name? If not, that's fine too.

Then try each tool:

```sh
go test -v ./...
go test -cover ./...        # expect: coverage: 100.0% of statements
go vet ./...                # expect: no output
gofmt -l .                  # expect: no output
```

To see `gofmt` in action, deliberately mis-indent a line in `greet.go`, run `gofmt -l .` (it lists the file), then `gofmt -w .` (it fixes it).

### Step 8 — Delete the throwaway package

`greet` was only for practice, and it doesn't belong in the Todo API. Delete it:

```sh
rm -r internal/greet
```

`go test ./...` now prints `go: warning: "./..." matched no packages` followed by `no packages to test`. That's expected; stage 2 adds the first real package.

## Checkpoint

- [ ] `go.mod` exists with your module path and a `go 1.27.x` line
- [ ] You watched a compile-error red, an assertion red, and a green
- [ ] You ran a single subtest with `-run`
- [ ] `go vet ./...` and `gofmt -l .` both printed nothing
- [ ] `internal/greet` has been deleted

## Commit

```sh
git add go.mod
git commit -m "chore: initialise Go module"
```

If you'd like a record of the practice work, commit `internal/greet` *before* you delete it, then commit the deletion. Both approaches are fine.

## Stretch goals

- **Example tests.** Write `func ExampleHello()` that calls `fmt.Println(Hello("Ada"))` and ends with a comment `// Output: Hello, Ada!`. `go test` runs it and checks the output, and `go doc` shows it as documentation.
- **Coverage report.** Run `go test -coverprofile=coverage.out ./... && go tool cover -html=coverage.out`.
- **Failing on purpose.** Change a `want` so it's wrong, and compare the output of `t.Errorf` with `t.Fatalf` when two cases fail.

## Further reading

- [Tutorial: Get started with Go](https://go.dev/doc/tutorial/getting-started)
- [How to write Go code](https://go.dev/doc/code): modules, packages, and testing basics
- [Package `testing`](https://pkg.go.dev/testing)
- [Go blog: Using subtests and sub-benchmarks](https://go.dev/blog/subtests)
- [Effective Go: Names](https://go.dev/doc/effective_go#names)

Next: [02 — Domain](02-domain.md)
