# 06 — Running it properly: config, graceful shutdown, request IDs

## Goal

Make the server behave like a real service:

- **Configuration** from environment variables (`PORT`, `LOG_LEVEL`, `LOG_FORMAT`), with defaults and validation
- **Graceful shutdown**: on Ctrl-C or `SIGTERM`, stop accepting new connections, let in-flight requests finish, then exit
- a **request ID** for every request, returned in a response header and included in every log line for that request

Along the way, `main` shrinks to a few lines, and the real work moves into a testable `run` function.

Time: about 2 hours.

## Concepts

### Configuration from the environment

The [twelve-factor](https://12factor.net/config) convention is to configure services through **environment variables**. They work the same locally, in Docker, and in Kubernetes.

```go
os.Getenv("PORT")                // "" if unset or set to empty
v, ok := os.LookupEnv("PORT")    // ok == false only if unset
```

For most settings, "unset" and "empty" mean the same thing ("use the default"), so `Getenv` is fine.

**Make config testable by injecting the lookup function**:

```go
func LoadConfig(getenv func(string) string) (Config, error)
```

Production passes `os.Getenv`. Tests pass a closure over a map:

```go
env := map[string]string{"PORT": "9090"}
cfg, err := LoadConfig(func(k string) string { return env[k] })
```

This avoids changing real process-wide environment variables in tests. (`t.Setenv` exists and is safe, but it prevents a test from running in parallel.)

Validate config **at startup** and fail fast with a clear message. It's much better than finding a typo in `LOG_LEVEL` at 3 a.m.

### Parsing a log level

`slog.Level` can parse itself:

```go
var lvl slog.Level
err := lvl.UnmarshalText([]byte("debug"))   // also accepts "INFO", "warn", "ERROR", even "WARN+2"
// err for "loud": slog: level string "loud": unknown name
```

### The `run` pattern

A `main` that does everything is hard to test: it calls `os.Exit`, reads real environment variables, and writes to the real stdout. Instead:

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    if err := run(ctx, os.Getenv, os.Stdout); err != nil {
        fmt.Fprintln(os.Stderr, "error:", err)
        os.Exit(1)
    }
}

func run(ctx context.Context, getenv func(string) string, stdout io.Writer) error {
    // everything else
}
```

`main` is now three lines of glue. `run` takes all its dependencies as parameters and **returns an error instead of exiting**, so a test can call it.

### Signals and `signal.NotifyContext`

When you press Ctrl-C, the process receives `SIGINT` (`os.Interrupt`). When Kubernetes or Docker stop a container, they send `SIGTERM`. `signal.NotifyContext` gives you a **context that's cancelled when one of those signals arrives**:

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
<-ctx.Done()   // blocks until a signal arrives
```

### Graceful shutdown with `http.Server`

`http.ListenAndServe(addr, h)` is a shortcut with no way to stop it cleanly. Build an `http.Server` yourself:

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           h,
    ReadHeaderTimeout: 5 * time.Second, // protects against clients that never finish sending headers
}
```

- `srv.ListenAndServe()` **blocks** until the server stops. When it's stopped by `Shutdown`, it returns `http.ErrServerClosed`, which is the *normal* result and not a failure.
- `srv.Shutdown(ctx)` stops accepting new connections, **waits for in-flight requests to finish**, then returns. If `ctx` expires first, it gives up and returns the context's error.

Because `ListenAndServe` blocks, run it in a **goroutine**, and use a **channel** to get its result back:

```go
errCh := make(chan error, 1)        // buffered: the goroutine can send even if nobody is receiving yet
go func() { errCh <- srv.ListenAndServe() }()

select {
case err := <-errCh:                // server failed to start, e.g. port already in use
    return err
case <-ctx.Done():                  // a signal arrived
}
// … now call Shutdown with a deadline, then read errCh
```

`select` waits on several channel operations and runs whichever is ready first.

> **Shutdown deadline:** use a *fresh* context for `Shutdown`, such as `context.WithTimeout(context.Background(), 10*time.Second)`, not the already-cancelled signal context. Otherwise `Shutdown` gives up straight away.

### Request-scoped values in `context`

A **request ID** ties together all the log lines from one request. Middleware generates it, or keeps one sent by the client or an upstream proxy, and stores it in the request's context:

```go
type ctxKey struct{}   // unexported type, so no other package can collide with your key

ctx := context.WithValue(r.Context(), ctxKey{}, id)
next.ServeHTTP(w, r.WithContext(ctx))

// later, anywhere that has the context:
id, _ := ctx.Value(ctxKey{}).(string)   // the "comma ok" type assertion: no panic if missing
```

Rules:

- **Always use an unexported key type**, never a plain string like `"request_id"`. Two packages that both used the string `"id"` would silently overwrite each other's values.
- Put **only request-scoped data** in context: request IDs, auth identity, deadlines. **Never** use it to pass optional parameters or dependencies such as loggers or repositories.
- Provide a small accessor function (`RequestIDFrom(ctx) string`) so callers never see the key.

### Getting the request ID into every log line

You could add `"request_id", RequestIDFrom(ctx)` to every log call, but it's easy to forget. Remember the `...Context` variants from stage 5 (`log.InfoContext(ctx, ...)`)? A `slog.Handler` receives that `ctx`. So you can **wrap** the handler with one that adds the request ID automatically:

```go
type contextHandler struct{ slog.Handler }   // embed: pass everything through

func (h contextHandler) Handle(ctx context.Context, r slog.Record) error {
    // look something up in ctx; if present: r.AddAttrs(slog.String("key", value))
    return h.Handler.Handle(ctx, r)
}
```

It's the same embedding trick as the `statusRecorder` in stage 5. One catch: `slog.Handler` also has `WithAttrs` and `WithGroup` methods, which return a *new* handler. If you don't override them, calling `logger.With(...)` would return the *inner* handler, and your wrapper would disappear. Override both so they re-wrap the result.

### Middleware order

Middleware wraps from the inside out, so **the last one applied is the outermost**:

```go
h = httpapi.NewHandler(svc, log)   // innermost
h = RequestLogger(log)(h)
h = RequestID(h)                   // outermost: runs first
```

`RequestID` must be **outside** `RequestLogger`, because the logger needs the ID to already be in the context when it logs.

## Steps

What you're building, in `internal/platform`:

```go
type Config struct {
    Port      string     // PORT, default "8080"
    LogLevel  slog.Level // LOG_LEVEL, default info
    LogFormat string     // LOG_FORMAT, "text" (default) or "json"
}
func LoadConfig(getenv func(string) string) (Config, error)

const RequestIDHeader = "X-Request-ID"
func RequestID(next http.Handler) http.Handler
func RequestIDFrom(ctx context.Context) string
```

In `cmd/todo-api/main.go`: `main()` plus `run(ctx, getenv, stdout) error`.

### Step 1 — Config defaults (red → green)

In `internal/platform/config_test.go`: with an empty environment, `LoadConfig` returns `Port == "8080"`, `LogLevel == slog.LevelInfo`, `LogFormat == "text"`, and no error.

### Step 2 — Config overrides (red → green)

A table-driven test. Each case has an env map and the expected `Config`:

- `PORT=9090` gives `Port == "9090"`
- `LOG_LEVEL=debug` gives `slog.LevelDebug`, and `LOG_LEVEL=WARN` gives `slog.LevelWarn` (case-insensitive)
- `LOG_FORMAT=json` gives `"json"`

Since `Config` has only comparable fields, you can compare whole structs with `got != tt.want`.

### Step 3 — Invalid config (red → green)

- `LOG_LEVEL=loud` returns an error, and the message **mentions `LOG_LEVEL`**, so the user knows which variable is wrong.
- `LOG_FORMAT=xml` returns an error that mentions `LOG_FORMAT`.

<details><summary>Hint</summary>

Wrap the parse error to add the variable name: `fmt.Errorf("LOG_LEVEL: %w", err)`. The result reads `LOG_LEVEL: slog: level string "loud": unknown name`.

</details>

### Step 4 — Request-ID middleware (red → green)

In `internal/platform/requestid_test.go`, wrap an inner handler that saves `RequestIDFrom(r.Context())` into a variable declared in the test. Then write these tests:

1. **No incoming header:** the inner handler sees a non-empty ID, and the response header `X-Request-ID` has the **same** value.
2. **Incoming header `X-Request-ID: abc`:** the inner handler sees `"abc"`, and the response echoes `"abc"`.
3. **Two requests without a header** get **different** IDs.
4. `RequestIDFrom(context.Background())` returns `""` without panicking.

For generated IDs, 8 random bytes in hex is plenty. You already know how from `todo.NewID`. (Don't import `todo` from `platform` just for this; the platform package shouldn't depend on the domain.)

> Header names are case-insensitive. Go writes them in canonical form (`X-Request-Id`), and `Header.Get` finds them however you spell the name.

<details><summary>Hint</summary>

Set the response header *before* calling `next.ServeHTTP`. After the inner handler has written its response, it's too late to add headers.

</details>

### Step 5 — Request ID in the logs (red → green)

**Test:** build `RequestID(RequestLogger(logger)(inner))` with a JSON logger into a buffer. Send a request with `X-Request-ID: abc`. The request log line has `request_id == "abc"`.

Make it pass by adding a `contextHandler` wrapper inside `NewLogger`. Don't change `RequestLogger` itself: it already calls `LogAttrs(r.Context(), ...)`. (If yours calls `Info` without a context, switch it to a `...Context` or `LogAttrs` variant.)

Then add a second test: a record logged **without** a request ID in the context has **no** `request_id` key.

Finally, check the payoff: the ERROR line logged by `httpapi` for a 500 now includes `request_id` too, with no changes to `httpapi`, because it already uses `ErrorContext(r.Context(), ...)`.

<details><summary>Hint</summary>

`NewLogger` builds the text or JSON handler as before, then returns `slog.New(contextHandler{inner})`. Remember to override `WithAttrs` and `WithGroup` so they return `contextHandler{h.Handler.WithAttrs(attrs)}` and the equivalent for groups.

</details>

### Step 6 — Restructure `main` into `run`

Rewrite `cmd/todo-api/main.go` along the lines of the `run` pattern above:

1. `LoadConfig(getenv)` returns an error on invalid config.
2. Build the logger from the config.
3. Build the repository, service, and handler; wrap it with `RequestLogger`, then `RequestID`.
4. Build an `http.Server` with `Addr: net.JoinHostPort("", cfg.Port)` (which gives `":9090"`) and a `ReadHeaderTimeout`.
5. Start `ListenAndServe` in a goroutine, sending its result on a buffered channel.
6. `select` on that channel and `ctx.Done()`.
7. On shutdown: log `"shutting down"`, call `Shutdown` with a 10-second timeout context, read the goroutine's result from the channel (treating `http.ErrServerClosed` as success), log `"server stopped"`, and return `nil`.

This step has no unit test; the checkpoint below exercises it by hand. The stretch goal shows how to test `run` automatically.

<details><summary>Hint</summary>

Use `errors.Is(err, http.ErrServerClosed)` for the check, not `==`. It's a good habit even when `==` would work.

</details>

## Checkpoint

```sh
go test -race ./...
go vet ./...
gofmt -l .
go list -deps -f '{{if not .Standard}}{{.ImportPath}}{{end}}' ./internal/todo
```

**Config and request IDs.** Start the server with overrides:

```sh
LOG_LEVEL=debug PORT=9090 go run ./cmd/todo-api
```

In another terminal:

```sh
curl -si localhost:9090/todos | grep -i x-request-id
# X-Request-Id: 8c5f482bbb75e809

curl -s -H 'X-Request-ID: my-trace-123' localhost:9090/todos/nope
```

The server logs show the IDs:

```text
time=... level=INFO msg="server starting" addr=:9090
time=... level=INFO msg=request method=GET path=/todos status=200 duration=212.292µs request_id=8c5f482bbb75e809
time=... level=INFO msg=request method=GET path=/todos/nope status=404 duration=183.958µs request_id=my-trace-123
```

**Graceful shutdown.** Press Ctrl-C in the server terminal:

```text
time=... level=INFO msg="shutting down"
time=... level=INFO msg="server stopped"
```

The process exits with status 0 (`echo $?` prints `0`).

> `go run` builds a temporary binary and runs it. Ctrl-C reaches your program as normal. If you want to be sure you're testing exactly what ships, run `go build -o todo-api ./cmd/todo-api && ./todo-api` instead. `/todo-api` is already in `.gitignore`.

**Bad config fails fast:**

```sh
LOG_LEVEL=loud go run ./cmd/todo-api
# error: LOG_LEVEL: slog: level string "loud": unknown name
# exit status 1
```

**Port in use.** Start two servers on the same port. The second one exits straight away with `listen tcp :8080: bind: address already in use`. That's the `case err := <-errCh` branch of your `select`.

- [ ] Tests, `vet`, and `gofmt` are clean.
- [ ] Every request log line has a `request_id`, and the response has an `X-Request-Id` header.
- [ ] Ctrl-C prints the two shutdown lines and exits with status 0.

## Commit

```sh
git add cmd/ internal/
git commit -m "feat: add env config, graceful shutdown and request IDs"
```

Then tell Claude **"done with stage 6"**.

## Stretch goals

- **Prove in-flight requests finish.** Temporarily add a route that sleeps for 2 seconds before responding. Call it, and press Ctrl-C while it's running. The response still arrives, and the log shows `"shutting down"` *before* that request's log line, then `"server stopped"`. Remove the route afterwards.
- **Test `run` end to end.** In `cmd/todo-api/main_test.go` (`package main`), call `run` in a goroutine with a cancellable context, `PORT=0` (the OS picks a free port) and a buffer for stdout. Wait until it's listening, make a request, cancel the context, and check that `run` returns `nil`. The tricky part is finding out which port was chosen; think about how `run` could report it. This is a good design exercise.
- **More config.** Add a `SHUTDOWN_TIMEOUT` setting parsed with `time.ParseDuration` (for example `"15s"`), with validation.
- **`t.Setenv`.** Rewrite one config test to use `t.Setenv` with the real `os.Getenv`, and see what happens if you add `t.Parallel()`.

## Further reading

- [Package `os/signal`: `NotifyContext`](https://pkg.go.dev/os/signal#NotifyContext)
- [Package `net/http`: `Server.Shutdown`](https://pkg.go.dev/net/http#Server.Shutdown)
- [Package `context`](https://pkg.go.dev/context): read the package overview on `WithValue` and key types
- [Go blog: Contexts and structs](https://go.dev/blog/context-and-structs)
- [Effective Go: Channels](https://go.dev/doc/effective_go#channels)
- [Go Tour: Select](https://go.dev/tour/concurrency/5)

Next: [07 — SQLite adapter](07-sqlite.md)
