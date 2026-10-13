# CLAUDE.md: rapid.codes URL Shortener

Claude Code reads this file at the start of every session. It holds the project's
decisions and working rules. Keep it up to date when a decision changes.

## What this project is

A URL shortening service (like bitly) at **https://rapid.codes**, built as a
portfolio project for Denis's CV and hosted publicly on GitHub.
The focus is the backend, with a very simple frontend. The bar is "good enough to
run online and defend in a job interview".

Quality goals: SOLID design, high RPS, low latency, monitoring, structured
logging, a clean REST API, and clean documentation. Every performance number in
the README must come from a test that was actually run, with its context
(hardware, local or online, date).

## How we work together (most important section)

Denis is learning system design through this project. He must be able to explain
every line in the repo in an interview.

| Area | Who writes it | Claude's role |
|---|---|---|
| Core logic: Base62 encoder, short-code generation, `LinkService`, repositories | **Denis** | Explain the concept, give hints, review his code. **Do not write the implementation unless Denis explicitly asks for that specific piece.** |
| Unit tests for the core | **Denis** | Suggest test cases, review. Do not write them unless asked. |
| Boilerplate: Docker, docker-compose, CI, config, tooling | Claude | Write it and **explain every line**. |
| Design decisions and ADRs | **Denis decides and writes** | Lay out the options and trade-offs, then review. Never decide for him. |
| README and API docs | Claude drafts | Denis edits them into his own voice. |

Rules:
- Explain every new technical term the first time it appears, with a concrete
  example from this project.
- **Be brutally honest.** No flattery, no softening. If something is wrong,
  weak, or will not hold up in an interview, say so directly.
- Whenever an HTTP status code is mentioned, also say in words what it means,
  for example `410 Gone` (the link existed but was removed).
- Think step by step, show the reasoning, and be honest. If an idea of Denis's
  has a problem, say so clearly and explain why.
- Brainstorm before building. Present options, then wait for Denis's decision.
- Small, frequent commits using Conventional Commits (`feat:`, `fix:`, `test:`,
  `docs:`, `chore:`, `refactor:`).
- Do not commit, push, or merge unless Denis asks.

## Git workflow: GitHub Flow

- `main` is always working and deployable. **No `dev` branch.**
- Each change gets a short-lived branch such as `feat/base62-encoder` or
  `fix/empty-url`, then a Pull Request into `main`.
- CI (GitHub Actions) runs tests and lint on every PR. Merge only when green.
  Branch protection on `main` will be enabled later.

## Tech stack

- **Python** (latest stable, standard GIL build). The service is I/O-bound, so
  async plus multiple worker processes is the scaling model. Free-threaded
  Python is not needed.
- **FastAPI** with **Uvicorn** (ASGI server). Automatic OpenAPI/Swagger docs and
  Pydantic validation. Confirmed by Denis over Flask (the original brief), for
  native async and code-first OpenAPI docs that cannot drift from the code.
- **PostgreSQL**, **SQLAlchemy 2.0** (ORM), **Alembic** (migrations)
- **pytest** for tests
- **Docker** for running everything consistently
- Hosting: **free tier only**. Check the provider's current limits and its
  load-testing policy before deploying.

## Scope of v1

In scope:
- `POST /api/v1/links`: anonymous. Takes a long URL and returns a short code.
- `GET /{code}`: redirect to the long URL.

Out of scope for v1: custom aliases, expiration, click statistics, user
accounts, and the Redis cache.

## Decisions made

| # | Decision | Reason |
|---|---|---|
| D1 | Redirects use **302**, sent with `Cache-Control: no-store` | Every click reaches the server, so statistics can be added later and a disabled link stops working immediately. Cost: more load, which the cache will solve later. |
| D2 | **Same long URL gives the same short code** (deduplication) | Denis's choice. Disabling targets the destination, so it applies to everyone. |
| D3 | Dedup lookup uses a `url_hash` column (SHA-256 of the normalized URL) with a **UNIQUE** index | Long URLs can exceed Postgres's index entry size limit. |
| D4 | Concurrent creation is protected by the database constraint, using `INSERT ... ON CONFLICT (url_hash) DO NOTHING` | "Check, then insert" is not safe under concurrency (race condition). |
| D5 | `201 Created` for a new link, `200 OK` when it already existed, with the same response body | Makes `POST` idempotent in practice. |
| D6 | A **disabled** link returns **`410 Gone`** on redirect, whatever the reason it was disabled | It existed and was intentionally removed. `404` is for codes that never existed. |
| D7 | URL normalization: only safe changes (lowercase scheme and host). Never change the path, trailing slash, or query order. | A wrong guess would redirect people to the wrong page. |
| D8 | Cache comes later, but the code is designed for it now: `LinkService` depends on a `LinkRepository` interface, and `CachedLinkRepository` will wrap the Postgres one | Make it work, make it right, make it fast. Shows Open/Closed and Dependency Inversion. |
| D9 | API on the same domain under `/api/v1/`. Reserved words (`api`, `docs`, `health`, `metrics`, ...) can never be short codes. | Simpler for v1. Avoids collisions with real paths. |
| D10 | Errors use **RFC 9457 Problem Details** (`application/problem+json`) plus a machine-readable `code` field (for example `"invalid_url"`) | A recognized standard instead of an invented format. `code` lets clients branch without parsing text. |
| D11 | `POST` with a URL whose link is disabled returns **`422`** with `code: "url_disabled"` | The request is valid but this URL is not allowed. Same shape a future Safe Browsing rejection will use. `410` would wrongly suggest the `/links` collection is gone. |
| D12 | URL limits: maximum **2048** characters. Invalid request bodies return **`422`** (FastAPI's default). | Common practical URL limit. Keeps one status code for "understood but invalid". |
| D13 | **Reject any URL with an explicit port**, including default ones (`:80`, `:443`) | Denis's choice. Simplest rule, closes off links to unusual services. Cost: a few legitimate URLs (sites on `:8080`, or a pasted `:443`) are refused. |
| D14 | `201 Created` includes `Location: <short_url>` | Denis's choice. No extra endpoint in v1. Cost: strictly, `Location` should point to the created resource, not to the redirect; `GET /api/v1/links/{code}` can be added later. |

## Abuse protection (required before going online)

- Rate limiting per IP for link creation (for example, 10 per minute)
- URL validation: only `http` and `https` schemes, a maximum length (D12), no
  explicit port (D13), and reject
  links that point back to `rapid.codes` itself (redirect loops)
- The ability to disable a link (see D6)
- Later: check new URLs against Google Safe Browsing

## Open decisions

- **Short-code strategy.** Denis proposed `encode(salt + counter)`. Under
  discussion: adding a salt only shifts the counter, so codes stay sequential.
  Alternatives are a keyed scramble of the counter (a bijection) or 7 random
  Base62 characters with a retry on collision.

## Roadmap

0. Requirements (done)
1. API design: `docs/api.md` (done)
2. LLD: data model, classes, interfaces  ← **next**
3. Core implementation and unit tests
4. Observability: structured JSON logs, request IDs, Prometheus metrics, Grafana
5. HLD and scaling: Redis cache, multiple workers or instances
6. Load testing (k6 or Locust), locally and online. Report p50, p95, and p99 with context.
7. Deployment and documentation: README, ADRs in `docs/adr/`, live demo
