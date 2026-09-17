# StudentLab

Local-first web platform for managing student projects under teacher supervision:
student work reports, per-student tabs with calendars, projects executed in
Docker-sandboxed backends, teacher dashboard, and Git-based multi-student
development integrated via PRs (with an AI agent as branch integrator).

> **Status: Step 0 — architecture & specification only.**
> There is **no application code in this repository yet**: no backend, no frontend,
> no database, no tests. Do not look for features; read the specification first.
> Intended workflow once implemented: GitHub → `git pull` → run locally → teacher
> reviews in browser (no hosting, no domain — by design).

## Documentation

| Document | Contents |
|----------|----------|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Audited current state, ADRs, target repo layout, layering contract, StudentTab design, gap analysis, risks |
| [docs/DATABASE.md](docs/DATABASE.md) | ERD, table contracts, SQLite conventions, migration discipline |
| [docs/API.md](docs/API.md) | REST surface (`/api/v1`), conventions, per-module endpoint specs + permission matrix |
| [docs/SECURITY.md](docs/SECURITY.md) | Threat model, authN (cookie JWT), RBAC enforcement design, checklists |
| [docs/EXECUTION.md](docs/EXECUTION.md) | Docker sandbox contract: isolation, limits, lifecycle, why `subprocess` is banned |
| [docs/TESTING.md](docs/TESTING.md) | pytest/Vitest/Playwright strategy, authz-matrix tests, E2E scenarios, CI lanes |
| [docs/GIT_WORKFLOW.md](docs/GIT_WORKFLOW.md) | Branch model, protections, ownership map, files students must not touch |
| [docs/AI_AGENT_WORKFLOW.md](docs/AI_AGENT_WORKFLOW.md) | 13-step PR integration procedure for the agent |
| [CONTRIBUTING.md](CONTRIBUTING.md) | The 10 rules for student contributors |

## Planned stack (target — not yet installed)

React 19 + TypeScript + Vite · FastAPI + SQLAlchemy 2 + SQLite + Alembic ·
pytest · Playwright · Docker for project execution.

Every document distinguishes **[CURRENT]** (what actually exists: nothing) from
**[TARGET]** (what Steps 1+ implement). Start with ARCHITECTURE.md §1 and §10.
