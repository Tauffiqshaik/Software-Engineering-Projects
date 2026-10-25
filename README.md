# CodeReview AI

**Automated code review for snippets, diffs and GitHub pull requests.**

[![CI](https://github.com/YOUR_USERNAME/codereview-ai/actions/workflows/ci.yml/badge.svg)](https://github.com/YOUR_USERNAME/codereview-ai/actions/workflows/ci.yml)
![Python](https://img.shields.io/badge/python-3.12-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-009688)
![React](https://img.shields.io/badge/React-Vite-61DAFB)
![Docker](https://img.shields.io/badge/docker-compose-2496ED)

Submit a code snippet, a pasted diff, or a GitHub pull request and get review comments with **line references**, **severity** and a **suggested fix**. Static analysis catches bugs, security problems and maintainability issues; an optional **LLM pass** adds context-aware findings. Reviewers mark findings as *confirmed* or *false positive*, and the system learns from it: suppressed findings stay suppressed, and per-rule accuracy is tracked.

## Table of contents

- [Features](#features)
- [Tech stack](#tech-stack)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [How a review works](#how-a-review-works)
- [False-positive tracking](#false-positive-tracking)
- [Security decisions](#security-decisions)
- [API reference](#api-reference)
- [Project structure](#project-structure)
- [Testing and CI](#testing-and-ci)
- [Troubleshooting](#troubleshooting)
- [Known limitations](#known-limitations)

## Features

- **Three input modes:** paste a snippet, paste a unified diff, or point at a GitHub PR (`owner/name` + PR number).
- **25+ static rules:** Python AST checks plus line-based checks for secrets and risky JavaScript. List them with `GET /rules`.
- **Optional LLM review** via the Anthropic API. If it fails, the review still succeeds with static results.
- **Changed-lines only in PR mode:** the whole file is analysed (so the AST parses), but only findings on lines the author touched are reported.
- **Post back to GitHub:** publish findings as a single PR review with inline comments.
- **Webhook support:** automatically review PRs on open, sync and reopen.
- **Triage workflow:** mark findings `open`, `confirmed` or `false_positive`; false positives are suppressed in future reviews.
- **Rule accuracy stats:** per-rule false-positive rate, worst rules first.
- **React UI** for submitting reviews and triaging findings.

## Tech stack

| Layer | Technology |
|---|---|
| Backend | Python 3.12, FastAPI, SQLAlchemy 2, Pydantic v2, httpx |
| Database | PostgreSQL 16 (SQLite for local development) |
| Frontend | React 18, Vite |
| Integrations | GitHub REST API, Anthropic API |
| Tooling | Docker Compose, GitHub Actions, pytest |

## Quick start

### With Docker (recommended)

```bash
git clone https://github.com/YOUR_USERNAME/codereview-ai.git
cd codereview-ai
cp .env.example .env          # optional: add GITHUB_TOKEN / ANTHROPIC_API_KEY
docker compose up --build
```

| Service | URL |
|---|---|
| UI | http://localhost:3000 |
| API docs (Swagger) | http://localhost:8000/docs |
| Health check | http://localhost:8000/health |

### Without Docker

```bash
# Backend (uses SQLite unless DATABASE_URL is set)
cd backend
pip install -r requirements-dev.txt
uvicorn app.main:app --reload --port 8000

# Frontend (in a second terminal)
cd frontend
npm install
npm run dev                   # http://localhost:5173, proxies /api to :8000
```

> Static analysis works with **no keys at all**. `ANTHROPIC_API_KEY` enables the AI pass and `GITHUB_TOKEN` enables PR reviews.

## Configuration

Copy `.env.example` to `.env`. Every variable is optional.

| Variable | Purpose | Default |
|---|---|---|
| `GITHUB_TOKEN` | Fine-grained PAT with *Pull requests: read/write* and *Contents: read*. Needed for PR reviews. | – |
| `GITHUB_WEBHOOK_SECRET` | Random string; use the same value in the GitHub webhook settings. The webhook endpoint refuses to run without it. | – |
| `ANTHROPIC_API_KEY` | Enables the LLM review pass. | – |
| `LLM_MODEL` | Model used for the LLM pass. | `claude-sonnet-5-5` |
| `API_KEY` | If set, clients must send an `X-API-Key` header. | – (open) |
| `DATABASE_URL` | SQLAlchemy URL. | `sqlite:///./codereview.db` |
| `CORS_ORIGINS` | Allowed browser origins. | `http://localhost:5173` |

Built-in limits: 200 KB per snippet or file, 50 files per review.

## How a review works

```
snippet / diff / PR ──► fetch files ──► static analyzers ──► (LLM pass) ──► validate ──► fingerprint ──► store
                         (GitHub API)    AST + patterns       optional       line must      suppress known
                                                                             be changed     false positives
```

1. **PR mode** reads the PR's files, parses each patch to find the changed lines, fetches the file at the head commit, analyses the whole file and keeps only findings on changed lines.
2. **Static rules** cover, among others: `eval`/`exec`, `shell=True`, unsafe `pickle` / `yaml.load`, weak hashes, SQL built with f-strings or concatenation, `verify=False`, bare or swallowed `except`, mutable default arguments, long functions, cyclomatic complexity, too many parameters, `== None`, debug mode, `mktemp`, hard-coded secrets, AWS/GitHub tokens, private keys, risky JavaScript (`eval`, `innerHTML`, command injection, `==`, `console.log`) and TODOs.
3. **LLM pass** sends numbered lines and asks for JSON. The reply is treated as untrusted: it must parse, lines must exist and be in the changed set, severity and category must come from the allowed enums, and duplicates of static findings are dropped.
4. **Optionally posts** findings back to the PR as one GitHub review with inline comments (`post_comments: true`).
5. A **webhook** (`POST /webhooks/github`) reviews PRs automatically.

## False-positive tracking

Each finding has a **fingerprint**:

```
sha256(rule | file | normalised source line)
```

It is based on the line's *text*, not its number. When a finding is marked `false_positive`, the same fingerprint is auto-suppressed in future reviews, even if unrelated edits moved it to a different line. Suppressed findings don't count toward severity totals, and re-opening one lifts the suppression.

`GET /stats/rules` reports per-rule counts and the **false-positive rate = FP ÷ (FP + confirmed)**, worst rules first, so you know which checks to tune.

## Security decisions

- **Webhook:** requires a valid `X-Hub-Signature-256` HMAC, compared in constant time.
- **Repo names** are validated against GitHub's character set (blocks `../`, query strings and other path tricks); file paths are URL-encoded.
- **Secrets are never echoed:** a hard-coded credential finding doesn't include the value. GitHub error bodies are not passed to clients.
- **Prompt injection:** reviewed code is wrapped in `<code>` tags with an instruction to treat it as data, and the output is validated, so a malicious comment can't produce arbitrary findings or fake line numbers.
- **Limits** on snippet size, file size, files per review and comments per review; minified lines are skipped to avoid regex blow-ups.
- **Optional API key** (`X-API-Key`, constant-time compare), least-privilege token guidance in `.env.example`, and containers that run as non-root.

## API reference

Interactive docs are available at `/docs` when the backend is running.

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/reviews/snippet` | Review a snippet: `{filename, code, use_llm}` |
| `POST` | `/reviews/diff` | Review a unified diff: `{diff}` |
| `POST` | `/reviews/pull-request` | Review a PR: `{repo: "owner/name", pr_number, use_llm, post_comments}` |
| `GET` | `/reviews` | List reviews |
| `GET` / `DELETE` | `/reviews/{id}` | Fetch or delete a review |
| `PATCH` | `/findings/{id}` | Set status: `open`, `confirmed` or `false_positive` |
| `GET` | `/stats/rules` | Per-rule counts and false-positive rate |
| `GET` | `/rules` | List all static rules |
| `POST` | `/webhooks/github` | GitHub webhook receiver (signature required) |
| `GET` | `/health` | Health check |

**Example**

```bash
curl -X POST http://localhost:8000/reviews/snippet \
  -H "Content-Type: application/json" \
  -d '{"filename": "app.py", "code": "import os\nos.system(input())", "use_llm": false}'
```

## Project structure

```
codereview-ai/
├── backend/
│   ├── app/
│   │   ├── main.py             # FastAPI routes and webhook
│   │   ├── review_service.py   # Orchestrates a review
│   │   ├── analyzers/          # engine, python_ast, patterns, base
│   │   ├── llm.py              # LLM pass + output validation
│   │   ├── github_client.py    # GitHub REST client
│   │   ├── diffparse.py        # Unified diff parsing
│   │   ├── models.py, schemas.py, database.py, config.py
│   └── tests/
├── frontend/                   # React + Vite UI (served by nginx in Docker)
├── .github/workflows/ci.yml    # Backend tests + frontend build
├── docker-compose.yml
└── .env.example
```

## Testing and CI

```bash
cd backend
pytest
```

The suite has **136 tests**. GitHub and the LLM are mocked, so no network access or keys are needed.

- **Every rule** has positive and negative cases (parameterised SQL and `yaml.safe_load` are *not* flagged), plus placeholder-secret handling and changed-line filtering.
- **Diff parsing** edge cases: deleted and new files, added lines that look like headers, "No newline" markers.
- **GitHub client** over a mock transport: auth header, pagination, error mapping without leaking bodies, binary and missing files, URL encoding.
- **LLM output validation:** malformed JSON, hallucinated lines, bad enums, caps, merging, failure tolerance.
- **False-positive workflow:** suppression survives line shifts, is scoped per file and rule, can be lifted, and drives the stats.
- **Webhook security:** missing, forged and wrong-body signatures rejected; unconfigured secret; malformed payloads; background failures contained.

GitHub Actions (`.github/workflows/ci.yml`) runs the backend tests on Python 3.12 and builds the frontend on Node 20 for every push to `main` and every pull request.

## Troubleshooting

- **"Request failed (500)" or "API is unavailable" in the UI:** the frontend can't reach the backend. With `npm run dev`, the backend must be running separately on port 8000. The Vite dev server answers `/api/*` with an empty 500 when nothing is listening. Check http://localhost:8000/health.
- **With Docker:** run `docker compose logs backend`. The backend waits for Postgres to be healthy before starting.
- **PR review fails:** confirm `GITHUB_TOKEN` is set and has *Pull requests* and *Contents* access to the repository.

## Known limitations

- Static rules are heuristics (no data-flow or taint analysis), so some findings are false positives. That is why triage and suppression exist.
- Only Python gets AST analysis; other languages use line-based rules.
- The webhook reviews in a background task rather than a durable queue. For production, use a worker queue (Celery or RQ) and Alembic migrations.
