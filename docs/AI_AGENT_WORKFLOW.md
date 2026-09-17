# StudentLab — AI-Agent Branch Integration Procedure

[CURRENT] No code, no PRs, and no agent tooling exist yet (empty repository). This
document specifies the workflow the integration agent must implement and follow once
student branches/PRs exist. It is the [TARGET] procedure; its "hard rules" section
binds any human maintainer acting as merger too.

Mission: integrate student branches into `main` **semantically**, never blindly.
The agent is a *maintainer*, not a button: it judges, tests, and refuses.

## 1. Hard Rules — [DECISION]

1. **Never blindly merge.** No `gh pr merge` without the full §3 pipeline.
2. **Never choose `ours`/`theirs` wholesale.** Every conflicting hunk is resolved
   by reading both sides and re-synthesizing intent (documented in the merge notes).
3. **Never touch student branches destructively** — no force-pushes, no history
   rewrites of `student/*`; conflicts are resolved on an integration scratch branch
   `integrate/<pr-number>`.
4. **Never weaken security to make tests pass** — no disabling authz checks, no
   loosening `EXECUTION_*` limits, no `--no-verify`, no flaky-test deletion. If a
   security test fails, the PR is *rejected*, not "fixed around".
5. **Refuse over resolve** when in doubt: >15 conflicting files, changed
   public contracts (schemas/routes/limits) without the PR describing it, or any
   unresolved ambiguity ⇒ close-with-requests or escalate to the teacher (human).
   The agent's decision log records every refusal with evidence.
6. Everything the agent does is recorded: checks run, commands, outcomes,
   reasoning per resolved hunk (PR comment; persistent `docs/integrations/` log
   optional later).

## 2. Inputs & Preconditions

- Trigger: PR `ready for review`, authored from `student/<username>/*` (GIT_WORKFLOW
  §2 regex) targeting `main`.
- Preconditions checked first (fail fast, don't burn a pipeline):
  `gh pr view <n> --json headRefName,author,...`; PR is open, mergeable state
  known; **the final head state of the branch is what is reviewed — the agent may
  NOT rely on commit-by-commit history being what was originally approved**
  (students may have pushed/reorganized);
  branch last-updated < 7 days (stale ⇒ rebase request first).
- Fetch: `git fetch origin pull/<n>/head:pr/<n>` and `git fetch origin main`.

## 3. The 13-Step Pipeline — [TARGET]

### Step 1 — Inspect branch
`gh pr view <n>`; branch name pattern valid; PR template sections filled
(module/closes/checklist); linked issue exists if claimed; author's username
matches branch namespace (else the "ownership" model (§5) is moot → request rename).

### Step 2 — Inspect commits
`git log --oneline origin/main..pr/<n>`; Conventional Commits present (soft
signal); spot any commit message hinting at scope creep ("also fixed auth") —
flag for step 4/6 attention. History readability only; content is judged at head.

### Step 3 — Inspect diff
`git diff --stat origin/main...pr/<n>` then full `git diff`. Build the touched-file
set; classify each file: `own-module | registry | locked | docs | deps | migration`.

### Step 4 — Identify affected modules
Map files → modules (backend `<module>` per ARCHITECTURE.md §6 table; frontend
`features/<f>`). **Alert flags:** touched a locked path (GIT_WORKFLOW §6 list —
auto-reject to teacher unless PR carries the waiver issue), touched another
student's module files, added a dependency, changed `core/config` defaults,
touched `conftest.py`/`fixtures`, deleted a test, touched `permissions.py`.
Any alert ⇒ security-sensitive path, go to Step 12 gate.

### Step 5 — Identify conflicts (mechanically)
```
git switch -c integrate/<n> origin/main
git merge --no-commit --no-ff pr/<n>     # dry merge
git diff --name-only --diff-filter=U     # conflict list
```
Zero conflicts ⇒ Step 6 test passes may run on the merge state directly.
Expected conflict hotspots and their *meaning*: `alembic/versions` (two heads ⇒
serialize, §7), `router.py` / `routes/index.tsx` (both added lines ⇒ keep both,
re-sort), `package.json` deps (union, then Step 12 review), fixtures/conftest
(needs human-grade judgment; often ⇒ refuse).

### Step 6 — Run tests (pre-resolution baseline)
Run full backend + frontend lane **on the raw merge state** (§4 command set).
Purpose: separate "branch was broken" from "my conflict resolution broke it".
If the PR head alone fails the same way ⇒ bounce to student with log excerpt
(agent does NOT fix feature logic in student code — that's their learning).

### Step 7 — Resolve conflicts semantically
For each conflicted file: read *both* sides, the PR description, and the relevant
modules' specs in this `docs/`; re-implement the union of intents — e.g. two added
router includes ⇒ both, alphabetically; two model field additions ⇒ both + one new
combined migration if the conflict touched `alembic/versions` (then re-run heads
check). **Never** `git checkout --ours|--theirs` at file level (hard rule 2);
mechanical hunks (import ordering, both-add) may be auto-resolved only when the
agent can state *why* it's safe, in the log.

### Step 8 — (covered by 7) — never blind ours/theirs
Retained as its own numbered law for auditability; a reviewer should find zero
instances of `--ours`/`--theirs` in agent logs.

### Step 9 — Run tests again
Full §4 set must now be green — this is the resolution's proof, not Step 6's.

### Step 10 — Playwright E2E
`npx playwright test` (non-`@exec` lane) + `@exec` lane on docker runner
(TESTING.md §5.4/§7). E2E is required whenever the PR touches frontend features,
authorization, or execution; teacher may waive for docs-only PRs — the *only*
allowed skip, recorded.

### Step 11 — Frontend build & type checks
`npm run typecheck` (`tsc --noEmit` strict), `npm run lint`, `npm run build` in
`frontend/` — build failures (not just tsc) gate merge; bundle size regression
reported (soft).

### Step 12 — Security-sensitive verification
Checklist against SECURITY.md/EXECUTION.md, triggered by Step 4 alerts or any diff
in `core/ execution/ e2e/ .github/`:

- authz matrix rows still complete for new endpoints (TESTING.md §3); no new
  endpoint merged without its negative tests
- ownership checks present in every touched service (grep `can_access_`);
  no 403-instead-of-404 regression
- execution: limits/`network none`/cleanup assertions still in config tests;
  `subprocess` grep of `backend/app/` outside allowed zones (EXECUTION.md §8 law)
- no secret-looking strings added (`JWT_SECRET`, `AKIA…`, private keys) —
  gitleaks scan; no `.env`/`.db` staged
- diff adds no dependency, or dependency justified + pinned + advisory-checked
- CORS/cookie flags/`ENV` defaults unchanged unless the PR *is* that change (⇒ human)
Any doubt ⇒ **human escalation, not judgment call.** The agent proposes, the
teacher disposes, on security.

### Step 13 — Merge only when required checks pass
`integrate/<n>` is green end-to-end ⇒ push the integration branch, merge **the
integration branch** into `main` with the squash-style merge commit message
(`chore(merge): pr/<n> <title>` + PR/author trailers + conflict-resolution notes),
then `git push origin main`. GitHub-side: agent posts the §1 log as PR comment,
marks PR merged (or closes with reasons), deletes `integrate/<n>`, tags
per-release cadence (§ GIT_WORKFLOW §9).

## 4. Canonical Command Set (run identically every time)

```bash
# backend
cd backend && source .venv/bin/activate
alembic upgrade head && alembic heads            # exactly one head
ruff check . && ruff format --check . && mypy app
pytest -q -m "not e2e_lane"                        # unit + api incl. authz matrix
pytest -q -m e2e_lane --docker-ok-or-skip
# frontend
cd frontend && npm ci && npm run typecheck && npm run lint && npm run test -- --run && npm run build
# e2e
cd ../e2e && npm ci && npx playwright test [--grep @exec]
```

CI mirror (`ci.yml`) must run the same commands — agent and CI agree by
construction (TESTING.md §7).

## 5. Ownership & Sequencing Rules

- One PR per student per module-area; agent merges **oldest-first**
  (prevents the second mover always resolving). Two open PRs touching the same
  file? The newer one rebases *after* the older merges — the agent requests it
  rather than resolving 3-way against an in-flight branch.
- Migration serialization: if 2+ open PRs add revisions, the agent rebases and
  linearizes in merge order, re-running upgrade/downgrade tests (§ GIT_WORKFLOW
  §5, DATABASE.md §7).
- The agent NEVER merges a PR it can't map to this procedure's green states;
  "partially green" always means closed-with-asks, not "risk it".

## 6. Failure Playbooks — [TARGET]

| Situation | Action |
|-----------|--------|
| Student branch broken (Step 6 red) | bounce with excerpt + which doc section applies; no agent fixes to student logic |
| Semantic conflict in shared file > trivial (Step 7 unsure) | comment asking *both* students to align in the thread (teacher moderates); hold merge |
| Migration fork unresolvable safely | refuse, ask the later PR to rebase; provide the exact rebase recipe in the comment |
| E2E flaky (Step 10 fails once, passes on rerun) | 1 rerun only; second flake = block + quarantine tag + flake issue — never merge-with-quarantine silently |
| Security check doubt (Step 12) | stop; escalate to teacher with the exact file/line concerns; merge is off the table until human sign-off |
| main broke after merge | `git revert` on main immediately (not rewrites), reopen PR, incident note in log |

## 7. Extensibility — [TARGET]

- `scripts/integrate.sh` (or GitHub Actions `workflow_dispatch` job) automates
  Steps 1–6 + 9–11 mechanically, leaving 7/12 for judgment — build it in Step 9,
  *after* at least two manual runs calibrated the checklist.
- Same procedure covers `feature/*` PRs (teacher's own) minus ownership checks.
- If per-student repos appear (GIT_WORKFLOW §8 open item), Step 1–2 add
  `git -C ../repo fetch` before pipeline; nothing else changes — the interface
  between "branch" and `main` is all this procedure depends on.
