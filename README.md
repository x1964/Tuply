<div align="center">
  <img src="./public/logo.svg" width="72" alt="Tuply logo — a green T crossed by a heartbeat pulse" />
  <h1>Tuply</h1>
  <p><strong>Know when your SaaS breaks — before your users do.</strong></p>
  <p>Developer-friendly monitoring for websites and APIs: scheduled health checks, response-time tracking, incident detection, and outage alerts.</p>
  <p>
    <a href="https://github.com/x1964/Tuply/actions/workflows/ci.yml"><img src="https://github.com/x1964/Tuply/actions/workflows/ci.yml/badge.svg" alt="CI status" /></a>
    <img src="https://img.shields.io/badge/Next.js-16-black" alt="Next.js 16" />
    <img src="https://img.shields.io/badge/TypeScript-5-3178C6" alt="TypeScript 5" />
    <img src="https://img.shields.io/badge/Supabase-Postgres_+%20Auth-3FCF8E" alt="Supabase" />
  </p>
</div>

---

## About

Every SaaS breaks eventually — a deploy goes sideways, a certificate expires, an
upstream API slows to a crawl. The question is whether you hear about it from
your monitoring or from an angry user.

Tuply probes your endpoints on a schedule and keeps the receipts: each check
records its outcome, status code and response time. When failures stack up, an
incident opens and you get alerted. When the endpoint recovers, the incident
resolves. A dashboard shows uptime, latency trends and incident history; a
public status page can share a safe subset of that with your users.

Built for indie hackers, Micro-SaaS founders, freelance developers and small
teams who want reliability without operating a monitoring stack.

**Project status:** working application, verified locally where possible —
`lint`, `typecheck`, 101 tests and the production build all pass. The database
schema is idempotent and self-verifying but must be applied to a Supabase
project first. Live end-to-end authentication and real alert delivery still
need verification against a configured project.

---

## Features

### Implemented

| Capability | Details |
| ---------- | ------- |
| Website & API monitoring | HTTP(S) endpoints, `GET` / `HEAD` / `POST`, configurable expected status |
| Scheduled health checks | Long-running worker (`npm run worker:monitoring`) or one-shot ticks via `POST /api/checks/run` (cron) |
| Response-time tracking | Per-check timings, per-monitor slow threshold, response-time charts |
| Uptime history | Check history, uptime aggregates, latest-check-per-monitor in one query |
| Incident management | Opens after 3 consecutive failures, resolves after 3 consecutive successes; timeline with causes and durations |
| Email alerts | Failure + recovery notifications via Resend, with timeout, retry and a delivery audit trail in Settings |
| Public status pages | Published-only pages with a deliberately narrow projection (no URLs, timings or error details leak) |
| SSL certificate monitoring | Expiry readings for HTTPS monitors, refreshed on a slow cadence |
| Dashboard | Overview stats, needs-attention list, recent incidents, live auto-refresh |
| Authentication | Sign up, sign in (remember-me), password reset, protected routes and APIs |

### Partially implemented

| Capability | State |
| ---------- | ----- |
| Alert channels | Email works; the dispatcher has a `webhook` channel extension point with no webhook notifier yet |
| Billing | Placeholder route only — nothing is charged, Stripe is not wired up |

### Planned

Heartbeat / cron-job monitoring · Discord / Slack / Telegram alerts · check-history
retention job (history is currently unbounded) · team management · AI-assisted
incident diagnosis · status-page customisation.

---

## Screenshots

No screenshots are committed yet. When real captures exist they will live in
[`docs/screenshots/`](./docs/screenshots/) per the checklist there — this
project does not ship fabricated dashboards.

---

## Tech stack

| Technology | Role |
| ---------- | ---- |
| Next.js 16 (App Router) + React 19 | Routing, Server Components, Server Actions, Route Handlers |
| TypeScript 5 (strict) | App and test code |
| Tailwind CSS 4 | Styling |
| Supabase (Postgres + Auth) | Persistence, authentication, Row Level Security |
| `@supabase/ssr` / `@supabase/supabase-js` | Server/browser auth clients |
| Zod | Input validation (server-enforced) |
| `undici` | Outbound probes with a pinned DNS dispatcher |
| Resend | Email alert delivery (optional) |
| Node built-in test runner | Unit + integration tests, no extra framework |
| ESLint + Prettier | Lint and formatting |

---

## Architecture

Feature-oriented layers with one-way dependencies:

```
app/  →  features/  →  services/  →  repositories/ports  →  Supabase
```

- **`app/`** — routing and rendering only. Pages authenticate, call services, render.
- **`features/`** — one product area each (auth, monitors, incidents, dashboard, status-pages, settings): screens, Server Actions, view models.
- **`services/`** — business rules independent of caller: the monitoring engine (`checker`, SSRF guard usage, `classify`, `health`, `process-monitor`, `scheduler`, incident engine) and notifications (dispatcher, email notifier).
- **`repositories/`** — ports plus Supabase adapters; the only layer that queries the database.
- **`lib/`** — framework plumbing: Supabase clients, SSRF guard, validation helpers, logging.
- **`workers/`** — monitoring worker lifecycle (claim → probe → record → incident → notify).
- **`proxy.ts`** — session middleware: refreshes the Supabase session, guards protected routes.

Full rationale, boundary rules and the security model:
[`docs/architecture.md`](./docs/architecture.md).

---

## Getting started

### 1. Prerequisites

- Node.js 20+ and npm
- A Supabase project (free tier works)

### 2. Clone and install

```bash
git clone https://github.com/x1964/Tuply.git
cd Tuply
npm install
```

### 3. Configure the environment

```bash
cp .env.example .env.local     # Windows: copy .env.example .env.local
```

Fill in `.env.local` — see [Environment variables](#environment-variables).

### 4. Set up the database

Open your Supabase project → **SQL Editor** → paste the entire
[`schema.sql`](./schema.sql) → **Run**. Read the PASS/FAIL table it prints:
every row must say PASS. The file is idempotent and never deletes data, so
re-running it is always safe.

### 5. Start the app

```bash
npm run dev                    # http://localhost:3000
```

Sign up, add a monitor, then start probing (the dev server does not run checks
by itself):

```bash
npm run worker:monitoring      # long-running worker
```

Without Supabase configured, the app still builds and the landing page
explains what is missing instead of failing.

---

## Environment variables

Placeholders for all of these live in [`.env.example`](./.env.example). Never
commit `.env.local`.

**Public configuration** (safe in the browser, `NEXT_PUBLIC_` prefix):

| Variable | Purpose |
| -------- | ------- |
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL (Project Settings → Data API) |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Anon key — safe to expose; RLS protects the data |
| `NEXT_PUBLIC_APP_URL` | Deployment origin for auth redirects and alert links (defaults to `http://localhost:3000`) |

**Server-only secrets** (never prefixed with `NEXT_PUBLIC_`):

| Variable | Purpose |
| -------- | ------- |
| `SUPABASE_SERVICE_ROLE_KEY` | Bypasses all RLS — worker and admin console only |
| `MONITORING_WORKER_SECRET` | Bearer secret for `POST /api/checks/run` (generate with `openssl rand -hex 32`) |
| `ADMIN_EMAILS` | Comma-separated allowlist for the admin console |
| `RESEND_API_KEY` / `ALERT_FROM_EMAIL` | Email delivery; the app runs without alerting if blank |
| `ALLOW_PRIVATE_TARGETS` | Dev/self-host probing of private hosts; forced off in production |
| `MONITORING_BATCH_SIZE` | Optional worker batch-size override |
| `STRIPE_SECRET_KEY` / `STRIPE_WEBHOOK_SECRET` | Reserved — billing is not implemented |
| `LOG_LEVEL` | `debug` / `info` / `warn` / `error` |

---

## Project structure

```
app/                 routes, layouts, API handlers (rendering only)
  (auth)/            sign in · sign up · password reset
  (dashboard)/       dashboard · monitors · incidents · status-pages · settings · billing
  admin/             allowlisted admin console
  api/               /monitors · /incidents · /checks/run
  status/[slug]/     public status pages
features/            screens + Server Actions per product area
components/          ui/ (generic) · layout/ · shared/ (domain-aware)
services/            monitoring/ (checker → health → incidents) · notifications/
repositories/        ports + Supabase adapters (only DB access)
workers/             monitoring worker entrypoint + lifecycle
lib/                 supabase/ · security/ (SSRF guard) · validation/ · core/ · logger/
config/              env · app routes · monitoring thresholds
types/               database · monitoring · common
supabase/            migrations · seed
docs/                architecture · database · monitoring · development · deployment
tests/               unit/ · support/
schema.sql           the whole database: tables, RLS, grants, self-checks
proxy.ts             session refresh + route guards
```

---

## Testing and quality checks

| Command | Purpose |
| ------- | ------- |
| `npm run lint` | ESLint |
| `npm run typecheck` | TypeScript, app + tests |
| `npm test` | Unit + integration suites (Node's built-in runner) |
| `npm run build` | Production build |
| `npm run worker:monitoring` | Long-running monitoring worker |
| `npm run db:start` | Local Supabase stack (needs Supabase CLI + Docker) |

Quality gate before pushing (also enforced by CI on every PR):

```bash
npm run lint && npm run typecheck && npm test && npm run build
```

Last verified locally: lint, typecheck, **101/101 tests** and production build
all green. CI has not run on GitHub yet — it runs on the first push/PR.

---

## Security

- **SSRF first.** Tuply fetches user-supplied URLs, so `lib/security/url-guard.ts`
  is the mandatory choke point: private/loopback/link-local/metadata ranges
  refused, connections pinned to validated addresses (DNS-rebinding defence),
  every redirect hop re-validated, response sizes and timeouts bounded.
- **Secrets.** Service-role key is server-only (`server-only` guard);
  `NEXT_PUBLIC_` values are assumed published. Never commit `.env.local`.
- **Auth.** Sessions via Supabase SSR cookies; `getUser()` (not `getSession()`)
  on the server; auth endpoints rate-limited; login errors stay generic to
  avoid account enumeration.
- **Data.** Row Level Security on every table; ownership re-checked
  server-side; logs redact tokens and secrets.
- **Reporting.** Do not open public issues for vulnerabilities — see
  [SECURITY.md](./SECURITY.md) and use private reporting.

No formal security audit or certification has been completed.

---

## Roadmap

Done now: monitoring, incidents, email alerts, status pages, dashboard, auth.
Up next (honestly labelled): check-history retention, webhook alert channel,
heartbeat/cron monitoring, chat-app alerts, team management, billing,
AI-assisted incident diagnosis. The
[Features](#features) table is the source of truth for what works today.

---

## Contributing

Bug reports and pull requests are welcome:

1. Open an issue first for anything beyond a trivial fix.
2. Keep to the layering rules in [`docs/architecture.md`](./docs/architecture.md).
3. Pass the [quality gate](#testing-and-quality-checks) and add tests for changed behaviour.
4. Never weaken a security control to make something work.

Details: [CONTRIBUTING.md](./CONTRIBUTING.md). For setup and deployment see
[`docs/development.md`](./docs/development.md) and
[`docs/deployment.md`](./docs/deployment.md).

---

## License

No license has been chosen yet, so the code is currently **not** licensed for
reuse. (Maintainer: MIT is the usual default for a project like this.)

---

<div align="center">
  <p><strong>Tuply</strong> — reliability you can read in one glance, alerts you actually get.</p>
</div>
