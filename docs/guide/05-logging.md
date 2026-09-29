# 05 — Logging: `log/slog` and middleware

## Goal

Add structured logging:

- a logger built once in `main` and **passed in** to what needs it
- **middleware** that logs every request's method, path, status, and duration
- the handler logs the **real error** behind every 500, while the client still gets a generic message

All of it is tested by capturing log output in a buffer.

Time: about 1.5 hours.

## Concepts

### `log` vs `log/slog`

The old `log` package writes plain lines: `2026/09/29 10:00:00 something happened: user=42`. That's fine for humans, but hard for machines to search.

`log/slog` (Go 1.21) writes **structured** records made of a message plus key/value **attributes**:

```go
logger.Info("payment accepted", "order_id", 1234, "amount_cents", 999)
```

With a **text handler**, that becomes:

```text
time=2026-09-29T10:00:00.000Z level=INFO msg="payment accepted" order_id=1234 amount_cents=999
```

And with a **JSON handler**, it becomes this. Log systems such as Datadog, Loki, or CloudWatch can then filter on any field:

```json
{"time":"2026-09-29T10:00:00Z","level":"INFO","msg":"payment accepted","order_id":1234,"amount_cents":999}
```

> **If you remember old Go…** structured logging used to mean choosing between `logrus`, `zap`, and `zerolog`. `slog` is now the standard, and those libraries can plug into it as handlers.

### Loggers, handlers, levels

- A **`*slog.Logger`** is what you call (`Info`, `Warn`, `Error`, `Debug`).
- A **`slog.Handler`** decides the format and destination: `slog.NewTextHandler(w, opts)` or `slog.NewJSONHandler(w, opts)`, where `w` is any `io.Writer`.
- **Levels** are `slog.LevelDebug < LevelInfo < LevelWarn < LevelError`. The handler drops records below its minimum:

```go
h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelWarn})
logger := slog.New(h)
logger.Info("not shown")
logger.Warn("shown")
```

### Attributes

Plain alternating key/value pairs work: `"user", id, "ok", true`. For speed and type safety, `LogAttrs` with typed attributes avoids reflection and catches mismatched pairs:

```go
logger.LogAttrs(ctx, slog.LevelInfo, "cache miss",
    slog.String("key", k),
    slog.Int("size", n),
    slog.Duration("took", d),
)
```

`logger.With("component", "billing")` returns a child logger that adds those attributes to every record.

`go vet` checks `slog` calls and warns about a key without a value, such as `logger.Info("msg", "user")`.

Every method has a `...Context` variant (`InfoContext(ctx, ...)`, `ErrorContext`). Pass `r.Context()` in handlers. It costs nothing today, and in stage 6 it lets you pull a request ID out of the context.

### Inject the logger; don't use a global

`slog.SetDefault(logger)` plus `slog.Info(...)` everywhere is convenient, but it's hidden global state: tests can't capture or silence it, and parallel tests would share it. **Pass a `*slog.Logger` explicitly** to the components that log, as you did with the repository and the clock.

In tests where you don't care about logs, `slog.New(slog.DiscardHandler)` (Go 1.24) discards everything.

### Where logging belongs in a hexagon

Logging is an **infrastructure** concern, so the **core doesn't log**. The service returns errors, and the adapters decide what to log. That keeps `internal/todo` importing only the standard library, and keeps the domain logic readable. (`log/slog` *is* in the standard library, but the dependency-rule check is about responsibilities as well as imports: domain code with logging calls mixed in is harder to read and test.)

We'll put reusable infrastructure in `internal/platform`.

### Middleware

Middleware is a function that **wraps** a handler to add behaviour before or after it:

```go
func RequireHeader(name string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if r.Header.Get(name) == "" {
                http.Error(w, "missing "+name, http.StatusBadRequest)
                return
            }
            next.ServeHTTP(w, r)   // call the wrapped handler
        })
    }
}

handler = RequireHeader("X-Api-Key")(handler)
```

The shape `func(http.Handler) http.Handler` is the Go convention. Middleware in this shape can be composed freely: `a(b(c(handler)))`.

### Capturing the status code

`http.ResponseWriter` doesn't let you *read* the status after it's written. The usual approach is to wrap it:

```go
type countingWriter struct {
    http.ResponseWriter   // embedded: all methods pass through automatically
    bytes int
}

func (c *countingWriter) Write(b []byte) (int, error) {
    n, err := c.ResponseWriter.Write(b)
    c.bytes += n
    return n, err
}
```

**Embedding** the interface gives you every method for free, and you override just the ones you need. For a status recorder, override `WriteHeader(code int)`.

**Watch out:** if a handler never calls `WriteHeader` and just calls `Write`, net/http sends **200**. Your recorder must report 200 in that case too, so start it at 200.

### Testing log output

Make the logger write to a `bytes.Buffer` with a **JSON** handler, then decode the buffer and check the fields:

```go
var buf bytes.Buffer
logger := slog.New(slog.NewJSONHandler(&buf, nil))
// ... run the code under test ...
var entry map[string]any
if err := json.Unmarshal(buf.Bytes(), &entry); err != nil {
    t.Fatalf("log output is not JSON: %v\n%s", err, buf.String())
}
if entry["status"] != float64(404) { ... }   // JSON numbers decode to float64
```

This checks structure, not formatting, so the test won't break if the time format or field order changes. If a test produces several log lines, split `buf.String()` on `"\n"` and decode each line.

In JSON, `slog.Duration` is written as an integer number of **nanoseconds**, for example `"duration":792`.

## Steps

What you're building:

```go
// internal/platform/logger.go
func NewLogger(w io.Writer, level slog.Level, json bool) *slog.Logger
func RequestLogger(log *slog.Logger) func(http.Handler) http.Handler

// internal/adapters/httpapi — signature change
func NewHandler(svc TodoService, log *slog.Logger) http.Handler
```

The request log line has the message `"request"` and the attributes `method`, `path`, `status`, and `duration`.

### Step 1 — `NewLogger` (red → green)

In `internal/platform/logger_test.go` (`package platform_test`):

- **JSON output:** `NewLogger(&buf, slog.LevelInfo, true)`. Log `Info("hello", "k", "v")`. `buf` decodes as JSON with `msg == "hello"` and `k == "v"`.
- **Level filtering:** with `LevelWarn`, an `Info` call writes **nothing** (`buf.Len() == 0`), and a `Warn` call writes something.
- **Text output:** with `json == false`, the output contains `msg=hello`.

### Step 2 — Request-logging middleware (red → green)

**Test:** wrap a tiny inline handler, `http.HandlerFunc(func(w, r) { w.WriteHeader(http.StatusNotFound) })`, with `RequestLogger(logger)`. Serve `GET /todos/x` through it using `httptest`. Decode the log line and check:

- `msg == "request"`
- `method == "GET"`
- `path == "/todos/x"`
- `status == 404` (as `float64`)
- `duration` is present and is a number

Also check that the **response** still has status 404. The middleware must pass the response through, not swallow it.

<details><summary>Hint 1</summary>

Record `start := time.Now()`, call `next.ServeHTTP(...)` with your wrapped writer, then log `time.Since(start)`.

</details>

<details><summary>Hint 2</summary>

`type statusRecorder struct { http.ResponseWriter; status int }`, with a `WriteHeader(code int)` method that saves `code` and then calls the embedded `WriteHeader`. Pass `&statusRecorder{ResponseWriter: w, status: http.StatusOK}` to `next`.

</details>

### Step 3 — Implicit 200 (red → green)

**Test:** an inner handler that only calls `w.Write([]byte("ok"))`, with no `WriteHeader`. The logged `status` must be `200`.

If you initialised the recorder's status to 0, this goes red, which is exactly why this test exists.

### Step 4 — Log the real error behind a 500 (red → green)

Change `httpapi.NewHandler` to take a `*slog.Logger` too.

**This is a deliberate signature change, and your existing handler tests will stop compiling.** Update your `newTestHandler` helper to pass `slog.New(slog.DiscardHandler)`. Because almost every test builds its handler through that one helper, the fix is mostly one line. That's what the helper was for. The exception is the stage 4 Step 8 test, which calls `httpapi.NewHandler(failingService{})` directly; add the logger argument there too (you're about to change that test anyway).

**Test:** reuse the `failingService` from stage 4 (its `List` returns `errors.New("db exploded")`). Give `NewHandler` a JSON logger writing into a buffer. After `GET /todos`:

- the response is still 500 with the generic message, and does **not** contain `db exploded`
- the log contains a record at `level == "ERROR"` whose `err` attribute is `"db exploded"`

Also add a check that a **404 doesn't log at ERROR level**. Expected client errors aren't server failures; the request logger already records them at INFO.

<details><summary>Hint</summary>

Only the `default:` (500) branch of your error-mapping function logs. The function needs the request so it can call `h.log.ErrorContext(r.Context(), "request failed", "method", r.Method, "path", r.URL.Path, "err", err)`.

</details>

### Step 5 — Wire it into `main`

In `cmd/todo-api/main.go`:

1. Build the logger: `platform.NewLogger(os.Stdout, slog.LevelInfo, false)`. Use text for now, because it's easier to read locally. Stage 6 makes this configurable.
2. Pass it to `httpapi.NewHandler(svc, logger)`.
3. Wrap the handler: `platform.RequestLogger(logger)(apiHandler)`.
4. Replace the old `log.Println` and `log.Fatal` calls with `logger.Info("server starting", "addr", ":8080")`, and if `ListenAndServe` returns, `logger.Error(...)` followed by `os.Exit(1)`.

> `slog` has no `Fatal`. That's deliberate: exiting is a control-flow decision, not a logging one. Log, then call `os.Exit(1)`.

### Step 6 — Refactor

- If a test decodes log lines in several places, extract a helper such as `decodeLogLines(t, buf) []map[string]any`.
- Is `"request"` a good message? The attributes carry the detail, so the message should be a short, constant string that's easy to search for.

## Checkpoint

```sh
go test -race ./...
go vet ./...
gofmt -l .
go list -deps -f '{{if not .Standard}}{{.ImportPath}}{{end}}' ./internal/todo   # still only the core
```

Run the server and make a few requests:

```sh
go run ./cmd/todo-api
```

```sh
curl -s localhost:8080/todos
curl -s -X POST localhost:8080/todos -d '{"title":"buy milk"}'
curl -s localhost:8080/todos/nope
curl -s -X PUT localhost:8080/todos
```

The server terminal shows something like this:

```text
time=2026-09-29T22:40:40.978+01:00 level=INFO msg="server starting" addr=:8080
time=2026-09-29T22:40:42.012+01:00 level=INFO msg=request method=GET path=/todos status=200 duration=342.083µs
time=2026-09-29T22:40:42.037+01:00 level=INFO msg=request method=POST path=/todos status=201 duration=565.417µs
time=2026-09-29T22:40:42.056+01:00 level=INFO msg=request method=GET path=/todos/nope status=404 duration=42µs
time=2026-09-29T22:40:42.074+01:00 level=INFO msg=request method=PUT path=/todos status=405 duration=61.375µs
```

The 405 is logged too, even though your handlers never ran for it. The middleware wraps the whole mux, so it sees every request.

Switch `json` to `true` in `main`, restart, and compare:

```json
{"time":"2026-09-29T22:39:57.343501+01:00","level":"INFO","msg":"request","method":"GET","path":"/todos/x","status":404,"duration":792}
```

- [ ] Tests, `vet`, and `gofmt` are clean. The dependency rule still holds.
- [ ] Every request produces one log line with the correct status.

## Commit

```sh
git add cmd/ internal/
git commit -m "feat(logging): add slog logger, request logging middleware and 500 error logging"
```

Then tell Claude **"done with stage 5"**.

## Stretch goals

- **`ReplaceAttr`.** Use `HandlerOptions.ReplaceAttr` to write `duration` in milliseconds as a float (`duration_ms`), or to drop `time` in tests so that the text output is predictable.
- **Response size.** Extend your recorder to count bytes written, and log `bytes`.
- **A `Chain` helper.** Write `func Chain(h http.Handler, mw ...func(http.Handler) http.Handler) http.Handler`. Think about the order: which middleware should end up outermost?
- **A panic recovery middleware.** Catch panics with `defer func() { if v := recover(); v != nil { ... } }()`, log them at ERROR level with the stack (`debug.Stack()`), and return 500. Test it with a handler that panics. (net/http already recovers panics in each request's goroutine, but it writes to stderr and closes the connection, which isn't what you want.)

## Further reading

- [Go blog: Structured Logging with slog](https://go.dev/blog/slog)
- [Package `log/slog`](https://pkg.go.dev/log/slog)
- [Effective Go: Embedding](https://go.dev/doc/effective_go#embedding)

Next: [06 — Running it properly](06-running-properly.md)
