# Deploy: Supabase (database) + Vercel (API and web)

```
Browser ──► Vercel project "web"  (Next.js)   ── /api/* and /feeds/* proxied ──►  Vercel project "api" (FastAPI)
                                                                                         │
                                                                              Supabase Postgres (transaction pooler :6543)
                                                                              pg_cron runs expire_due_leads() every hour
```

Two Vercel projects from one GitHub repo. The browser only ever sees the **web** domain, so the session cookie is first-party and no CORS is needed.

## 0. Push to GitHub
Unzip and push the whole thing as one repo, keeping the layout: the repo root holds `.gitignore` and the `lead-system/` folder (with `backend/`, `frontend/`, `db/`). The Vercel root directories below assume that layout.

## 1. Supabase
1. **New project** → pick a region near your users → save the database password.
2. **Connect** button (top bar) → copy two strings and put your password in each (URL-encode special characters: `@`→`%40`, `#`→`%23`, `/`→`%2F`):
   - **Session pooler** (port 5432): used on your machine to run migrations.
   - **Transaction pooler** (port 6543): used by the API on Vercel.
   The direct `db.<ref>.supabase.co` host is IPv6-only and won't work from Vercel; use the poolers.
3. **Create the tables** (from your computer):
   ```bash
   cd backend
   python3 -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   export DATABASE_URL='<session pooler string>'
   python -m scripts.migrate          # applies 001, 002, 003 (safe to re-run)
   MANAGER_PASSWORD='choose-a-long-passphrase' \
     python -m scripts.create_manager --email you@company.com --name "Your Name"
   ```
   Migration 003 adds `expire_due_leads()` and turns on Row Level Security with no policies, so the public Supabase Data API/anon key can read nothing. The app talks to Postgres directly and is unaffected.
4. **Schedule the Day-22 job**: Dashboard → Database → Extensions → enable `pg_cron`, then paste `db/supabase/schedule_expiry.sql` into the SQL Editor and run it. Check it later with `SELECT * FROM cron.job_run_details ORDER BY start_time DESC LIMIT 5;`.

## 2. Vercel project "api" (FastAPI)
Add New → Project → import the repo → **Root Directory: `lead-system/backend`** (Framework: Other). Environment variables:

| Name | Value |
|---|---|
| `DATABASE_URL` | transaction pooler string (port 6543); `postgresql://` is fine |
| `JWT_SECRET` | `python -c "import secrets; print(secrets.token_urlsafe(64))"` |
| `ENVIRONMENT` | `production` |
| `DB_NULL_POOL` | `true` |
| `MAX_CONCURRENT_HASHES` | `2` |
| `ALLOWED_ORIGINS` | `["https://YOUR-WEB-DOMAIN"]` (set after step 3, then redeploy) |
| `PUBLIC_BASE_URL` | `https://YOUR-WEB-DOMAIN` |
| `CRON_SECRET` | any 32+ char random string (only for the cron fallback) |

Deploy, then open `https://<api-domain>/healthz`. Expect `{"status":"ok"}`.

## 3. Vercel project "web" (Next.js)
Add New → Project → same repo → **Root Directory: `lead-system/frontend`**. Environment variables:

| Name | Value |
|---|---|
| `BACKEND_URL` | `https://<api-domain>` (no trailing slash) |
| `SESSION_COOKIE_NAME` | `lead_session` |

Deploy. Then go back to the **api** project, set `ALLOWED_ORIGINS` and `PUBLIC_BASE_URL` to the web URL, and **redeploy** the api project (env changes apply to new deployments only).

Use a custom domain on **web** if you have one (Settings → Domains) and put that exact URL in `ALLOWED_ORIGINS`.

## 4. Smoke test
1. Open the web URL → sign in as the manager → **Team** → add an employee.
2. Sign in as the employee (private window) → **Add client** → see the countdown.
3. Employee → **Calendar reminders** → create link → subscribe on a phone.
4. Manager → **Overview** shows totals; **All clients** shows the lead with its input time.
5. Force an expiry to see the pool work: in Supabase SQL Editor
   `UPDATE leads SET assigned_at = now() - interval '22 days' WHERE client_name = 'TEST';` then `SELECT expire_due_leads();`. The client moves to **Expired archive**, and the employee can re-claim it by typing the number again.

## Troubleshooting
- **`Origin not allowed` (403) on every save** → `ALLOWED_ORIGINS` doesn't exactly match the web URL (scheme + host, no trailing slash, inside a JSON list), or you didn't redeploy api.
- **Login works but you're sent back to /login** → `SESSION_COOKIE_NAME` differs from the API's cookie name, or you're testing the API domain directly instead of the web domain.
- **API returns an HTML "Authentication Required" page** → Vercel Deployment Protection is on for the api project; turn it off for Production (Settings → Deployment Protection).
- **`prepared statement ... already exists`** → the app is on the transaction pooler without pgbouncer mode: use port 6543 on the `*.pooler.supabase.com` host, or set `DB_PGBOUNCER=true`.
- **Cold start + login takes ~1 s** → normal: Argon2id hashing (64 MiB) is deliberate.

## Costs and limits to know
- Vercel Hobby crons run at most once a day. That's why hourly expiry lives in Supabase `pg_cron` (free). `vercel.json` keeps a daily cron as a backup; it needs `CRON_SECRET` set.
- Supabase free projects pause after a week of inactivity and have small connection limits; use Pro for a production sales team.
- Not built here: per-IP login rate limiting (needs Redis/Upstash; per-account lockout is active), and the Google Calendar API push. The `.ics` subscription feed works today.
- Audit-log IPs will show Vercel's proxy addresses, not employees' real ones.
