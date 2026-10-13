# Technical plan — Manager Library (Laravel + React)

Translation of [`../spec-python/plan.md`](../spec-python/plan.md) to the challenge stack. The business rules, user stories,
acceptance criteria, API contract and data model are **not** changed: they still live in `../spec-python/` (identical to
`../spec-.net/` except for technology). Only technology decisions are translated here. Where the original spec is ambiguous or
inconsistent, the interpretation applied is recorded in [`interpretations.md`](interpretations.md). Task status lives in
[`tasks.md`](tasks.md).

Precedence when in doubt: `constitution.md` → `spec.md` → `data-model.md` / `api-contract.md` → this plan.
If this plan and the original spec disagree, the original spec wins and this plan is wrong.

## 1. Technical decisions

| # | Decision | Original (Python) | Laravel / React translation | Why |
|---|---|---|---|---|
| D-01 | Language and runtime | Python 3.12 | PHP 8.3 (`php:8.3-cli` image). Laravel version: whatever `composer create-project laravel/laravel` installs on PHP 8.3; the exact version is recorded in `composer.lock`. Frontend: React with Vite on Node 22 | Challenge stack (`../README.md`) |
| D-02 | HTTP framework | FastAPI | Laravel routing. All routes in `routes/api.php` under the `api/v1` prefix (`apiPrefix: 'api/v1'` in `bootstrap/app.php`). `php artisan install:api` is **not** used because it installs Sanctum | Contract base path `/api/v1` |
| D-03 | Persistence | SQLAlchemy 2 + Alembic on PostgreSQL 16 | Laravel migrations + query builder / Eloquent on PostgreSQL 16, schema `library`. `CHECK` constraints and the partial unique index are created with `DB::statement(...)` inside the same migration (Laravel has no fluent API for them) | Article V, ADR-001 |
| D-04 | Concurrency | Version token per book + range `CHECK` (ADR-002) | Conditional `UPDATE book SET available_copies = available_copies - 1 WHERE id = ? AND available_copies > 0` inside `DB::transaction`; 0 affected rows → 409 `BOOK_NOT_AVAILABLE`, no retry. `ck_book_available_range` remains the last barrier. No version column is added (see interpretations I-01, I-02) | RNF-2 without changing the data model |
| D-05 | Dates | UTC instants (`timestamptz`); `Clock` port | `config('app.timezone') = 'UTC'`; columns `timestampTz`; `Clock` is an interface in `app/Application/Ports`, real adapter returns `CarbonImmutable::now('UTC')`, tests use a fixed clock | Test overdue without waiting 14 days |
| D-06 | Report | One aggregated query behind a read port (ADR-004) | `ReportReader` port; adapter runs one `GROUP BY book_id, book_title, category_name` query over `loan_item` joined to `loan`. Never joins the live catalog | Article VIII, ADR-004 |
| D-07 | Value objects | `Isbn`, `Quantity`, `DocumentNumber`, `DateRange` in the owner's row | `final readonly` PHP classes in `app/Domain`; stored as plain columns of the owner, no table | They have no identity |
| D-08 | Covers | External storage (disk locally, S3 in production); DB keeps `cover_key` | `CoverStorage` port; local adapter uses a Laravel filesystem disk rooted at `COVERS_DIR`. DB stores only the opaque key; `null` when absent, never `''` | CA-03.4 |
| D-09 | Passwords | Argon2id behind `PasswordHasher` | `password_hash($plain, PASSWORD_ARGON2ID)` / `password_verify` behind the `PasswordHasher` port. **Verified 2026-10-13 15:24:** `php:8.3-cli` (PHP 8.3.35) returns `true` for `defined("PASSWORD_ARGON2ID")` | Article IX |
| D-10 | Bootstrap admin | App creates the initial admin from `ADMIN_USERNAME` / `ADMIN_PASSWORD` | Idempotent Artisan command `library:bootstrap-admin`, run by the API container entrypoint after `php artisan migrate --force`. Not a seeder: the hash can only be produced by the `PasswordHasher` port | `data-model.md` §9 |
| D-11 | Authentication | JWT HS256, 8 h, `JWT_SECRET` | `firebase/php-jwt`, HS256, `exp = now + 28800`, claims `sub` and `role` only. Custom middleware validates the `Bearer` token and the role. **Sanctum is not used**: it issues opaque tokens, not JWT | `api-contract.md` §4 |
| D-12 | Tests | pytest, fakes for ports, real DB only in integration | PHPUnit (ships with Laravel) with in-memory fakes for ports; PostgreSQL only for integration and concurrency tests. Architecture tests (T-19): Pest `arch()` or a PHPUnit test that scans `use` statements — decided when T-19 is taken | Test pyramid |

## 2. Architecture mapping (hexagonal)

| Original (`src/library/…`) | Laravel (`app/…`) |
|---|---|
| `domain/` | `app/Domain/` — plain PHP only: no `Illuminate\*`, no Eloquent |
| `application/ports/outbound/` | `app/Application/Ports/` — interfaces, no framework types in signatures |
| `application/services/` | `app/Application/Services/` — one use case per class (`LendBooksService`, …) |
| `adapters/inbound/http/` | `app/Adapters/Http/` — controllers, form requests, error rendering |
| `adapters/outbound/{persistence,storage,security}/` | `app/Adapters/Persistence/`, `app/Adapters/Storage/`, `app/Adapters/Security/` |
| `bootstrap/composition/port_bindings.py` | `app/Bootstrap/LibraryServiceProvider.php` — the **only** place where a port is bound to an adapter |

## 3. Laravel defaults that would break the spec (and what is done instead)

| Laravel default | Spec rule it breaks | What is done |
|---|---|---|
| Migrations `users`, `password_reset_tokens`, `sessions`, `cache`, `jobs` | Six tables only (`data-model.md` §2) | Deleted before the first migration runs |
| `$table->timestamps()` and Eloquent `$timestamps = true` | No `created_at`/`updated_at` (`data-model.md` §3) | Never used; models set `public $timestamps = false` |
| `$table->uuid('id')->default(...)` / any `->default()` / `->useCurrent()` | No column has `DEFAULT` (`data-model.md` §3) | IDs generated by the application (UUID v4); no defaults in migrations |
| Laravel's own `migrations` table | Six tables in schema `library` | Kept outside schema `library` (e.g. `public.migrations`). **To verify** with the `data-model.md` §10 queries |
| Sanctum (`install:api`) | JWT with `sub` and `role` | Not installed; see D-11 |
| Validation 422 body `{message, errors}` | Single error body `{code, message, details}` (RNF-4) | Exception rendering in `bootstrap/app.php` maps every exception to the contract body |
| Paginator JSON (`data`, `links`, `meta`, …) | `{items, page, size, total}` | Responses built by hand from `LengthAwarePaginator` |
| `users` model / `User` auth guard | Table name is `user` in schema `library` | Own `user` table and JWT middleware; Laravel auth guard not used |
| Route order `/loans/{id}` before `/loans/overdue` | `GET /loans/overdue` must resolve | `/loans/overdue` declared first and `{id}` constrained with `whereUuid` |

## 4. Configuration per environment

| Original variable | Laravel variable | Required | Example |
|---|---|---|---|
| `DATABASE_URL` | `DB_URL` (or `DB_CONNECTION=pgsql` + `DB_HOST`/`DB_PORT`/`DB_DATABASE`/`DB_USERNAME`/`DB_PASSWORD`) | yes | `postgresql://library:library@db:5432/library` |
| `JWT_SECRET` | `JWT_SECRET` | yes | long random string |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | same | yes | set in `.env`, never in the repo |
| `COVERS_DIR` | same | no | `storage/app/covers` |
| `CORS_ORIGIN` | same, read by `config/cors.php` | yes | `http://localhost:5173` |
| — | `APP_KEY` | yes (Laravel) | generated with `php artisan key:generate`, never committed |

Only `.env.example` is versioned (Article IX).

## 5. Test strategy

| Level | What | Against |
|---|---|---|
| Domain | Max 3, 14-day due date, copies in range, ISBN 13 digits | Plain PHP objects |
| Application | Lend, return, report | Fakes of the ports |
| Integration | Repositories and report query | Real PostgreSQL 16 (container) |
| Contract | Single error body and status codes | Running API |
| Architecture | Articles I, II and III | See D-12 |
| Concurrency | Two loans of the last copy | Real PostgreSQL 16 |

## 6. Build order

Same as the original: skeleton and health → initial migration with seed → domain → use cases with fakes → adapters → API →
contract and concurrency tests. Time-boxed priority for this delivery: the **core** (T-01 to T-17), see [`tasks.md`](tasks.md).
