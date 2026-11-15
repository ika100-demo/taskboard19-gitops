---
plan_id: 001-persistent-tasks-in-postgres-and-task-ui
spec: 001-persistent-tasks-in-postgres-and-task-ui
feature: Keep tasks in a system-owned PostgreSQL database and let people list and create them in the web UI
gitops_app: ika100-demo/taskboard19-gitops
status: draft
repos:
  - id: taskboard19-gitops
    shape: gitops-app
    summary: Declare the postgres database, wire it to the Task API and wire the web UI to the Task API
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.6]
    order: before-merge
    gitops:
      - {resource: postgres, type: database}
      - {uses: postgres, component: taskboard19-api}
      - {env: {TASK_API_URL: 'http://taskboard19-api'}, component: taskboard19-web}
    depends_on: []
    done: false
  - id: taskboard19-api
    shape: service-python
    summary: Keep tasks in PostgreSQL so they survive restarts and are shared by every instance
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.6]
    criteria:
      - "Given tasks \"A\" and \"B\" were created through `POST /tasks`, when the service is stopped and a new process is started against the same database (`DATABASE_URL`), then `GET /tasks` answers 200 with both tasks in creation order, each with the same `id`, `title`, `due_date` and `created_at` returned by its `POST`, and the next `POST /tasks` returns an `id` greater than every existing one; ids are positive integers that are never reused but may have gaps (system AC-001.1) (replaces AC-001.7)"
      - "Given the database holds no task, when a client sends `GET /tasks`, then the response is 200 with an empty JSON array (system AC-001.1) (replaces AC-001.6)"
      - "Given an empty database, when two service instances start at the same time against it, then both create or find the task schema themselves, both answer `GET /ready` with 200, and no manual migration step is needed (system AC-001.1, AC-001.2)"
      - "Given two service instances use the same database, when a task is created through one of them, then `GET /tasks` on the other includes it, and tasks created concurrently through both instances get distinct ids (system AC-001.2)"
      - "Given the database is unreachable or does not answer within 3 seconds, when a client sends `POST /tasks` or `GET /tasks`, then the response is 503 with the JSON body `{\"detail\": \"Task storage is unavailable\"}`, no task is stored, `GET /ready` answers 503 and `GET /health` keeps answering 200 (system AC-001.3)"
      - "Given the database was unreachable and is reachable again, when a client sends the next `POST /tasks` or `GET /tasks`, then it succeeds and `GET /ready` answers 200, without restarting the service (system AC-001.3)"
      - "Given the service runs against a database, when a client sends `POST /tasks` with `{\"title\": \"Buy milk\", \"due_date\": \"2026-12-01\"}` and then `GET /tasks`, then the list ends with that task exactly as the 201 response returned it, and a `POST /tasks` that fails validation (422) stores nothing in the database (system AC-001.6)"
    arguments: |
      The HTTP contract of spec 001 stays as it is (system A2); only the store behind `TaskStore` changes.
      Connection settings come from DATABASE_URL / PG* (injected by the gitops `uses` operation). Tests run
      against a real PostgreSQL (e.g. a test container or a devbox service), not a fake.
      Schema creation must be idempotent and safe when several instances start at once (e.g. an advisory lock).
      There is no data carry-over from the in-memory store; the database starts empty.
    depends_on: []
    done: false
  - id: taskboard19-web
    shape: web-nextjs
    summary: Show all tasks and a create form on the home page, backed by the Task API
    acs: [AC-001.4, AC-001.5, AC-001.6, AC-001.7, AC-001.8]
    criteria:
      - "Given the Task API at `TASK_API_URL` answers `GET /tasks` with tasks, when a person opens the home page, then it replaces the template's landing content with every task in the order the API returned them (oldest first), each showing its title and its `due_date` as `YYYY-MM-DD`, or \"No due date\" when `due_date` is null; the page is rendered at request time, so a reload shows tasks created elsewhere (system AC-001.4)"
      - "Given the Task API answers `GET /tasks` with `[]`, when a person opens the home page, then it shows \"No tasks yet\" and the create form (system AC-001.5)"
      - "Given the home page is open, when a person enters the title \"Buy milk\", optionally picks a due date in the date picker and submits, then the web server sends `POST /tasks` to the Task API with `{\"title\": \"Buy milk\", \"due_date\": \"YYYY-MM-DD\" or null}`, and on 201 the returned task appears at the end of the list without a page reload and the form is cleared (system AC-001.6)"
      - "Given the home page is open, when a person submits the form with a title that is empty, only spaces or longer than 200 characters, then \"Title must be 1 to 200 characters and not only spaces.\" is shown next to the title field, the entered values are kept and no request is sent to the Task API (system AC-001.7)"
      - "Given the Task API is unreachable, does not answer `GET /tasks` within 5 seconds, or answers it with a 5xx status or a body that is not a task list, when a person opens the home page, then it shows \"Tasks could not be loaded. Please try again later.\" instead of the list (system AC-001.8)"
      - "Given the Task API is unreachable, does not answer `POST /tasks` within 5 seconds, or answers with any status other than 201, when a person submits a valid create form, then the form shows \"The task could not be saved. Please try again.\", keeps the entered values and appends nothing to the list (system AC-001.8)"
    arguments: |
      The browser never calls the Task API directly: server components read the list and a server action
      (or a route handler of this app) performs the create, so no CORS and no NEXT_PUBLIC_ variable is needed.
      Read TASK_API_URL at runtime on the server (document it in docs/env-vars.md); tests stub the Task API.
      Smoke: a read-only check that the home page loads, and an @writes check creating a `smoke-<uuid>` task.
    depends_on: []
    done: false
gitops_pin: []
---

## Decomposition

Three entries, one per repo the spec's stories name. The gitops entry runs first (`before-merge`) so that the
database exists and the Task API receives `DATABASE_URL` before its database-backed version deploys. The Task API
and the web UI then build **in parallel**: the API keeps its HTTP contract unchanged (system A2), so the web UI
builds against the contract below without waiting for the API. No library is needed (no shared types beyond the
small JSON contract), and no repo has to be created, so no `/component:new` is needed.

## taskboard19-gitops — Declare the postgres database, wire it to the Task API and wire the web UI to the Task API

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.6 · **Depends on:** []

- `resource postgres` (type `database`): one PostgreSQL per environment in the system's own cluster (system A1).
  It exists wherever the Task API runs, today `dev` only; backups and staging/prod are non-goals.
- `uses postgres` on `taskboard19-api`: the API gets `DATABASE_URL`, `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`,
  `PGPASSWORD` from the connection Secret. This is the infrastructure half of AC-001.1 to AC-001.3.
- `env TASK_API_URL=http://taskboard19-api` on `taskboard19-web`: the in-namespace Service of the Task API (port 80),
  so the web UI reaches the API inside each environment (the infrastructure half of AC-001.4 and AC-001.6).
- The API's existing readiness probe (`/ready`, every 5 s, 1 s timeout) is what takes an instance out of rotation
  when the database is down (AC-001.3); no probe change is needed.

## taskboard19-api — Keep tasks in PostgreSQL so they survive restarts and are shared by every instance

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.6 · **Depends on:** []

The store behind `POST /tasks` and `GET /tasks` moves from process memory to PostgreSQL. The API owns durability and
ordering across restarts (AC-001.1), sharing between instances (AC-001.2), the 503 and readiness behaviour when the
database is unavailable and its self-recovery (AC-001.3), and the API side of a create made from the UI (AC-001.6).
Its component spec 001 criteria **AC-001.7** (a restart empties the list and restarts ids at 1) and **AC-001.6**
(an empty list "since the service started") contradict durable storage; the new criteria replace both, as the
system spec's notes for the architect require for AC-001.7 (AC-001.6's wording conflicts the same way).

## taskboard19-web — Show all tasks and a create form on the home page, backed by the Task API

**Shape:** `web-nextjs` · **Criteria:** AC-001.4, AC-001.5, AC-001.6, AC-001.7, AC-001.8 · **Depends on:** []

The home page replaces the template's landing content (system A5) with the task list and the create form. The web
UI owns listing and wording (AC-001.4, AC-001.5, A6, A8), creating without reload (AC-001.6), client-side title
validation with the API's rules (AC-001.7, A7) and the two error states (AC-001.8). It talks to the Task API only
from its server side, through `TASK_API_URL`.

## Contract

Between `taskboard19-web` (caller, from its Next.js server side) and `taskboard19-api` (unchanged from Task API spec
001 except where tasks are kept). Base URL: `TASK_API_URL` (in the cluster `http://taskboard19-api`). All bodies are
`application/json`. No authentication (system A9).

**Task object**

| Field | Type | Rule |
|---|---|---|
| `id` | integer | positive, assigned by the API, never reused, increasing in creation order, may have gaps (system A3) |
| `title` | string | 1 to 200 characters, at least one non-whitespace character, stored exactly as sent |
| `due_date` | string or null | calendar date `YYYY-MM-DD`, or `null` when none |
| `created_at` | string | ISO 8601 / RFC 3339 UTC timestamp with `Z` suffix; fractional seconds may be present |

**`GET /tasks`**

- `200`: JSON array of every task in creation order (oldest first), each exactly as its `POST` returned it; `[]`
  when there is none. No query parameters, no pagination, no limit (system A10).
- `503`: `{"detail": "Task storage is unavailable"}` when the database is unreachable (AC-001.3).

**`POST /tasks`**

- Request: `{"title": <string>, "due_date": <"YYYY-MM-DD" | null>}`; `due_date` may be omitted (means `null`).
  The web UI always sends both fields, `due_date` from its date picker or `null`.
- `201`: the created task object.
- `422`: FastAPI's standard validation body `{"detail": [{"loc": ["body", "title" | "due_date"], ...}]}`; nothing
  stored. Only `loc` is contract; `type` and `msg` are not.
- `503`: `{"detail": "Task storage is unavailable"}`; nothing stored.

**Probes of the API** (used by the platform, not by the web UI): `GET /health` 200 while the process runs, regardless
of the database; `GET /ready` 200 when the database answers, 503 when it does not.

**Web UI states** (the web UI's own surface, stated here so both specs use the same words): "No tasks yet",
"No due date", "Title must be 1 to 200 characters and not only spaces.", "Tasks could not be loaded. Please try again
later.", "The task could not be saved. Please try again." (system A8).

### Errors

- The API's error bodies are JSON with a `detail` field: a string for 503 (`"Task storage is unavailable"`), a list of
  `{loc, msg, type}` for 422. The web UI never shows API error text to a person; it shows its own wording.
- On `GET /tasks` the web UI treats anything other than a `200` with a JSON array of task objects (a 5xx, a 4xx, an
  unlisted status, a body that does not parse, a network error, a timeout) as "Tasks could not be loaded. Please try
  again later.".
- On `POST /tasks` the web UI treats anything other than `201` with a task object (including a `422` it should never
  provoke, a 5xx, an unlisted status, a network error, a timeout) as "The task could not be saved. Please try again.",
  keeps the entered values and appends nothing.
- The API answers an unexpected internal failure with 500 (FastAPI default); database unavailability is always 503,
  never 500.

### Timeouts

- Web UI to API, `GET /tasks`: 5 s total per request, no retry; past that the page shows the load error.
- Web UI to API, `POST /tasks`: 5 s total per request, **no retry** (a create is not idempotent; a retry could create
  the task twice). A timed-out create shows the save error; the task may still have been stored and appears on reload.
- API to database: connection attempt at most 3 s, each statement at most 5 s; on either the request answers 503.
  The API keeps no broken connection: the next request opens or validates a connection, so it recovers without a
  restart.
- API `/ready` database check: at most 500 ms, so it answers within the probe's 1 s timeout; a slow database makes the
  instance not ready, never restarts it (`/health` does not touch the database).

## gitops-app PR

- The entry PR (before any component merges) adds the `postgres` database resource, `uses postgres` on
  `taskboard19-api` and `TASK_API_URL` on `taskboard19-web`, applied by `/system:ship` with `cplat`.
- **No pins.** Both components run only in `dev`, which tracks `main`; staging/prod rollout is a non-goal of the spec,
  so `gitops_pin` is empty.
- After Argo reconciles dev: the database cluster is healthy and its `-app` Secret exists; the `taskboard19-api` pod
  has `DATABASE_URL` and `/ready` is 200; the `taskboard19-web` pod has `TASK_API_URL`. After both components deploy:
  create a task in the web UI at `taskboard19-web.taskboard19-dev.localhost`, restart the API deployment
  (`kubectl rollout restart`), and confirm the task is still listed by `GET /tasks` and on the home page.
