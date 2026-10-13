# Tasks — Manager Library (Laravel + React)

Translation of [`../spec-python/tasks.md`](../spec-python/tasks.md) to the challenge stack. Task IDs, dependencies and the
expected evidence are the original ones; only the technology wording changes. Interpretations referenced as `I-xx` are in
[`interpretations.md`](interpretations.md).

**Rule (Article XII):** a task is **done** only when its evidence exists and can be executed. Each done task links to that
evidence (commit, test or command). Nothing is marked done because "it is coded".

Status values: **pending** (not started) · **in progress** (started, evidence not yet available) · **done** (evidence linked).

Last updated: 2026-10-13, 15:20 (UTC-5). At this point no implementation exists in `api`, `app`, `page`, `infra` or `tool`:
each one contains only its initial `README.md`.

## Core (T-01 to T-17)

| # | Task | Laravel translation | Depends on | Evidence required | Status | Evidence link |
|---|---|---|---|---|---|---|
| T-01 | Project skeleton and `GET /health` | Laravel app in `api`, route `GET /api/v1/health` | — | `GET /api/v1/health` returns `{"status":"ok"}`; the container starts with one command | pending | — |
| T-02 | Initial migration: six tables, FKs, unique indexes, five seeded categories | One Laravel migration in schema `library`; `CHECK` and partial index via `DB::statement`; default Laravel migrations removed; no `DEFAULT`, no timestamps | T-01 | The `data-model.md` §10 queries return what the document states; no column has `DEFAULT` | pending | — |
| T-03 | Domain: `Isbn`, `Quantity`, `DateRange` | `final readonly` classes in `app/Domain` | T-01 | Unit tests: ISBN 13 digits, quantity > 0, end ≥ start | pending | — |
| T-04 | Domain: `Book` (`withdraw`, `restock`, soft delete) | `app/Domain/Book.php` | T-03 | Tests: never < 0, never > total | pending | — |
| T-05 | Domain: `Loan` (`addItem`, `returnItem`, computed status, 14 days, max 3) | `app/Domain/Loan.php`, `LoanItem.php`; `Clock` port | T-04 | Tests with a fake `Clock`: `open`, `overdue`, `closed` | pending | — |
| T-06 | Domain: `Member` and `User` (name normalization, valid role) | `app/Domain/Member.php`, `User.php` | T-03 | `"Ana "` becomes `ana`; role `admin` via API rejected | pending | — |
| T-07 | Auth ports and use cases (`LoginService`, `CreateLibrarianService`) with fakes | `app/Application/Services`, ports in `app/Application/Ports` | T-06 | HU-01 and HU-02 pass against fakes | pending | — |
| T-08 | Book ports and use cases (create, search, edit, soft delete, cover) with fakes | Same layout | T-04 | HU-03 and HU-04 pass against fakes | pending | — |
| T-09 | Lend use case (`LendBooksService`) with fakes | Same layout | T-05, T-08 | HU-06 complete: limits, overdue, no copies (I-05) | pending | — |
| T-10 | Return and overdue use cases | Same layout | T-09 | HU-07 and HU-09; returning twice → `ITEM_ALREADY_RETURNED` (I-06 for `days_late`) | pending | — |
| T-11 | Most-borrowed report use case and its read port | `ReportReader` port | T-09 | HU-10; old title kept after rename (ADR-004) | pending | — |
| T-12 | Persistence adapter: book, member, user repositories | `app/Adapters/Persistence` | T-02, T-08 | Integration tests against real PostgreSQL 16 | pending | — |
| T-13 | Persistence adapter: loan repository + concurrency control | Conditional `UPDATE` on `available_copies` (I-01, I-02) | T-12, T-09 | Two loans of the last copy: one 201, the other 409 (RNF-2) | pending | — |
| T-14 | Security adapter: password hash and JWT | Argon2id via `password_hash`; `firebase/php-jwt` (D-09, D-11) | T-07 | The password never appears in logs or responses (RNF-3) | pending | — |
| T-15 | Cover adapter (disk) | Laravel filesystem disk behind `CoverStorage` | T-08 | 2 MB PNG ok; 3 MB → 422; no cover → `cover_key` null | pending | — |
| T-16 | HTTP adapter: routes of `api-contract.md`, single error body, role authorization | `routes/api.php` under `/api/v1`; exception rendering in `bootstrap/app.php`; JWT/role middleware (I-03, I-04) | T-10, T-11, T-14, T-15 | Contract test covers every route and every code of `api-contract.md` §2 | pending | — |
| T-17 | Composition and startup (initial admin from environment) | `LibraryServiceProvider`; `library:bootstrap-admin` command (D-10) | T-16 | The system starts with an empty DB and the admin can log in | pending | — |

## Beyond the core (T-18 to T-23)

| # | Task | Laravel / React translation | Depends on | Evidence required | Status | Evidence link |
|---|---|---|---|---|---|---|
| T-18 | Move domain-only invariants to the DB: `ck_book_total_positive`, `ck_user_role` | New Laravel migration with `DB::statement` | T-12 | An `INSERT` via `psql` that violates them fails; `data-model.md` §4 is updated by the spec owner | pending | — |
| T-19 | Architecture tests (Articles I, II, III) | Pest `arch()` or PHPUnit scan of `use` statements (D-12) | T-17 | Test passes, and fails if a forbidden import is introduced | pending | — |
| T-20 | `infra`: `docker compose up` starts empty engine, API and frontend | Relative build contexts; migrations at API startup (I-08) | T-17 | One command; the DB starts without tables until migrations run | pending | — |
| T-21 | `tool`: demo seeder **through the API** (≥ 40 books, 10 members, 15 loans, 3 overdue) | Node or PHP script calling `/api/v1` | T-17 | After running it, `/loans/overdue` returns 3 lines and the report has rows | pending | — |
| T-22 | `app`: login, catalog, members, lend, return, overdue, report | React + Vite | T-16 | Manual end-to-end walkthrough | pending | — |
| T-23 | `page`: public static presentation site | Static HTML/CSS (content undefined, M-04) | — | Opens without the API; correct on mobile and desktop | pending | — |

## Delivery priority for 2026-10-13 17:00 (UTC-5)

1. `docs/spec-laravel/` (this folder) — translation and interpretations.
2. `infra` minimal compose (part of T-20) so the API has an empty PostgreSQL 16.
3. T-01, T-02, then T-03 to T-07, T-14, T-17 (login as admin on an empty DB).
4. T-08 to T-13 if time allows; then T-23 as the cheapest independent task.
5. Last step before 16:45: update the **Status** and **Evidence link** columns of this file to the real state.

Tasks not reached by the deadline stay **pending** in this file.
