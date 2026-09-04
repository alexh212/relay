# Relay

A hosted webhook inspector: you create a throwaway `/hooks/{uuid}` URL, anything POSTed to it is stored in Postgres and pushed to the browser over a WebSocket, and you can replay any captured request to another URL with automatic retries on failure.

**Status:** prototype
**Live:** https://webhook-inspector-sx1y.onrender.com serves the frontend and returns 200. It's a static site only — the API it talks to is a separate Render service, and that URL doesn't appear anywhere in the repo; it's only recoverable from the compiled JS bundle.

## The problem

There are two real trust boundaries here, and the rest of the app is CRUD around them. Replay is a user-controlled outbound HTTP request from the server, which is a standard SSRF sink. And there are no user accounts, so every read has to be scoped by an unauthenticated, client-generated session string instead of a login — that scoping has to live in the query layer, not in a handler's `if` statement, or it isn't real. On top of that, the retry logic has to run identically from a live HTTP request and from a detached background worker, which is why it's factored into one shared core function called by both.

## How it works

**Ingress.** `POST /hooks/{endpoint_id}` (`app.py:168`, also GET/PUT/PATCH/DELETE, rate-limited 60/min per IP) takes no session header — anyone with the UUID can post. `flows.capture_webhook` loads the endpoint, rejects bodies over 1MB (checked on both the `content-length` header and the actual bytes), and, if the endpoint has a `secret` configured, verifies `x-webhook-signature` as `hex(HMAC_SHA256(secret, raw_body))` via `hmac.compare_digest` (`security.verify_hmac_signature`). It writes a `CapturedRequest` row and publishes a summary to Redis channel `endpoint:{id}`; the publish is wrapped in try/except so a Redis outage degrades the live feed but doesn't fail the capture.

**Live feed.** `WS /ws/endpoints/{endpoint_id}?session_id=…` (`app.py:235`) loads the endpoint and closes with 4004 if it's not found or 4001 if the session ID doesn't match, then subscribes to the Redis channel and forwards messages verbatim. `frontend/src/RequestFeed.tsx` prepends incoming messages and reconnects with exponential backoff capped at 30s.

**Session isolation.** There's no auth. `frontend/src/utils.ts:48` generates a `crypto.randomUUID()` on first load, stores it in `localStorage`, and sends it as `x-session-id` on every request. `app.py:70` just checks the header is present and 16+ characters — no lookup. The actual enforcement is in `store.py`: `list_endpoints` filters `WHERE session_id = :sid`, and `get_session_endpoint_or_404` / `assert_request_session_access` join back to `endpoints` on `(id, session_id)` and 404/403 on a miss. I checked this against the live API — reading a request UUID with a different `x-session-id` returns 403. It holds. What it's worth is a separate question: the session ID is a bearer token with no signature, no rotation, and no expiry.

**Replay.** `POST /api/requests/{id}/replay` (10/min) loads the request unscoped, validates the destination via `security.validate_destination_url` (resolves the hostname, rejects if any address falls in one of 24 blocked ranges — RFC1918, loopback, link-local/169.254, CGNAT, IPv4-mapped IPv6), *then* checks session access, then runs `flows._execute_replay`: strip hop-by-hop headers (`security.sanitize_headers`), fire an `httpx.AsyncClient(timeout=15.0)` request, persist a `DeliveryAttempt`. `flows._should_retry` retries on a transport error or any status ≥500, and `enqueue_retry` scores the job in the Redis sorted set `retry_queue` at `now + 5**attempt_number` (5s, 25s, 125s, 625s), up to 5 attempts.

**Worker.** `worker.py` is a 40-line loop: `zpopmin(retry_queue, count=10)`, re-`zadd` anything not yet due, otherwise call `flows.process_retry_job` (same `_execute_replay` core) and re-enqueue on failure. Sleeps 1s between passes.

**Deploy.** `render.yaml` defines one service: the FastAPI app, `alembic upgrade head && uvicorn app:app`, health check on `/health`. The frontend is a separate Render static site not in the blueprint. `worker.py` is not in the blueprint at all — see Known limitations.

## Setup

Needs Python 3.12, Node 20+, a running PostgreSQL 15+, and a running Redis 7 — the backend won't start without `DATABASE_URL` and the test suite hits a real database.

```bash
git clone https://github.com/alexh212/relay.git && cd relay
createdb webhookinspector

cd backend
python3.12 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # edit DATABASE_URL if your Postgres needs a user/password
alembic upgrade head
uvicorn app:app --reload          # :8000

# second terminal, same venv, from backend/
python worker.py                  # required for retries to actually process

# third terminal
cd frontend && npm install && npm run dev   # Vite on :5173
```

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Required, no default. Must resolve to an async driver — the code rewrites plain `postgresql://`/`psycopg` URLs to `postgresql+asyncpg://` and force-enables SSL for any host containing `neon.tech`. |
| `REDIS_URL` | Pub/sub for the live feed and the retry queue. Defaults to `redis://localhost:6379` — a typo'd value falls back silently instead of failing. |
| `ALLOWED_ORIGINS` | CORS origins, comma-separated. Defaults to the two local Vite ports. Must be set to the real frontend origin in production. |
| `DEBUG` | Optional. Turns on SQLAlchemy statement logging, which logs captured webhook bodies. Off in the Render deploy. |
| `VITE_API_URL` | Frontend build-time only, baked into the bundle. Defaults to `localhost:8000`. No `.env.example` exists for the frontend. |

## Tests

```bash
cd backend
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost/webhookinspector_test REDIS_URL=redis://localhost:6379 python -m pytest tests/ -v
```

Requires a live Postgres — `conftest.py` builds a real engine and only Redis is mocked. This is the command CI actually runs. A frontend vitest suite exists (`frontend/src/utils.test.ts`) but nothing in CI invokes it, and eslint is configured but never run either.

## Known limitations

- **The retry worker isn't deployed.** `render.yaml` runs only `uvicorn app:app`. Nothing starts `worker.py` in production, so every failed or 5xx replay on the live site writes into the Redis `retry_queue` and nothing ever consumes it. The queue is write-only in prod and grows.
- **Replay forwards Authorization and Cookie headers to whatever URL you type.** `security.sanitize_headers` only strips hop-by-hop headers; `test_app.py:259` explicitly asserts `Authorization: Bearer token` survives a replay. Replay a real signed webhook to a URL you don't control and you've handed over that provider's credentials.
- **The retry queue drops jobs on crash.** `zpopmin` then re-`zadd` has no acknowledgement or in-flight set — a worker killed between pop and re-add (or completed replay) silently loses the job. Two workers would also duplicate work on the same jobs.
- **The public demo shares one session across every visitor.** `Demo.tsx:4` hardcodes a single `DEMO_SESSION` string for the whole landing page, so every browser that loads it can read every other visitor's demo endpoints and payloads. It also creates a fresh endpoint per mount and fires a synthetic webhook every 3.5s; cleanup is a best-effort DELETE on unmount that doesn't run if the tab is just closed, so abandoned tabs leave orphaned endpoints and rows behind permanently.
- **No data retention.** No TTL, no pruning job, no size cap on `captured_requests` or `delivery_attempts`; full response bodies are stored on every replay attempt. This grows until a free-tier database fills up.
- **The session ID has no security properties.** Any client-supplied string 16+ characters long is accepted — no signature, no server record, no expiry. Anyone who learns a session ID has full read/delete on it. Clearing `localStorage` permanently orphans that session's data with no recovery.
- **Idle WebSockets leak Redis connections.** A client disconnect is only noticed when the next `send_text` fails, which only happens when a new webhook arrives on that channel — an endpoint whose viewer closed the tab keeps its pubsub subscription open indefinitely. No ping/keepalive, no connection cap.
- **SSRF check has a TOCTOU window.** `validate_destination_url` resolves the hostname once; `httpx` resolves it again independently for the actual request. A DNS record that changes in between (rebinding) bypasses the blocklist. Redirect-based bypass happens to be closed because `httpx` doesn't follow redirects by default — that's incidental, not enforced.
- **`/health` fails the whole API on a Redis blip**, even though capture itself degrades gracefully when the pub/sub publish fails — `render.yaml` points its health check straight at `/health`.
- **`status_code` and `duration_ms` are stored as strings**, not integers, so `_should_retry` has to `int()`-cast a column and there's no way to aggregate attempt latency in SQL.
- **A malformed `Content-Length` header 500s instead of 400s** — `flows.capture_webhook` does an unguarded `int()` on it.
- **Rate limiting is in-memory and resets on every deploy.** Only endpoint creation, capture, and replay are limited; reads and the WebSocket are not, and the UI polls `/attempts` every 5 seconds per selected request.
- **The HMAC happy path is untested.** There's a test for a rejected signature, none for an accepted one — the branch that would catch an encoding bug is uncovered.
- **`frontend/dist/index.html` is a stale, non-functional artifact** force-committed past `.gitignore`; the JS bundle it references isn't in the repo and doesn't match what the live site serves.
- **`requirements.txt` isn't actually pinned.** `slowapi` is `>=0.1.9` with its transitive deps absent, so installs aren't reproducible; CI then reinstalls `pytest`/`httpx` unpinned on top.
- No LICENSE file.

## What I'd build next

- Add the worker as a second Render service (`type: worker`, `startCommand: python worker.py`) so the retry path that's already written and tested actually runs in production. This is the biggest gap between the code and the deployment.
- Strip `Authorization`/`Cookie`/`x-api-key` from replay by default, with an explicit opt-in checkbox or header allowlist in the UI, instead of forwarding credentials by default.
- Add retention: a TTL or periodic delete on `captured_requests` older than N hours, plus a per-endpoint row cap. The public demo is actively filling the database right now.
- Make the retry queue crash-safe — an in-flight set removed only after the attempt persists, or a Redis Stream with a consumer group instead of a sorted set. Add an `attempt_number` column to `delivery_attempts` so history shows which retry produced which result.
