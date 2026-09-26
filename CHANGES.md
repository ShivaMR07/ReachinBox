# Fixes applied

I read through every file in `backend/` and `frontend/` against the README's
stated requirements and found three real bugs. Everything else (auth flow,
BullMQ delayed-job scheduling, restart reconciliation, Slack notifications,
Elasticsearch indexing, bull-board mount, the Next.js dashboard) was already
correct and needed no changes.

## 1. Scheduling more than one recipient at once crashed outright
`backend/src/services/emailService.ts`

The old code inserted every `ScheduledEmail` row with a placeholder
`jobId: ""`, then patched each row's `jobId` to its own `id` in a second
pass. `jobId` has a DB-level `@unique` constraint, and Postgres enforces
unique constraints immediately (not at commit) inside a transaction — so the
moment a campaign had **more than one recipient**, the second insert with the
same `""` placeholder violated the constraint and the whole request failed.
Since the entire point of this app is scheduling many (1000+) recipients at
once, this broke the core feature.

**Fix:** generate each row's `id` up front with `crypto.randomUUID()` and use
it as both `id` and `jobId` in the same insert, so every row is unique from
the start — no placeholder, no second pass.

## 2. The hourly rate limit stopped enforcing itself once reached
`backend/src/services/rateLimiter.ts`

The Lua script returned the current count unmodified when a sender was
already at/over its cap. But since sends are blocked *before* incrementing,
that returned value is always exactly equal to `limit` — indistinguishable
from a legitimately-allowed send that just hit `newVal === limit`. The
JS side's `result > limit` check was therefore never true, so once a sender
hit its cap, every subsequent send that hour was silently treated as
**allowed**, defeating the entire point of the per-hour rate limit.

**Fix:** the Lua script now returns the sentinel `-1` when blocked, which
can never come out of a real `INCR`, so the two cases are distinguishable.

## 3. Default "start time" in the compose form used UTC, not local time
`frontend/components/ComposeModal.tsx`

The default start time was built with `date.toISOString().slice(0, 16)`,
which produces UTC clock digits. The `datetime-local` input displays those
digits as if they were local time, so anyone outside UTC would see (and
silently schedule from) a wall-clock time that was actually offset from what
they intended.

**Fix:** added `toLocalDatetimeInputValue()`, which builds the string from
the `Date` object's local getters (`getFullYear`, `getHours`, etc.) instead
of `toISOString()`.

## Verified, not changed
- All 34 TypeScript/TSX files parse with zero syntax errors.
- Every local `import` across both `backend/` and `frontend/` resolves to a
  real export (checked programmatically, not just by eye).
- `docker-compose.yml`, both `.env.example` files, both `package.json`s, and
  the Prisma schema were all internally consistent with the code that
  reads/writes them.
- I could not run `npm install` in this sandbox (no network egress), so this
  wasn't validated with a live `tsc`/`next build` — only static analysis.
  Once you run `npm install` in each of `backend/` and `frontend/`, the
  Quick Start steps in the README should work as written.
