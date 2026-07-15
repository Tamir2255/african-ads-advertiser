# AfriAd Platform

Twin-platform African digital advertising network:

- `backend/` — Central API (Node.js/Express + PostgreSQL). Deploy to **Render**.
- `afriad-advertiser/` — Advertiser console (static HTML/JS). Deploy to **Vercel**.
- `afriad-earner/` — Earner dashboard (static HTML/JS). Deploy to **Vercel** (separate project).
- `afriad-admin/` — Internal review console for social-proof approvals (static HTML/JS). Deploy to **Vercel** (separate project) or keep it unlisted/password-protected via Vercel's access controls since it's staff-only.
- Database: **Railway** managed PostgreSQL.

## ⚠️ Before you do anything else

The credentials that were previously sitting in a plain `.env` file (Railway DB password,
JWT secret) must be treated as **compromised** — rotate the Railway database password and
generate a brand-new `JWT_SECRET` before deploying this. Never commit a real `.env` file;
use `backend/.env.example` as your template and set real values only in Render's
environment variable dashboard.

## Deployment order

1. **Railway** — create a PostgreSQL database, then run `backend/schema.sql` against it
   (Railway's built-in query console, or `psql "$DATABASE_URL" -f schema.sql`).
2. **Render** — create a Web Service pointing at `backend/`. Set env vars from
   `backend/.env.example` (`DATABASE_URL` from Railway, a new `JWT_SECRET`,
   `FLUTTERWAVE_SECRET_KEY` / `FLUTTERWAVE_WEBHOOK_HASH` once you have real gateway keys).
   Build command: `npm install`. Start command: `npm start`.
3. **Vercel (x3)** — deploy `afriad-advertiser/`, `afriad-earner/`, and `afriad-admin/`
   as three separate Vercel projects. In each project's HTML files, replace the
   placeholder `API_BASE_URL` (`https://afriad-backend.onrender.com`) with your actual
   Render URL.
4. Register a normal account through `afriad-advertiser` or `afriad-earner` (either
   works — role only affects what wallets/campaigns it can touch), then promote it to
   admin directly in the database:
   ```sql
   UPDATE users SET role = 'admin' WHERE email = 'you@example.com';
   ```
   Sign in at `afriad-admin/index.html` with that same email/password to review pending
   social-proof submissions and approve or reject each one with one click.

## What changed from the previous version

- Advertisers no longer pre-load a wallet — each campaign generates an invoice and a
  mobile money checkout link (`paymentGateway.js`), matching the pay-as-you-go spec.
- Earners have a real wallet (`balance` + `pending_balance` per currency) credited by
  `/api/tasks/*` routes, with atomic SQL transactions so a video view or click can't
  double-credit or leave the campaign budget in an inconsistent state.
- Passwords are hashed with bcrypt (was plain text comparison).
- `multer` was being used but was missing from `package.json` — this alone would have
  failed the Render build.
- Frontend fetch calls now point at an explicit Render URL instead of
  `window.location.origin`, which silently broke every request since Vercel and Render
  are different domains.
- The old standalone `create-campaign.html` mockup (with its own disconnected pricing
  script) was folded into `dashboard.html`'s campaign form, which is now wired to the
  real API — keeping a second, unconnected copy of the same form would just drift out
  of sync again.
