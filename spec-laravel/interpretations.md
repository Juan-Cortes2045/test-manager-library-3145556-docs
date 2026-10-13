# Interpretations log — Manager Library (Laravel + React)

The original spec (`../spec-python/`, identical to `../spec-.net/` except for technology) is **not edited**. This log records
gaps and inconsistencies found while translating it, the evidence for each one, and the interpretation the implementation
follows. None of them adds tables, columns, endpoints or features: the data model and scope stay as specified.

Status values: **applied** (the implementation follows it), **proposed** (needs confirmation from the spec owner).
Line references point to `../spec-python/`; the same lines exist in `../spec-.net/`.

## Findings with an interpretation

### I-01 · ADR-002 mentions a version token that the data model does not have

- **Finding:** ADR-002 says "the book carries a version token", but table `book` has no version column.
- **Evidence:** `adr/adr-002-concurrencia-optimista.md:6` vs. `data-model.md:63-75` (columns of `book`).
- **Interpretation:** the data model wins (Article X). Optimistic concurrency is implemented with a conditional update
  `UPDATE book SET available_copies = available_copies - 1 WHERE id = ? AND available_copies > 0`; zero affected rows means
  the copy is gone. `ck_book_available_range` (`data-model.md` §4) remains the last barrier. No column is added.
- **Status:** applied.

### I-02 · ADR-002 offers two outcomes on conflict

- **Finding:** on a version conflict the server "responds 409 `BOOK_NOT_AVAILABLE` (or retries once if the copy is still available)".
- **Evidence:** `adr/adr-002-concurrencia-optimista.md:7`.
- **Interpretation:** respond **409 `BOOK_NOT_AVAILABLE` without retry**. With the conditional update of I-01 a retry adds nothing:
  if the update affected zero rows, no copy is available. Nothing is persisted (CA-06.3).
- **Status:** applied.

### I-03 · 401 covers three cases but only one error code exists

- **Finding:** 401 is used for "no token, invalid or expired token, invalid credentials", but the only 401 code listed is
  `INVALID_CREDENTIALS` (origin CA-01.2, i.e. login).
- **Evidence:** `api-contract.md:36` and `api-contract.md:46`.
- **Interpretation:** `POST /auth/login` with wrong credentials → `INVALID_CREDENTIALS`. Missing, malformed, invalid or expired
  token on any other endpoint → 401 with code `UNAUTHENTICATED` and the same single error body. The message never says
  which part failed.
- **Status:** proposed (introduces a `code` value not listed in the contract; the status code and body shape are unchanged).

### I-04 · `GET /books/{id}/cover` is used but not listed

- **Finding:** the contract says covers are rendered with `GET /books/{id}/cover` (binary or 404), but the endpoint table only lists
  `PUT` and `DELETE` for `/books/{id}/cover`, and no role is given for `GET`.
- **Evidence:** `api-contract.md:121` vs. `api-contract.md:101-109`.
- **Interpretation:** treat it as part of the contract: `GET /books/{id}/cover` → 200 with the image (`image/jpeg` or `image/png`)
  or 404 (no cover, unknown or soft-deleted book). Role: **authenticated** (same as `GET /books/{id}`), to be confirmed.
- **Status:** proposed (role to be confirmed).

### I-05 · "422 with the current count" does not say where the count goes

- **Finding:** CA-06.4 requires a 422 "with the current count" but no field is defined.
- **Evidence:** `spec.md:63`; error body in `api-contract.md:19-28`; code `MEMBER_LIMIT_EXCEEDED` in `api-contract.md:56`.
- **Interpretation:** 422 `MEMBER_LIMIT_EXCEEDED` with
  `details: [{ "field": "member_id", "issue": "tiene N libros sin devolver; máximo 3" }]`, where `N` is the current count of
  unreturned items. The `message` also states the count in Spanish.
- **Status:** applied.

### I-06 · `days_late` and `from`/`to` in `GET /loans` are not fully defined

- **Finding:** CA-09.1 asks for "days late" without unit or rounding. `GET /loans` accepts `from`/`to` without format; only the
  report defines them as "date, both inclusive".
- **Evidence:** `spec.md:80`; `api-contract.md:136`; `api-contract.md:160`; `api-contract.md:167`.
- **Interpretation:**
  - `days_late` = number of **started** 24-hour periods since `due_at` (`ceil((now - due_at) / 24 h)`), so any item listed as
    overdue has `days_late >= 1`. Computed, never stored (Article VII).
  - `from` and `to` in `GET /loans` use the report's convention: dates `YYYY-MM-DD`, both inclusive, in UTC, i.e.
    `loaned_at >= from 00:00:00Z AND loaned_at < (to + 1 day) 00:00:00Z`. Both optional. `to < from` → 422 `INVALID_DATE_RANGE`.
- **Status:** applied.

### I-07 · `active_loans` does not say whether it counts loans or books

- **Finding:** member detail returns `active_loans` described as "count of unreturned".
- **Evidence:** `api-contract.md:128`; RN-2 in `spec.md:95`; `spec.md:63` (CA-06.4 counts books).
- **Interpretation:** `active_loans` = number of **unreturned loan items (books)** of the member, the same number CA-06.4 checks
  against the limit of 3. The field name is kept as in the contract.
- **Status:** applied.

### I-08 · `infra` must start the API and the frontend, which live in other repositories

- **Finding:** T-20 requires `docker compose up` to start the empty engine, the API and the frontend, but they are separate repos,
  and nothing says who runs the migrations.
- **Evidence:** `tasks.md:26`; Article V in `constitution.md:37-41`; ADR-001.
- **Interpretation:** the six repos are cloned side by side; `infra/docker-compose.yml` uses relative build contexts
  (`../test-manager-library-3145556-api`, `../test-manager-library-3145556-app`). The API container entrypoint runs
  `php artisan migrate --force` and then `php artisan library:bootstrap-admin`. `infra` contains no DDL and no `CREATE TABLE`.
- **Status:** applied (to be documented in the `infra` README).

## Minor inconsistencies recorded without resolution

These do not block the delivery and are left for the spec owner.

| # | Inconsistency | Evidence |
|---|---|---|
| M-01 | RNF-1 measures search with 5 000 books using the `tool` seeder, but T-21 only asks the seeder for 40 or more books | `spec.md:105` vs. `tasks.md:27` |
| M-02 | CA-10.4 talks about renaming a category, but categories are a fixed seed "without maintenance" and no endpoint changes them | `spec.md:87` vs. `data-model.md:30` |
| M-03 | "Wins" is used with two scopes: `data-model.md` wins over other documents, while the engine wins over `data-model.md` | `data-model.md:7` vs. `constitution.md:64-66` |
| M-04 | The content of the public static site (`page`) is not defined beyond "presentation", "no API", "mobile and desktop" | `tasks.md:29`; `arquitectura-panoramica.md:26` |
