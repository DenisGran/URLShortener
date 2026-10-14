# rapid.codes low-level design (v1)

This document describes the inside of the service: the data model, then the
classes and interfaces. The decisions it refers to (D1, D2, ...) are listed in
`CLAUDE.md`.

## Data model

One table.

```sql
CREATE TABLE links (
    code        TEXT COLLATE "C" PRIMARY KEY
                CHECK (code ~ '^[0-9a-zA-Z]{7}$'),
    url         TEXT COLLATE "C" NOT NULL UNIQUE
                CHECK (octet_length(url) <= 2048),
    disabled    BOOLEAN     NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

| Column | Why it looks like this |
|---|---|
| `code` | 7 random Base62 characters (D15). Primary key (D17), so its index serves every redirect lookup. `COLLATE "C"` compares raw bytes: fast and case-sensitive (`aZ3` and `Az3` are different codes). The `CHECK` makes the database reject a malformed code even if the generator has a bug. |
| `url` | The normalized destination (D7). `UNIQUE` gives deduplication (D2, D3) and makes concurrent creation safe (D4). The byte-length `CHECK` (D12) keeps every value below the B-tree index entry limit (~2700 bytes). |
| `disabled` | `true` makes the redirect return `410 Gone` and a new `POST` return `422` with `url_disabled` (D6, D11, D18). |
| `created_at` | Set by the database clock, stored as a UTC instant (D19). |

### Indexes

| Index | Created by | Used for |
|---|---|---|
| `links_pkey` on `code` | `PRIMARY KEY` | `GET /{code}`, and rejecting duplicate codes (D15) |
| `links_url_key` on `url` | `UNIQUE` | Deduplication in `POST /api/v1/links` |

No other indexes in v1. Every index slows down inserts and takes space, so one
is added only when a query needs it.

### Queries

**Redirect** (`GET /{code}`):

```sql
SELECT url, disabled FROM links WHERE code = $1;
```

No row means `404 Not Found`. `disabled = true` means `410 Gone`. Otherwise
`302 Found` to `url`.

**Create** (`POST /api/v1/links`), for a URL that has already been validated
and normalized:

```sql
INSERT INTO links (code, url) VALUES ($1, $2)
ON CONFLICT (url) DO NOTHING
RETURNING code;
```

| Outcome | Meaning | Next step |
|---|---|---|
| A row is returned | New link | `201 Created` |
| No row is returned | The URL already exists (`ON CONFLICT (url)`) | `SELECT code, disabled FROM links WHERE url = $1`. Then `200 OK`, or `422` with `url_disabled` if it is disabled. |
| Unique violation on `links_pkey` | The random code is already taken | Generate a new code and try again, at most 3 attempts, then `503` (D15) |

Notes for the implementation:
- `ON CONFLICT (url)` only covers the `url` constraint. A collision on `code`
  is still raised as an error, which is what triggers the retry.
- In Postgres, an error aborts the current transaction. Each retry needs a new
  transaction, or a `SAVEPOINT` to roll back to.

## Classes and interfaces

To be designed next.
