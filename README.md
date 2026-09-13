# AI Football Analyst

Automated football analytics platform: daily fixture ingestion, statistical
prediction models, value-vs-market-odds comparison, and generated ticket
suggestions (Safe / Balanced / High Odds).

> Analytical tool, not a guarantee of outcomes. No feature in this codebase
> should ever present a prediction or ticket as a certain win — see
> [`docs/DATA_QUALITY.md`](docs/DATA_QUALITY.md) and the `PredictionResult`
> model for how outcomes are tracked and evaluated honestly over time.

## Phase status

- [x] **Phase 1 — Architecture**: folder structure, database schema,
      provider interface, API contract skeleton, environment config,
      Docker Compose for local dev.
- [x] **Phase 1 review & corrections** (this delivery): audited the above
      against a formal review checklist and fixed the issues found — see
      the engineering report delivered alongside this repo for the full
      audit trail. Highlights: the daily-ticket endpoint now actually
      filters by date (it previously didn't have a date field at all),
      predictions/tickets are never overwritten in place, several missing
      duplicate-prevention constraints were added, and provider/config
      errors now fail clearly instead of surfacing as generic 500s.
- [x] **Phase 2 — Backend foundation**: connection pooling + commit/rollback
      session pattern; a repository layer (`app/repositories/`); a thin
      service layer (`app/services/`) that owns the one real cross-cutting
      decision that existed (timezone-correct "today", via `app/core/clock.py`)
      instead of scattering `date.today()` calls; a reusable idempotent-job
      wrapper (`app/services/job_runner.py`) wired into the scheduler
      placeholder; a catch-all exception handler; migrations auto-apply via
      `entrypoint.sh` in dev. The spec for this phase was cut off after
      section 3 (Repository layer) — sections 4-11 had no detailed content,
      so anything built for them (service layer, job runner) follows this
      repo's own conventions rather than instructions that were never sent.
      Auth/CRUD were not mentioned in what was received, so neither was added.
- [ ] Phase 3 — Data ingestion from Sportmonks, sync jobs, caching
- [ ] Phase 4 — Prediction / probability / value / risk engines
- [ ] Phase 5 — Ticket generator + daily orchestrator + scheduling
- [ ] Phase 6 — Frontend (Next.js)
- [ ] Phase 7 — AI chat + tool calling
- [ ] Phase 8 — Backtesting, monitoring, production hardening

## Stack

- **Frontend**: Next.js, TypeScript, Tailwind (scaffolded in `apps/web`,
  built out in Phase 6)
- **Backend**: Python, FastAPI, SQLAlchemy, Alembic, PostgreSQL, Redis,
  APScheduler
- **Data provider**: Sportmonks (behind a provider interface — swappable)
- **Infra**: Docker Compose locally; Render (API + worker + cron + Postgres +
  Redis) and Vercel (frontend) recommended for hosting

## Running locally

Requires Docker + Docker Compose installed.

1. Copy the environment template and fill in real values:
   ```bash
   cp .env.example .env
   ```
   At minimum for Phase 1 you need `POSTGRES_*` and `SECRET_KEY` — the app
   now fails immediately with a clear error if `SECRET_KEY` is missing,
   instead of a confusing failure later. `SPORTMONKS_API_KEY` isn't
   required until Phase 3, but if you do set `FOOTBALL_DATA_PROVIDER=sportmonks`
   without a key, the app now raises a clear `ConfigurationError` rather
   than failing mysteriously on the first request.

2. Build and start everything:
   ```bash
   docker compose up --build
   ```
   This starts `db` (Postgres 16), `redis` (Redis 7), `api` (FastAPI on
   http://localhost:8000, docs at `/docs`), and `scheduler` (the
   APScheduler skeleton — see "What's still a placeholder" below).

3. Generate and run the initial migration. This repo ships the Alembic
   *setup* but deliberately not a pre-baked migration file — autogenerate
   needs to run against a live DB to diff against, and generating one
   blind in a sandbox without Postgres/SQLAlchemy installed would risk
   baking in a wrong migration silently:
   ```bash
   docker compose exec api alembic revision --autogenerate -m "initial schema"
   docker compose exec api alembic upgrade head
   ```

4. Check it's alive:
   ```bash
   curl http://localhost:8000/health
   ```

5. Run the test suite:
   ```bash
   docker compose exec api pytest -v
   ```

## Project layout

```
ai-football-analyst/
├── apps/
│   ├── api/                  # FastAPI backend
│   │   ├── app/
│   │   │   ├── core/         # settings, logging, exception types
│   │   │   ├── db/           # SQLAlchemy engine + session (pooled, commit/rollback per request)
│   │   │   ├── models/       # ORM models = the database schema
│   │   │   ├── repositories/ # data access — routes never query the DB directly
│   │   │   ├── providers/    # FootballDataProvider interface + adapters
│   │   │   ├── schemas/      # Pydantic request/response contracts
│   │   │   ├── api/routes/   # FastAPI routers
│   │   │   ├── scheduler.py  # APScheduler skeleton (Phase 5 fills in the job)
│   │   │   └── main.py       # app entrypoint
│   │   ├── alembic/          # DB migrations
│   │   └── tests/
│   └── web/                  # Next.js frontend (Phase 6)
├── docs/
│   └── DATA_QUALITY.md       # binding data-integrity principles (STEP 13)
├── docker-compose.yml
└── .env.example
```

Note on future phases: Phase 4's prediction/value/risk engines and Phase 7's
AI explanation layer are expected to land as new top-level packages under
`app/` (e.g. `app/engine/`, `app/ai/`) alongside the existing ones — nothing
in the current layout needs restructuring to accommodate them.

## What's still a placeholder (intentionally)

- **Sportmonks adapter**: HTTP client, auth, and error handling are real;
  every fetch method raises `NotImplementedError` on purpose (Phase 3).
- **Scheduler**: fires a real cron job on a real timezone/hour from config,
  but the job body only logs — no fixture sync, no predictions (Phase 5).
  Its docstring documents the idempotency contract Phase 5 must follow.
- **Prediction/value/risk math**: none exists yet (Phase 4).
- **Auth enforcement**: routes are open, no login required yet (Phase 2).
- **Frontend**: `apps/web` contains only a placeholder README (Phase 6).
- **AI chat**: not started (Phase 7).

## Next step

**Phase 2 is on hold until you explicitly ask for it**, per your review
instructions. In the meantime, worth double-checking on your end: the two
things this review could not verify by actually running them (rather than
reading them) are the Alembic migration and the full pytest suite — both
need dependencies this sandbox can't install. See the engineering report's
TEST RESULTS section for exactly what was and wasn't executed.

## Phase 3 — Real football data ingestion

Phase 3 implements the documented Sportmonks Football API v3 integration and provider-neutral ingestion services. It supports fixture-by-date sync, team/league/season enrichment, match statistics, lineups, standings, and pre-match odds. No fake football data is generated. Fields unavailable from a provider/subscription remain null or are reported as unsupported.

Configure `FOOTBALL_DATA_PROVIDER=sportmonks` and `SPORTMONKS_API_KEY` only with a valid Sportmonks subscription/token. Use `SPORTMONKS_LEAGUE_IDS` to restrict fixture ingestion when desired. The sync endpoint is `POST /api/v1/sync/fixtures?target_date=YYYY-MM-DD`.

## Phase 5 — Value/Risk Engine + Daily Ticket Generator

Phase 5 adds a deterministic value/risk layer and daily ticket orchestration on top of Phase 4 predictions.

### Implemented
- model probability vs decimal market odds
- implied probability = `1 / odds`
- edge = model probability minus implied probability
- EV = `probability * odds - 1`
- bounded value ranking score
- conservative risk/confidence filtering
- three daily ticket classes: safe, balanced, high_odds
- avoids duplicate matches inside a ticket
- avoids repeated selections from the same market family inside a ticket
- preserves ticket history by superseding old active tickets instead of overwriting them
- API: `POST /api/v1/ticket-engine/date/{YYYY-MM-DD}`
- automated scheduler job at 01:00 in the configured application timezone
- no-odds selections are never treated as value selections

Value is a long-run pricing concept, not a guarantee for a single match. A positive EV selection can still lose. citeturn0search0turn0search1


## Phase 6
DailyAnalysisOrchestrator now runs predictions -> value/risk -> opportunity ranking -> daily tickets. See `docs/PHASE6_ENGINEERING_REPORT.md`.

## Phase 8 — Final web integration
The project now includes a mobile-first Next.js dashboard under `apps/web`, connected to the FastAPI API. It presents daily matches, ranked opportunities, AI summary, generated tickets, and AI Analyst chat. Docker Compose now includes the web service on port 3000.
