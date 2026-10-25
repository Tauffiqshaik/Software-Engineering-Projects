# SpendWise

**Personal finance REST API: track transactions, set category budgets, get monthly reports.**

[![CI](https://github.com/YOUR_USERNAME/spendwise/actions/workflows/ci.yml/badge.svg)](https://github.com/YOUR_USERNAME/spendwise/actions/workflows/ci.yml)
![Python](https://img.shields.io/badge/python-3.12-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-009688)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791)
![Docker](https://img.shields.io/badge/docker-compose-2496ED)

## Table of contents

- [Features](#features)
- [Tech stack](#tech-stack)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Usage example](#usage-example)
- [Design decisions](#design-decisions)
- [API reference](#api-reference)
- [Project structure](#project-structure)
- [Testing and CI](#testing-and-ci)
- [Possible extensions](#possible-extensions)

## Features

| Area | What it does |
|---|---|
| **Auth** | Register and login, PBKDF2-SHA256 password hashing, JWT bearer tokens, per-user data isolation |
| **Categories** | CRUD, plus **keywords** for auto-categorisation (`"swiggy order"` → *Food*) |
| **Transactions** | CRUD, **filter** (type, category, date range, amount range, text search), **sort**, **paginate**, **CSV export**, **bulk import** |
| **Budgets** | Monthly limit per category; status is `ok`, `warning` (≥ 80%) or `exceeded` |
| **Reports** | Monthly summary (income, expense, net, per-category, budget status) and N-month trend |

## Tech stack

Python 3.12 · FastAPI · SQLAlchemy 2 · PostgreSQL 16 (SQLite for local dev) · Pydantic v2 · PyJWT · Docker Compose · pytest · GitHub Actions

## Quick start

### With Docker

```bash
git clone https://github.com/YOUR_USERNAME/spendwise.git
cd spendwise
docker compose up --build
```

API: http://localhost:8000 · Swagger docs: http://localhost:8000/docs · Health: http://localhost:8000/health

### Locally (SQLite, no Docker)

```bash
pip install -r requirements-dev.txt
uvicorn app.main:app --reload
```

Tables are created automatically on startup.

## Configuration

| Variable | Purpose | Default |
|---|---|---|
| `SECRET_KEY` | JWT signing key. **Set this for anything other than local development.** | `dev-only-change-me` |
| `DATABASE_URL` | SQLAlchemy URL | `sqlite:///./spendwise.db` |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | Token lifetime | `60` |
| `PBKDF2_ITERATIONS` | Password hashing cost (OWASP recommends ≥ 600k; tests lower it for speed) | `600000` |

With Docker Compose, pass the key like this: `SECRET_KEY=your-long-random-string docker compose up --build`.

## Usage example

```bash
# 1. Register and log in
curl -X POST localhost:8000/auth/register -H "Content-Type: application/json" \
  -d '{"email": "me@example.com", "password": "a-strong-password"}'

TOKEN=$(curl -s -X POST localhost:8000/auth/login -H "Content-Type: application/json" \
  -d '{"email": "me@example.com", "password": "a-strong-password"}' | python -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# 2. Add a transaction (Idempotency-Key makes retries safe)
curl -X POST localhost:8000/transactions \
  -H "Authorization: Bearer $TOKEN" -H "Idempotency-Key: demo-1" \
  -H "Content-Type: application/json" \
  -d '{"type": "expense", "amount": "250.50", "occurred_on": "2026-03-05", "description": "swiggy order"}'

# 3. Monthly report
curl -H "Authorization: Bearer $TOKEN" "localhost:8000/reports/monthly?month=2026-03"
```

> Passwords must be at least 8 characters. Open `/docs` to see every request and response shape and try each endpoint from the browser.

## Design decisions

- **Money is `NUMERIC(12,2)` / `Decimal`, never float.** Input with more than 2 decimals (`10.999`) is rejected rather than silently rounded.
- **Idempotent POST.** Send an `Idempotency-Key` header and a retried request returns the original response (with `Idempotent-Replay: true`) instead of creating a duplicate. The key and the transaction row are committed in the **same DB transaction**, a unique `(user_id, key)` constraint resolves concurrent races, and reusing a key with a *different* body returns `409`.
- **Atomic bulk import.** `POST /transactions/bulk` is all-or-nothing: one bad row and nothing is saved.
- **Safe category deletion.** Deleting a category un-categorises its transactions and removes its budgets in one transaction, with explicit statements so behaviour is identical on SQLite and PostgreSQL.
- **Stable pagination.** Sorting adds `id` as a tiebreaker so pages never overlap when many rows share a value.
- **Portable month queries.** Reports use half-open date ranges `[start, next_month)` instead of `strftime` / `date_trunc`, so they run on both databases and can use the `(user_id, occurred_on)` index.
- **Security basics.**
  - Ownership checks return `404` (not `403`) so IDs can't be probed.
  - Login errors don't reveal whether an email exists.
  - `LIKE` wildcards in search are escaped and sort fields are whitelisted.
  - CSV export neutralises spreadsheet formula injection (`=`, `+`, `-`, `@`).

## API reference

Interactive docs are at `/docs`. All endpoints except register, login and health require `Authorization: Bearer <token>`.

| Area | Endpoints |
|---|---|
| Auth | `POST /auth/register` · `POST /auth/login` · `GET /auth/me` |
| Categories | `POST /categories` · `GET /categories` · `PATCH /categories/{id}` · `DELETE /categories/{id}` |
| Transactions | `POST /transactions` (supports `Idempotency-Key`) · `POST /transactions/bulk` · `GET /transactions` · `GET /transactions/export` (CSV) · `GET` / `PATCH` / `DELETE /transactions/{id}` |
| Budgets | `POST /budgets` · `GET /budgets?month=` · `GET /budgets/status?month=2026-03` · `PATCH` / `DELETE /budgets/{id}` |
| Reports | `GET /reports/monthly?month=2026-03` · `GET /reports/trend?end_month=2026-03&months=6` |
| Meta | `GET /health` |

**Transaction list query parameters** (also accepted by `/transactions/export`):

```
type, category_id, date_from, date_to, min_amount, max_amount, q
sort_by = occurred_on | amount | created_at | id
order   = asc | desc
page, page_size
```

## Project structure

```
spendwise/
├── app/
│   ├── main.py            # App setup, router registration, /health
│   ├── config.py          # Environment-based settings
│   ├── database.py        # Engine and session
│   ├── models.py          # SQLAlchemy models
│   ├── schemas.py         # Pydantic request/response models
│   ├── security.py        # Password hashing and JWT
│   ├── services.py        # Shared business logic
│   └── routers/           # auth, categories, transactions, budgets, reports
├── tests/                 # pytest suite
├── .github/workflows/ci.yml
├── Dockerfile
└── docker-compose.yml
```

## Testing and CI

```bash
pip install -r requirements-dev.txt
pytest
```

The suite has **68 tests** covering:

- **Duplicate requests:** same idempotency key, key reuse with a different body, per-user key scoping, failed requests not burning a key, duplicate emails (case-insensitive), duplicate categories and budgets.
- **Incorrect data:** zero, negative and over-precise amounts, bad enums, impossible and future dates, oversized fields, unknown or foreign category IDs, bad month formats, invalid sort fields, inverted ranges, out-of-range pagination.
- **Transactionality:** bulk import rollback, category deletion cascade.
- **Business logic:** budget thresholds (exactly 80% and exactly 100%), month and year boundaries, empty months in trends, user isolation.

GitHub Actions (`.github/workflows/ci.yml`) runs the tests on every push to `main` and every pull request.

## Possible extensions

Alembic migrations, recurring transactions, multi-currency support, rate limiting, refresh tokens, and a Postgres service container in CI.
