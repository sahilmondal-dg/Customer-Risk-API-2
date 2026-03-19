# Claude.md — v1.0 · FROZEN · 2026-03-05
**Project:** Customer Risk API · **Classification:** Training Demo System

---

## 1. System Intent

A FastAPI service wraps a Postgres database of pre-assessed customer risk profiles, enforcing API key authentication on every request and returning structured JSON containing a customer's risk tier and contributing factors. A single-page HTML/JS UI, served by the same FastAPI container, allows operations staff to query by customer ID without writing code. The system is read-only: it surfaces pre-assessed values and computes nothing.

**Success:** `docker compose up` from a clean clone (with `.env` present) starts all services without manual intervention, the UI is reachable at `http://localhost:8000/`, and all nine invariants hold end to end.

---

## 2. Hard Invariants

**INVARIANT: The system never writes to the database. No API endpoint, application code path, or error handler issues INSERT, UPDATE, DELETE, or DDL statements against the database. The application's database connection is used exclusively for SELECT queries.** This is never negotiable.

**INVARIANT: The `risk_tier` and `risk_factors` fields in every API response are taken directly from the database row corresponding to the requested `customer_id`. No default values are substituted, no tier is inferred from factors, and no factors are added, removed, or reordered by application logic.** This is never negotiable.

**INVARIANT: The database query that retrieves customer data is parameterised such that exactly the row matching the supplied `customer_id` is returned. No other row is accessible via the query, regardless of what value is supplied as `customer_id`.** This is never negotiable.

**INVARIANT: Every request to any API endpoint is rejected with HTTP 401 if a valid API key is not present. No API route is accessible without authentication. The UI HTML page and `/health` are the only routes exempt from this check.** This is never negotiable.

**INVARIANT: The API key never appears in any API response body, log output, or error message. This applies to all code paths including startup logs, request logging, and exception handlers.** This is never negotiable.

**INVARIANT: Every SQL query that incorporates a value derived from user input uses psycopg2's parameterised query syntax (`%s` placeholders passed as a separate argument tuple). String concatenation and f-string interpolation into SQL are never used.** This is never negotiable.

**INVARIANT: All error responses return only a structured JSON payload containing a human-readable message and the appropriate HTTP status code. Database exceptions, query text, table names, column names, and Python tracebacks are logged internally only and never included in the response body.** This is never negotiable.

**INVARIANT: After the init script has executed, the database contains a minimum of one customer record with `risk_tier = 'LOW'`, one with `risk_tier = 'MEDIUM'`, and one with `risk_tier = 'HIGH'`. This condition is established at first startup and is not modified by any application operation.** This is never negotiable.

**INVARIANT: The FastAPI application does not begin serving requests until a successful connection to Postgres can be established and the database contains the expected seed records. The container startup sequence enforces this via Docker Compose health check dependency — the API container does not start until Postgres reports healthy.** This is never negotiable.

---

## 3. Scope Boundary

**Claude Code is permitted to create and modify exactly these files:**

```
customer-risk-api/
├── .env.example
├── .gitignore
├── README.md
├── docker-compose.yml
├── db/
│   └── init.sql
└── api/
    ├── Dockerfile
    ├── requirements.txt
    ├── main.py
    ├── db.py
    └── static/
        └── index.html
```

**Claude Code must not:**
- Create files outside this manifest without flagging it first
- Add routes, endpoints, or application features not specified in the execution plan
- Add any ORM, SQLAlchemy, or database abstraction layer — psycopg2 direct only
- Add write operations (INSERT, UPDATE, DELETE, DDL) anywhere in application code
- Add external service calls, outbound HTTP, or third-party API integrations
- Add user management, per-user keys, or role-based access control
- Add TLS configuration, rate limiting, or production secrets management
- Modify `docker-compose.yml` to expose additional ports or services beyond `postgres` and `api`

**If a task prompt conflicts with an invariant: the invariant wins. Flag the conflict — do not resolve it silently.**

---

## 4. Execution Contract

- One task at a time. Do not begin the next task until the current task is verified and committed.
- No scope expansion. If completing a task would require building something not specified: stop and flag it.
- No silent deviations. If the implementation requires a decision not covered by this document or the execution plan, flag it — do not resolve it with your own judgment.
- No architectural decisions. Stack choices, library selections, and design patterns are fixed. If a fixed choice creates a problem, flag it.
- No editing Claude.md. If a genuine contract change is needed, flag it and return to planning. This document is not edited inline.

---

## 5. Fixed Stack

| Concern | Fixed Choice |
|---|---|
| Orchestration | Docker Compose (docker-compose.yml) |
| Database | Postgres 15 (image: postgres:15) |
| API runtime | Python 3.11 (image: python:3.11-slim) |
| API framework | FastAPI 0.111.0 |
| ASGI server | uvicorn[standard] 0.29.0 |
| DB driver | psycopg2-binary 2.9.9 · no ORM · no SQLAlchemy |
| UI | Plain HTML5 + vanilla JavaScript · no frameworks · no CDN imports |
| API port | 8000 (host and container) |
| Auth mechanism | `X-API-Key` request header · single shared key · `hmac.compare_digest` comparison |
| customer_id format | `VARCHAR(20)` · pattern `^[A-Za-z0-9_-]{1,20}$` |
| risk_factors schema | Postgres `TEXT[]` · returned as JSON array of strings |
| Seed volume | 15 records: CUST-001–CUST-005 (LOW), CUST-006–CUST-010 (MEDIUM), CUST-011–CUST-015 (HIGH) |
| Health endpoint | `GET /health` → `{"status": "ok"}` · exempt from auth |
| UI route | `GET /` → serves `/app/static/index.html` · exempt from auth |
| Key injection | Startup event replaces literal `REPLACE_ME` in `index.html` with `API_KEY` env value |
| Init mechanism | `db/init.sql` mounted at `/docker-entrypoint-initdb.d/init.sql:ro` |
| Startup dependency | `depends_on: postgres: condition: service_healthy` · `pg_isready` health check |

**Environment variables (all required, all sourced from `.env`):**

| Variable | Used By | Purpose |
|---|---|---|
| `API_KEY` | FastAPI, UI (injected at startup) | Shared authentication secret |
| `POSTGRES_USER` | Postgres, FastAPI | Database username |
| `POSTGRES_PASSWORD` | Postgres, FastAPI | Database password |
| `POSTGRES_DB` | Postgres, FastAPI | Database name |
| `POSTGRES_HOST` | FastAPI | Postgres service hostname (Docker service name: `postgres`) |
| `POSTGRES_PORT` | FastAPI | Postgres port (default: `5432`) |

If a variable is missing at startup, the application must fail fast with a clear error — not start in a degraded state.