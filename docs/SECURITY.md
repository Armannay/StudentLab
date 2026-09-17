# StudentLab — Security Specification (AuthN, AuthZ, Threats)

[CURRENT] **No authentication or authorization exists** — no users table, no login,
no session handling, no middleware (repository contains only `README.md`). This is
a [TARGET] specification. Nothing here is enforced today because nothing runs today.

## 0. Threat Model in One Paragraph — [DECISION]

The threat model is **not** "the internet is attacking us". It is: a classroom of
students, each holding full read access to the frontend source and free access to
the local network/port the API listens on. Students are *semi-adversarial*:
curiosity-driven, technically capable, and explicitly learning to break things.
Everything the browser can see is public; therefore **the browser enforces nothing**.
Secondary threats: untrusted *code* execution (the main one — EXECUTION.md),
secret leakage through Git (the repo is public on GitHub), and multi-user
data visibility on one shared local instance. The teacher's machine itself is
trusted enough to store the DB and run Docker, not trusted enough to run student
code unsandboxed.

## 1. Roles — [TARGET]

```
role = "student" | "teacher"        (users.role, CHECK-constrained)
```

No finer-grained roles in v1 — [DECISION]. Capability matrix:

| Capability | student | teacher |
|------------|:-------:|:-------:|
| Login/logout, `/auth/me` | ✅ | ✅ |
| Read own student profile | ✅ | ✅ (any) |
| Read another student's profile | ❌ 404 | ✅ |
| Create/update/delete own projects | ✅ | ✅ (any) |
| Touch another student's project | ❌ 404 | ✅ |
| Trigger execution on own project | ✅ | ✅ (any) |
| Read execution output | own | all |
| Create/edit own activities (reports) | ✅ | ✅ (any) — **see grading-integrity note, §5.4** |
| Read any student's calendar | ❌ | ✅ |
| Teacher dashboard, user admin, password reset | ❌ 403 | ✅ |
| Read `GET /api/docs` (OpenAPI) | ✅ | ✅ (local tool; docs are not a secret) |

Two enforcement rules make this real:

1. **Role checks** at the dependency level (`require_teacher`) for *collections*
   (roster, dashboard, user admin).
2. **Ownership checks** in the *service layer* for every single-resource access:
   `student_id` of the row vs `current_user`. Role gates say "students may use
   `/projects/{id}`"; only the ownership check says "…and {id} must be yours".

## 2. Session & Login Design — [TARGET]

| Item | Value |
|------|-------|
| Algorithm | JWT HS256 via PyJWT; secret `JWT_SECRET` (≥32 random bytes, env-only; dev fallback generated into gitignored `backend/.dev/JWT_SECRET` on first boot and warned about loudly) |
| Access token | `sub=user_id`, `role`, `iat`, `exp` **30 min**, `jti`, `typ:"access"` |
| Refresh token | `typ:"refresh"`, **7 days**, stored hash (SHA-256) in `refresh_tokens` table with `family_id`; **rotation on every use**; reuse of a rotated token ⇒ whole family revoked (theft signal). Logout = revoke family |
| Transport | Both tokens in `httpOnly` cookies (never in JS-readable storage): `sl_access` (Path=/, SameSite=Lax), `sl_refresh` (Path=/api/v1/auth). No `Secure` on http localhost; **config flip required before any LAN exposure** (§8) |
| CSRF | SameSite=Lax + **`X-Requested-With: StudentLab` header required on all unsafe methods**, checked in middleware (double-submit is unnecessary because no token is readable by JS) |
| Login flow | `POST /auth/login` → verify argon2id hash → issue family. SPA then re-fetches `/auth/me` (no user object trusted from the login response) |
| 401 handling | API returns 401; SPA refreshes once, else redirects `/login`. `ProtectedRoute` in `routes/` gates all app routes; the route guard is UX, backend checks are security |
| Failed-login UX | Uniform 401 `unauthenticated` for unknown user or bad password; identical response shape/timing; user enumeration via `GET /users?username=` is impossible (endpoint teacher-only) |
| Brute force | Local scope ⇒ modest defense: per-username counter in memory — 10 failures/15 min ⇒ 5 min lockout. No IP blocking (localhost would lock out the whole class) |

Password rules (v1): Argon2id (`argon2-cffi`, default params), min length 10, no
composition rules (teacher-issued initial passwords; students are told not to reuse
important passwords). `password_hash` never leaves the process: no schema field, no
log line, no fixture dump — CODEOWNERS review catches regressions.

## 3. Authorization Implementation (contract for Step 3) — [TARGET]

```
core/deps.py
  get_current_user(token cookie) -> User        # 401 if absent/expired/bad
  require_teacher(user) -> User                 # 403 if role != teacher

core/permissions.py   (pure functions, unit-testable, no FastAPI imports)
  can_access_student(user, student_id) -> bool
  can_access_project(user, project) -> bool     # project.student_id == user.student_profile.id or teacher
  can_access_activity(user, activity) -> bool
  can_access_execution(user, execution) -> bool # via execution.project
```

Service-layer pattern (every single-resource read/write):

```
obj = repo.get(id)
if obj is None or not can_access(user, obj):
    raise NotFoundError()          # 404 — deliberately, see below
```

**404-over-403 rule for cross-student access:** returning 403 would confirm the row
exists → sequential probing of `/students/{id}` becomes an enumeration oracle.
Students get 404 for *everything* not theirs. Teachers get real 403/404 on
teacher-scoped routes. This rule is a [DECISION]; the E2E scenario
"Student A cannot access Student B" asserts 404 specifically.

Collection list endpoints do not rely on path ids at all: the service **forces**
`student_id = current_user`'s for students and ignores/overrides any client-supplied
value — belt (scoping) and braces (per-row checks) both required by tests.

Anti-patterns forbidden in review: `@login_required`-style single global dependency;
permission decisions inside React (hiding buttons ≠ security — fine as UX, never as
the control); DB views instead of ownership checks; trust in the frontend-sent
`student_id`.

## 4. Student Tab Exploit Walkthrough (normative example) — [DECISION]

Student Sara (id 7) edits the URL `/teacher/students/8` → client-side guard renders
403 — *this layer is cosmetic and may be bypassed*. Sara calls
`GET /api/v1/students/8` with her own valid cookie → `student_service.get(8)` →
`can_access_student(sara, 8)` false → **404**. Same for `/projects/{id}`,
`/activities/{id}`, `/executions/{id}`, and `/calendar/8/month`. Forging
`student_id=8` in query/body → silently scoped to 7. The only field students send
that identity-relevant state hinges on is the *path id*, and that is checked.

## 5. Data & Privacy — [TARGET]

1. **SQLite file contains every student's reports in plaintext.** Accepted for a
   single-teacher local machine; document to the class; file permission `0600` set
   by seed script. Do not commit `*.db` (root `.gitignore` already prevents —
   added Step 0).
2. `.env` files excluded by `.gitignore`; only `.env.example` (no secrets, only
   keys/defaults) is committed — created in Step 1.
3. Execution stdout/stderr can contain secrets students printed; capped tails are
   stored and visible to teacher — class policy: "don't print credentials"; no
   attempt at secret-scanning in v1.
4. **Grading integrity:** teachers can edit student reports (§1 matrix) — a
   deliberate simplification. If auditability matters, add `edited_by_user_id` on
   `activities` in a later step; flagged as human decision.
5. Logs: request-id + user-id + route only; never bodies (login), never outputs.
   No analytics, no external calls — the app must run fully offline (local-first).

## 6. IDOR & Enumeration Checklist — [TARGET]

Every new endpoint must answer: (a) which user may call it? (b) which rows may they
see? (c) what happens with a forged id → 404? (d) what happens with forged
`student_id` in body/query → scoped? The API test template in TESTING.md §3
implements exactly these four assertions per resource and is mechanically applied
when endpoints are added.

## 7. Project Source & Files (git-adjacent risks) — [TARGET]

- `projects.source_repo` is validated against an allowlist pattern: repo under
  `github.com/Armannay/` or `WORKSPACE_ROOT`-relative path. `source_ref` must match
  the requesting student's own branch namespace (`student/<username>/*`) or be the
  shared base ref — enforced in `project_service` (see API.md §6.3).
- The API **never** accepts: file uploads, absolute paths, shell strings, Docker
  images, "entry commands". `entrypoint` must match `^[\w./-]{1,200}\.(py|js)$` and
  resolve inside the materialized workspace (path-traversal guard in
  `execution/workspace.py`; tests assert `../../etc/passwd` and absolute-path
  rejection).
- Git operations (future integration): only `git -C <workspace> fetch/checkout` with
  argument lists (never `shell=True`), never store GitHub tokens in the app DB —
  the teacher's own git credentials (agent/SSH) operate outside the app.

## 8. If This Ever Leaves Localhost — [TARGET, dormant]

Required flips before LAN or public exposure (log them, this list is the tripwire):
`Secure` cookie flag + HTTPS termination; real CSRF token; login rate limit by IP;
CORS locked to exact origin; consider moving DB to Postgres (DATABASE.md §8).
**Until then, "local-first, no domain" is a security assumption, not just a
convenience** — documented in README so nobody silently changes it.

## 9. Docker Execution Security — cross-reference — [TARGET]

The container hardening contract, resource caps, and the case against
`subprocess` live in **EXECUTION.md** (authoritative there; §8 output caps feed
DATABASE.md `executions`). Summary of non-negotiables: `--network none`, read-only
rootfs + tmpfs workdir, non-root user, `--cap-drop ALL`, `--security-opt
no-new-privileges`, cpu/mem/pids limits, wall-clock timeout with kill, output cap
64 KiB/stream, destroy-after-run, project directory owned by student ⇒ only that
directory ever enters the container.

## 10. OWASP Top-10 Mapping (for this app specifically) — [TARGET]

| OWASP | StudentLab exposure | Control |
|-------|--------------------|---------|
| A01 Broken access control | cross-student ids, teacher routes | §1–§4; negative E2E scenarios mandatory |
| A02 Crypto failures | password storage | argon2id; JWT secrets env-only; no custom crypto ever |
| A03 Injection | SQL via ORM only; no shell in request path; path traversal §7 | SQLAlchemy params, arglists, allowlist validators |
| A04 Insecure design | executing untrusted code as core feature | EXECUTION.md is the design review of that decision |
| A05 Misconfiguration | debug mode, open CORS, default secret | prod-like CI build; config fails startup if `JWT_SECRET` missing in non-dev; `/docs` accepted as public-by-design (rationale API.md §1) |
| A06 Vulnerable components | unpinned deps | lockfiles + Dependabot enabled in Step 1 (public repo ⇒ free) |
| A07 Auth failures | brute force, session fixation | §2 table; rotate on login |
| A08 Integrity failures | unreviewed merges of student PRs | GIT_WORKFLOW + AI_AGENT_WORKFLOW gates |
| A09 Logging failures | no audit trail | §5.5; request-ids; execution rows are the real audit log |
| A10 SSRF | `source_repo` fetch by URL | §7 allowlist; v1 fetches only from github.com/Armannay allowlist |

## 11. Security Open Questions (human decisions)

1. LAN exposure now or never? (Changes §2 cookie `Secure`, §8.)
2. Password self-service change/reset for students?
3. Teacher editing student reports — acceptable trust model, or require the
   `edited_by` column from day one? (§5.4)
4. Are `stdout` tails kept forever? (Storage + privacy; suggest a prune policy in
   Step 7.)
