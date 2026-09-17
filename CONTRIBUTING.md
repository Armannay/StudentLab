# Contributing to StudentLab

[CURRENT repo state] The repository currently contains **architecture
specification only** — no application code exists yet (no backend/, frontend/,
tests, or Docker setup). Please read [docs/ARCHITECTURE.md §1](docs/ARCHITECTURE.md)
before looking for anything to run. This file tells you how to work here once
implementation steps begin — and the rules below **already** apply to Step-1+ PRs.

## Who is this for?

Students contributing code to the platform, under teacher supervision, integrated
via Pull Requests (often by the AI integration agent — see
[docs/AI_AGENT_WORKFLOW.md](docs/AI_AGENT_WORKFLOW.md)). The short version: **you
own your feature module; the platform core is not free territory.**

## The 10 rules

1. **Branch:** `student/<your-username>/<feature-slug>` — kebab-case, from fresh
   `main` (`git switch main && git pull` first). Never push to `main`; it is
   (will be) protected.
2. **Commits:** Conventional Commits — `feat(projects): add duplicate-title error`,
   `fix(calendar): mark empty days`. One logical change per commit.
3. **PRs early, as draft:** open the PR as soon as you have a branch, mark
   "ready for review" when checks are green and you filled the template.
4. **Stay in your module:** your files are listed in the ownership map
   ([docs/GIT_WORKFLOW.md §5](docs/GIT_WORKFLOW.md)). Touching shared files
   (`router.py` include line, routes registry) is allowed only as the *last*
   commit and gets extra review.
5. **Never edit the locked list**
   ([docs/GIT_WORKFLOW.md §6](docs/GIT_WORKFLOW.md)): `core/`, `execution/`, CI,
   e2e infra, lockfiles, `docker-compose.yml`. Need a change there? Open an issue.
   Dependencies in particular: **you do not add packages** — we won't merge PRs
   that quietly add `requests` or a React chart lib.
6. **Tests are the feature.** Backend new endpoint = service tests + API tests +
   the authorization-matrix rows for it (docs/TESTING.md §3). UI components get
   accessible roles/labels and `data-testid`s from the registry
   (`e2e/helpers/selectors.ts` pattern).
7. **No fake green.** Do not claim tests pass unless you ran them. If a suite
   can't run yet (feature not existing), say so — the repo's docs practice this
   honesty; so should your PRs.
8. **Security is not yours to relax:** no `shell=True`, no running student code
   outside the Executor, no `except` swallowing authz errors, no secrets in code
   (the repo is public!), no storing tokens in `localStorage`.
   Read [docs/SECURITY.md](docs/SECURITY.md) §11-style checklists for your module.
9. **Migrations:** at most one alembic revision per PR; before marking ready:
   `git rebase main` and `alembic heads` shows exactly one head. Never hand-edit
   someone else's migration.
10. **Docs update with code:** behavior change ⇒ touch the relevant `docs/*.md`
    section marked [TARGET]→[CURRENT] in the same PR (that's how these docs stay
    truthful).

## Local workflow cheat-sheet

```bash
git switch main && git pull
git switch -c student/sara/report-edit
# ...code + tests...
cd backend  && pytest -q && ruff check .            # must pass locally too
cd frontend && npm run typecheck && npm run lint && npm test
git commit  # feat(reports): ...
git push -u origin student/sara/report-edit         # draft PR → CI runs → review
# before "ready for review":
git fetch origin main && git rebase origin/main && git push --force-with-lease
```

(Commands refer to the tooling specified for Step 1+; until scaffolding exists,
they will fail — that is expected, not a repo bug.)

## What lives where (map)

| You want to… | You edit | You read |
|--------------|----------|----------|
| Add/extend a backend feature | `backend/app/{models,schemas,repositories,services}/<module>*` + one include line | docs/ARCHITECTURE.md §5–§6, docs/API.md |
| Change data model | your module's model + your own alembic revision | docs/DATABASE.md |
| Add a UI area | `frontend/src/features/<feature>/**` + one route line | docs/ARCHITECTURE.md §7–§8 |
| Run projects (the sandbox!) | `backend/app/execution/**` — **teacher territory; propose in an issue first** | docs/EXECUTION.md |
| E2E coverage | specs under `e2e/tests/<area>/` (your feature only) | docs/TESTING.md §5 |
| Understand PR/branch rules | — | docs/GIT_WORKFLOW.md |
| Understand who merges and how | — | docs/AI_AGENT_WORKFLOW.md |

## Getting unblocked

Stuck on a merge conflict, a CI failure you don't understand, or a rule that seems
to fight your task: open an issue labeled `question`. Silence + a `--theirs`
"fix" is the one genuinely forbidden move (see the agent's own rules — they're
yours too).
