# Relay

Relay gives you a URL for capturing webhooks and a browser inspector for viewing and replaying them. Captured requests are stored in PostgreSQL and appear in the browser through Redis Pub/Sub and WebSockets.

**Status: prototype. Use synthetic data in the hosted demo.** The landing-page demo shares a session across visitors and generates sample traffic automatically. Replay can forward sensitive headers, including `Authorization` and `Cookie`, to the destination you choose. Do not send production credentials or customer payloads.

[Demo](https://webhook-inspector-sx1y.onrender.com/) · [API](https://webhook-inspector-api.onrender.com/) · [Health](https://webhook-inspector-api.onrender.com/health)

The deployment hostnames still use the project's former name, `webhook-inspector`. This repository is the canonical source.

## How it works

**Capture.** Create an endpoint and send a request to `/hooks/{endpoint_id}`. The backend saves its headers and body, then publishes an update through Redis. Capture accepts bodies up to 1 MB. Endpoints with a configured secret require an HMAC-SHA256 signature in `x-webhook-signature`; other endpoints can receive requests from anyone with their URL.

**Live inspection.** The browser subscribes to the endpoint's WebSocket feed and displays new captures without a refresh. If Redis publication fails after the database save, the capture can still succeed without a live update.

**Sessions.** The browser generates a session ID, stores it in `localStorage`, and sends it with requests. Database queries and WebSocket access are scoped to that ID. This is bearer-token access control, not user-account authentication: anyone with the session ID can access its data. The public landing-page demo uses one shared session.

**Replay and retries.** A captured request can be sent to a destination URL after session-access and destination checks. The backend records the delivery attempt. Transport errors and server-error responses can be queued in Redis for retry. A separate `worker.py` process consumes that queue; its live deployment remains unverified.

The backend is FastAPI with SQLAlchemy. The frontend uses React and Vite. See [backend/](backend/) for routes, storage, replay, and worker code, and [frontend/src/](frontend/src/) for the inspector.

## Run locally

Requirements: Python 3.12, Node.js 20 or later, PostgreSQL 15 or later, and Redis 7. Start PostgreSQL and Redis before running the backend.

```bash
git clone https://github.com/alexh212/relay.git
cd relay
createdb webhookinspector
```

From the repository root:

```bash
cd backend
python3.12 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# Set DATABASE_URL for your local database and check the Redis/CORS settings.
alembic upgrade head
uvicorn app:app --reload
```

For retry processing, open another terminal at the repository root:

```bash
cd backend
source venv/bin/activate
python worker.py
```

For the frontend, open another terminal at the repository root:

```bash
cd frontend
npm install
npm run dev
```

The API runs on port 8000; Vite normally uses port 5173.

| Variable | Purpose |
| --- | --- |
| `DATABASE_URL` | Required PostgreSQL connection. The backend uses the asyncpg driver and normalizes supported PostgreSQL URL forms. Neon hosts are configured with SSL. |
| `REDIS_URL` | Live-feed publication and retry queue. Defaults to `redis://localhost:6379` only when absent, not when an invalid value is supplied. |
| `ALLOWED_ORIGINS` | Comma-separated CORS origins. Include the deployed frontend origin when hosting the app. |
| `DEBUG` | SQL logging, which can include captured payloads. Keep it off when handling sensitive data. |
| `VITE_API_URL` | Frontend API base URL, set at build time. Defaults to the local API on port 8000. |

The backend example settings are in [backend/.env.example](backend/.env.example). The frontend has no example environment file; configure `VITE_API_URL` in its environment when using a different backend.

## Tests

Use a dedicated, disposable local test database, not your development or hosted database. From the repository root, with the backend dependencies already installed:

```bash
createdb -h localhost -U postgres webhookinspector_test
cd backend
source venv/bin/activate
(
  export DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost/webhookinspector_test
  export REDIS_URL=redis://localhost:6379
  alembic upgrade head && python -m pytest tests/ -v
)
```

Adjust the database-creation command and test database credentials for your local PostgreSQL installation. The test fixtures do not create the schema, so a fresh database requires migrations before pytest, as in CI. The subshell supplies the same explicit local database URL to both commands without changing your shell's configuration afterward. Backend tests use a real database, mock the application Redis client, and mock outbound replay requests. Worker tests mock their database/session dependencies. CI runs the backend tests; the frontend utility tests and lint checks are not included in that workflow. The accepted-signature HMAC path does not yet have a test.

## Deployment

[render.yaml](render.yaml) defines the API service and runs migrations before startup. The static frontend is configured separately. The blueprint does not define a retry worker, but it also cannot establish whether one was configured elsewhere.

`/health` checks PostgreSQL and Redis. A failed check can return 503 even when a route that does not require the failed dependency still works. Health checks do not establish end-to-end delivery or retry guarantees.

## Limits to keep in mind

- **Data handling:** replay currently preserves sensitive headers. Stored requests and delivery responses have no automatic expiry or retention policy. The landing-page demo creates sample records while open; closing a tab does not guarantee their cleanup.
- **Access control:** session IDs have no built-in rotation, expiry, or recovery mechanism. There are no user accounts. The public sample session is deliberately shared.
- **Outbound requests:** destination validation blocks several private and special-use address ranges, but DNS is resolved separately for validation and the outgoing request. That leaves a DNS-rebinding risk. Replay should be limited to destinations you control.
- **Retries:** worker operation must be verified separately. The queue removes jobs before processing and has no acknowledgement/recovery step, so a worker crash can lose a job.
- **Connections and limits:** idle WebSocket disconnects may leave Redis subscriptions open until another event is sent. Rate limits are in-memory, reset on restart, and do not cover reads or WebSockets.
- **Validation and storage:** malformed `Content-Length` values can cause a server error. Status codes and durations are stored as strings, requiring numeric conversion for comparisons and aggregation.
- **Packaging:** the committed `frontend/dist/index.html` is stale; build the frontend from source. Some dependencies are unpinned, and the repository has no license file.

Next priorities are safer replay-header defaults, data retention, and recoverable retry processing. Automatic retries should only be advertised for a deployment after confirming that its worker is running and testing delivery.
