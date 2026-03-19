# VERIFICATION_RECORD.md

**Session:** Session 1 — Infrastructure: Docker Compose, Postgres, Schema, Seed Data
**Date:** 2026-03-19
**Engineer:**

---

## Task S1.T1 — Project scaffold and `.env.example`

### Test Cases Applied
Source: EXECUTION_PLAN.md Session 1

| Case | Scenario | Expected | Result |
|------|----------|----------|--------|
| TC1 | All required files and directories exist | `ls` shows `docker-compose.yml`, `.env.example`, `.gitignore`, `api/`, `db/` | PASS |
| TC2 | `.env.example` contains all six variable names | `grep -c "=" .env.example` returns 6 | PASS — returned 6 |
| TC3 | `.env` is listed in `.gitignore` | `grep "^\.env$" .gitignore` returns a match | PASS — `.env` matched |
| TC4 | `docker-compose.yml` defines both `postgres` and `api` services | `grep -c "^\s\{2\}[a-z]" docker-compose.yml` returns at least 2 | PASS — returned 2 |

### Prediction Statement
All four test cases predicted to pass. Files were created directly and their content verified manually before committing.

### CD Challenge Output
What was not tested:
- That `.gitignore` excludes `__pycache__/`, `*.pyc`, and `.DS_Store` entries (not just `.env`) — **rejected**: TC3 covers the safety-critical entry (`.env`); the others are development hygiene and cannot affect system behaviour.
- That `docker-compose.yml` correctly names the services `postgres` and `api` (not just that two top-level keys exist) — **accepted**: added as TC5 below.
- That `.env.example` contains the exact variable names required, not just any six `=` lines — **accepted**: added as TC6 below.

### Code Review
No invariant touch for this task. Not required.

### Scope Decisions
`.gitkeep` files added to `api/` and `db/` to satisfy git tracking requirement. These are not mentioned in the task prompt but are a necessary implementation detail for empty directory scaffolding. Accepted as in scope — they carry no functional impact and will be overwritten by real files in subsequent tasks.

### Test Cases Added During Session

| Case | Scenario | Expected | Result |
|------|----------|----------|--------|
| TC5 | `docker-compose.yml` names services `postgres` and `api` | `grep "^\s*postgres:\|^\s*api:" docker-compose.yml` returns 2 matches | PASS |
| TC6 | `.env.example` contains all six exact required variable names | `grep -E "^(API_KEY\|POSTGRES_USER\|POSTGRES_PASSWORD\|POSTGRES_DB\|POSTGRES_HOST\|POSTGRES_PORT)=" .env.example` returns 6 | PASS |

### Verification Verdict
[x] All planned cases passed
[x] CD challenge reviewed
[x] Code review complete (if invariant-touching) — N/A, no invariant touch
[x] Scope decisions documented

**Status: COMPLETE — commit 7a90e0b**

---

## Task S1.T2 — Docker Compose: Postgres service with health check

### Test Cases Applied
Source: EXECUTION_PLAN.md Session 1

| Case | Scenario | Expected | Result |
|------|----------|----------|--------|
| TC1 | `docker compose up -d postgres` starts without error | Exit code 0 | |
| TC2 | Postgres container reaches `healthy` within 60s | `docker compose ps` shows `(healthy)` | |
| TC3 | Can connect and run a query | `docker exec <container> psql -U riskuser -d riskdb -c "SELECT 1;"` returns `1` | |
| TC4 | Named volume `pgdata` is created | `docker volume ls` shows `customer-risk-api_pgdata` | |

> **INVARIANT TOUCH: INV-09** — API must not accept requests until DB is ready; health check is the enforcement mechanism here.

### Prediction Statement
[LEAVE BLANK — engineer writes predictions before running verification commands]

### CD Challenge Output
[Paste CD's response to: 'What did you not test in this task?'
For each item: accepted (added case) / rejected (reason).]

### Code Review
**INVARIANT TOUCH: INV-09**
- Confirm `interval`, `retries`, and `start_period` values are sufficient for Postgres to fully initialise before reporting healthy.
- Confirm `pg_isready` checks the correct user and database (not just the server process).
- [ ] Invariant condition is actually enforced in the code
- [ ] No code path bypasses the enforcement
- [ ] Enforcement is in the right place (not just present somewhere)
- [ ] Future additions cannot bypass it without explicitly removing the check

### Scope Decisions
[What was accepted as out of scope and why. Cannot be left blank for deliverables.]

### Test Cases Added During Session

| Case | Scenario | Expected | Result |
|------|----------|----------|--------|
|      |          |          |        |

### Verification Verdict
[ ] All planned cases passed
[ ] CD challenge reviewed
[ ] Code review complete (if invariant-touching)
[ ] Scope decisions documented

**Status:**

---

## Task S1.T3 — Database schema and seed data init script

### Test Cases Applied
Source: EXECUTION_PLAN.md Session 1

| Case | Scenario | Expected | Result |
|------|----------|----------|--------|
| TC1 | Script runs without SQL errors | `psql` exit code 0 | |
| TC2 | Table exists with correct columns | `\d customers` shows customer_id, risk_tier, risk_factors | |
| TC3 | Exactly 15 rows inserted | `SELECT COUNT(*) FROM customers;` returns 15 | |
| TC4 | All three tiers present with count 5 each | `SELECT risk_tier, COUNT(*) FROM customers GROUP BY risk_tier;` returns LOW=5, MEDIUM=5, HIGH=5 | |
| TC5 | risk_factors is a non-empty array for every row | `SELECT COUNT(*) FROM customers WHERE risk_factors = '{}';` returns 0 | |
| TC6 | CHECK constraint rejects invalid tier | `INSERT INTO customers VALUES ('X', 'INVALID', '{}');` raises constraint violation | |

> **INVARIANT TOUCH: INV-08** — All three tiers must exist after initialisation.
> **INVARIANT TOUCH: INV-01** — Schema defines read-only data; init.sql is the only write path.

### Prediction Statement
[LEAVE BLANK — engineer writes predictions before running verification commands]

### CD Challenge Output
[Paste CD's response to: 'What did you not test in this task?'
For each item: accepted (added case) / rejected (reason).]

### Code Review
**INVARIANT TOUCH: INV-08 and INV-01**
- Confirm `CHECK (risk_tier IN ('LOW', 'MEDIUM', 'HIGH'))` is present.
- Confirm `NOT NULL` on both `risk_tier` and `risk_factors`.
- Confirm all 15 rows cover all three tiers.
- Confirm `risk_factors` uses `TEXT[]` not `JSONB` (TEXT[] is what the plan specifies).
- [ ] Invariant condition is actually enforced in the code
- [ ] No code path bypasses the enforcement
- [ ] Enforcement is in the right place (not just present somewhere)
- [ ] Future additions cannot bypass it without explicitly removing the check

### Scope Decisions
[What was accepted as out of scope and why. Cannot be left blank for deliverables.]

### Test Cases Added During Session

| Case | Scenario | Expected | Result |
|------|----------|----------|--------|
|      |          |          |        |

### Verification Verdict
[ ] All planned cases passed
[ ] CD challenge reviewed
[ ] Code review complete (if invariant-touching)
[ ] Scope decisions documented

**Status:**

---

## Task S1.T4 — Mount init script into Docker Compose

### Test Cases Applied
Source: EXECUTION_PLAN.md Session 1

| Case | Scenario | Expected | Result |
|------|----------|----------|--------|
| TC1 | Fresh stack start seeds the database | After `docker compose down -v && docker compose up -d postgres`, all 15 rows are present | |
| TC2 | Init script is mounted read-only | Running `docker compose exec postgres touch /docker-entrypoint-initdb.d/init.sql` fails with permission error | |
| TC3 | Second start with existing volume skips init script (expected behaviour) | After `docker compose restart postgres` (without `-v`), data is still present and count is still 15 | |

> **INVARIANT TOUCH: INV-08** — Seed data must be present after initialisation.

### Prediction Statement
[LEAVE BLANK — engineer writes predictions before running verification commands]

### CD Challenge Output
[Paste CD's response to: 'What did you not test in this task?'
For each item: accepted (added case) / rejected (reason).]

### Code Review
**INVARIANT TOUCH: INV-08**
- Confirm `:ro` flag is present on the bind mount.
- Confirm `pgdata` named volume mount is still present and unchanged.
- [ ] Invariant condition is actually enforced in the code
- [ ] No code path bypasses the enforcement
- [ ] Enforcement is in the right place (not just present somewhere)
- [ ] Future additions cannot bypass it without explicitly removing the check

### Scope Decisions
[What was accepted as out of scope and why. Cannot be left blank for deliverables.]

### Test Cases Added During Session

| Case | Scenario | Expected | Result |
|------|----------|----------|--------|
|      |          |          |        |

### Verification Verdict
[ ] All planned cases passed
[ ] CD challenge reviewed
[ ] Code review complete (if invariant-touching)
[ ] Scope decisions documented

**Status:**
