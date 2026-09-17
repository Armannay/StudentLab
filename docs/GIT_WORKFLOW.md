# StudentLab — Git Collaboration Workflow

[CURRENT] The repository has **one commit** (`3abc304 Initial commit`, README
only), one branch (`main`), one remote (`origin → https://github.com/Armannay/StudentLab`,
**public**), no tags, no CI, no CODEOWNERS, no `.gitignore` until Step 0. So: no
existing Git-related files to preserve; everything below is [TARGET] and becomes
binding the moment student code arrives.

## 1. The Local-First Flow (why this repo looks unusual) — [DECISION]

```
GitHub (origin/main)  ←── merges only via PR
      │  git pull (teacher's machine)
      ▼
Teacher's clone ──► runs backend+frontend locally (docker compose up)
      ▼
Teacher reviews the platform in a browser — no hosting, no domain, by design
```

Consequences:

- The repo **is** the deliverable; a merged PR is "deployed" as soon as the teacher
  pulls and restarts. There is no environment to promote to ⇒ release process is
  just **tags**: `step-N`, `v0.x` annotated tags on main.
- `main` must *always* run on a fresh clone (`git pull && docker compose up` is a
  non-negotiable invariant; CI enforces build+test, teacher enforces by using it).
- Secrets never enter this public repo: `.env` gitignored (done in Step 0),
  `.env.example` committed from Step 1; GitHub repo stays **public-safe forever**
  (students' code is in it too).

## 2. Branch Model — [TARGET]

```
main ─────────────●────────────●──────────►   protected, always runnable
                   \          ▲
 feature/<area>-<slug> ───────┤  teacher/platform work (rarely external)
 student/<username>/<slug> ───┘  students' work, the norm
                    (each from and back to main via PR + AI-agent gate)
```

- Naming (enforced by branch-protection regex + CI check):
  `student/<username>/<slug>`, username = lowercase `users.username`, slug =
  kebab-case ≤ 4 words, e.g. `student/sara/calendar-day-click`. One PR = one
  student = one branch; short-lived (target ≤ 3 days open — rebase discipline).
- Students never branch off other students' branches. Cross-student dependency ⇒
  raise an issue; teacher sequences it.
- Student branches live on `origin` (pushed), not just laptops — the AI agent and
  teacher must see them.

## 3. main Protection — [TARGET] (GitHub settings, do in Step 1)

Repo is public on GitHub Free ⇒ these are available:

- Branch protection for `main`: require PR (no direct push, also no push of
  merges by non-admins), require status checks (the CI suite), require branch
  up to date before merge, forbid force pushes & deletions.
- `.github/CODEOWNERS` (review request enforcement):
  `*  @Armannay` default; scoped entries per §6 ownership; students then *cannot*
  self-approve shared areas.
- Merge method: **squash only** ⇒ main history = one commit per PR = exactly the
  granularity the AI-agent workflow (AI_AGENT_WORKFLOW.md) reasons about.
  Message = PR title (Conventional Commits enforced by CI check).

## 4. Commits & PRs — [TARGET]

Commits: Conventional Commits, `<type>(<module>): <summary>`;
types: `feat fix test docs refactor chore ci`. Scope = module dir name
(`projects`, `execution`, `frontend-teacher`, …). Body = *why*, bulleted,
no requirement to mirror the diff.

PR template (`.github/PULL_REQUEST_TEMPLATE.md`, Step 1):

```markdown
Module: [projects|reports|calendar|execution|...]
Closes: #
- [ ] Feature lives only inside my feature module(s) (see CONTRIBUTING.md map)
- [ ] Backend: service tests + API tests incl. authorization-matrix rows for any new endpoint
- [ ] Frontend: no new shared-component file; StudentTab untouched unless assigned
- [ ] Migration included (if models changed) + heads check green after rebase
- [ ] No edits to protected areas (GIT_WORKFLOW.md §6) — or explicit waiver link
- [ ] Local: pytest, tsc, build, `npx playwright test --grep <tag>` all actually run
Screenshots / notes:
```

Size: ≤ ~400 changed lines or split. Students push early as **draft PR** (CI runs,
teacher sees trajectory).

## 5. Conflict-Minimizing Ownership Map — [DECISION]

The architecture (ARCHITECTURE.md §4–§5) exists largely to serve this table:
module-per-feature means the *typical* two student PRs touch disjoint files.

| Area | Edit rights | Conflict risk & mitigation |
|------|------------|----------------------------|
| `backend/app/{models,schemas,repositories,services}/<own module>*` | owning student | low by construction |
| `frontend/src/features/<own feature>/**` | owning student | low |
| `backend/app/api/v1/router.py` (one include line/module) | any student, **teacher/agent review required** | the designed choke point: one-line diffs; sort alphabetically to make them near-deterministic |
| `frontend/src/routes/index.tsx` (one route line/module) | same | same |
| `frontend/src/components/*` (shared primitives) | teacher/agent only | high — hence locked |
| `backend/app/core/**`, `app/execution/**` | teacher/agent only | locked (§6) |
| `alembic/versions/` | owning student adds; **rebase + `alembic heads` before merge** | the #1 expected conflict: linear revision chain ⇒ agent merges serially, oldest PR first (AI_AGENT_WORKFLOW §7) |
| `pyproject.toml` / `package.json` (deps) | request-via-issue, teacher/agent applies | see §6; lockfiles never hand-edited |
| `docs/**` | anyone, prose-only, review-required | merge-conflicts cheap to resolve |
| `.github/**`, `docker-compose.yml`, `Dockerfile*`, `e2e/**`, root config | teacher/agent only | locked (§6) |

## 6. Files Students Must Not Modify — [TARGET] (enforced via CODEOWNERS; CI check duplicates the list)

```
backend/app/core/**           backend/app/execution/**       backend/app/main.py
backend/app/api/v1/router.py  (may ADD one line; not reorder/edit others)
backend/alembic/env.py        backend/alembic/script.py.mako
docker-compose*.yml           runner/Dockerfile              Dockerfile*
.github/**                    e2e/**                         .gitignore  .editorconfig
frontend/src/services/api.ts  frontend/src/types/**          frontend/vite.config.ts
frontend/tsconfig*.json       frontend/eslint.config.*       package-lock.json
**/pyproject.toml  **/requirements*.txt  **/package.json    (dependency changes = teacher)
```

Rationale: authz plumbing, sandbox config, CI, and test infra are exactly where a
"helpful" student edit silently disables a security control (SECURITY.md §3,
EXECUTION.md §4). A student needing a change there: open an issue; the teacher or
the AI agent implements it in a `feature/` branch.

**Special-review triggers** (any diff touching these ⇒ second reviewer named in PR):
`login`/cookies/token code, `permissions.py`/deps, anything under `execution/`,
CHECK/FK/constraint changes, new dependencies (public repo ⇒ supply-chain care),
CI/workflow edits, `settings` defaults that weaken limits (e.g. timeout, network).

## 7. Student Daily Workflow — [TARGET] (also in CONTRIBUTING.md)

```bash
git switch main && git pull            # always start fresh
git switch -c student/sara/report-edit
# work; small commits; run `pytest` / `npm run check` locally (they must PASS —
# report honestly, never claim green that wasn't run)
git push -u origin student/sara/report-edit        # → draft PR early
# teacher/agent comments → push commits (no force-push to own PR? allowed on
# your branch pre-merge, discouraged: keep history readable for review)
git fetch origin main && git rebase origin/main    # before "ready", esp. if migrations
# mark ready for review → checks + approval → squash merge by maintainer/agent
```

## 8. Multi-Student Git Plan (where this is going) — [TARGET, forward-looking]

Student **code projects** are tracked on their own branch namespace
(`student/<username>/<feature>`) inside this same repo (simplest for one class;
split to per-student repos only if projects outgrow the monorepo — decision Step 5+
noted as open). Convention that keeps the graph merge-friendly:

- Each student's features live under their feature folders only (§5) ⇒
  `git merge` of two students' branches is mechanically trivial; **integration
  conflicts are rare by architecture, not by luck**.
- Integration into `main` is done **exclusively** via the AI-agent procedure
  (AI_AGENT_WORKFLOW.md) or teacher — students never merge main *into* their
  branch with `-s ours` shortcuts, never resolve others' conflicts.
- Long-lived branch per student for their *project source* (e.g.
  `student/sara/project-maze`) is optional and owned by the student — the platform
  records it in `student_profiles.git_branch` / `projects.source_ref` (DATABASE.md
  §3) so execution can pin it (EXECUTION.md §7).

## 9. Housekeeping — [TARGET]

- `.editorconfig` + `ruff format`/`prettier` on staged files (tooling Step 1) —
  formatting debates banned by automation, not by policy.
- Stale student branches: agent/teacher deletes on merge (`delete branch on merge`
  setting on). Branches > 14 days with no activity ⇒ issue pinged, then prune.
- Commit hygiene: no merge commits on main (squash-only achieves this);
  `git log --oneline main` readable as the project's changelog ⇒ release notes
  from `git range-log` later.

## 10. Git-Related Risks — [CURRENT audit]

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Student commits `.env`/`.db` | high (no `.gitignore` existed before Step 0 — now added) | `.gitignore`; secret-scan in CI (gitleaks optional); teacher history-rewrite runbook documented in CONTRIBUTING |
| Students push to main | medium | §3 protection — **not yet in place** (empty repo ⇒ do it before first student) |
| Migration-chain conflicts | near-certain once 2+ students add columns | §5 rule + CI heads check + agent serializes |
| Shared-registry churn (router/route files) | certain early | designed one-line pattern §5; alphabetical ordering |
| Lockfile conflicts | medium | deps via teacher (PR template, §6) |
| Force-push overwrites on student branch | medium | pre-merge allowed but agent must diff-review final state, not commit history (AI_AGENT_WORKFLOW §2) |
| Public repo exposes student code | certain | consent + assignment design question for the teacher — flagged as human decision, report §13 |
