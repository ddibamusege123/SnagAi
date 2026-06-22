# Snag Ai

An AI agent workspace with real accounts, persistent history, streaming
replies, a working task scheduler, file attachments (images, PDFs, text),
voice input, Stripe billing, structured logging, and Redis-backed rate
limiting — backed by a stateless Express API that proxies Claude.

## Run it locally

```bash
# Terminal 1 — backend
cd backend
cp .env.example .env   # fill in ANTHROPIC_API_KEY and JWT_SECRET at minimum
npm install
npm run dev             # http://localhost:4000

# Terminal 2 — frontend
cd frontend
npm install
npm run dev             # http://localhost:5173
```

Open `http://localhost:5173`, sign up with any email/password (8+ chars),
and you're in. The Vite dev server proxies `/api/*` and `/healthz` to the
backend, so it never needs to know the backend's real address in development
— and streaming responses pass through that proxy cleanly (verified: real
SSE headers and chunked transfer arrive intact on the other side).

Or run both with Docker: `docker compose up --build`.

**Node 22.5+ required** — the backend uses Node's built-in `node:sqlite`
module (still experimental, hence the warning on boot) instead of a
native-compiled dependency, specifically so the database "just works"
without a build toolchain. If you're on an older Node, either upgrade or
swap `src/db.ts` for a real client like `pg` — every other file talks to
the data layer through `src/data/*.ts`, so that swap doesn't touch route
code. Both Dockerfiles are already pinned to `node:22-alpine`.

## What's real right now

- **Accounts.** Email/password signup and login, bcrypt-hashed passwords,
  JWT sessions — no server-side session storage, keeping the backend
  stateless.
- **Persistence.** Conversation history, tasks, and catches live in SQLite
  and survive a restart. The chat route loads history from the database
  itself; the client only ever sends the latest turn.
- **Streaming replies.** `/api/chat` streams over SSE — the assistant
  bubble grows token-by-token instead of appearing all at once
  (`backend/src/lib/anthropic.ts`'s `streamSnagReply`,
  `frontend/src/lib/api.ts`'s `streamChat`). If the connection drops
  mid-reply, whatever already streamed in stays visible with a note rather
  than vanishing.
- **A real scheduler.** `node-cron` jobs run every active task in the
  background; creating, pausing, resuming, or deleting a task live-updates
  its job, no restart needed.
- **Attachments.** Images, PDFs, and plain text/markdown files (15MB cap).
  Images become real vision content blocks; PDFs use Claude's native
  document support; text files get folded into the message text. Other
  document types (.docx, etc.) aren't parsed yet.
- **Voice input.** Browser Web Speech API, no backend involved. Tells you
  plainly if your browser doesn't support it.
- **Billing.** Stripe Checkout + webhook, free tier capped at 20
  catches/day, enforced server-side. Needs your own Stripe test-mode keys
  to actually process a payment — see below.
- **Structured logging.** Every log line is JSON (via `pino`), tagged with
  this instance's region, so multi-region log aggregation can filter by
  it. Request logging captures method, full path, status, duration, and
  the authenticated user.
- **Distributed rate limiting.** Set `REDIS_URL` and the limiter switches
  from per-instance in-memory counting to a shared Redis-backed window —
  the same limit holds regardless of which instance answers a request. If
  Redis is unset, or briefly unreachable, it transparently falls back to
  in-memory and logs a warning rather than failing requests.

**To actually test billing**, you need your own Stripe account in test
mode: a secret key, a price (Stripe Dashboard → Products), and a webhook
endpoint pointed at `/api/billing/webhook` with its signing secret —
`stripe listen --forward-to localhost:4000/api/billing/webhook` is the
fastest way to get that locally. Without those three env vars set, billing
routes return a clear "not set up" error instead of crashing.

**To actually test Redis rate limiting**, run a local Redis
(`docker run -p 6379:6379 redis`) and set `REDIS_URL=redis://localhost:6379`.
Without it, everything still works — just per-instance instead of shared.

## Why it's structured this way

Same core decision as before: **the backend is stateless app logic in
front of a shared database.** Any instance can answer any request as long
as it can reach the same DB — true whether that's the local SQLite file
here or a replicated Postgres cluster in production. JWTs extend that same
property to auth: no session table to keep in sync across regions.

**Tasks, catches, and uploads go through a small repository layer**
(`backend/src/data/`). The SQL lives there and nowhere else, so swapping
SQLite for a production database later is a contained change.

**Streaming is additive, not a replacement.** The scheduler and manual
task runs still use the non-streaming `getSnagReply` — nothing is watching
those in real time, so there's no reason to pay for the complexity of a
stream there. Only the interactive chat route streams.

## Deploying this globally

The shape from before still holds — CDN for the static frontend, geo-routed
traffic to regional backend instances, one shared data layer, now joined by
a shared Redis for rate limiting and a centralized destination for the
structured logs every instance already emits. SQLite doesn't replicate
across regions on its own; for production, point `backend/src/db.ts` at a
replicated Postgres (Aurora Global Database, CockroachDB) instead, and
swap uploaded-file storage from local disk to S3-compatible object storage
for the same reason — both currently assume a single machine's disk.

## What's deliberately not here yet

- **Non-image, non-PDF file uploads.** Word docs, spreadsheets, etc.
  aren't parsed.
- **Multi-conversation history.** There's one ongoing thread per user, not
  a list of separate conversations to switch between.
- **Centralized log shipping.** Every instance logs structured JSON to
  stdout; nothing yet ships those logs to a central place across regions
  (that's normally a platform-level concern — e.g. your container
  orchestrator's log driver — not application code).
- **Subscription edge cases.** The webhook handles the common paths
  (checkout completed, subscription updated/canceled) but not every Stripe
  event type (e.g. payment retries, proration).
