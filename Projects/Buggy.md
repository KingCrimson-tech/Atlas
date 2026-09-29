
# PROJECT: Buggy - Mini-Sentry Error Tracker (Standalone Observability)

## 1. Context for Agent
You are building a fresher-level, resume-worthy standalone observability project.
No Prometheus/Grafana/OTel. Build the tool itself: SDK + ingest backend + dashboard.
Stack: Python 3.11+ + FastAPI + sqlite3 stdlib + vanilla JS + Chart.js CDN.
Single repo, single Dockerfile, `docker compose up` must run everything.
Build STRICTLY phase-by-phase. Do not skip ahead. Verify each phase before next.

## 2. Goal
Any Python app can report errors with 3 lines:
from sdk.buggy import init, capture_exception
init(dsn="http://localhost:8000/api/events", service="demo-api")
capture_exception(e, route="POST /checkout")

Backend groups duplicate errors by fingerprint, stores in SQLite, dashboard shows grouped errors + detail.

## 3. Non-Goals (DO NOT BUILD)
Auth, multi-tenancy, React, ORM (use raw sqlite3), K8s, Terraform, Prometheus, source-maps, Slack alerts, rate-limiting. List these as Future Work in README only.

## 4. Architecture
[demo/app.py + sdk/buggy.py] --POST JSON--> [app/main.py :8000] --> [buggy.db SQLite]
[Browser] --GET--> [app/main.py serves public/ + /api/*]

Ports: main API 8000, demo app 8001.

## 5. File Tree (must match)
buggy/
  requirements.txt # fastapi, uvicorn[standard], requests
  app/__init__.py
  app/main.py # FastAPI app, all routes, static mount
  app/db.py # sqlite init + fingerprint helpers
  app/models.py # Pydantic models
  sdk/buggy.py # zero FastAPI dep, only requests/stdlib
  demo/app.py # FastAPI demo on 8001 using sdk
  public/index.html
  public/app.js
  Dockerfile
  docker-compose.yml
  README.md

## 6. Data Model (exact SQL)
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
  user_id TEXT DEFAULT '', timestamp TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_events_fp ON events(fingerprint);
CREATE INDEX IF NOT EXISTS idx_errors_last ON errors(last_seen);

Fingerprint algorithm (must implement exactly):
1. normalize_message(msg): replace r'\d+' -> '*', r"'[^']*'" -> "'*'", r'"[^"]*"' -> '"*"', collapse whitespace, lower? NO - keep case, truncate 500 chars.
2. extract_frames(stack: str): lines containing 'File "', take first 3, strip line numbers `, line \d+` -> `, line *`, join with '|'. If no frames, use ''.
3. fingerprint = sha1(f"{service}|{normalized}|{frames}".encode()).hexdigest()[:16]

## 7. API Contracts
POST /api/events -> 202
  In: {service:str required, message:str required, stack:str="", route:str="", request_id:str="", user_id:str="", environment:str="dev", timestamp:str? ISO, default now UTC}
  Logic: normalize, fingerprint, now=ISO UTC, INSERT errors ON CONFLICT(fingerprint) DO UPDATE SET count=count+1, last_seen=now | INSERT event.
  Out: {fingerprint:str, count:int}
  Must never 500 on bad SDK payload - validate with Pydantic, return 422.

GET /api/errors?search=&service=&limit=50 -> 200 [{fingerprint, service, message, count, first_seen, last_seen} ORDER last_seen DESC, filter message LIKE %search%]

GET /api/errors/{fingerprint}?events_limit=20 -> 200 {error:{...}, events:[{id, route, stack, request_id, user_id, timestamp} ORDER id DESC]} 404 if not found.

GET /api/stats -> 200 {total_groups:int, total_events:int, events_today:int}
GET /health -> 200 {ok:true}

## 8. SDK Contract (sdk/buggy.py, <120 lines)
_config = {"dsn":None, "service":None, "environment":"dev"}
def init(dsn:str, service:str, environment:str="dev"): store, strip trailing /
def capture_exception(exc:BaseException, route:str="", user_id:str="", request_id:str="") -> str|None:
  try: build payload {service, message=str(exc) or type name, stack=traceback.format_exception, route, ...}, requests.post(dsn, json=payload, timeout=2), return fingerprint from resp else None. Except ALL -> return None, never raise.
def handle_exception(route:str=""): decorator for demo functions.
def install_excepthook(): set sys.excepthook to auto-capture.
No import of fastapi inside SDK.

## 9. Dashboard Spec (public/)
index.html: header stats (3 cards), search input + service filter, table Count|Message|Service|Last Seen, click row opens detail pane with <pre> stack + event list, Chart.js bar errors/day computed client-side from /api/errors. Plain fetch(), no build. Must work when served via FastAPI StaticFiles at /.

## 10. Demo App (demo/app.py, port 8001)
FastAPI, imports sdk via sys.path. On startup init(dsn="http://localhost:8000/api/events"...). Routes: GET /ok, GET /fail?msg=, GET /user-fail/{id} (raises ValueError(f"Payment failed for order {id}")), global exception_handler -> sdk.capture_exception + return 500 JSON. Must prove grouping: /user-fail/1 and /user-fail/2 produce same fingerprint.

## 11. Phases + Acceptance Criteria
P0 Scaffold: uvicorn app.main:app serves /health + /docs. FAIL if not.
P1 DB+Ingest: curl POST twice same message diff numbers -> same fingerprint count=2, 1 row in errors, 2 rows in events.
P2 Query APIs: /api/errors search + /api/errors/{fp} + /api/stats return correct JSON in /docs.
P3 SDK: python -c "init+capture_exception" creates row. SDK with dead server does not raise.
P4 Demo: hit 8001/user-fail/1,2 -> 8000 shows 1 group count 2.
P5 UI: open localhost:8000/ shows table, search, detail drawer, chart.
P6 Ship: Dockerfile + docker-compose.yml (buggy:8000, demo:8001) fresh `docker compose up --build` works. README with arch diagram, ports table, 3 demo curls, screenshot placeholder, Future Work.

## 12. Agent Rules
- Use only stdlib sqlite3, no SQLAlchemy.
- All timestamps UTC ISO8601 `datetime.utcnow().isoformat()+"Z"`.
- Backend must use parameterized queries (? placeholders).
- Keep functions small, add docstrings.
- After each phase, print VERIFY commands + expected output, stop and wait for user to confirm before next phase.
- If stuck, do minimal to pass acceptance, note tradeoff.

## 13. README Outline to Generate in P6
Title, 1-line pitch, arch ASCII, quickstart (docker compose up), ports table, demo script, API table, fingerprint explanation, screenshots, Future Work, resume bullet.