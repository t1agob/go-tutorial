# Learn modern Go by building a Todo API

This is a guided, hands-on path from "I used Go a while ago" to a small but well-structured REST API. **You write all the code.** The guides in [`docs/guide/`](docs/guide/) explain each concept, tell you which test to write next, and give hints when you get stuck.

By the end you'll have:

- a Todo REST API running locally, using only the standard library for HTTP
- structured logging with `log/slog`, request-logging middleware, and request IDs
- configuration from environment variables and a graceful shutdown
- a hexagonal (ports and adapters) layout, where the business logic knows nothing about HTTP or databases
- two interchangeable storage adapters (in-memory and SQLite), both checked by the same test suite
- every piece built test-first

## Prerequisites

- **Go 1.27+.** Check with `go version`.
- **git**
- **curl**, to try the API from stage 4 on
- An editor with Go support. VS Code with the official Go extension (which uses `gopls`) works well.

## How to use the guide

1. Start with [00 — Overview](docs/guide/00-overview.md). It explains the architecture and the TDD loop that every stage follows.
2. Do the stages **in order**, one at a time. Each stage builds on the code you wrote in the previous one.
3. In every stage, follow the loop: **write a test → watch it fail → make it pass → tidy up → repeat**.
4. Try to solve each step before you open a hint. Hints go from a gentle nudge to near-solution.
5. Finish each stage by running the **Checkpoint** commands and making the suggested **commit**.
6. Ask for a review (see below), then go on to the next stage.

## Asking for a review

When you've finished a stage, tell Claude **"done with stage N"**. Claude will run the tests, read your code, check the architecture rules, and give feedback in three groups:

- **Must fix**: bugs, failing checks, or broken layering
- **Idiomatic Go**: how an experienced Go developer would usually write it
- **Optional polish**

Claude won't change your code unless you ask. Before you start the next stage, Claude may update that stage's guide to match the names you actually chose.

## Stages

| # | Guide | You'll learn |
| --- | --- | --- |
| 0 | [Overview](docs/guide/00-overview.md) | Hexagonal architecture, the dependency rule, the TDD loop, what's changed in Go |
| 1 | [Tooling](docs/guide/01-tooling.md) | Modules, packages, `go test`, table-driven tests |
| 2 | [Domain](docs/guide/02-domain.md) | Structs, methods, errors as values, an injected clock |
| 3 | [Ports and service](docs/guide/03-ports-and-service.md) | Interfaces, fakes, `context`, an in-memory adapter, `sync` |
| 4 | [HTTP adapter](docs/guide/04-http.md) | `net/http` routing, JSON, `httptest`, error mapping |
| 5 | [Logging](docs/guide/05-logging.md) | `log/slog`, middleware, testing log output |
| 6 | [Running it properly](docs/guide/06-running-properly.md) | Config, graceful shutdown, request IDs via `context` |
| 7 | [SQLite adapter](docs/guide/07-sqlite.md) | `database/sql`, `embed`, one contract test suite run against both adapters |
