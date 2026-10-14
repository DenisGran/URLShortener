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
- Hosting: **Azure for Students** (GitHub Student Pack): $100 credit per year
  plus 12 months of free services (B1s VM, PostgreSQL Flexible Server B1ms).
  The Azure account is shared with Denis's other projects, so keep costs minimal.
  Create the account only close to deployment (the 12 months start at sign-up).
  Check Azure's load-testing rules before running load tests.
- Domain: **rapid.codes**, already owned by Denis.

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
| D3 | Dedup uses a **UNIQUE** index directly on `url` (`COLLATE "C"`). No hash column. | D12 caps URLs at 2048 bytes, below the B-tree entry limit (~2700 bytes). Measured: SHA-256 costs ~0.4 µs vs ~1 ms for the DB round trip, and the space difference is negligible at our scale, so the simpler design wins. Revisit (add a SHA-256 `url_hash`) if URLs become unbounded or the index must fit in RAM at large scale. |
| D4 | Concurrent creation is protected by the database constraint, using `INSERT ... ON CONFLICT (url) DO NOTHING` | "Check, then insert" is not safe under concurrency (race condition). |
| D5 | `201 Created` for a new link, `200 OK` when it already existed, with the same response body | Makes `POST` idempotent in practice. |
| D6 | A **disabled** link returns **`410 Gone`** on redirect, whatever the reason it was disabled | It existed and was intentionally removed. `404` is for codes that never existed. |
| D7 | URL normalization: only safe changes (lowercase scheme and host). Never change the path, trailing slash, or query order. | A wrong guess would redirect people to the wrong page. |
| D8 | Cache comes later, but the code is designed for it now: `LinkService` depends on a `LinkRepository` interface, and `CachedLinkRepository` will wrap the Postgres one | Make it work, make it right, make it fast. Shows Open/Closed and Dependency Inversion. |
| D9 | API on the same domain under `/api/v1/`. Reserved words (`api`, `docs`, `health`, `metrics`, ...) can never be short codes. | Simpler for v1. Avoids collisions with real paths. |
| D10 | Errors use **RFC 9457 Problem Details** (`application/problem+json`) plus a machine-readable `code` field (for example `"invalid_url"`) | A recognized standard instead of an invented format. `code` lets clients branch without parsing text. |
| D11 | `POST` with a URL whose link is disabled returns **`422`** with `code: "url_disabled"` | The request is valid but this URL is not allowed. Same shape a future Safe Browsing rejection will use. `410` would wrongly suggest the `/links` collection is gone. |
| D12 | URL limits: maximum **2048 bytes** (`octet_length`, not characters: non-ASCII characters take 2-4 bytes). Invalid request bodies return **`422`** (FastAPI's default). | Common practical URL limit. Keeps one status code for "understood but invalid". |
| D13 | **Reject any URL with an explicit port**, including default ones (`:80`, `:443`) | Denis's choice. Simplest rule, closes off links to unusual services. Cost: a few legitimate URLs (sites on `:8080`, or a pasted `:443`) are refused. |
| D14 | `201 Created` includes `Location: <short_url>` | Denis's choice. No extra endpoint in v1. Cost: strictly, `Location` should point to the created resource, not to the redirect; `GET /api/v1/links/{code}` can be added later. |
| D15 | Short codes are **7 random Base62 characters** (from `secrets`), with a **UNIQUE** index on `code`. On a collision, generate a new code and retry, at most 3 attempts, then `503`. | Unpredictable codes (no enumeration). Retry chance = links in use / 62^7, about 1 in 3.5 million at 1M links. Rejected: counter, salt, epoch+counter (predictable), hash slice (same as random), multiplicative scramble (easily reversed), Feistel (too complex), KGS and an in-memory pool (operational cost, RAM), 5 chars (too dense), 3 words (too long). |
| D16 | Rate limits per IP (per `/64` prefix for IPv6): `POST` 10/min, redirects 100/min, `404` responses 10/min. `429` with `Retry-After`. Counters must be shared across workers. | Enumeration produces mostly `404`s, so a strict `404` limit stops it without hurting real users. MAC addresses are not visible to the server. Known cost: users behind a shared NAT IP share the limit. |
| D17 | `code` is the **primary key** of `links`. No surrogate `id`. | The lookup index on `code` is needed anyway, so a separate `id` would only add a second index. Revisit if other tables need foreign keys, a link can have several codes (custom aliases), or a code must be changeable. |
| D18 | Disabling is a `disabled BOOLEAN NOT NULL DEFAULT false` column | Denis's choice: simplest. Cost: no record of when or why a link was disabled. |
| D19 | `created_at TIMESTAMPTZ NOT NULL DEFAULT now()` | The database clock is the single source of time for all workers. Postgres stores the instant in UTC (the offset is not kept). Local time is a display concern. |

## Abuse protection (required before going online)

- Rate limiting per IP for creation and redirects (D16)
- URL validation: only `http` and `https` schemes, a maximum length (D12), no
  explicit port (D13), and reject
  links that point back to `rapid.codes` itself (redirect loops)
- The ability to disable a link (see D6)
- Later: check new URLs against Google Safe Browsing

## Open decisions

- None right now.

## Azure cost savings (remind Denis before deployment, roadmap step 7)

The free 12-month allowances are per account, not per project: one B1s VM and
one Postgres server shared by all of Denis's projects.

1. One VM for all projects: a Docker Compose stack per project, one Caddy in
   front routing by domain (free Let's Encrypt HTTPS).
2. Slim images (`python:3.x-slim`) and 1-2 Uvicorn workers (1 GB RAM is shared).
3. Check the free list for a larger free Arm/AMD size (B2pts v2 / B2ats v2).
4. No Azure Load Balancer, Application Gateway, or Front Door. Caddy does the job.
5. One Postgres server, one database and one user per project.
6. Keep Postgres backup retention at 7 days (32 GB backup storage included).
7. After the free 12 months: consider Postgres in Docker on the VM.
8. DNS at the registrar or Cloudflare free, not Azure DNS. Behind Cloudflare,
   rate limiting must read the client IP from `CF-Connecting-IP`.
9. Check whether the public IPv4 address is billed.
10. Everything in one region (cross-region traffic costs money).
11. Images on GitHub Container Registry, not Azure Container Registry.
12. CI on GitHub Actions (free for public repos).
13. Monitoring off the VM (Datadog / New Relic from the Student Pack). Avoid
    Azure Monitor / Log Analytics (billed per GB).
14. Docker log rotation (`max-size`, `max-file`).
15. Redis (later) in Docker on the VM, not Azure Cache for Redis.
16. Load tests from the laptop or GitHub Actions, not Azure Load Testing.
17. Budget alert in Azure Cost Management (for example at $5 and $10).
18. Create the Azure account only close to deployment.

Open for step 6: load tests on a shared VM are distorted by the other projects.
Options: stop the other projects, publish local numbers, or use a temporary VM.

## Roadmap

0. Requirements (done)
1. API design: `docs/api.md` (done)
2. LLD: data model, classes, interfaces  ← **next**
3. Core implementation and unit tests
4. Observability: structured JSON logs, request IDs, Prometheus metrics, Grafana
5. HLD and scaling: Redis cache, multiple workers or instances
6. Load testing (k6 or Locust), locally and online. Report p50, p95, and p99 with context.
7. Deployment and documentation: README, ADRs in `docs/adr/`, live demo
