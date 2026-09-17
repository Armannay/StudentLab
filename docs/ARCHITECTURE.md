# StudentLab — Architecture Specification

Status legend — every section in the docs is one of:

- **[CURRENT]** — verified state of this repository today.
- **[TARGET]** — specification for implementation in Steps 1+. **Not built yet.**
- **[DECISION]** — a design choice fixed by this specification.

> ⚠️ Nothing described under [TARGET] exists in the repository yet. Do not read this
> document as documentation of implemented code. It is the implementation contract
> that Steps 1+ must satisfy.

---

## 1. Current State (audited) — [CURRENT]

Audit performed 2026-09-17 against branch `main` @ `3abc304a` and the GitHub remote
`Armannay/StudentLab` (public repo, created 2026-09-17).

Method: full file listing, `git ls-files`, `git ls-tree -r main`, `git show-ref`,
`git stash list`, `git fsck --lost-found`, object count (`git count-objects -v` →
3 total objects: 1 commit, 1 tree, 1 blob), working-tree check including ignored
files, and `gh api` for remote metadata.

**Finding: the repository is empty.** It contains exactly one file, `README.md`,
whose entire content is the single line `# StudentLab`. There is no commit history
beyond the initial commit, no other branches on the remote, no stashes, no dangling
git objects, and no untracked or ignored files in the working tree.

Checklist from the Step 0 audit brief:

| # | Audit item | Finding |
|---|------------|---------|
| 1 | Directory structure | Root contains only `README.md` |
| 2 | Frontend architecture | **Does not exist** (no `frontend/`, no `package.json`, no Vite config) |
| 3 | Backend architecture | **Does not exist** (no `backend/`, no Python files) |
| 4 | API endpoints | None |
| 5 | Database models | None; no database file present |
| 6 | Database access layer | None |
| 7 | Authentication | None |
| 8 | Project / activity / user functionality | None |
| 9 | Tests | None (no `pytest.ini`, no test dirs, no Playwright) |
| 10 | Docker configuration | None (`Dockerfile*`, `docker-compose*.yml` absent) |
| 11 | Configuration / environment handling | None (no `.env.example`, no settings module) |
| 12 | Git-related files | No `.gitignore`, no `.gitattributes`, no CODEOWNERS, no hooks committed |
| 13 | CI/CD | None (no `.github/workflows/`) |

Important note on the planning brief: the brief describes a "current technology"
stack (React/TypeScript/Vite, FastAPI, SQLite, pytest, Playwright). That is the
**intended** stack; the audit found no code using it. The entire stack must
therefore be created from scratch in Steps 1+.

Nothing can be reused. Nothing should be preserved — except `README.md` itself,
which is correct but minimal. Consequently:

- "What is already good" = the choice of stack and the local-first premise
  (sound for the stated workflow: GitHub → `git pull` → local app).
- "What can be reused" = nothing.
- "What should NOT be changed" = repository name/remote, `main` as default branch,
  the one-line README (only extended, see README change rationale in Step 0 report).
- "What should be refactored" = N/A — everything is greenfield, so every
  "refactor" in this spec is actually a **"build it right the first time"**
  requirement. This is an advantage: no legacy debt, and the guardrails in this
  spec can be laid down *before* the first line of feature code exists.

---

## 2. Architecture Decision Record — [DECISION]

| ID | Decision | Choice | Rejected alternatives (why) |
|----|----------|--------|------------------------------|
| ADR-1 | Overall shape | **Modular monolith** (one FastAPI app, enforced internal module boundaries) | Microservices: unjustified for one teacher + a class of students, kills local-first simplicity. Unstructured single-file app: guaranteed merge-conflict hell and unenforceable boundaries |
| ADR-2 | Internal structure | **Layered / Clean**: `Router → Service → Repository → DB`, one direction only | FastAPI-`Depends`-everything, fat route handlers, or "just one file per endpoint": business logic leaks into transport layer, untestable, students copy bad patterns |
| ADR-3 | DB engine | **SQLite** file (dev & classroom), SQLAlchemy 2.x ORM, Alembic migrations | Postgres now: extra ops burden with no need. Keep the option open — see DATABASE.md §8 |
| ADR-4 | Frontend | **React 19 + TypeScript + Vite**, feature-oriented folders, TanStack Query for server state | Redux (overkill); giant shared components (merge-conflict magnets); SSR/Next (there is no server-side rendering requirement in a local app) |
| ADR-5 | API style | Versioned REST under `/api/v1` | GraphQL: complexity without benefit at this scale; WebSockets for CRUD: deferred |
| ADR-6 | AuthN | JWT **access + refresh in `httpOnly` cookies**, Argon2id password hashing | Tokens in `localStorage` (XSS-stealable); session-in-DB (more moving parts, not needed); server-side redirect auth (SPA + API split makes it awkward) |
| ADR-7 | AuthZ | Role enum (`student`, `teacher`) + **object-level ownership checks in the service layer**; deny as 404 for cross-student access | Frontend-only hiding (trivially bypassable — explicitly forbidden); global ACL table (over-engineered for 2 roles) |
| ADR-8 | Code execution | **Docker containers via a pluggable `Executor` interface**; never run student code in-process | `subprocess`/`os.system` in the API server: categorically rejected — see EXECUTION.md §9 |
| ADR-9 | E2E | **Playwright in a root-level `e2e/` npm package** (own `package.json`), testing real browser workflows | Co-locating with `frontend/` (couples test infra to app deps, entangles students); Cypress (Playwright's tracing/auto-wait fits multi-role scenarios better) |
| ADR-10 | Unit/integration | Backend: `pytest` + `httpx` `ASGITransport` tests. Frontend: minimal `Vitest` + Testing Library, component-level only where it pays | Large frontend test suites before components exist: waste; "e2e only": too slow a feedback loop for backend logic |
| ADR-11 | CI | Single GitHub Actions PR pipeline: backend → frontend typecheck/tests/build → Playwright E2E | Nightly matrix, deploy pipelines, hosted envs: not needed for local-first |
| ADR-12 | Merge policy | Students never touch `main`; all integration via PR, squash-merge, human/AI-agent gate (AI_AGENT_WORKFLOW.md) | Direct pushes: unacceptable; long-running shared integration branches: more conflict surface |

Version anchors at spec time (2026-09) — **pin exact versions at Step 1 scaffold and
commit lockfiles**: Python 3.12+ (3.13 OK), FastAPI 0.13x with Pydantic v2,
SQLAlchemy 2.0.x, Alembic 1.14.x, pytest 8.x, React 19.x, Vite 7.x,
TypeScript 5.x, Playwright (latest stable, `@playwright/test`).

---

## 3. Target System Overview — [TARGET]

```
Browser (student or teacher)
        │  HTTPS not required; http://localhost
        ▼
React + TypeScript SPA  (frontend/, Vite dev server :5173 / static build)
        │  JSON REST under /api/v1  (same origin in prod; Vite proxy in dev)
        ▼
FastAPI modular monolith  (backend/, uvicorn :8000)
        │
        ├──► SQLite database  (backend/studentlab.db)  via Repository layer
        │
        └──► Execution module  →  Executor interface  →  DockerExecutor
                                        │
                                        ▼
                            Temporary sandbox container
                         (one per run, destroyed after)
```

Deployment model: teacher clones/pulls this repo, runs `docker compose up` (or the
documented run commands), opens `http://localhost:5173` (dev) or the served build.
No public hosting, no domain, no TLS — by design. Security assumptions of a
localhost app apply, but the *students are semi-adversarial users* (they can see
all client code): all authorization therefore lives in the backend (SECURITY.md).

## 4. Target Repository Layout — [TARGET]

```
StudentLab/
├── backend/
│   ├── app/
│   │   ├── main.py                 # create_app(), router registration, lifespan
│   │   ├── api/
│   │   │   └── v1/
│   │   │       ├── router.py       # THE aggregator: includes each feature router (1 line each)
│   │   │       ├── auth.py         # login/logout/refresh/me
│   │   │       ├── users.py        # teacher-only user admin
│   │   │       ├── students.py     # /students/me, teacher listing, per-student overview
│   │   │       ├── projects.py
│   │   │       ├── activities.py   # reports/activities (calendar reuses this data)
│   │   │       ├── calendar.py     # month aggregation endpoint
│   │   │       └── executions.py   # run a project, fetch execution + output
│   │   ├── core/
│   │   │   ├── config.py           # pydantic-settings; all env config here
│   │   │   ├── database.py         # engine, session factory, PRAGMA hooks
│   │   │   ├── security.py         # hashing, JWT encode/decode, cookie helpers
│   │   │   ├── deps.py             # get_db, get_current_user, require_teacher, ...
│   │   │   ├── permissions.py      # pure ownership predicates (see SECURITY.md §3)
│   │   │   ├── time.py             # utcnow() — the one timestamp helper (DATABASE.md §4)
│   │   │   └── errors.py           # domain errors → HTTP mapping (registered in main.py)
│   │   ├── models/                 # SQLAlchemy declarative ORM only — no queries here
│   │   │   ├── user.py  student_profile.py  project.py  activity.py  execution.py
│   │   │   └── refresh_token.py    # rotation families (SECURITY.md §2)
│   │   ├── schemas/                # Pydantic request/response DTOs
│   │   ├── repositories/           # ONLY place allowed to build SQL/ORM queries
│   │   ├── services/               # business logic + authorization decisions
│   │   │   └── student_service.py  project_service.py  activity_service.py
│   │   │       calendar_service.py execution_service.py  auth_service.py  user_service.py
│   │   └── execution/              # sandbox subsystem (EXECUTION.md)
│   │       ├── base.py             # Executor Protocol, RunSpec, RunResult, ExecutionStatus
│   │       ├── docker_executor.py
│   │       ├── workspace.py        # project dir → ephemeral workspace materialization
│   │       └── runner.py           # queue consumer / state machine driver
│   ├── alembic.ini
│   ├── alembic/versions/         # one file per PR; rebase rule in DATABASE.md §7
│   ├── tests/
│   │   ├── conftest.py           # app+db fixtures; tmp SQLite per test session
│   │   ├── unit/                 # services, permissions, execution spec-building
│   │   └── api/                  # full HTTP tests through TestClient/httpx
│   ├── pyproject.toml            # deps + pytest/ruff config (pinned)
│   └── requirements-dev.txt
├── frontend/
│   ├── src/
│   │   ├── app/                  # main.tsx, App.tsx, providers (Query, Auth), theme
│   │   ├── routes/               # route table, ProtectedRoute, role guards
│   │   ├── components/           # shared primitives ONLY (Button, Modal, ...)
│   │   ├── features/
│   │   │   ├── auth/             # Login page, useAuth(), session bootstrap
│   │   │   ├── students/       # ★ StudentTab + StudentInfo (+ teacher student list)
│   │   │   ├── projects/         # ProjectList, ProjectEditor (owns ProjectList)
│   │   │   ├── reports/          # ReportList + edit (owns calendar month aggregation view)
│   │   │   ├── calendar/         # Calendar (consumes reports' hook)
│   │   │   ├── execution/        # ExecutionHistory, RunOutputViewer
│   │   │   └── teacher/          # Dashboard
│   │   ├── services/             # api client (fetch wrapper), error mapping
│   │   └── types/                # API DTO types mirroring backend schemas
│   ├── index.html  vite.config.ts  tsconfig.json  package.json
├── e2e/                          # Playwright package (own package.json) — TESTING.md
│   ├── playwright.config.ts
│   ├── fixtures/  helpers/  seed/
│   └── tests/{auth,student,teacher,authorization,execution}/
├── runner/
│   └── Dockerfile                # sandbox image (pinned, teacher-owned) — EXECUTION.md §9
├── docker-compose.yml            # api, frontend, runner image build — EXECUTION.md
├── .github/workflows/ci.yml      # TESTING.md §7 — added in Step 1
├── .gitignore                    # ADDED IN STEP 0 (only tooling change)
├── CONTRIBUTING.md               # ADDED IN STEP 0
├── README.md                     # ADDED IN STEP 0 (pointer/status only)
└── docs/                         # this spec
```

Rules:

1. Every backend module owns its four files across the layers (`model`, `schemas`,
   `repository`, `service`) — a student feature = one file per folder. This makes
   merges almost disjoint.
2. `frontend/src/components/` is restricted to presentational primitives shared by
   ≥2 features. Anything specific to one domain lives in that feature.
3. Cross-feature imports are forbidden except via the feature's `index.ts` barrel.
4. Students work in *their* feature folder; the shared registry files
   (`api/v1/router.py`, `routes/index.tsx`, `components/*`) are edited only via
   teacher-reviews-required PRs (GIT_WORKFLOW.md §6).

## 5. Backend Layering Contract — [TARGET]

```
Router (api/v1/*.py)      HTTP only: parse request → call ONE service → return schema.
    │                     NO business rules, NO queries, NO session commits.
    ▼
Service (services/*.py)   Business logic, validation that needs the DB, ALL
    │                     authorization/ownership decisions, transaction boundary
    │                     (single commit/rollback per service method).
    ▼
Repository (repositories/*.py)  The ONLY layer building SQLAlchemy queries.
    │                     Plain functions/classes taking a Session; no FastAPI
    │                     imports; no HTTPException (raise domain errors instead).
    ▼
Database (models/*.py)    ORM schema; no query methods on models (no active-record).
```

Import direction (enforced; anything else is a review-blocking smell):

| Layer | May import | Must NOT import |
|-------|-----------|-----------------|
| api | schemas, core.deps, services | repositories, models, other routers |
| services | models, schemas, repositories, core | fastapi (no Request/HTTPException) |
| repositories | models, core | services, schemas, fastapi |
| models | core.config (types only), sqlalchemy | anything of ours above it |
| execution | models, core, repositories | api, services (execution_service is the caller *into* it) |

Cross-module access: `project_service` may import `project_repository`; it may NOT
import another module's repository. Cross-module reads go through that module's
service or repository-as-public-api (`repositories/__init__.py`). Domain errors
(`NotFoundError`, `PermissionDeniedError`, `QuotaExceededError`) live in
`core/errors.py`; `main.py` registers handlers mapping them to HTTP 404/403/422.

`main.py` responsibilities: create app, register middleware (request-id, no-auth
CORS only for same-origin dev proxy), include `api/v1/router.py`, startup/shutdown
lifespan (db init guard, execution worker start/stop). Never define routes in
`main.py`.

## 6. Module Catalog — [TARGET]

| Module | Route prefix | Service | Owner semantics |
|--------|-------------|---------|-----------------|
| auth | `/api/v1/auth` | `auth_service` | Anyone (login); self (me) |
| users | `/api/v1/users` | `user_service` | Teacher only (create students, reset passwords, set roles) |
| students | `/api/v1/students` | `student_service` | Self (`/me`, `/students/{id}` for own id) or teacher |
| projects | `/api/v1/projects` | `project_service` | Owner student (list own) or teacher (any) |
| activities | `/api/v1/activities` | `activity_service` | Owner student or teacher |
| calendar | `/api/v1/calendar` | `calendar_service` | Owner student or teacher; read-only aggregation over activities |
| execution | `/api/v1/executions` | `execution_service` → `execution/` | **Owner student only for triggering**; teacher may read results |

## 7. Frontend Data & State — [TARGET]

- Server state: TanStack Query. One `apiClient` in `services/api.ts`
  (`fetch` + `credentials: "include"`, JSON, error → `ApiError{status, code, message}`,
  401 → redirect to `/login`). Each feature defines its hooks (`useProjects`,
  `useCreateProject`, `useMonth`, …) calling typed endpoints in `types/api.ts`.
- Auth state: `AuthContext` (`user`, `role`, `isLoading`) populated once via
  `GET /auth/me`; consumed by route guards — never from components directly.
- Mutations invalidate by key convention: `["projects", studentId]`,
  `["activities", studentId, from, to]`, `["executions", projectId]`,
  `["calendar", studentId, y, m]`.
- Routing: `react-router`. `/login`, `/s` (student home = own StudentTab),
  `/teacher` (dashboard), `/teacher/students/:studentId` (same StudentTab read-only),
  `403` and `404`. Route-level guards only; deep components trust the backend.
- No global state library beyond context; no CSS-in-JS (plain CSS modules or a
  single small UI kit — decide in Step 1, don't churn).

## 8. StudentTab Architecture — [TARGET]

One reusable component renders everything a person needs to see about one student.
Identity comes from **data + route params, never from per-student code or files**.

```
StudentTab({ studentId, mode })          mode: "self" | "teacher-view"
├── StudentInfo       # name, username, created_at, summary stats
├── ProjectList       # projects of this student; create/edit/delete in "self" mode
├── Calendar          # month grid, day badges from activity aggregates; day click
├── ReportList        # activities/reports; edit in "self" mode; filter by month
└── ExecutionHistory  # executions of this student's projects (read-only in both modes)
```

- `StudentInfo` lives in `features/students` (the tab's owner).
- `ProjectList` is exported from `features/projects`; `Calendar` from
  `features/calendar`; `ReportList` from `features/reports`; `ExecutionHistory`
  from `features/execution`. The tab is pure composition.
- Each child takes `{ studentId, mode }` and calls the same hooks students and
  teachers use; the **backend** decides (via `student_id` + role) what data comes
  back. `mode` only toggles edit affordances in the UI — the API enforces truth.
- Routes: a student at `/s` renders `<StudentTab studentId={me.id} mode="self"/>`;
  a teacher at `/teacher/students/:studentId` renders
  `<StudentTab studentId={Number(params.studentId)} mode="teacher-view"/>`.
  Adding student #30 is a database row, not a code change. **A file like
  `StudentATab.tsx` failing to exist is a design requirement, not an accident.**
- Teacher student list = array of ids → the same tab. No duplicated layouts.

## 9. Execution Boundary (summary; see EXECUTION.md) — [TARGET]

`execution_service` validates ownership + quota, inserts an `Execution` row
(`PENDING`), and enqueues a `RunSpec`. A single in-process worker (started in the
app lifespan, concurrency capped at `EXECUTION_MAX_CONCURRENT`) pops specs, calls
`Executor.run()`, and persists `RUNNING → SUCCEEDED|FAILED|TIMEOUT|ERROR` with
capped stdout/stderr. The FastAPI request path never talks to Docker directly and
never executes student code. See EXECUTION.md for the full contract.

## 10. Gap Analysis

Because [CURRENT] is empty, every capability is a gap. The work items below are the
ordered backlog (matches the final Step 0 report §11).

| # | Work item | Builds | Exit criteria |
|---|-----------|--------|---------------|
| 1 | Repo scaffolding & tooling | backend+frontend+e2e skeletons, ruff/eslint/tsconfig strict, CI skeleton green on "no tests yet" | `pytest`, `tsc`, `build`, `playwright install` all runnable |
| 2 | DB foundation | models per DATABASE.md, migrations, PRAGMA session, seed CLI | `alembic upgrade head` on clean DB; model tests green |
| 3 | AuthN | `/auth/*`, argon2id, cookie tokens, `get_current_user` | API tests: valid/invalid login, expiry, logout |
| 4 | RBAC + users | roles deps, ownership predicates, `/users` admin | Negative authorization tests (student↛other) pass |
| 5 | Students + Projects CRUD | modules end-to-end | API tests per endpoint; frontend tabs render real data |
| 6 | Activities + Calendar | month aggregation, reports UI | Calendar E2E scenario green |
| 7 | Execution engine | `DockerExecutor`, runner image, quota/cleanup | Unit: spec building; API: run+fetch output; E2E: run project |
| 8 | Teacher dashboard | `/teacher/dashboard` + UI | E2E teacher scenario green |
| 9 | E2E hardening + agent workflow | remaining scenarios in TESTING.md, CI gating | Full PR pipeline green; AI_AGENT_WORKFLOW dry-run on 2 student PRs |

## 11. Architectural Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Fat route handlers creep in under time pressure | Untestable logic, boundary rot | §5 import rules in CI-review checklist; service-layer tests only possible if logic is in services — reviewers ask "which service test covers this?" |
| Students edit shared registries / components | Constant merge conflicts | GIT_WORKFLOW.md §6 protected-files table + CODEOWNERS |
| SQLite write concurrency during class (30 students) | `database is locked` errors | WAL mode + busy_timeout + short transactions (DATABASE.md §6); migrate path documented if it becomes real |
| Docker unavailable/legacy on teacher machine | Execution feature dead | Executor interface + clear error state `ERROR: docker_unavailable`; pre-flight check in `/healthz`; Step 1 verification task |
| Playwright vs Vite dev-server flakiness | CI noise → students distrust tests | E2E runs against production build + uvicorn (TESTING.md §6); retries=1 in CI only |
| "Local-first" tempts skipping authz | Data leaks between students | SECURITY.md checklist; negative tests are merge-blocking |
| Frontend `types/` drift from backend schemas | Silent contract breaks | One hand-maintained `types/api.ts` mirroring `schemas/` + API doc; later option: generate from OpenAPI (decide Step 6, don't mix) |
| Alembic multi-head migrations | Broken `upgrade head` on main | One-migration-per-PR + rebase rule + CI `alembic heads` check (DATABASE.md §7) |

## 12. Non-Goals (explicit "do not build") — [DECISION]

No microservices/queues/Redis/Celery; no GraphQL; no Kubernetes; no public hosting,
TLS, or domain; no async SQLAlchemy / async DB driver (sync sessions throughout —
the only awaits are for the execution worker, and blocking Docker calls run in a
threadpool); no multi-tenancy, email, password-reset-by-email (teacher resets in
admin UI); no code editor in the browser (projects are folders + git).
