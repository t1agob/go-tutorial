# 04 — HTTP adapter: a REST API with the standard library

## Goal

Build the **driving adapter**: HTTP handlers that turn requests into service calls and service results into JSON. Test them with `httptest`, with no real network involved. Then write `main.go` to wire everything together, and try the API with `curl`.

At the end of this stage you have a running REST API.

Time: about 2 hours.

## Concepts

### Routing with the standard library

Since Go 1.22, `http.ServeMux` understands **methods** and **path wildcards**, so you no longer need a third-party router for this:

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /books", listBooks)
mux.HandleFunc("GET /books/{isbn}", getBook)
mux.HandleFunc("POST /books/{isbn}/reviews", addReview)

func getBook(w http.ResponseWriter, r *http.Request) {
    isbn := r.PathValue("isbn")
    // ...
}
```

- `"GET /books/{isbn}"` matches `/books/978-...` but not `/books` or `/books/x/y`.
- A registered `GET` pattern also handles `HEAD` requests automatically.
- If the path matches but the method doesn't, the mux answers **`405 Method Not Allowed`** by itself, with an `Allow` header listing the valid methods.
- If nothing matches, it answers **`404 page not found`**.

> **If you remember old Go…** you probably used `gorilla/mux`, `chi`, or `httprouter` for `{id}` and method matching, and wrote `if r.Method != "POST"` checks by hand. You don't need them for this kind of API any more.

### Handlers

An `http.Handler` is anything with a `ServeHTTP(http.ResponseWriter, *http.Request)` method. `http.HandlerFunc` adapts an ordinary function into one. A `*ServeMux` is itself a `Handler`, and that's what your adapter will return.

When writing a response, **the order matters**:

```go
w.Header().Set("Content-Type", "application/json") // 1. headers
w.WriteHeader(http.StatusCreated)                   // 2. status (once)
w.Write(body)                                       // 3. body
```

After `WriteHeader` or the first `Write`, header changes are ignored. If you never call `WriteHeader`, the first `Write` sends `200 OK`.

### JSON

```go
type book struct {
    ISBN      string     `json:"isbn"`
    Title     string     `json:"title"`
    Published *time.Time `json:"published,omitempty"` // left out when nil
}
```

- **Struct tags** control the JSON field names. Only **exported** fields are encoded.
- `time.Time` is encoded as RFC 3339, for example `"2026-09-29T10:00:00Z"`.
- A `nil` slice is encoded as `null`, and an empty slice as `[]` (see stage 3).

Decoding a request body:

```go
r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // refuse bodies over 1 MiB
dec := json.NewDecoder(r.Body)
dec.DisallowUnknownFields()                    // {"titel": "..."} becomes an error, not silently ignored
var req createBookRequest
if err := dec.Decode(&req); err != nil {
    // 400 Bad Request
}
```

Encoding a response: `json.NewEncoder(w).Encode(v)`. It writes the JSON followed by a newline.

### Keep HTTP types separate from domain types

It's tempting to put `json:"..."` tags straight onto `todo.Todo`. **Don't.** In hexagonal terms, the JSON shape belongs to the HTTP adapter, not the core:

- The core stays free of transport concerns.
- You can rename a domain field without breaking API clients, or change the API without touching the domain.
- Request types (`createRequest{Title}`) accept only what clients are *allowed* to send. A client can't set `ID` or `Completed` by adding fields to the body.

So the adapter defines its own **DTOs** (data transfer objects), such as a `todoResponse` struct, plus a small function to convert a `todo.Todo` into one.

### The adapter declares what it needs

The handler needs "something with Create, Get, List, Complete, and Delete". Following the stage 3 rule ("interfaces belong to the consumer"), declare a small `TodoService` interface **in the adapter package**. `*todo.Service` satisfies it automatically, and in tests you can substitute something else, for example a service that always fails, to test 500 responses.

### Mapping errors to status codes

This is the adapter's main job. Use one function to translate domain errors to HTTP:

| Domain error | HTTP status |
| --- | --- |
| `todo.ErrNotFound` | 404 Not Found |
| `todo.ErrEmptyTitle`, `todo.ErrTitleTooLong` | 400 Bad Request |
| Malformed JSON body | 400 Bad Request |
| **Anything else** | 500 Internal Server Error, with a **generic** message |

Never send unexpected error text to clients. It can leak internal details such as SQL or file paths. From stage 5 on, you'll log the real error instead.

The error body is always `{"error": "<message>"}`.

### Testing handlers with `httptest`

```go
req := httptest.NewRequest(http.MethodGet, "/books/123", nil)
rec := httptest.NewRecorder()

handler.ServeHTTP(rec, req)

rec.Code                        // status code
rec.Body.String()               // response body
rec.Header().Get("Content-Type")
```

No network or port is involved, and it runs in microseconds. The request goes through the real mux, so routing, path values, and 405 handling are all tested for real.

(`httptest.NewServer` starts a real server on a random port, when you need a real network client. We don't need it yet.)

**Real or fake service in handler tests?** Use the **real** `todo.Service` with the **real** `memory.Repository`, fixed clock and ID. It's fast, and it tests the full path through the core. Use a small fake `TodoService` only for cases that are hard to trigger with the real one, such as "the service returned an unexpected error".

### Package naming

The standard library already has a package called `http`. If your adapter were also named `http`, every file importing both would need an alias. Name the directory and package **`httpapi`**: `internal/adapters/httpapi`, `package httpapi`. In Go, the directory name and package name should match.

## Steps

What you're building, in `internal/adapters/httpapi`:

```go
type TodoService interface { /* the five methods of *todo.Service */ }

func NewHandler(svc TodoService) http.Handler
```

The contract:

| Method and path | Success | Errors |
| --- | --- | --- |
| `POST /todos` body `{"title": "..."}` | **201** + todo JSON | 400 invalid JSON or validation error |
| `GET /todos` | **200** + JSON array (**`[]` when empty, never `null`**) | — |
| `GET /todos/{id}` | **200** + todo JSON | 404 |
| `PATCH /todos/{id}/complete` | **200** + todo JSON | 404 |
| `DELETE /todos/{id}` | **204**, empty body | 404 |

The todo JSON shape:

```json
{"id":"id-1","title":"buy milk","completed":false,"created_at":"2026-09-29T10:00:00Z"}
```

When completed, it also has `"completed_at":"..."`. The field is omitted when not completed.

All JSON responses set `Content-Type: application/json`.

### Step 1 — Test helpers

Create `internal/adapters/httpapi/handler_test.go` (`package httpapi_test`) with:

- a fixed `t0` (`time.Date(2026, 9, 29, 10, 0, 0, 0, time.UTC)`)
- `newTestHandler() http.Handler`, which builds `memory.New()` → `todo.NewService(repo, fixedClock, fixedID)` → `httpapi.NewHandler(svc)`
- a helper like `do(t, h, method, path, body string) *httptest.ResponseRecorder`, using `strings.NewReader(body)` for the body and `t.Helper()`

Then create `handler.go` with a `TodoService` interface, a `NewHandler` that returns an empty `http.NewServeMux()`, and nothing else yet.

### Step 2 — `GET /todos` on an empty store (red → green)

**Test:** status 200, `Content-Type: application/json`, and the body is `[]`. The encoder adds a trailing newline, so compare with `strings.TrimSpace(body)`.

Red first: with no route registered, you'll get a 404 from the mux.

<details><summary>Hint 1</summary>

Keep the service in a small struct (`type handler struct{ svc TodoService }`) and make each route a method on it, such as `h.list`. Register them in `NewHandler` with `mux.HandleFunc("GET /todos", h.list)`.

</details>

<details><summary>Hint 2</summary>

Write two helpers now, because you'll use them everywhere: `writeJSON(w, status, v any)`, which sets the header, writes the status, and encodes, and `writeError(w, status, msg string)`, which calls `writeJSON` with `map[string]string{"error": msg}`.

</details>

<details><summary>Hint 3</summary>

If you get `null` instead of `[]`, check how you build the response slice. `make([]todoResponse, 0, len(todos))` is never nil.

</details>

### Step 3 — `POST /todos` (red → green)

**Test:** POST `{"title":"buy milk"}` returns 201 and exactly this body:

```json
{"id":"id-1","title":"buy milk","completed":false,"created_at":"2026-09-29T10:00:00Z"}
```

Comparing the whole JSON string is fine here, because the output is fully predictable (fixed clock, fixed ID, fixed field order). If you'd rather compare fields, decode the body into a struct first.

This step introduces the `createRequest` and `todoResponse` DTOs and the conversion function.

### Step 4 — `POST /todos` errors (red → green)

A table-driven test, with a status code and an error message for each case:

| body | status | `error` |
| --- | --- | --- |
| `{"title":` (truncated) | 400 | `invalid JSON body` (your wording) |
| `{"title":"x","extra":1}` | 400 | `invalid JSON body` |
| `{"title":"  "}` | 400 | `title must not be empty` (the domain message) |
| 201 × `"a"` | 400 | the too-long message |

This is where you write the `errors.Is` → status mapping function. Keep it to **one** function that every handler calls when an error comes back from the service.

<details><summary>Hint</summary>

A `switch` with no value and `case errors.Is(err, todo.ErrNotFound):` branches reads well. Put the 500 case in `default`.

</details>

### Step 5 — `GET /todos/{id}` (red → green)

- After creating a todo, `GET /todos/id-1` returns 200 and the todo.
- `GET /todos/nope` returns 404 and `{"error":"todo not found"}`.

Remember `r.PathValue("id")`.

### Step 6 — `PATCH /todos/{id}/complete` (red → green)

- After creating, PATCH returns 200 with `"completed":true` and a `completed_at`.
- An unknown ID returns 404.

### Step 7 — `DELETE /todos/{id}` (red → green)

- After creating, DELETE returns **204 with an empty body**, and a following GET returns 404.
- An unknown ID returns 404.

For 204, call `w.WriteHeader(http.StatusNoContent)` and write nothing else.

### Step 8 — Unexpected errors become a generic 500 (red → green)

Here a fake is the right tool. In the test file:

```go
type failingService struct{ httpapi.TodoService } // embeds the interface
```

Then give it **one** method, `List`, that returns `nil, errors.New("db exploded")`. Embedding the interface means the type satisfies `TodoService` without you writing the other four methods. If one of them were called, it would panic on a nil interface, which in a test is a loud and useful failure.

**Test:** `GET /todos` with this service returns **500**, and the body does **not** contain `"db exploded"`.

### Step 9 — Method not allowed (verify)

**Test:** `PUT /todos` returns **405**, with an `Allow` header containing `GET` and `POST`. You wrote no code for this; the mux does it. Write the test anyway, because it documents the behaviour and would catch a routing mistake.

Note that the 405 body is plain text (`Method Not Allowed`), not your JSON error format. That's acceptable for this project; see the stretch goals.

### Step 10 — Wire it up in `main`

Create `cmd/todo-api/main.go` (`package main`). It's the **composition root**, the one place that knows about every package:

1. build the memory repository
2. build the service with the real clock and `todo.NewID`
3. build the handler
4. `http.ListenAndServe(":8080", handler)`, and if that returns an error, log it and exit (`log.Fatal`)

**Clock:** pass a function that returns `time.Now().UTC()`, not `time.Now` itself. Otherwise timestamps come out in your local time zone (`"2026-09-29T22:38:21+01:00"`). That's valid, but inconsistent, and it will matter when you store times in SQLite in stage 7.

`main` has no unit test. It's pure wiring, and every piece it wires is already tested. Stage 6 will restructure it so that it can be tested.

> Stage 5 replaces `log` with `slog`, and stage 6 adds graceful shutdown. For now, keep `main` minimal.

## Checkpoint

```sh
go test -race ./...
go vet ./...
gofmt -l .
go list -deps -f '{{if not .Standard}}{{.ImportPath}}{{end}}' ./internal/todo   # still only the core
```

Then run the server in one terminal:

```sh
go run ./cmd/todo-api
```

…and in another terminal, try it out. Your IDs and timestamps will differ.

```sh
curl -i localhost:8080/todos
```

```text
HTTP/1.1 200 OK
Content-Type: application/json
Date: ...
Content-Length: 3

[]
```

```sh
curl -i -X POST localhost:8080/todos -d '{"title":"buy milk"}'
```

```text
HTTP/1.1 201 Created
Content-Type: application/json
...

{"id":"c8e4fd3f4ec002b3fd8a41a970cba2da","title":"buy milk","completed":false,"created_at":"2026-09-29T21:38:21.686591Z"}
```

Copy the `id` from that output, then:

```sh
ID=c8e4fd3f4ec002b3fd8a41a970cba2da   # paste yours

curl localhost:8080/todos/$ID
curl -X PATCH localhost:8080/todos/$ID/complete
# {"id":"...","title":"buy milk","completed":true,"created_at":"...","completed_at":"..."}

curl localhost:8080/todos
# [{"id":"...", ... }]

curl -i -X DELETE localhost:8080/todos/$ID
# HTTP/1.1 204 No Content

curl -i localhost:8080/todos/$ID
# HTTP/1.1 404 Not Found
# {"error":"todo not found"}

curl -X POST localhost:8080/todos -d '{"title": ""}'
# {"error":"title must not be empty"}

curl -X POST localhost:8080/todos -d 'nope'
# {"error":"invalid JSON body"}

curl -i -X PUT localhost:8080/todos
# HTTP/1.1 405 Method Not Allowed
# Allow: GET, HEAD, POST
```

Stop the server with Ctrl-C. Everything is in memory, so the todos are gone. Stage 7 fixes that.

- [ ] Tests, `vet`, and `gofmt` are clean. The dependency rule still holds.
- [ ] Every `curl` above behaves as shown.

## Commit

```sh
git add cmd/ internal/
git commit -m "feat(http): add REST adapter and main entrypoint"
```

Then tell Claude **"done with stage 4"**.

## Stretch goals

- **Filtering.** Support `GET /todos?completed=true`. Read it with `r.URL.Query().Get("completed")` and `strconv.ParseBool`, and return 400 for an invalid value. Where should the filtering happen: in the handler, the service, or the repository? There's a trade-off, so be ready to explain your choice.
- **Pagination.** `?limit=10&offset=20`. Validate the values and cap `limit`.
- **JSON 405s and 404s.** Wrap the mux in your own handler so that unmatched routes return the `{"error": ...}` format. This is harder than it sounds; look at what information the mux gives you.
- **Location header.** On 201, set `Location: /todos/{id}`. It's a common REST convention.
- **`httptest.NewServer`.** Write one test that starts a real server and uses `http.Get(srv.URL + "/todos")`. When is this worth the extra cost?

## Further reading

- [Go blog: Routing enhancements for Go 1.22](https://go.dev/blog/routing-enhancements)
- [Package `net/http`: `ServeMux` patterns](https://pkg.go.dev/net/http#hdr-Patterns-ServeMux)
- [Package `net/http/httptest`](https://pkg.go.dev/net/http/httptest)
- [Go blog: JSON and Go](https://go.dev/blog/json)
- [Package `encoding/json`](https://pkg.go.dev/encoding/json): see `Decoder.DisallowUnknownFields`
- [Tutorial: Developing a RESTful API with Go](https://go.dev/doc/tutorial/web-service-gin): it uses Gin, which is interesting for comparing with the standard library

Next: [05 — Logging](05-logging.md)
