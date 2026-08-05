# SESDIAN - Asset Management

Lightweight asset-management app: a static, zero-build frontend (vanilla
HTML/CSS/JS) backed by a Neon Postgres database through small Node serverless
functions. The frontend stays instant; the backend gives real auth, roles, and
data persistence.

> Replaces the original Laravel + React build that shipped a ~1.1 MB JS bundle per page.

## Architecture
```
Browser  -->  /api/* (Node handlers)  -->  Neon Postgres
 public/        register, login, me, settings        users, settings
 *.html         categories, rooms, assets            categories, rooms
 assets/app.js  borrowings (list/create/approve)     assets, borrowings
 assets/db.js   users (list/role)
```
- Auth: JWT (HS256) signed server-side; passwords hashed with scrypt; token in
  localStorage, sent as a Bearer header. `JWT_SECRET` is required (no fallback).
- The Neon connection string lives only in the server env var `DATABASE_URL`,
  never in the browser. Every data endpoint checks the token; writes check admin.
- DB-first: pages render entirely from the database (or an in-memory demo set);
  the static templates are hidden, so no seed data flashes before the DB loads.

## Roles
- First account to log in (when no admin exists yet) becomes the admin; everyone
  else is a normal user. Admins can promote/demote others on the Kelola User page.
- User: browse assets/categories/rooms, submit borrowing requests.
- Admin: approve / reject / lend / return borrowings (stock adjusts automatically),
  add/edit/delete assets (with image upload), categories and rooms, manage users,
  and set the WhatsApp notification number.

## WhatsApp notifications
- An admin sets the notification number on the Kelola User page (`settings` table,
  created automatically on first use - no manual migration needed).
- New borrowing request -> the admin is notified.
- Overdue item (past due date, not returned) -> admin can press "Ingatkan WA" on
  that row to remind the borrower.
- On return, the borrower gets a confirmation; a borrower can also press
  "Konfirmasi Pengembalian" to tell the admin they have returned an item.
- Delivery: if `FONNTE_TOKEN` (https://fonnte.com gateway) is set, messages are
  auto-sent server-side. Otherwise the app opens a `wa.me` click-to-send link.
  Borrower reminders need the user's phone (captured at registration / `users.phone`).

## UI
- Loading: protected pages show a spinner until the database data is rendered,
  so no hardcoded/placeholder content flashes first.
- Icons: the interface uses inline SVG icons (no emoji); `public/assets/icons.js`
  holds the set and `applyBinds` maps category icons by name.

## Project layout
```
public/             the static site (served by server.mjs, or any static host)
  *.html            pages
  assets/app.css    design tokens + animations
  assets/app.js     runtime: auth, rendering, search/filter, admin UI, toasts
  assets/db.js      data layer (calls /api, manages the JWT, demo dataset)
  assets/config.js  BACKEND mode + optional API base (no secrets)
  favicon.svg
api/                serverless functions
  register, login, me, categories, rooms, assets, borrowings, users, settings
  _auth.js _db.js _wa.js   (shared helpers, not routed)
  each file is a plain (req, res) handler, so it runs on Vercel and on Node alike
db/                 SQL: schema, migrations, seed data
  neon-schema.sql     full schema + seed (fresh install)
  migrate*.sql        incremental migrations (v5/v6 are destructive data loads)
docs/               guides (setup, local dev, deployment, features, etc.)
tests/              node:test unit tests (npm test)
lib/load-env.mjs    reads .env.local / .env into process.env
server.mjs          the HTTP server: static public/ + /api handlers (npm start)
app.js              fixed entry point for Hostinger hPanel / Passenger
dev-server.mjs      starts server.mjs in dev mode (npm run dev)
scripts/            db:check / db:run helpers (see docs/LOCAL-DEV.md)
package.json        declares @neondatabase/serverless
vercel.json         Vercel-only settings, kept so old deployments still work
public/.htaccess    Apache/LiteSpeed rules for Hostinger shared hosting
```

## Setup
1. Create a Neon database (https://neon.tech) and copy the connection string.
2. Fresh DB: run `db/neon-schema.sql` in the Neon SQL Editor, then `db/migrate-v3.sql`.
   Existing DB (already has data): run `db/migrate.sql` then `db/migrate-v3.sql`
   (they only add columns/tables; nothing is dropped).
   To run locally instead, see `docs/LOCAL-DEV.md`.
3. Deploy and set these environment variables on the host:
   - `DATABASE_URL`  (required) - your Neon connection string
   - `JWT_SECRET`    (required) - a long random string, e.g. `openssl rand -hex 32`
   - `FONNTE_TOKEN`  (optional) - enables automatic WhatsApp sending
   Hostinger (VPS or hPanel Node.js app): see `docs/HOSTINGER-DEPLOY.md`.
   Any other Node host: `npm install --omit=dev && npm start`.
   Vercel still works unchanged via `vercel.json`.
4. Log in with your existing account; since it is the only account and there is
   no admin yet, it is promoted to admin automatically. New sign-ups are users.

## Local development
- Full stack: `npm run dev` (needs `.env.local`; see `docs/LOCAL-DEV.md`).
- UI only (no backend): set `BACKEND: 'demo'` in `public/assets/config.js`, then
  open `public/index.html` or run `npx serve public`. Demo signs you in as an
  admin so every feature is previewable; data lives in memory for the session.

## Notes
- Borrowing is atomic: stock is reserved and the record inserted in a single SQL
  statement; status changes adjust stock in one statement with a re-reserve guard.
- Asset images are downscaled client-side (max 800px, JPEG) and stored as a data
  URL in `assets.image`; keep uploads modest (the API caps a body at 5 MB, and
  Vercel's own request limit is ~4.5 MB).
