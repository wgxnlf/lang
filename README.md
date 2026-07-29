# Docebo — online language school (full-stack)

A complete, sellable web product: marketing site, learning app, accounts,
subscriptions, and payments through **bePaid** — in one small Node.js codebase
with zero native dependencies.

## What's inside

**Frontend (12 pages, vanilla HTML/CSS/JS):**

| Page | Purpose |
|---|---|
| `index.html` | Landing: hero, languages, pricing, features, testimonials |
| `languages.html` | Catalog of 11 languages with region filters |
| `pricing.html` | 4 plans, monthly/yearly toggle, comparison table, FAQ — **buy buttons are live** |
| `about.html`, `contact.html` | Company pages; the contact form posts to the API |
| `register.html`, `login.html` | Accounts (JWT) |
| `dashboard.html` | Plan, streak, per-language progress, payment history |
| `learn.html` | The product: language grid → lesson list → lesson player with vocabulary, phrases and a mini-quiz |
| `admin.html` | Admin panel: stats, learners, payments, support inbox |
| `payment-result.html` | Post-checkout success/failure page (polls payment status) |
| `payment-mock.html` | Built-in simulated checkout (used only until bePaid is configured) |

**Backend (Express + SQLite via `node:sqlite`):**
- JWT auth (bcrypt password hashing), roles (`user` / `admin`)
- Plans & subscriptions with real access enforcement:
  Start = 1 language / 5 lessons · Basic = 1 language, all lessons ·
  Progress = 3 languages · Unlimited = everything
- 11 languages × 8 lessons seeded automatically (lessons 1–2 of every course
  ship with real vocabulary and phrases; 3–8 are structured outlines ready for
  your own content)
- Progress tracking, streaks, enrollments
- **bePaid** checkout + webhook, with a mock gateway fallback
- Contact-form inbox and an admin API

## Quick start

Requires **Node.js ≥ 22.5** (uses the built-in SQLite module — nothing to compile).

```bash
npm install
cp .env.example .env
npm start
```

Open http://localhost:3000. The database is created and seeded on first run.

- Admin panel: sign in with `ADMIN_EMAIL` / `ADMIN_PASSWORD` from `.env`
  (defaults: `admin@docebo.local` / `admin12345` — **change them**), then open `/admin.html`.
- Payments work immediately in **mock mode**: pressing "Choose plan" leads to a
  built-in simulated checkout that exercises the exact same activation logic as
  the real webhook.

## Connecting bePaid (real payments)

1. Register a shop at [bepaid.by](https://bepaid.by) and get your **Shop ID**
   and **Secret Key** from the merchant back office.
2. Put them in `.env`:
   ```
   BEPAID_SHOP_ID=12345
   BEPAID_SECRET_KEY=your-secret-key
   BEPAID_TEST_MODE=true          # test transactions first
   CURRENCY=BYN                   # or EUR/USD/RUB — whatever your shop accepts
   BASE_URL=https://your-domain.example
   ```
3. `BASE_URL` **must be a public HTTPS URL** — bePaid redirects buyers back to
   `payment-result.html` and posts server-to-server notifications to
   `/api/payments/webhook`. For local testing use a tunnel (e.g. ngrok) and set
   `BASE_URL` to the tunnel URL.
4. Run a test purchase with bePaid's test cards, then set `BEPAID_TEST_MODE=false`.

How the flow works:

```
pricing.html ──POST /api/payments/checkout──▶ server creates a pending payment
        ◀── redirect_url ──                    and a bePaid checkout token
buyer pays on checkout.bepaid.by
        │
        ├─ browser → payment-result.html (success/decline/fail/cancel URL)
        └─ bePaid → POST /api/payments/webhook (Basic-auth verified, idempotent)
                    └─ payment marked successful → subscription activated
```

## API overview

```
POST /api/auth/register            {name, email, password}
POST /api/auth/login               {email, password}
GET  /api/auth/me                  → user + effective plan access

GET  /api/languages                catalog with lesson counts
GET  /api/languages/:code          lessons with per-user lock flags
GET  /api/lessons/:id              lesson content (enforces plan, auto-enrolls)
POST /api/lessons/:id/complete
GET  /api/me/overview              plan, progress, streak, payments

GET  /api/plans
POST /api/payments/checkout        {plan_id, period: month|year} → redirect_url
POST /api/payments/webhook         bePaid notification endpoint
GET  /api/payments/status/:trackingId

POST /api/contact                  {name, email, topic, message}

GET  /api/admin/stats|users|payments|messages     (admin role)
PATCH /api/admin/messages/:id      {status: read}
```

## Project layout

```
server/
  index.js            Express app: API + static frontend
  config.js           .env → config (bePaid, JWT, currency…)
  db.js               schema, seed data, access rules
  middleware/auth.js  JWT verification, admin guard
  services/bepaid.js  checkout creation + webhook auth
  routes/             auth · catalog · payments · contact · admin
public/               the whole frontend (styles.css, api.js, script.js, pages)
data/                 SQLite database (created at runtime, git-ignored)
```

## Production notes

- Set a long random `JWT_SECRET` and change the admin credentials.
- Run behind HTTPS (nginx/Caddy or a PaaS) — required for bePaid callbacks.
- Back up `data/docebo.db`; it's the entire state.
- Lesson content lives in the `lessons` table as JSON — edit it directly or
  extend `server/db.js` seeding to plug in your own curriculum.
