# Lead Follow-Up & Ownership Management System — Step 1

## Confirmed tech stack

| Layer | Choice | Notes |
|---|---|---|
| Backend API | **Python 3.12 + FastAPI** | Async SQLAlchemy 2.0 + asyncpg |
| Database | **PostgreSQL 15+** | Raw SQL migrations (validated on PG 16) |
| Queue / scheduler | **Redis + Celery (Celery Beat)** | Chosen over BullMQ to stay in one language with the API |
| Auth | **Argon2id** via `argon2-cffi` | HTTP-only, Secure, SameSite cookie holding a short-lived JWT |
| Calendar | Google Calendar API + tokenised `.ics` feed | Google OAuth credentials must come from your Google Cloud project |
| Frontend | **Next.js 14 (App Router) + Tailwind CSS** | Employee and Systems Manager views |

## Project structure

```
lead-system/
├── db/
│   └── migrations/
│       ├── 001_init_schema.up.sql        <- written + tested (this step)
│       ├── 001_init_schema.down.sql      <- written + tested (this step)
│       ├── 002_lead_queries.up.sql       <- Step 3: search indexes, conversion view fix
│       └── 002_lead_queries.down.sql
├── backend/                               <- Step 2+
│   └── app/
│       ├── main.py
│       ├── core/        config.py, security.py (Argon2id, JWT), db.py
│       ├── middleware/  auth.py, rbac.py (EMPLOYEE / SYSTEM_MANAGER guards)
│       ├── models/      SQLAlchemy models
│       ├── schemas/     Pydantic request/response models
│       ├── routers/     auth.py, leads.py, manager.py, calendar.py, feeds.py
│       ├── services/    leads.py, expiry.py, ics.py, google_calendar.py
│       └── workers/     celery_app.py, tasks.py (hourly expiry, calendar sync)
├── frontend/                              <- Step 5+
│   └── src/
│       ├── app/         (employee)/dashboard, (manager)/admin
│       └── components/  LeadCard, CountdownTimer, StatsCards, ArchiveTable
├── docker-compose.yml                     <- postgres, redis, api, worker, beat, web
└── .env.example
```

## Schema summary (10 tables, 2 reporting views)

| Table | Purpose |
|---|---|
| `users` | Accounts, role, Argon2id hash (salt embedded per user), lockout counters, `token_version` for revocation |
| `leads` | The clients. `state` (ACTIVE/EXPIRED) tracks ownership; `status` (NEW/PENDING/SAVING/SOLVED) tracks the pipeline |
| `lead_assignments` | Full ownership history; one open row per lead |
| `expired_clients_archive` | Snapshot at expiry: previous owner, timestamps, JSON ownership history, reassignment outcome |
| `lead_activities` | Per-lead follow-up log |
| `google_calendar_connections` | Per-employee OAuth link; refresh token stored encrypted |
| `calendar_feed_tokens` | `.ics` feed tokens; only the SHA-256 hash is stored |
| `calendar_events` | Day 1 / 7 / 14 / 20 follow-ups and their sync state |
| `audit_log` | Logins, reassignments, archive views |
| `worker_runs` | Health record for each hourly expiry run |
| `v_client_totals` | Total, solved, new, pending, saving, active, expired counts (deliverables 6, 7, 8) |
| `v_employee_conversion` | Per-employee solved / lost / conversion % (corrected in migration 002 so archived solved clients keep their credit) |

## Design decisions to review

1. **Day 22 rule.** `expires_at = assigned_at + 21 days` (set by trigger). A lead is visible while `now() < expires_at`. The worker only runs hourly, so the API must also filter `expires_at > now()`. That way a lead never shows on Day 22 even if the worker is late.
2. **Entry time is permanent.** `entered_at` never changes. `assigned_at` restarts the 21-day window on reassignment, which is how a manager gives an expired lead a fresh window.
3. **Database enforces the rules.** A check constraint makes ACTIVE leads require an owner and EXPIRED leads forbid one. Employees can never see an unowned lead, even through a bug.
4. **One owner per client.** A partial unique index on the normalised phone number blocks two employees from owning the same active client. This goes beyond your spec. Drop `uq_leads_active_phone` if you want duplicates allowed.
5. **Optimistic locking.** `leads.version` increments on every update, so two simultaneous edits cannot silently overwrite each other.

## Questions that affect Step 3 and 4

- Should a `SOLVED` lead also expire on Day 22? I assumed yes (the rule applies to every lead). If solved clients should stay with the employee permanently, the worker needs a `status <> 'SOLVED'` filter.
- "Claim" implies an unowned pool. In this design that pool is the expired archive: employees would claim from it, and managers can assign directly. Is that what you intend?

## Validation performed

Applied to a throwaway PostgreSQL 16 instance:
- Up migration applies cleanly; down then up round-trips (0 tables, then 12).
- 21-day trigger, owner/state constraint, duplicate-owner guard, solved timestamp and version increment all behave as specified.
- With 100,000 leads across 200 employees: the expiry-worker scan takes about 0.9 ms (uses `idx_leads_due_for_expiry`), an employee dashboard query about 0.6 ms (uses `idx_leads_owner_active`), and the manager totals view about 8 ms.


## Deployment layer (added)

| Path | Purpose |
|---|---|
| `db/migrations/003_expiry_and_supabase.*.sql` | `expire_due_leads()` (the Day-22 job) + RLS lockdown for Supabase |
| `db/supabase/schedule_expiry.sql` | pg_cron hourly schedule (run once in Supabase) |
| `backend/api/index.py`, `backend/vercel.json` | Vercel serverless entrypoint, rewrites, daily cron fallback |
| `backend/app/routers/cron.py` | `GET /api/cron/expire` (Bearer `CRON_SECRET`) |
| `backend/scripts/migrate.py` | Applies migrations to Supabase and records them |
| `frontend/` | Next.js 14 + Tailwind: login, employee dashboard, manager console |
| `DEPLOY.md` | Step-by-step Supabase + Vercel deployment |

The Celery/Redis worker from the original plan is replaced by the SQL function + pg_cron, so there is no Redis or long-running process to host.
