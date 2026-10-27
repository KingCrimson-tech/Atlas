# Buggy — Full Project Report

> A self-hosted miniature Sentry: a Python SDK that reports exceptions, a FastAPI backend that groups duplicates, SQLite for storage, and a zero-build dashboard for triage.

## Contents

1. [What Buggy is, in plain language](#1-what-buggy-is-in-plain-language)
2. [Who uses it and what they see](#2-who-uses-it-and-what-they-see)
3. [System architecture](#3-system-architecture)
4. [The core idea: events, groups, and fingerprints](#4-the-core-idea-events-groups-and-fingerprints)
5. [End-to-end walkthrough: one crash, from request to dashboard](#5-end-to-end-walkthrough-one-crash-from-request-to-dashboard)
6. [Data layer in detail](#6-data-layer-in-detail)
7. [API surface](#7-api-surface)
8. [SDK internals](#8-sdk-internals)
9. [Dashboard internals](#9-dashboard-internals)
10. [Demo application](#10-demo-application)
11. [Packaging and operations](#11-packaging-and-operations)
12. [Verification performed](#12-verification-performed)
13. [Security, privacy, and limitations](#13-security-privacy-and-limitations)
14. [Design decisions and why they fit](#14-design-decisions-and-why-they-fit)
15. [Future work](#15-future-work)
16. [File-by-file map](#16-file-by-file-map)
17. [Glossary](#17-glossary)
18. [Bottom line](#18-bottom-line)

---

## 1. What Buggy is, in plain language

Every application fails sometimes. A checkout endpoint rejects a card, a background worker cannot reach an email server, a frontend crashes because a field is missing. When that happens, the developer needs to know:

- **What** went wrong (the message and stack trace)
- **How often** it happens
- **Where** it happens (which service, route, user, request)
- **When** it started and when it was last seen
- **Whether** two crashes are the same underlying bug or different ones

Without a tool, this information is scattered across console logs and disappears when a container restarts. A naive solution — storing every crash as a separate row — makes the problem worse: a bug hit 10,000 times fills the screen with 10,000 identical entries.

Buggy solves that by separating **incidents** from **occurrences**:

- One row per distinct error pattern ("error group"), with a counter and first/last seen timestamps.
- One row per actual crash ("event"), with its own stack trace, route, request ID, and user ID.

The grouping key is a **fingerprint**: a short hash computed from the service name, a normalized error message, and the first few stack frames. Number 101 and number 202 in "Payment failed for order 101/202" normalize to the same text, so both crashes land in one group with `count = 2`, while the raw events preserve the exact messages and stacks.

Buggy is deliberately small and self-contained. There is no cloud account, no external database, no JavaScript build pipeline, and no Prometheus/Grafana/OTel stack. It runs with one Docker Compose command.

---

## 2. Who uses it and what they see

### The developer integrating it

Three lines are enough:

```python
from sdk.buggy import init, capture_exception

init(dsn="http://localhost:8000/api/events", service="demo-api")
capture_exception(exc, route="POST /checkout")
```

Optionally they can wrap a function with `@handle_exception(...)` or call `install_excepthook()` to report uncaught crashes. The SDK never raises: if Buggy is down, the host application keeps working.

### The person triaging errors

They open <http://localhost:8000> and see:

1. **Three counters** — error groups, total events, events today.
2. **A search box and service filter** — narrow down by message text or service name.
3. **A table** — count, message, service, last seen, newest first.
4. **A detail drawer** — click a row to see the group's metadata, stack trace, and the latest raw events. Click an event to view its specific stack.
5. **A bar chart** — activity over the last seven days.

The API behind the dashboard is also usable directly, and FastAPI exposes interactive documentation at `/docs`.

---

## 3. System architecture

```
┌──────────────────────┐     POST /api/events      ┌─────────────────────────┐
│ Demo app :8001       │ ────────────────────────► │ Buggy API :8000         │
│ FastAPI + SDK        │                           │ FastAPI                 │
│                      │ ◄──── 202 fingerprint ─── │                         │
└──────────────────────┘                           └───────────┬─────────────┘
                                                               │ sqlite3
                                                   ┌───────────▼─────────────┐
                                                   │ SQLite: buggy.db        │
                                                   │ errors + events         │
                                                   └───────────┬─────────────┘
                                                               │ GET /api/*
┌──────────────────────┐          GET /            ┌───────────▼─────────────┐
│ Browser              │ ◄──────────────────────── │ Static dashboard        │
│ public/index.html    │                           │ public/app.js           │
└──────────────────────┘                           └─────────────────────────┘
```

| Component | File(s) | Technology | Responsibility |
|---|---|---|---|
| SDK | `sdk/buggy.py` | `requests` + stdlib | Build and POST error payloads; never raise |
| Ingest/query API | `app/main.py` | FastAPI | HTTP routes, validation, static mount |
| Persistence/grouping | `app/db.py` | stdlib `sqlite3` | Schema, fingerprint, upsert, queries |
| Request/response models | `app/models.py` | Pydantic v2 | Payload validation and typed responses |
| Dashboard | `public/index.html`, `public/app.js` | Vanilla JS, Chart.js CDN | Stats, search, table, detail drawer, chart |
| Demo application | `demo/app.py` | FastAPI + SDK | Generates real exceptions on demand |
| Packaging | `Dockerfile`, `docker-compose.yml` | Docker Compose | One image, two services, persistent volume |

Ports:

| Port | Service | Contents |
|---|---|---|
| 8000 | `buggy` | Dashboard, `/api/*`, `/health`, `/docs` |
| 8001 | `demo` | Demo application that reports its own errors |

Inside Compose, the two containers share a private network. The demo reaches the API at `http://buggy:8000/api/events`; from the host, both are published on `localhost`.

---

## 4. The core idea: events, groups, and fingerprints

### 4.1 Two tables, two jobs

- `errors` answers: **"What distinct problems exist and how bad are they?"**
  It holds one row per fingerprint with `count`, `first_seen`, `last_seen`, `service`, and the first message seen.
- `events` answers: **"What exactly happened each time?"**
  It holds one row per crash with the full stack, route, request ID, user ID, and timestamp.

This split is what makes the dashboard fast and readable. Listing 50 groups is cheap even when there are 500,000 events behind them. Opening one group fetches only that group's latest events via the `events.fingerprint` index.

### 4.2 How the fingerprint is computed

Implemented in `app/db.py`:

1. **Normalize the message**
   - replace runs of digits with `*`
   - replace `'...'` with `'*'`
   - replace `"..."` with `"*"`
   - collapse all whitespace runs to a single space
   - truncate to 500 characters
   - **case is preserved** (no lowercasing)
2. **Extract stack frames**
   - keep lines containing `File "`
   - keep the first three only
   - replace `, line 123` with `, line *`
   - join them with `|`
3. **Hash**
   - `sha1(f"{service}|{normalized_message}|{frames}")`
   - keep the first 16 hex characters

Worked example:

```
service:  checkout
message:  Payment failed for order 101
stack:    File "/srv/checkout/service.py", line 88, in charge
          File "/srv/checkout/gateway.py", line 41, in charge
          ...
```

becomes:

```
normalized: Payment failed for order *
frames:     File "/srv/checkout/service.py", line *, in charge|File "/srv/checkout/gateway.py", line *, in charge
raw:        checkout|Payment failed for order *|File "/srv/checkout/service.py", line *, in charge|File "/srv/checkout/gateway.py", line *, in charge
sha1:       993d8f6d4dce0abf7883cdaad459277b0a730efb
fingerprint: 993d8f6d4dce0abf
```

The same error for order 202, and the same stack with a different line number, produces **the same** `993d8f6d4dce0abf`. Changing the service to `payments` produces `1b3bb01235141ef6`; changing the function from `charge` to `refund` produces `d866503b4460fd48`. This is exactly the intended behavior: variable data (order numbers, line numbers) does not split a group, but a different service or call site does.

### 4.3 Why these choices

| Choice | Reason | Tradeoff |
|---|---|---|
| Include `service` | Two services can emit identical messages; they are different bugs | Renaming a service creates new groups |
| First 3 frames | Enough to distinguish call paths, short enough to stay stable | Deep-frame differences are invisible; moving code changes the group |
| Mask line numbers | Line numbers shift with unrelated edits | Genuinely different lines at the same call site are merged |
| Mask digits and quoted values | IDs, order numbers, ports, IPs are variable | Messages that differ only by a meaningful number are merged |
| Preserve case | "Timeout" and "timeout" can be different sources; avoids surprise merging | Case-only changes split a group |
| SHA-1 truncated to 64 bits | Short, fixed-size key; collision probability is negligible at this scale | Not cryptographic; collisions are theoretically possible |
| Store the first message | The group needs a readable title | The title can show an old ID (e.g., order 1) even when later events say order 202 |

### 4.4 Time handling

- All server timestamps are UTC ISO-8601 with a trailing `Z`.
- `errors.first_seen` is set on the first insert; `errors.last_seen` is updated on every occurrence.
- `events.timestamp` uses the SDK-provided timestamp when present, otherwise server time.
- "Events today" compares the first 10 characters of the event timestamp with today's UTC date. This is a string comparison that works because the format is fixed-width ISO-8601.

---

## 5. End-to-end walkthrough: one crash, from request to dashboard

Take `GET http://localhost:8001/user-fail/1`.

### Step 1 — the demo app raises

`demo/app.py` routes `/user-fail/{user_id}` and deliberately raises:

```python
raise ValueError(f"Payment failed for order {user_id}")
```

FastAPI turns that into a 500. Before the response is sent, the app-level exception handler runs:

```python
@app.exception_handler(Exception)
async def report_exception(request, exc):
    capture_exception(exc, route=f"{request.method} {request.url.path}")
    return JSONResponse(status_code=500, content={"ok": False, ...})
```

The caller still receives a 500 — reporting does not hide the failure — but the error is now on its way to Buggy.

### Step 2 — the SDK builds and sends a payload

`capture_exception` uses the config set at startup by `init()`. It builds:

```json
{
  "service": "demo-api",
  "message": "Payment failed for order 1",
  "stack": "Traceback (most recent call last):\n  File \"/app/demo/app.py\", line 42, ...",
  "route": "GET /user-fail/1",
  "request_id": "",
  "user_id": "",
  "environment": "dev",
  "timestamp": "2026-10-25T08:10:29.504101Z"
}
```

The message falls back to the exception class name if `str(exc)` is empty. The stack is produced by `traceback.format_exception`. The SDK posts with a 2-second timeout, checks the status, reads `fingerprint` from the JSON response, and returns it.

If anything fails — DNS, timeout, 500, malformed JSON — the broad `except Exception` logs at debug level and returns `None`. The host application is unaffected.

### Step 3 — FastAPI validates

`EventIn` requires non-empty `service` and `message`. Everything else has defaults. Bad input (missing message, wrong type, empty string) becomes a **422** before any database code runs. This is why a broken SDK payload cannot crash the ingest endpoint with a 500.

### Step 4 — the database groups it

`record_event` computes the fingerprint and executes two statements in one transaction:

```sql
INSERT INTO errors(fingerprint, service, message, first_seen, last_seen, count)
VALUES (?, ?, ?, ?, ?, 1)
ON CONFLICT(fingerprint) DO UPDATE SET
  count = count + 1,
  last_seen = excluded.last_seen;

INSERT INTO events(fingerprint, service, route, stack, request_id, user_id, timestamp)
VALUES (?, ?, ?, ?, ?, ?, ?);
```

The first insert creates a group or increments the existing one. The second always records the raw occurrence. The connection is then queried for the new count and the transaction commits.

For `/user-fail/1` and `/user-fail/2`, the normalized message is identical (`Payment failed for order *`) and the stack frame (same file, same function, line number masked) is identical, so the second request only increments the count.

### Step 5 — the API responds

```json
{"fingerprint": "37dc66b514fdde28", "count": 2}
```

with HTTP **202 Accepted**.

### Step 6 — the dashboard reads it

The browser loads `/` and `/app.js`. On boot, `app.js` runs two parallel fetches:

- `/api/errors?limit=500` → stats-independent group list for the chart, service options, and initial table
- `/api/stats` → the three counters

It renders the counters, the table, and the chart. Clicking a row fetches `/api/errors/{fingerprint}?events_limit=20`, then fills the drawer: message, service, count, first/last seen, the newest event's stack, and a list of events. Clicking another event swaps the `<pre>` content to that event's stack.

Search input is debounced by 250 ms and refetches `/api/errors?search=...&service=...&limit=50`; changing the service filter refetches immediately.

---

## 6. Data layer in detail

### 6.1 Schema

```sql
CREATE TABLE IF NOT EXISTS errors(
  fingerprint TEXT PRIMARY KEY,
  service TEXT NOT NULL, message TEXT NOT NULL,
  first_seen TEXT NOT NULL, last_seen TEXT NOT NULL,
  count INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE IF NOT EXISTS events(
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  fingerprint TEXT NOT NULL,
  service TEXT NOT NULL, route TEXT DEFAULT '',
  stack TEXT DEFAULT '', request_id TEXT DEFAULT '',
  user_id TEXT DEFAULT '', timestamp TEXT NOT NULL,
  FOREIGN KEY (fingerprint) REFERENCES errors(fingerprint)
);

CREATE INDEX IF NOT EXISTS idx_events_fp ON events(fingerprint);
CREATE INDEX IF NOT EXISTS idx_errors_last ON errors(last_seen);
```

Notes:

- `fingerprint` is the natural primary key of a group; no surrogate ID is needed.
- `events.fingerprint` is a foreign key to `errors.fingerprint`; connections enable `PRAGMA foreign_keys = ON`, and ingestion always writes the group before its event.
- `idx_events_fp` makes "latest events for this group" cheap.
- `idx_errors_last` makes "newest groups first" cheap.
- SQLite is opened with a 10-second timeout and WAL mode, which allows dashboard reads during ingest writes. Connections are short-lived and created per operation.

### 6.2 Ingest transaction

The upsert and event insert run inside a `with get_conn() as conn:` block, so both commit or neither does. If two requests race for the same new fingerprint, SQLite serializes the writes and the `ON CONFLICT` path increments correctly.

### 6.3 Query construction

`list_errors` appends `WHERE` clauses only for non-empty filters and always parameterizes user input:

```python
clauses.append("message LIKE ?")
params.append(f"%{search}%")
```

`ORDER BY last_seen DESC LIMIT ?` is applied last. `limit` is clamped by FastAPI's `Query(50, ge=1, le=500)`. Detail events use `ORDER BY id DESC LIMIT ?`.

### 6.4 Stats

```sql
SELECT count(*) FROM errors;                                       -- groups
SELECT count(*) FROM events;                                       -- all events
SELECT count(*) FROM events WHERE substr(timestamp,1,10) = ?       -- today
```

All three run in one connection. "Today" is the current UTC date, not the server's local date.

---

## 7. API surface

| Method | Path | Parameters | Success | Failure |
|---|---|---|---|---|
| `POST` | `/api/events` | JSON `EventIn` | `202 {fingerprint, count}` | `422` for invalid payload |
| `GET` | `/api/errors` | `search`, `service`, `limit` (1–500) | `200 [ErrorItem]` | `422` for bad limit |
| `GET` | `/api/errors/{fingerprint}` | `events_limit` (1–500) | `200 {error, events}` | `404` unknown fingerprint |
| `GET` | `/api/stats` | — | `200 {total_groups, total_events, events_today}` | — |
| `GET` | `/health` | — | `200 {ok: true}` | — |
| `GET` | `/` and static files | — | Dashboard | `404` for unknown files |
| `GET` | `/docs`, `/openapi.json` | — | Swagger UI / schema | — |

Request model (`app/models.py`):

```python
class EventIn(BaseModel):
    service: str = Field(min_length=1)
    message: str = Field(min_length=1)
    stack: str = ""
    route: str = ""
    request_id: str = ""
    user_id: str = ""
    environment: str = "dev"
    timestamp: str | None = None
```

Response models (`EventOut`, `ErrorItem`, `EventItem`, `ErrorDetail`, `StatsOut`) give FastAPI typed serialization and useful `/docs`.

Route ordering matters: all `/api/*`, `/health`, and `/docs` routes are registered **before** `app.mount("/", StaticFiles(...))`. Starlette checks routes in order, so the static mount is a fallback, not an override.

`environment` is accepted and sent by the SDK but is not stored: the fixed schema has no column for it. This is a deliberate schema/spec tradeoff, not an accident; adding the column is listed as future work.

---

## 8. SDK internals

`sdk/buggy.py` is under 100 lines and has no FastAPI dependency.

- **Configuration** lives in a module-level dict: `dsn`, `service`, `environment`. `init()` strips a trailing slash from the DSN so `.../api/events/` and `.../api/events` behave the same. One configuration per process.
- **`_payload`** converts an exception into the API body. It uses `str(exc)` or the class name, formats the full traceback, adds request metadata, environment, and a timezone-aware UTC timestamp.
- **`capture_exception`** returns the fingerprint on success. On *any* failure it logs a debug message with the route and traceback, then returns `None`. The broad catch is intentional: telemetry must never become a second outage. There are no retries, queues, or batching, so a down server loses that report.
- **`handle_exception(route="")`** is a decorator that captures and **re-raises**. The caller's normal error handling still runs.
- **`install_excepthook()`** wraps `sys.excepthook`, reports the exception, then calls the original hook so the terminal traceback is not lost.

Because the SDK uses only `requests` and the stdlib, it can be copied into a script, worker, CLI, or any framework. It is synchronous; in an async application `capture_exception` blocks for up to roughly the 2-second timeout when Buggy is unavailable.

---

## 9. Dashboard internals

The dashboard is plain HTML/CSS/JS with Chart.js from jsDelivr. There is no bundler and no framework.

### Boot and state

```js
loadOverview(); // /api/errors?limit=500 + /api/stats → cards, services, chart
loadTable();    // /api/errors?limit=50 → table
```

`state` holds only the selected fingerprint and the Chart instance. Service options are derived from the 500-group list and preserved across filtering. The table uses `<tr tabindex="0" role="button">` and handles both click and Enter/Space.

### Detail drawer

A row click fetches the group with `events_limit=20`. The drawer shows group metadata, renders the newest event's stack, and lists events. Each event button replaces the stack. Server data is inserted with `textContent`/`createElement`, never `innerHTML`, so an error message containing HTML cannot execute.

### Chart

It builds the last seven UTC dates, zero-fills them, sums each group's `count` into the bucket for the group's `last_seen` date, and renders a bar chart. This is an approximation computed from `/api/errors` as specified: it is not a historical per-day event query, so a group seen today contributes all of its lifetime count to today's bar. The README's "Errors per day" label is best read as "group activity attributed to last-seen day".

### Accessibility and resilience

- Search and service controls have `aria-label`s; the table has headers; rows expose an accessible label.
- Toolbar controls have explicit focus outlines; table rows are keyboard-activatable and use the browser's default focus indicator; the drawer has `aria-live="polite"`.
- An inline status banner appears if the API cannot be reached; the page does not silently show stale/empty data.
- The chart canvas has no text alternative, and the color contrast of muted text is modest — those are known improvements, not implemented features.
- Chart.js is loaded from a CDN, so the chart requires network access even though the rest of the dashboard is local.

---

## 10. Demo application

`demo/app.py` exists to prove the system end to end:

- `GET /ok` returns `{"ok": true}`.
- `GET /fail?msg=...` raises a `RuntimeError`.
- `GET /user-fail/{id}` raises `ValueError(f"Payment failed for order {id}")`.
- The global `Exception` handler reports to the SDK and returns a JSON 500.

The DSN comes from `BUGGY_DSN` (default `http://localhost:8000/api/events`), which lets the same file run locally and inside Compose. The intentional 500s are the demonstration: the caller sees a real failure, while Buggy receives the message, route, and traceback. Hitting `/user-fail/1` and `/user-fail/2` produces **one group with count 2**, which is the visible proof that fingerprint normalization works.

The demo does not report `HTTPException`s (FastAPI has a separate handler for those); only unexpected errors reach the reporter.

---

## 11. Packaging and operations

### Dockerfile

One image for both services:

- `python:3.12-slim`
- installs `requirements.txt` (`fastapi`, `uvicorn[standard]`, `requests`, `pydantic>=2,<3`)
- copies `app/`, `sdk/`, `demo/`, `public/`
- default command: `uvicorn app.main:app --host 0.0.0.0 --port 8000`

### Compose

- `buggy`: publishes 8000, sets `BUGGY_DB=/data/buggy.db`, mounts the `buggy_data` volume at `/data`, and has a healthcheck that calls `/health`.
- `demo`: publishes 8001, overrides the command to run `demo.app:app`, sets `BUGGY_DSN=http://buggy:8000/api/events`, and waits for `buggy` to become healthy.
- `depends_on: condition: service_healthy` prevents the demo from starting before the API can accept writes.
- The named volume means `docker compose down` keeps data; `docker compose down -v` deletes it.

### Configuration

| Variable | Default | Effect |
|---|---|---|
| `BUGGY_DB` | `buggy.db` | SQLite file path |
| `BUGGY_DSN` | `http://localhost:8000/api/events` | Demo app's report target |

### Local development

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/uvicorn app.main:app --port 8000
.venv/bin/uvicorn demo.app:app --port 8001
```

The local venv used during development ran Python 3.14 with FastAPI 0.162.0, Uvicorn 0.54.0, and Pydantic 2.14.1. The container pins Python 3.12. The code uses `str | None` and built-in generics, so Python 3.10+ is required.

---

## 12. Verification performed

The project was built phase by phase and checked at each step.

| Check | Result |
|---|---|
| `GET /health`, `/docs` | 200; Swagger UI available |
| Same message with different numbers | Same fingerprint; 1 errors row, 2 events rows; count 2 |
| Same message with changed line number | Same fingerprint (line masking works) |
| Changed function/service | Different fingerprint |
| Invalid payload (missing `message`) | 422, no 500 |
| Search, service filter, limit bounds | Correct subsets; `limit=0` → 422 |
| Unknown fingerprint | 404 |
| Stats | Correct group/event/today counts |
| SDK against live API | Fingerprint returned; row created |
| SDK against dead server | `None`, no exception; debug log emitted |
| Decorator | Captured then re-raised |
| Demo `/user-fail/1`, `/2` | One group, count 2, two distinct stacks |
| Dashboard | Real browser render; row click opens drawer; search filters after debounce |
| Fresh `docker compose down -v && docker compose up --build -d` | Build, healthcheck, demo dependency, ingest, and queries all pass |
| Restart | Data persists in the named volume |
| FixMySlop scans | 0 issues after fixes for the swallowed exception, undeclared Pydantic dependency, and two-tone headline |

**No automated test suite or CI is present.** Everything above was verified manually or with ad-hoc scripts. This is the largest quality gap in the project: a handful of pytest tests around fingerprinting, the ingest transaction, and the API contract would make future changes much safer.

---

## 13. Security, privacy, and limitations

Buggy is a local/demo tool, not a hardened public service.

- **No authentication or authorization.** Anyone who can reach port 8000 can read all errors and inject events. Do not expose it to the internet as-is.
- **No rate limiting.** A client can flood `/api/events`; each request is a database write. Payload sizes are not explicitly capped by the app.
- **Sensitive data can be captured.** Messages, stack traces, routes, request IDs, and user IDs may contain credentials, tokens, emails, or personal data. Treat the SQLite file as sensitive and avoid sending it to an instance you do not control.
- **The DSN is unauthenticated.** If it leaks, anyone can write fake errors.
- **SQL injection is addressed** through parameterized queries throughout.
- **No TLS** in the Compose setup; use a reverse proxy if network-exposed.
- **Fingerprint collisions** are theoretically possible with 64 bits, and over-grouping can happen when messages differ only in masked values.
- **SDK delivery is best-effort.** No retries/offline queue; reports are lost if Buggy is down or slower than the 2-second timeout.
- **Unbounded storage.** There is no retention or vacuum policy; the `events` table grows forever.
- **Single node.** SQLite with WAL is excellent for this scale but does not horizontally scale.
- **Chart is approximate**, as explained in section 9.
- **Environment is not persisted**, despite being accepted by the API and sent by the SDK.
- **Client timestamps are trusted** for `events.timestamp` and can skew "events today" if an SDK sends an odd value.
- **First message wins** for the group title; later variants are only visible in event rows.
- **CDN dependency** for Chart.js; the dashboard otherwise has no external runtime dependencies.
- **Small code-health items**: `app/db.py` currently has two unused imports (`contextlib.closing`, `webbrowser.get`) that should be removed, and there is no linter/type-check configuration in the repo.

---

## 14. Design decisions and why they fit

| Decision | Why |
|---|---|
| Raw `sqlite3`, no ORM | Zero migration/dependency overhead; the schema is small and the queries are explicit |
| Two tables instead of one | Aggregates stay small and fast; raw evidence is retained separately |
| Fingerprint as primary key | Natural idempotent grouping key; upsert is a single statement |
| SHA-1 truncated to 16 hex chars | Compact URLs/keys; adequate uniqueness for a self-hosted tool |
| Thin synchronous SDK | Easy to drop into any Python process; no framework coupling |
| Fire-and-forget reporting | Observability must not take down the app it observes |
| Vanilla JS + StaticFiles | No build step, no Node toolchain; the API and UI ship as one container |
| One image, two commands | Keeps the Dockerfile simple while still demonstrating a separate demo app |
| Compose volume + healthcheck | Data survives restarts and startup order is deterministic |
| Keep `/docs` and typed Pydantic responses | The API is explorable and self-documenting |

---

## 15. Future work

These are the natural next steps, roughly in order of value:

1. **Automated tests + CI** — fingerprint unit tests, ingest integration tests, API contract tests.
2. **Authentication and project keys** — required before any shared deployment. The DSN is currently unauthenticated.
3. **Rate limiting and payload limits** — protect ingestion and the database.
4. **Retention and pagination** — delete or archive old events; cursor-based event listing; dashboard pagination beyond the 500-group cap.
5. **Environment column and filtering** — currently accepted but discarded.
6. **True per-day event counts** — query `events` grouped by date instead of attributing group counts to `last_seen`.
7. **SDK improvements** — background queue, batching, retries with backoff, configurable timeout, async support, scrubbing hooks for PII.
8. **Alerting** — Slack/email on new groups or spikes.
9. **Source maps** for frontend stacks.
10. **Multi-tenancy** and per-project isolation.
11. **Operational tooling** — Prometheus/OTel export, Kubernetes/Terraform, backups.

---

## 16. File-by-file map

| File | Role |
|---|---|
| `app/main.py` | FastAPI app, lifespan DB init, five API routes, static mount |
| `app/db.py` | Schema, connection helper, normalization, frame extraction, fingerprint, ingest transaction, list/detail/stats queries |
| `app/models.py` | `EventIn`, `EventOut`, `ErrorItem`, `EventItem`, `ErrorDetail`, `StatsOut` |
| `sdk/buggy.py` | `init`, `capture_exception`, `handle_exception`, `install_excepthook`, payload builder |
| `demo/app.py` | Demo FastAPI app on 8001 that intentionally fails and reports through the SDK |
| `public/index.html` | Dashboard markup, CSS variables, layout, accessible controls |
| `public/app.js` | Fetching, filtering, table rendering, detail drawer, Chart.js chart |
| `public/favicon.svg`, `public/favicon.ico` | Browser icon (removes the automatic `/favicon.ico` 404) |
| `requirements.txt` | Direct dependencies |
| `Dockerfile`, `.dockerignore` | Image definition and build-context trimming |
| `docker-compose.yml` | Two services, volume, healthcheck, env wiring |
| `README.md` | Quickstart, demo script, API reference, design notes |
| `asset.jpg` | Dashboard screenshot used by the README |
| `context.md` | Original project brief (git-ignored, not part of the runtime) |

---

## 17. Glossary

| Term | Meaning |
|---|---|
| **API** | The HTTP interface applications use to send and read data |
| **SDK** | A small library an application imports to report errors |
| **Endpoint** | A URL + method, e.g. `POST /api/events` |
| **Event** | One actual crash occurrence |
| **Group / issue** | All events sharing a fingerprint, shown as one row |
| **Fingerprint** | A short hash that identifies an error pattern |
| **Stack trace** | The list of function calls that led to an exception |
| **Payload** | The JSON body sent to the API |
| **SQLite** | A file-based relational database built into Python |
| **FastAPI / Uvicorn** | Web framework / server that runs the API |
| **Docker Compose** | Tool that builds and runs the API and demo as containers |
| **UTC / ISO-8601** | Standard time zone / standard timestamp text format |
| **WAL** | SQLite write-ahead logging; lets reads proceed during writes |

---

## 18. Bottom line

Buggy is a complete, runnable observability product at small scale: an app reports an exception through a minimal SDK, FastAPI validates and fingerprints it, SQLite maintains both a cheap aggregate and the raw evidence, and a no-build dashboard turns those rows into a searchable issue list with stacks, counts, and a trend chart. Its architecture is intentionally boring — one database file, one image, two ports — which is what makes it easy to run, explain, and extend.

> Built **Buggy**, a self-hosted error tracker (FastAPI + SQLite + vanilla JS) with a 3-line Python SDK, SHA-1 fingerprinting that groups duplicate crashes into actionable issues, searchable stack-trace inspection, and a Chart.js trends dashboard, fully containerized behind `docker compose up --build`.
