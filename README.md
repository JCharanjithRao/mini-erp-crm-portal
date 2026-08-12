# Mini ERP + CRM Operations Portal

An internal operations portal for a wholesale/distribution company — customers,
product inventory, and sales challans (delivery notes), with role-based access
for Admin, Sales, Warehouse, and Accounts staff.

## Live deployment

| | |
|---|---|
| GitHub repo | https://github.com/Akhilesh-2007/mini-erp-crm-portal |
| Frontend (Vercel) | https://frontend-six-liart-80.vercel.app |
| Backend API (Render) | https://erp-crm-backend-kj4h.onrender.com (`/api` prefix for all endpoints, `/health` for a liveness check) |
| Postgres | Render managed Postgres (free tier) |

> **Free-tier note:** the backend spins down after ~15 min idle; the first
> request after that can take 30–50s and occasionally returns a stray 404
> while it wakes up — retry once. See Known Limitations.

## Test login credentials

Seeded by `backend/prisma/seed.ts`. Same password for all four:

| Role | Email | Password |
|---|---|---|
| Admin | admin@erp.test | `Password123!` |
| Sales | sales@erp.test | `Password123!` |
| Warehouse | warehouse@erp.test | `Password123!` |
| Accounts | accounts@erp.test | `Password123!` |

## API documentation

A Postman collection covering every endpoint (auth, customers, follow-ups,
products, stock movements, challans) is at
[`postman/ERP-CRM.postman_collection.json`](postman/ERP-CRM.postman_collection.json).
Import it into Postman and set the collection variable `baseUrl` (defaults to
`http://localhost:4000/api`; point it at the live Render URL + `/api` to test
production). Run **Auth → Login as ...** first — it auto-saves the JWT into
the `token` variable used by every other request.

## Architecture

```
/backend   Node.js + TypeScript + Express + Prisma + PostgreSQL
/frontend  React + TypeScript + Vite
```

**Backend** — layered as `routes → middleware → controllers → Prisma`.
- `authenticate` middleware verifies the JWT and attaches `req.user`;
  `requireRole(...)` is a middleware factory that gates individual routes by
  role. Every route file composes these per CLAUDE.md's role matrix (e.g.
  customer/challan writes are Sales+Admin, product/stock writes are
  Warehouse+Admin, reads are open to any authenticated role).
- Validation is `zod` schemas per module (`src/validation/*`), parsed inside
  each controller; failures throw an `AppError(400, ...)` with per-field
  messages, caught by a single centralized error-handling middleware that
  produces the `{ error: { message, details? } }` shape for every failure
  mode (400/401/403/404/500) instead of leaking raw exceptions.
- Money and business-critical writes (confirming a challan, logging a stock
  movement) run inside Prisma interactive transactions: for a challan
  confirm, every line item's demand is checked against current stock
  **before** any write happens, so a rejection never leaves stock partially
  decremented.
- `ChallanItem` stores a **snapshot** of `productName`/`sku`/`unitPrice` at
  creation time — editing a `Product` later never rewrites historical
  challans.

**Frontend** — a shell (`AppLayout`: sidebar + top bar) wraps role-protected
routes (`ProtectedRoute` + React Router). Auth state lives in a `Context`
(`AuthContext`) holding the JWT **in memory only** — a full page reload logs
you out by design (see Known Limitations). Each domain module
(`customers/`, `products/`, `challans/`) follows the same shape: a typed
`api/*.ts` client, a list page (search/filter/pagination), a create/edit
form, and a detail page.

## Data model

See [CLAUDE.md](CLAUDE.md) for the full spec this was built against. Six
Prisma models: `User`, `Customer` (+ `FollowUp` for the "multiple follow-up
notes over time" requirement — a proper child table rather than a single
text field), `Product`, `StockMovement`, `Challan`, `ChallanItem`.

## Local setup

### Prerequisites
- Node.js 20+
- A PostgreSQL instance (local install, Docker, or a free Neon/Supabase DB)

### 1. Backend

```bash
cd backend
npm install
cp .env.example .env      # then edit DATABASE_URL / JWT_SECRET
npx prisma migrate deploy # applies the committed migrations
npm run seed               # creates the 4 test users above
npm run dev                 # http://localhost:4000
```

Env vars (`backend/.env`, see `.env.example`):

| Var | Required | Purpose |
|---|---|---|
| `DATABASE_URL` | yes | Postgres connection string |
| `JWT_SECRET` | yes | Signs/verifies login JWTs |
| `PORT` | no (default 4000) | Express listen port |
| `FRONTEND_URL` | no | Restricts CORS to this origin in production; unset = allow all (local dev) |

### 2. Frontend

```bash
cd frontend
npm install
npm run dev   # http://localhost:5173, talks to http://localhost:4000/api by default
```

`frontend/.env.development` already points `VITE_API_BASE_URL` at the local
backend. For a production build pointed at a different API, set
`VITE_API_BASE_URL` (see `.env.example`) before `npm run build`.

## Deployment

### Backend + Postgres → Render

The repo root has a `render.yaml` Blueprint that provisions both the
Postgres database and the web service in one step:

1. Push this repo to GitHub.
2. In the Render dashboard: **New → Blueprint**, connect the repo. Render
   reads `render.yaml` and provisions `erp-crm-db` (Postgres) and
   `erp-crm-backend` (web service) together, wiring `DATABASE_URL`
   automatically and generating a random `JWT_SECRET`.
3. Click **Apply**. The build runs `npm install && npm run build`; the start
   command runs `npx prisma migrate deploy && npm start`, so migrations
   apply automatically on every deploy.
4. Once live, seed the production database once:
   `render.yaml`'s free-tier web service has an interactive shell in the
   Render dashboard (**Shell** tab) — run `npm run seed` there.
5. Note the backend's `*.onrender.com` URL for the frontend step below.

### Frontend → Vercel

1. In Vercel: **New Project**, import the same GitHub repo, set **Root
   Directory** to `frontend`.
2. Add an environment variable `VITE_API_BASE_URL` =
   `https://<your-backend>.onrender.com/api`.
3. Deploy. `frontend/vercel.json` adds the SPA rewrite so client-side routes
   (e.g. `/customers/123`) work on refresh.
4. Back in Render, set the backend's `FRONTEND_URL` env var to the resulting
   `https://<your-app>.vercel.app` origin and redeploy, so CORS is locked
   down to just that origin instead of allowing all.

## Assumptions

- **Cancelling a Confirmed challan does not restore stock automatically**
  (documented in `challanController.ts`). By the time a challan is
  Confirmed, goods may have already physically left the warehouse; silently
  reversing the `StockMovement` on cancel could mask a real discrepancy. Any
  stock reversal is meant to be a deliberate, separately-logged IN movement.
- Challan numbers are per-calendar-year sequential (`CH-2026-0001`,
  resetting each year), generated read-max-then-increment inside the same
  transaction as the insert, with a small retry loop on unique-constraint
  collisions rather than serializing all challan creation behind a lock.
- A product can only appear once per challan — the backend validates this
  and rejects duplicate line items rather than silently merging quantities.
- `currentStock` is never directly editable via the product edit form; it
  only changes through logged `StockMovement`s (including the ones a
  confirmed challan generates), so the audit trail is always complete.

## Known limitations

- **Auth token is in-memory only** (no localStorage/cookie persistence) — a
  page refresh logs the user out. This was an explicit Phase 1 scope
  decision, not an oversight; adding persistence is a small, isolated change
  if needed later (guard against XSS token theft before doing so).
- No automated test suite (unit/integration) — verification for this
  submission was done via direct API testing and a scripted headless-browser
  pass through each module's UI, not a committed test suite.
- The customer/product pickers in the challan form load up to 200 records
  client-side rather than being a searchable async combobox — fine at demo
  scale, would need a proper typeahead against the paginated endpoints for a
  large catalog.
- No PDF/invoice export, no product image upload, no Docker/CI setup —
  called out as stretch goals in CLAUDE.md and left out to keep the core
  modules solid within the time box.
- No dedicated reporting/analytics views for the Accounts role beyond
  read-only access to the same Customer/Product/Challan screens everyone
  else uses.
- Render's free-tier web service spins down after inactivity — the first
  request after idle can take 30–50s (cold start). Render's free Postgres
  tier also expires after 30 days and needs manual upgrade/renewal.
