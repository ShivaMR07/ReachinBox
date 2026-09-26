# ReachInbox — Email Job Scheduler

A production-structured email scheduler + dashboard: schedule emails to send at a
specific time, at scale, with per-sender rate limiting, live queue visibility,
Slack alerts, and full restart-safety — no cron anywhere.

```
reachinbox-scheduler/
├── backend/     Express + TypeScript API, BullMQ worker, Postgres, Redis, Elasticsearch
└── frontend/    Next.js + TypeScript + Tailwind dashboard
```

## Quick start

### 1. Infra (Docker)
```bash
cd backend
docker compose up -d      # Postgres, Redis, Elasticsearch
```

### 2. Backend
```bash
cd backend
cp .env.example .env      # fill in GOOGLE_CLIENT_ID/SECRET at minimum (see below)
npm install
npm run prisma:migrate    # creates tables
npm run dev                # API on :4000
# in a second terminal:
npm run worker             # BullMQ worker process
```

### 3. Frontend
```bash
cd frontend
cp .env.local.example .env.local
npm install
npm run dev                 # dashboard on :3000
```

Open `http://localhost:3000`. Live BullMQ dashboard: `http://localhost:4000/admin/queues`.

### Credentials you must supply yourself
This sandbox can't mint real OAuth credentials for you — these are the only manual steps:
- **Google OAuth** (login): create a Web OAuth client at console.cloud.google.com →
  APIs & Services → Credentials. Redirect URI: `http://localhost:4000/api/auth/google/callback`.
- **Slack OAuth** (rate-limit alerts): create an app at api.slack.com/apps, add the
  `incoming-webhook` scope, redirect URL `http://localhost:4000/api/slack/callback`.
- **Ethereal SMTP**: needs nothing — the app auto-generates a throwaway Ethereal
  inbox on first boot and logs its login + a preview link per sent email to the
  console. Set `ETHEREAL_USER`/`ETHEREAL_PASS` in `.env` if you want a stable inbox
  across restarts instead of a fresh one each boot.

---

## Design decisions

### No cron — BullMQ delayed jobs
Every scheduled email is a single BullMQ **delayed job**, keyed by `jobId = ScheduledEmail.id`.
`queue.add(jobId, data, { jobId, delay })` schedules it for the exact send instant.
There is no polling loop, no `setInterval`, no `node-cron` anywhere in the codebase —
BullMQ's own delayed-job timer (backed by Redis' sorted sets) is the entire scheduling
mechanism.

### Restart safety & idempotency
- **Postgres is the source of truth.** A `ScheduledEmail` row is written *before* any
  BullMQ job is created (`emailService.scheduleCampaign`).
- **The BullMQ `jobId` is fixed to the row's own `id`.** BullMQ enforces unique job IDs
  per queue, so calling `enqueueEmailJob` twice for the same row is a guaranteed no-op —
  this is what makes scheduling idempotent, not application-level locking.
- **On every boot** (`emailService.reconcileOnStartup`, called from `server.ts`), the
  app scans for rows still `SCHEDULED`/`QUEUED` and re-enqueues *only* those whose
  BullMQ job is actually missing from Redis. If Redis has AOF/RDB persistence enabled
  (recommended in prod), a normal restart finds every job already present and touches
  nothing. Only if Redis itself lost its data (e.g. a fresh container) does reconciliation
  re-create jobs — and because it checks Postgres status first, an email that already
  sent (`status: SENT`) is never re-queued, and one still pending never gets duplicated.
- **Inside the worker**, before sending, we re-read the row and skip immediately if
  `status === SENT` — a second defense against a retried/duplicated job (e.g. BullMQ's
  own at-least-once retry semantics after a crash mid-send) ever double-sending.

### Throughput, concurrency & rate limiting
- **Worker concurrency** is configurable via `WORKER_CONCURRENCY` (default 5) — passed
  straight into `new Worker(..., { concurrency })`.
- **Minimum delay between sends**, per sender, defaults to **2000ms** (`DEFAULT_MIN_DELAY_MS`),
  overridable per campaign from the compose form. Enforced in the worker via a Redis-backed
  "last send timestamp" check (`waitForSenderPacing`) so it holds even across multiple
  concurrent worker slots/processes, not just within one.
- **Emails-per-hour** is enforced **per sender**, with a Redis fixed-window counter
  (`services/rateLimiter.ts`): key = `sender + hour-bucket`, incremented with a single
  atomic Lua script (`GET` → compare → `INCR`) so concurrent workers across any number
  of processes/machines can never race past the limit — the guarantee lives entirely in
  Redis, never in an in-memory counter. Limits are fully configurable per campaign
  (`hourlyLimit` in the compose form) or via `DEFAULT_MAX_EMAILS_PER_HOUR`.
  - *Trade-off:* this is a fixed window, not a sliding window/token bucket — a small
    double-burst is possible right at a window boundary. Documented as an accepted
    trade-off for this scope; swapping in a sliding-window algorithm is localized to
    `rateLimiter.ts`.
- **When the limit is hit**, the job is *never* dropped or failed. The worker calls
  `job.moveToDelayed(nextWindowStart)` (BullMQ's `DelayedError` pattern) on the **same
  job / same jobId**, re-delaying it to the start of the next hour window — this is
  itself idempotent (no new job is created) and preserves relative order between
  deferred jobs.
- **Slack notification** fires live (a real `axios.post` to the stored incoming-webhook
  URL) the instant a sender's hourly cap is reached. If the user never connected Slack,
  this is a silent no-op — checked fresh from Postgres on every call, so connecting Slack
  later starts notifications immediately with no redeploy.

### 1000+ emails scheduled at once
Recipients are parsed from the uploaded CSV/text file (`/api/emails/parse-leads`, a
tolerant regex scan so it works on messy exports), deduped, and each gets its own
`ScheduledEmail` row + delayed BullMQ job in one transaction + loop. At send time the
combination of (a) `minDelayMs` pacing and (b) the per-hour cap means a 1000-recipient
campaign submitted "all at once" naturally spreads itself across however many hour
windows the cap requires — excess jobs simply sit as BullMQ delayed jobs (cheap, just
Redis sorted-set entries) until their window arrives, rather than the server trying to
send them all simultaneously.

### Search — Elasticsearch
Every email is indexed the moment it's sent (`services/elasticService.ts`), with the
index auto-created on boot if missing. `/api/search?q=...&status=...` does a
`multi_match` across subject/body/recipient. Indexing failures never block or fail the
send pipeline itself (logged and swallowed) — Elasticsearch is treated as a derived,
rebuildable view, not a second source of truth.

### Live BullMQ dashboard
`@bull-board/express` is mounted at `/admin/queues`, giving real-time visibility into
active/delayed/completed/failed jobs without any custom polling code.

---

## API surface (backend)

| Method | Path | Description |
|---|---|---|
| GET | `/api/auth/google` | Start Google OAuth login |
| GET | `/api/auth/me` | Current session user |
| POST | `/api/auth/logout` | Logout |
| POST | `/api/emails/parse-leads` | Upload CSV/text → detected email addresses |
| POST | `/api/emails/schedule` | Schedule a campaign (subject, body, recipients, timing) |
| GET | `/api/emails/scheduled` | List pending/queued/deferred emails |
| GET | `/api/emails/sent` | List sent/failed emails |
| GET | `/api/slack/connect` | Start Slack OAuth |
| GET | `/api/slack/status` | Is Slack connected |
| POST | `/api/slack/disconnect` | Disconnect Slack |
| GET | `/api/search?q=` | Elasticsearch full-text search over emails |
| * | `/admin/queues` | Live BullMQ dashboard |

## What's deliberately out of scope for this submission
- Automated tests weren't included given the EOD deadline — the modules
  (`rateLimiter`, `emailService`, `emailWorker`) are written as small, pure-ish
  functions specifically so unit tests are easy to bolt on afterward.
- No CI/CD pipeline or cloud deploy config (e.g. Terraform) — the Docker Compose
  file covers local infra; deploying is a matter of pointing `DATABASE_URL`,
  `REDIS_URL`, and `ELASTIC_NODE` at managed services (Railway/Render/Neon/Upstash/
  Elastic Cloud all work with zero code changes) and running `npm run build && npm start`
  for both the API and worker processes.
- The frontend follows the written spec closely (header, tabs, compose modal, tables,
  loading/empty states) rather than a pixel-perfect Figma match, since no Figma link
  was reachable from the assignment text provided.
