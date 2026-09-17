# StudentLab — API Specification

[CURRENT] **There are zero API endpoints in this repository.** No FastAPI app, no
router, no `main.py`, no ASGI entrypoint (verified: `git ls-files` → `README.md`
only). Everything below is [TARGET] — the contract Steps 1+ implement. No
endpoint documented here is callable today.

## 1. Conventions — [TARGET]

| Concern | Contract |
|---------|----------|
| Base path | `/api/v1` (versioned from day one; breaking change ⇒ `/api/v2`, no in-place edits) |
| Format | JSON only, UTF-8. Requests `Content-Type: application/json` |
| Naming | Plural resources, snake_case fields, `id` in path always integer |
| Auth | `httpOnly` cookies `sl_access`, `sl_refresh` (SECURITY.md §3). No tokens in URLs |
| Transport | Same-origin. Dev: Vite proxies `/api` → `http://localhost:8000` (no CORS in prod; permissive CORS only in dev via `settings.CORS_ORIGINS`) |
| Errors | Uniform envelope, single shape everywhere |
| Validation | Pydantic v2; invalid body/query ⇒ 422 with field-level detail |
| Timestamps | ISO-8601 UTC (`2026-09-17T13:34:21Z`); calendar fields are `YYYY-MM-DD` |
| Pagination | `?limit=25&offset=0`, limit ≤ 100; response `{ items, total, limit, offset }` for every list |
| Idempotency | Not required for v1 (single local user per session) |
| Docs | FastAPI's OpenAPI at `/api/docs` — teacher-only view is NOT enforced (localhost, non-secret); do not rely on it being hidden |

Error envelope:

```json
{ "error": { "code": "not_found",
             "message": "Project not found.",
             "details": [ { "field": "title", "issue": "already exists for this student" } ] } }
```

| HTTP | `code` | When |
|------|--------|------|
| 400 | `bad_request` | malformed semantics not caught by validation |
| 401 | `unauthenticated` | missing/expired access token |
| 403 | `forbidden` | role not permitted (e.g. student → teacher dashboard) |
| 404 | `not_found` | missing **or** owned by another student (SECURITY.md §6 — never 403 on other students' resources) |
| 409 | `conflict` | duplicate title/username, project busy, optimistic-lock mismatch |
| 413 | `payload_too_large` | request body cap (64 KiB) |
| 422 | `validation_error` | Pydantic |
| 429 | `execution_quota` | run quota exceeded |
| 500 | `internal` | unexpected — response carries no internals, id = request-id for logs |

## 2. Endpoint Inventory — [TARGET]

`R` = any authenticated role, `S` = student (own resources only), `T` = teacher,
`P` = public. "S(own)" means: a student may only reference resources they own —
enforced in the service layer, not in the route.

### auth

| Method | Path | Access | Notes |
|--------|------|----------|-------|
| POST | `/api/v1/auth/login` | P | `{username, password}` → 204 + sets both cookies. Uniform failure: 401 `unauthenticated` for wrong username OR wrong password (SECURITY.md §4) |
| POST | `/api/v1/auth/logout` | R | Clears cookies, revokes refresh family |
| POST | `/api/v1/auth/refresh` | R | Rotates access token. Body empty; reads `sl_refresh` |
| GET | `/api/v1/auth/me` | R | → `{ id, username, role, student: {full_name, group_name} | null }` — the SPA's sole bootstrap call |

### users (teacher admin only)

| Method | Path | Access |
|--------|------|--------|
| GET | `/api/v1/users?role=&limit=&offset=` | T |
| POST | `/api/v1/users` | T — creates a student (`{username, full_name, role, initial_password}`) or teacher |
| PATCH | `/api/v1/users/{id}` | T — role, is_active, full_name via profile |
| POST | `/api/v1/users/{id}/reset-password` | T — `{new_password}`; no email flow (non-goal) |
| DELETE | `/api/v1/users/{id}` | T — soft delete (`is_active=0`); `?wipe=true` for the destructive path (DATABASE.md §5) |

Students get **no** self-service password change in v1 (add in Step 4+ if wanted;
listed as an open question).

### students

| Method | Path | Access |
|--------|------|--------|
| GET | `/api/v1/students/me` | S |
| GET | `/api/v1/students` | T — roster for the dashboard (`{id, username, full_name, group_name, project_count, last_activity_at}`) |
| GET | `/api/v1/students/{studentId}` | T or S(own, i.e. `studentId == me.student.id`) → `StudentTab` head data |
| PATCH | `/api/v1/students/{studentId}` | S(own) profile fields; T for any |

### projects

| Method | Path | Access |
|--------|------|--------|
| GET | `/api/v1/projects?student_id=&q=&limit=&offset=` | S: `student_id` forced to own id even if another value is sent; T: any |
| POST | `/api/v1/projects` | S (teacher may also create for a student via `student_id`) |
| GET | `/api/v1/projects/{id}` | S(own) / T |
| PATCH | `/api/v1/projects/{id}` | S(own) / T |
| DELETE | `/api/v1/projects/{id}` | S(own) / T — cascades executions (DATABASE.md §5) |

`ProjectOut` = `{ id, student_id, title, description, source_repo, source_ref,
entrypoint, created_at, updated_at, last_execution: {id,status,finished_at} | null }`.
Note: `source_repo`/`source_ref` are metadata + git pointers; **the API never accepts
file contents or shell commands** (SECURITY.md §7).

### activities (work reports)

| Method | Path | Access |
|--------|------|--------|
| GET | `/api/v1/activities?student_id=&project_id=&from=&to=&limit=&offset=` | S(own) / T |
| POST | `/api/v1/activities` | S(own project ids only) / T |
| GET | `/api/v1/activities/{id}` | S(own) / T |
| PATCH | `/api/v1/activities/{id}` | S(own) / T |
| DELETE | `/api/v1/activities/{id}` | S(own) / T |

`from`/`to` are inclusive `YYYY-MM-DD`. `POST` body: `{ student_id?, project_id?,
date, title, description, hours }` — for a student, `student_id` is ignored and
forced to own; a foreign `project_id` ⇒ 404.

### calendar

| Method | Path | Access |
|--------|------|--------|
| GET | `/api/v1/calendar/{studentId}/month?year=2026&month=9` | S(own) / T |
| GET | `/api/v1/calendar/heatmap?year=2026` | T — optional teacher overview, Step 8 |

Response: `{ days: [ { date, activity_count, total_hours, project_ids } ] }` —
pre-aggregated so the client never re-fetches all rows for a month view. Read-only
projection over `activities`; never a second source of truth (DATABASE.md §3).

### executions

| Method | Path | Access |
|--------|------|--------|
| POST | `/api/v1/projects/{id}/executions` | **S(own)** / T — 202 `{execution_id, status:"pending"}`; never synchronous |
| GET | `/api/v1/executions/{id}` | S(owner-of-project) / T |
| GET | `/api/v1/executions?project_id=&status=&limit=&offset=` | S (auto-scoped to own projects) / T |
| POST | `/api/v1/executions/{id}/cancel` | S(owner) / T — best-effort `docker rm -f` |

`ExecutionOut` = `{ id, project_id, status, requested_at, started_at, finished_at,
exit_code, stdout, stderr, truncated, error }`. `stdout`/`stderr` are the capped
persisted tails (EXECUTION.md §8); `truncated` is the honest flag. Poll
`GET /api/v1/executions/{id}` while status ∈ {pending, running} with backoff
(500 ms → 2 s, cap 10 s). SSE/websocket is deferred (ARCHITECTURE.md §12).

### system

| Method | Path | Access | Notes |
|--------|------|--------|-------|
| GET | `/api/v1/healthz` | P | `{status:"ok", version, db:"ok", docker:"ok|unavailable"}` — pre-flight used by E2E and by EXECUTION.md error surfacing |

## 3. Request/Response Examples — [TARGET]

```http
POST /api/v1/auth/login            Content-Type: application/json
{ "username": "sara", "password": "correct horse battery" }
→ 204  Set-Cookie: sl_access=...; HttpOnly; SameSite=Lax; Path=/
       Set-Cookie: sl_refresh=...; HttpOnly; SameSite=Lax; Path=/api/v1/auth

POST /api/v1/projects              (as sara, student id 7)
{ "title": "Maze solver", "description": "BFS exercise",
  "source_repo": "https://github.com/Armannay/StudentLab.git",
  "source_ref": "student/sara/maze", "entrypoint": "main.py" }
→ 201
{ "id": 12, "student_id": 7, "title": "Maze solver", "description": "BFS exercise",
  "source_repo": "...", "source_ref": "student/sara/maze", "entrypoint": "main.py",
  "created_at": "2026-09-17T13:40:02Z", "updated_at": "2026-09-17T13:40:02Z",
  "last_execution": null }

POST /api/v1/projects/12/executions
→ 202 { "execution_id": 88, "status": "pending" }

GET /api/v1/executions/88          → 200
{ "id": 88, "project_id": 12, "status": "running", "started_at": "2026-09-17T13:40:10Z",
  "finished_at": null, "exit_code": null, "stdout": "", "stderr": "",
  "truncated": false, "error": null }
```

Negative case that MUST hold (authorization is backend-enforced):

```http
# sara (student id 7) requests another student's tab data
GET /api/v1/students/8     → 404 { "error": { "code": "not_found", ... } }
GET /api/v1/projects/13    → 404 (project 13 belongs to student 8)
GET /api/v1/teacher/...    → 403 / 404
# Even with a spoofed query param, own-scoping wins:
GET /api/v1/activities?student_id=8   → 200 { items: [ only student 7's rows ] }
```

## 4. Handler Shape (what a route may contain) — [TARGET]

```python
@router.post("", status_code=201)
def create_project(payload: ProjectCreate, user: User = Depends(get_current_user),
                   db: Session = Depends(get_db)) -> ProjectOut:
    return ProjectService(db).create(user=user, payload=payload)
```

Three lines. No `if`, no query, no `db.commit()` — ARCHITECTURE.md §5. A reviewer
seeing logic in this layer rejects the PR.

## 5. Compatibility & Evolution — [TARGET]

- Additive changes only within `v1` (new optional fields, new endpoints). Renaming
  or removing a field = new version.
- `updated_at` on mutable resources; future `If-Match` support is a documented
  non-goal for v1 (single user per session ⇒ last-write-wins is acceptable).
- `GET /auth/me` is the version-skew safety valve: SPA and API ship from the same
  `git pull`, so drift is bounded to the browser cache — instruct students/teacher
  to hard-reload after pulls (CONTRIBUTING.md).

## 6. Open API questions (human decisions)

1. Student self-service password change? (v1: no.)
2. Real-time execution output streaming (SSE) vs the polling contract above?
   v1: polling; revisit if UX demands.
3. `POST /projects` accepting `source_repo` from students is convenient but lets a
   student point a project at another student's branch — SECURITY.md §7 requires the
   service to verify the ref belongs to the student's own branch namespace. Confirm
   teacher accepts this restriction (it constrains cross-student "borrowing").
