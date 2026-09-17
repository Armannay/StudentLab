# StudentLab — Testing Strategy

[CURRENT] **No tests exist in any form** — no `pytest` config or test files, no
Playwright installation or config, no `package.json` anywhere, no CI. Verified by
full repository listing (see ARCHITECTURE.md §1). This document is the [TARGET]
testing architecture; §8 states exactly what cannot be tested yet and why.

Because there is nothing to protect yet, this strategy lands **with** each
implementation step (test-first per module), not after.

## 1. Pyramid & Ownership — [TARGET]

| Layer | Tooling | Scope | Who writes it | Speed budget |
|-------|---------|-------|---------------|--------------|
| Unit (backend) | pytest | services, permissions, repositories-on-real-sqlite, execution spec/state-machine (NullExecutor) | students for their module, teacher for core | whole unit suite < 30 s |
| API/integration (backend) | pytest + `httpx.AsyncClient(ASGITransport)` (sync `TestClient` where simpler) | every endpoint incl. authz matrix, happy + negative | same as unit — **endpoint without its 4 authz tests doesn't merge** | < 90 s |
| Frontend | Vitest + Testing Library | feature components in isolation (forms, calendar math, error mapping); NO snapshot sprawl | students with template provided | < 60 s |
| E2E | Playwright (`e2e/`) | real browser, real stack, user journeys (§5) | teacher/agent write scenarios; students extend for their feature | CI lane < 8 min total |
| Static | `tsc --noEmit` strict, eslint, `ruff` (+ `ruff format --check`), `mypy` on `backend/app` (strict on core/execution, gradual elsewhere) | every PR | enforced in CI, not by humans | < 60 s |

Rules: every bugfix adds a regression test first; E2E never duplicates logic tests
(it proves wiring); no test depends on another test's order (no shared mutable DB
state between test files).

## 2. Backend Layout & Fixtures — [TARGET]

```
backend/tests/
├── conftest.py            # app, db, client, auth fixtures (below)
├── factories.py           # make_user/make_student/make_project/make_activity/make_execution
├── unit/
│   ├── test_project_service.py   ├── test_permissions.py
│   ├── test_calendar_service.py  ├── test_auth_service.py
│   └── execution/ test_spec_builder.py  test_state_machine.py  test_docker_config.py
└── api/
    ├── test_auth.py  test_users_admin.py  test_projects_crud.py
    ├── test_activities_calendar.py  test_executions.py
    └── test_authorization_matrix.py   # the §6 canonical file
```

`conftest.py` contract (fixed now so all steps write compatible tests):

- `app` fixture: `create_app(settings overriden)` with `DATABASE_URL` = tmp-path
  SQLite **per test function**, `EXECUTION_BACKEND=null`; schema applied via
  **running Alembic `upgrade head`** (not `Base.metadata.create_all`) — keeps
  migrations tested for free; `execution_lane` fixture opts a test into
  `docker` backend and is auto-skipped when docker absent.
- `client` = httpx AsyncClient over ASGITransport — **no real network, no
  docker-compose needed** in unit/API lanes.
- `as_user(user) -> client` helper: sets the cookie jar via real
  `POST /auth/login` (auth is tested through the front door, never by forging
  tokens — token forging reserved to one dedicated `test_security_tokens.py`).
- Factories write through repositories (exercises them) but bypass HTTP for
  *setup* only; assertions always go through HTTP.

SQLite notes for tests: same PRAGMA connect handler as prod (FK enforcement! —
DATABASE.md §5), tmp_path file DB per test for isolation + WAL realism.

## 3. The Authorization Matrix Test (canonical pattern) — [TARGET]

For each resource `R ∈ {student, project, activity, execution}`:

| Case | Actor | Target | Expected |
|------|-------|--------|----------|
| own | student A | A's R | 200/201/204 |
| cross-student | student A | B's R | **404** (SECURITY.md §3 — not 403) |
| anon | — | any R | 401 |
| teacher | teacher | A's or B's R | 200 |
| role-denied collection | student | `/users`, `/teacher/dashboard` | 403 |
| forged scoping | student A | list w/ `student_id=B` | 200 but **only A's rows** |

`test_authorization_matrix.py` parametrizes this table across every resource as
endpoints land. It is the single most important file in the backend test suite;
extending it is part of the definition of done for each CRUD module.

## 4. Frontend Tests — [TARGET]

Vitest + `@testing-library/react` + jsdom; per feature `__tests__/`. Only meaningful
units: `ProjectForm` validation, `Calendar` day→badge mapping (pure, test hard),
`apiClient` error mapping, `useAuth` bootstrap states. MSW (or a fetch stub) — no
tests requiring a live backend (that's Playwright's job). Component tests never mock
the backend *semantics* (404-stays-404); they mock the transport.

## 5. Playwright E2E — [TARGET]

### 5.1 Package & layout (root-level `e2e/`, its own `package.json` — ADR-9)

```
e2e/
├── package.json                 # @playwright/test (pinned, matches CI image)
├── playwright.config.ts
├── .env.example                 # E2E_BASE_URL, credentials of seed accounts
├── fixtures/
│   ├── auth.fixture.ts          # loginAs(page, 'student_a' | 'student_b' | 'teacher')
│   ├── project.fixture.ts       # createProject(page, seed) — via UI, not API (§5.4)
│   └── test.ts                  # extends base with fixtures + per-run namespace
├── seed/seed.ts                 # idempotent: ensures seed accounts + demo project
├── helpers/selectors.ts         # ONLY place allowed to name data-testid values
├── tests/
│   ├── auth/       login.spec.ts        logout.spec.ts
│   │               protected-routes.spec.ts
│   ├── student/    own-tab.spec.ts      create-project.spec.ts
│   │               run-project.spec.ts  inspect-output.spec.ts
│   │               create-report.spec.ts calendar.spec.ts
│   ├── authorization/  student-cannot-access-other.spec.ts
│   │                   student-cannot-access-dashboard.spec.ts
│   ├── teacher/    dashboard.spec.ts    student-tab.spec.ts inspect-all.spec.ts
│   └── execution/  run-and-report.spec.ts  limits.spec.ts   # timeout/OOM states
└── test-results/  playwright-report/      # gitignored already
```

Naming: `tests/<area>/<journey>.spec.ts`; `test("...")` titles read as user
outcomes ("Student A cannot open Student B's tab"), one journey per file, tags
`@auth @student @teacher @authz @exec` (CI lanes select by tag).

### 5.2 Selector policy — [DECISION]

- Priority: `getByRole` (+ name) → `getByLabel` → `getByText` → `getByTestId`.
- **Banned in review:** CSS structural selectors, `nth-child`/`nth-of-type`,
  class-based locators, positional `>>` chains. Exception registry: `helpers/selectors.ts`
  only (document why + link to issue).
- `data-testid`: used only where role/label cannot express intent (status chips,
  output viewer panes, calendar day cells `data-testid="calendar-day-2026-09-17"`,
  project row `data-testid="project-row-<id>"`). Convention: kebab-case
  semantic names, never styling hooks. Every testid must exist in the registry
  file — CI greps for testids used in specs but absent from registry (ratchet).
- Testability is a frontend obligation from the first component: a component
  landing without accessible roles/labels in a review is bounced.

### 5.3 Test data strategy — [TARGET]

- Seed accounts (fixed, class-visible): `teacher`, `student_a` (Sara),
  `student_b` (Bilal) — created by `seed/seed.ts` **through the real API**
  (`/users`) using a bootstrap teacher, or via `python -m app.scripts.e2e_seed`
  gated by `settings.E2E_SEED_ENABLED` (True only when `ENV=e2e`). No committed
  `.db`; CI/teacher run the seed.
- Every spec creates its own data through the UI (proves the UI), names it with a
  per-run uuid prefix (`e2e-<runid>-…`), and asserts on *that* data only ⇒ specs
  are parallel-safe (`fullyParallel: true`) and rerunnable without cleanup.
- `--headed` never in CI; trace `on-first-retry`, video `retain-on-failure`,
  screenshot `only-on-failure` — failure artifacts are required for student bug
  reports.
- Time: the app's clock is injectable (`settings.FROZEN_NOW` in e2e env) for the
  calendar "click a report date → verify report" scenario; never `new Date()` in
  assertions.

### 5.4 Environment strategy — [TARGET]

- Config `webServer` (CI + local `npm run e2e`): starts **uvicorn with the app**
  and serves the **built frontend** via FastAPI static mount or `vite preview` on
  one origin; DB = fresh tmp SQLite created by global setup → then `seed/seed.ts`.
  E2E tests real production builds, not Vite dev-server HMR quirks.
- Docker execution lane: `EXECUTION_BACKEND=docker` + `docker compose -f
  docker-compose.yml -f e2e/docker-compose.e2e.yml up` (CI has docker; local
  teacher too). `@exec`-tagged specs run only there; other lanes use
  `EXECUTION_BACKEND=null` and assert the queued-state transitions only.
- `.env.example` documents `E2E_BASE_URL` (default `http://localhost:4173`);
  remote/CI overrides without editing tests.

### 5.5 Required v1 scenarios (acceptance list) — [TARGET]

Auth: login page renders · valid login lands student on own tab / teacher on
dashboard · invalid login error (no token leak) · logout kills session (back
button → login) · protected route redirects anon.

Student: open own tab · create project (form validation incl. duplicate title) ·
open project · run project → status transitions visible · inspect result output ·
create report · open calendar · click report date → report listed · edit + delete
report reflected in calendar.

Authorization: student A → B's tab URL shows 404/empty state, and direct
`fetch('/api/v1/students/B')` from A's session asserts 404 (an API-level assert
inside a browser test — cheap and catches frontend-only "fixes") · student
opening `/teacher` sees no dashboard.

Teacher: dashboard lists students w/ counts · open student tab · inspect
projects/reports/calendar/execution history of *both* students · run-inspect
read-only affordances.

Execution (`@exec`): print-hello succeeds and shows in history · infinite loop
shows `timeout` badge within timeout+grace · (optional stretch) memory bomb shows
`failed`.

## 6. What CANNOT Be Tested Yet — [CURRENT, honest statement]

Everything above is scaffolding for code that does not exist: no endpoints to call,
no UI to drive, no executor to run. Therefore **no test suite was run during this
Step 0** and none can pass or fail meaningfully; the only executable verification
performed in Step 0 was repository/audit commands (ARCHITECTURE.md §1). Each future
step's DoD includes "its tests were actually run and are green in CI".

## 7. CI Strategy — [TARGET] (brief-level; pipeline lands Step 1)

```
PR → backend: uv sync/pip install → ruff+mypy → alembic heads check → pytest (unit+api)
   → frontend: npm ci → tsc --noEmit → eslint → vitest run → vite build
   → e2e: playwright install --with-deps → build+seed → playwright test --grep-invert @exec
   → e2e-exec (docker): compose up → playwright test --grep @exec
merge requirements: all green + approvals per GIT_WORKFLOW.md; also run on push:main.
Nightly (optional, Step 9): full suite + flake report; PRs do not gate on it.
```

Retries: 1 in CI, 0 locally (fast local signal). `failOnSkipped:false` but skipped
`@exec` count reported. Playwright shard only when runtime demands.

## 8. Minimal Tooling Added in Step 0 — [CURRENT change]

Permitted by the brief "if Playwright is not configured at all". Assessment: since
**no** tooling of any kind exists (not even a `package.json` or Python env), adding
a lone Playwright config would be unresolvable scaffolding divorced from the
frontend scaffold it must version-align with. Step 0 therefore adds only the root
`.gitignore` (protects the repo from `.env`/`.db`/`node_modules` from commit #2)
and defers `package.json`, `playwright.config.ts`, `pyproject.toml` to **Step 1
(repo scaffolding)**, whose exact contents are prescribed by §5.1–§5.4 and
§7 so Step 1 is mechanical. If the teacher prefers, they may pre-create
`e2e/` + `frontend/` + `backend/` skeletons directly from this spec before Step 2.

## 9. Anti-Goals — [DECISION]

No 100% coverage quotas (coverage reported, not gated, v1; API matrix §3 is the
gate). No load/perf suites until real class size demands. No mutation testing, no
visual regression — revisit post-Step 9 only if pain appears.
