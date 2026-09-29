# 07 — SQLite adapter: the hexagonal payoff

## Goal

Add a second driven adapter, `sqlite.Repository`, that stores todos in a SQLite file, and select it with `STORAGE=sqlite`. Along the way:

- turn the memory adapter's tests into a **contract test suite** that any `todo.Repository` must pass
- run that one suite against **both** adapters

When you're done, you will have changed **nothing** in `internal/todo` (apart from adding the test-support package) and **nothing** in `internal/adapters/httpapi`. That's the point of the architecture.

Time: about 2–3 hours.

## Concepts

### `database/sql` and drivers

`database/sql` is the standard library's database interface. Actual databases plug in as **drivers** that register themselves when their package is imported:

```go
import (
    "database/sql"
    _ "modernc.org/sqlite"   // blank import: run the package's init(), which registers "sqlite"
)

db, err := sql.Open("sqlite", "todo.db")
```

We use **`modernc.org/sqlite`**, a pure-Go SQLite with no cgo and no C compiler needed. (`mattn/go-sqlite3` is the other popular choice, but it needs cgo.)

If you forget the blank import, you get `sql: unknown driver "sqlite" (forgotten import?)`.

### Adding a dependency

```sh
go get modernc.org/sqlite@v1.60.1
```

This adds a `require` line to `go.mod` and checksums to `go.sum`. **Commit both files.** Until your code imports the package, `go.mod` marks it `// indirect`. Once you've imported it, run `go mod tidy` to clean that up and remove anything unused.

### `*sql.DB` is a pool, not a connection

`sql.Open` doesn't connect to anything; it prepares a **connection pool**. Create it once, share it, and `Close` it at shutdown. The methods you'll use:

```go
db.ExecContext(ctx, "INSERT ...", args...)        // no rows back; returns sql.Result (RowsAffected)
db.QueryRowContext(ctx, "SELECT ... WHERE id = ?", id).Scan(&a, &b)  // exactly one row
rows, err := db.QueryContext(ctx, "SELECT ...")    // many rows
```

For many rows, the loop always has this shape:

```go
rows, err := db.QueryContext(ctx, "SELECT name, age FROM people ORDER BY name")
if err != nil { return err }
defer rows.Close()                // releases the connection. Don't forget it
for rows.Next() {
    var p Person
    if err := rows.Scan(&p.Name, &p.Age); err != nil { return err }
    people = append(people, p)
}
if err := rows.Err(); err != nil { return err }  // errors during iteration show up here
```

**Always use `?` placeholders** for values, and never build SQL by concatenating strings. Placeholders prevent SQL injection and handle quoting for you.

Pass `ctx` to every call. When a client disconnects, `r.Context()` is cancelled, and the driver stops the query. This is where the `ctx` parameters you've been passing since stage 3 finally matter.

### SQLite and concurrency: one connection

SQLite allows only **one writer at a time**. With a pool of several connections, concurrent writes fail with:

```text
database is locked (5) (SQLITE_BUSY)
```

And with the special DSN `":memory:"`, **each connection gets its own separate empty database**. The schema you created on connection 1 doesn't exist on connection 2:

```text
SQL logic error: no such table: todos (1)
```

The simplest fix for both, and the right one for this project, is:

```go
db.SetMaxOpenConns(1)
```

The pool then hands out one connection at a time, and `database/sql` queues the other callers.

### Embedding the schema

`//go:embed` bakes a file into the compiled binary at build time:

```go
import _ "embed"

//go:embed schema.sql
var schema string
```

The file must be in the same directory as the Go file, or below it. Keep the SQL in a real `.sql` file, so your editor highlights it, and run it at startup with `ExecContext`. `CREATE TABLE IF NOT EXISTS` makes that safe to repeat.

### Mapping between Go types and columns

SQLite has only a few storage types. Decide explicitly how each field is stored:

| Go field | Column | Notes |
| --- | --- | --- |
| `ID string` | `TEXT PRIMARY KEY` | |
| `Title string` | `TEXT NOT NULL` | |
| `Completed bool` | `INTEGER NOT NULL` | 0 or 1; the driver converts `bool` both ways |
| `CreatedAt time.Time` | `INTEGER NOT NULL` | Unix **nanoseconds** (`t.UnixNano()`) |
| `CompletedAt *time.Time` | `INTEGER` (nullable) | `NULL` when not completed |

**Why integers for times?** They round-trip exactly (nanosecond precision), and they **sort correctly**, which `List` relies on (`ORDER BY created_at, id`). Text timestamps look friendlier, but RFC 3339 with nanoseconds drops trailing zeros, so the strings have different lengths and don't sort chronologically. For example, `"…T10:00:00.5Z"` sorts **before** `"…T10:00:00Z"` as a string (because `.` < `Z`), even though it's half a second later.

**Nullable columns** need a type that can represent "no value". `database/sql` provides `sql.NullInt64`, `sql.NullString`, `sql.NullTime`, and so on:

```go
var n sql.NullInt64            // after Scan: n.Valid == false means NULL
arg := sql.NullInt64{Int64: 42, Valid: true}   // as a query argument
```

**Time zones on the way back:** `time.Unix(0, n)` returns a time in the **local** time zone. The instant is right (`Equal` passes), but JSON would show `+01:00` and so on. Call `.UTC()` on everything you read back.

### "Not found" in SQL

- `QueryRow(...).Scan(...)` returns **`sql.ErrNoRows`** when nothing matched. Map it to `todo.ErrNotFound`. The core must never see `sql.ErrNoRows`, because that would leak the adapter's technology into the domain.
- `DELETE` doesn't fail when nothing matched. Check `res.RowsAffected()`, and if it's 0, return `todo.ErrNotFound`.

### Upsert

`Save` means "insert or replace". SQLite (3.24+) supports:

```sql
INSERT INTO people (id, name) VALUES (?, ?)
ON CONFLICT(id) DO UPDATE SET name = excluded.name
```

`excluded` refers to the row you tried to insert.

### Wrapping errors (for real this time)

Stage 2 introduced `%w`. Now there's a real reason to use it. A raw driver error such as `database is locked` doesn't tell you *which* operation failed. Wrap it with context:

```go
return fmt.Errorf("save todo %s: %w", t.ID, err)
```

The message reads `save todo abc: database is locked`, and `errors.Is` still sees the original error. **Don't** wrap `todo.ErrNotFound` with SQL details; return it as is, because it's a domain answer, not an infrastructure failure.

### Contract tests

Your memory adapter tests from stage 3 describe what **any** `todo.Repository` must do. Rather than copying them for SQLite, move them into a reusable function:

```go
// package todotest
func RunRepositoryContract(t *testing.T, newRepo func(t *testing.T) todo.Repository)
```

Each adapter's test file then contains one call:

```go
func TestContract(t *testing.T) {
    todotest.RunRepositoryContract(t, func(t *testing.T) todo.Repository {
        return memory.New()
    })
}
```

The `newRepo` factory is called **once per subtest**, so every subtest gets a fresh, empty repository and the subtests can't affect each other.

**Where does `todotest` live?** At `internal/todo/todotest`. It's a separate package (so importing `testing` there doesn't affect `todo`) that sits next to the port it tests. The standard library uses this pattern too, for example `net/http/httptest` and `testing/fstest`. It imports only `todo` and the standard library, so the dependency rule holds.

### Test databases with `t.TempDir()`

```go
path := filepath.Join(t.TempDir(), "test.db")
```

`t.TempDir()` creates a fresh directory for each test and **deletes it automatically** when the test ends. Combine it with `t.Cleanup(func() { repo.Close() })` to close the database first.

## Steps

What you're building:

```go
// internal/todo/todotest/contract.go
func RunRepositoryContract(t *testing.T, newRepo func(t *testing.T) todo.Repository)

// internal/adapters/sqlite/sqlite.go + schema.sql
func Open(ctx context.Context, dsn string) (*Repository, error)  // opens and applies the schema
func (r *Repository) Close() error
// + Save, Get, List, Delete satisfying todo.Repository

// internal/platform/config.go (extended)
//   STORAGE = "memory" (default) | "sqlite"
//   DB_PATH = path to the SQLite file, default "todo.db"
```

### Part A — the contract suite (a refactor: stays green)

#### Step 1 — Extract the suite

Create `internal/todo/todotest/contract.go` (`package todotest`). Move the **contract** tests from `internal/adapters/memory/memory_test.go` into `RunRepositoryContract` as `t.Run(...)` subtests, each starting with `r := newRepo(t)`:

- save then get; get missing gives `ErrNotFound`; save replaces
- list on empty; list order (including the tie broken by `ID`)
- delete; delete missing gives `ErrNotFound`
- the concurrent saves test (50 goroutines)

Leave the **memory-specific** test (the one about not leaking internal state through `CompletedAt`) in the memory package. That's a detail of how the memory adapter is built, not part of the port's contract.

Replace the moved tests in the memory package with the one-line `TestContract` shown above.

**Run `go test -race ./...`. It must still be green.** Moving tests while keeping them passing is a refactor. If something turns red, the move changed behaviour, so fix that before continuing.

#### Step 2 — Strengthen the contract with what SQLite will test

Add two checks to the suite. They pass for memory already, but they'll catch real bugs in SQLite:

1. **Nanosecond precision:** use a fixed time with nanoseconds, such as `time.Date(2026, 9, 29, 10, 0, 0, 123456789, time.UTC)`. `Get` returns a `CreatedAt` that is `Equal` to it.
2. **UTC on the way out:** `got.CreatedAt.Location() == time.UTC`.

Run the tests again. They're still green against memory.

### Part B — the SQLite adapter

#### Step 3 — Dependency, schema, skeleton

```sh
go get modernc.org/sqlite@v1.60.1
```

Create `internal/adapters/sqlite/schema.sql` with a `CREATE TABLE IF NOT EXISTS todos (...)` that follows the mapping table above.

Create `sqlite.go` with:

- the embedded schema
- the `var _ todo.Repository = (*Repository)(nil)` check
- `Open` (a real implementation: `sql.Open`, `SetMaxOpenConns(1)`, apply the schema, and on error close the database and return a wrapped error)
- `Close`
- **stub** versions of the four repository methods

Then `go mod tidy`.

#### Step 4 — Run the contract against SQLite (red)

`internal/adapters/sqlite/sqlite_test.go`: a `TestContract` whose factory opens a database under `t.TempDir()`, registers `t.Cleanup` to close it, and calls `t.Fatal` if `Open` fails.

Run it. Nearly every subtest fails, and that list of failures is your checklist. ("List on empty" may pass against the stubs, because a stub returning `nil` has length 0. That's fine: it gets checked properly once the other list tests pass.)

```sh
go test ./internal/adapters/sqlite
```

#### Step 5 — Make it green, one subtest at a time

Run a single subtest while you work on it:

```sh
go test -run 'TestContract/save_then_get' ./internal/adapters/sqlite
```

Suggested order: `Save` + `Get` → get missing → save replaces → `List` → `Delete` → delete missing → concurrent saves.

<details><summary>Hint: sharing scanning code between Get and List</summary>

`*sql.Row` and `*sql.Rows` both have `Scan(dest ...any) error`. Declare a tiny interface, `type scanner interface{ Scan(dest ...any) error }`, and write one `scanTodo(s scanner) (todo.Todo, error)` that both methods use.

</details>

<details><summary>Hint: the location check fails</summary>

```text
contract.go:29: CreatedAt location = Local, want UTC
```

`time.Unix(0, n)` returns local time. Add `.UTC()`.

</details>

<details><summary>Hint: <code>List</code> on an empty table</summary>

The `todo.Service` from stage 3 already turns `nil` into `[]`, but it's tidier for the adapter to start with `out := []todo.Todo{}` anyway.

</details>

<details><summary>Hint: concurrent saves fail with "database is locked"</summary>

Check that `Open` calls `db.SetMaxOpenConns(1)`.

</details>

#### Step 6 — Wire `STORAGE` into config and `main` (red → green for config)

Extend `Config` with `Storage` and `DBPath`, and add tests first.

> **Heads up:** adding fields with non-empty defaults changes what "the default config" is. If your stage 6 tests compare whole `Config` structs, they now fail (`want` has `Storage:""`, but `got` has `"memory"`). Update your expected defaults in **one** place (the `defaultConfig()` helper, if you followed stage 6's suggestion), and the whole table goes green again.

The new tests:

- the defaults are `Storage == "memory"` and `DBPath == "todo.db"`
- `STORAGE=sqlite` and `DB_PATH=/tmp/x.db` are read correctly
- `STORAGE=postgres` returns an error that mentions `STORAGE`

Then, in `run`, choose the repository:

1. Declare `var repo todo.Repository = memory.New()`.
2. If `cfg.Storage == "sqlite"`: call `sqlite.Open(ctx, cfg.DBPath)`, return any error, `defer` its `Close()`, and assign it to `repo`.
3. Log which storage is in use.

Nothing else in `run` changes, and **nothing at all in `httpapi` or `todo` changes**.

## Checkpoint

```sh
go test -race ./...
go vet ./...
gofmt -l .
go list -deps -f '{{if not .Standard}}{{.ImportPath}}{{end}}' ./internal/todo
# still only: <module>/internal/todo, and no sqlite driver
```

**Data survives a restart:**

```sh
STORAGE=sqlite go run ./cmd/todo-api
```

```sh
curl -s -X POST localhost:8080/todos -d '{"title":"survive a restart"}'
# {"id":"469a…","title":"survive a restart","completed":false,"created_at":"2026-09-29T21:48:29.316709Z"}
```

Ctrl-C the server, start it again with the same command, then:

```sh
curl -s localhost:8080/todos
# [{"id":"469a…","title":"survive a restart","completed":false,"created_at":"2026-09-29T21:48:29.316709Z"}]
```

Server log:

```text
time=... level=INFO msg="storage ready" storage=sqlite
time=... level=INFO msg="server starting" addr=:8080
...
```

A `todo.db` file now exists in the repo root, and `.gitignore` already excludes it. If you have the `sqlite3` command-line tool installed, try `sqlite3 todo.db 'select * from todos'`.

**Bad storage fails fast:**

```sh
STORAGE=postgres go run ./cmd/todo-api
# error: STORAGE: must be memory or sqlite, got "postgres"
# exit status 1
```

- [ ] One contract suite runs against both adapters, and both pass with `-race`.
- [ ] `git diff` against your stage 6 commit shows **no** changes in `internal/todo/*.go` (only the new `todotest/` directory) and **no** changes in `internal/adapters/httpapi/`.
- [ ] Your final tree looks like this:

```text
.
├── README.md
├── go.mod
├── go.sum
├── cmd/todo-api/main.go
├── docs/guide/…
└── internal/
    ├── todo/
    │   ├── todo.go, todo_test.go
    │   ├── ports.go
    │   ├── service.go, service_test.go
    │   ├── id.go, id_test.go
    │   └── todotest/contract.go
    ├── adapters/
    │   ├── memory/   memory.go, memory_test.go
    │   ├── httpapi/  handler.go, handler_test.go
    │   └── sqlite/   sqlite.go, schema.sql, sqlite_test.go
    └── platform/     logger.go, config.go, requestid.go (+ tests)
```

## Commit

```sh
git add go.mod go.sum cmd/ internal/
git commit -m "feat(sqlite): add SQLite repository behind the same port, with shared contract tests"
```

Then tell Claude **"done with stage 7"** for the final review.

## Stretch goals

- **Migrations.** Replace the single schema with numbered files (`001_init.sql`, `002_add_due_date.sql`), embedded with `//go:embed migrations/*.sql` into an `embed.FS`, plus a `schema_version` table recording which ones have run. Then add a `DueDate` field all the way through: domain, both adapters, DTO, and contract.
- **Build tags.** Put the SQLite tests behind `//go:build integration` and run them with `go test -tags integration ./...`. When is this split worth it? (For pure-Go SQLite with temporary files, it's arguably *not* worth it. Think about why.)
- **A context-cancellation contract test.** Cancel a context before calling `List`. Both adapters should return an error that `errors.Is(err, context.Canceled)`. For memory, that means adding the check from the stage 3 stretch goal.
- **WAL mode.** Open with `?_pragma=journal_mode(WAL)` and read about what it changes for concurrent readers.
- **A third adapter.** Try Postgres using `github.com/jackc/pgx/v5/stdlib` and a Docker container. The contract suite tells you when you're done.

## Further reading

- [Tutorial: Accessing a relational database](https://go.dev/doc/tutorial/database-access)
- [Accessing relational databases (full guide)](https://go.dev/doc/database/): see especially "Querying for data" and "Managing connections"
- [Package `database/sql`](https://pkg.go.dev/database/sql)
- [Package `embed`](https://pkg.go.dev/embed)
- [Package `modernc.org/sqlite`](https://pkg.go.dev/modernc.org/sqlite)
- [Managing dependencies](https://go.dev/doc/modules/managing-dependencies)

---

**You've finished the guide.** Look back at your commit history: every stage added one layer, and the core from stage 2 has barely changed since. The core didn't need to change because of HTTP, logging, config, or SQLite, and that's what hexagonal architecture is for.
