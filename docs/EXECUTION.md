# StudentLab — Project Execution Architecture (Docker Sandbox)

[CURRENT] **Nothing is implemented.** No `backend/app/execution/`, no Dockerfile, no
`docker-compose.yml`, no `docker` SDK dependency, no execution code path of any kind.
Every statement below is [TARGET]; §9 states why a specific tempting shortcut is
forbidden; §11 lists the "done when" criteria for the implementation step.

## 1. Purpose & Threat Statement — [DECISION]

Students will submit **arbitrary untrusted code** (`entrypoint` file plus its
imports) for execution, written by semi-adversarial users (SECURITY.md §0), on the
teacher's own machine. The platform's whole security failure budget is spent here:
one escape, one `rm -rf ~`, one mined-crypto incident, and the project loses trust.
Therefore execution is: **out-of-process always, containerized always, capped
always, destroyed always.**

## 2. Component Architecture — [TARGET]

```
Browser ──POST /projects/{id}/executions──► FastAPI route (thin)
      │                                        │
      │                                        ▼
      │                            execution_service.py
      │                            (ownership, quota, workspace check,
      │                             INSERT Execution(PENDING), enqueue)
      │                                        │  in-process asyncio queue (bounded)
      │                                        ▼
      │                              runner.py (worker, lifespan-managed)
      │                              materialize workspace → temp dir
      │                                        │
      │                                        ▼
      │                              Executor interface (base.py)
      │                              ├── DockerExecutor  (v1, the only one)
      │                              └── (NullExecutor / dev fake — tests only)
      │                                        │ docker SDK (blocking calls in threadpool)
      │                                        ▼
      │                              container: studentlab-runner:<tag>
      │                              one run = one container, never reused
      │                                        │
      └── poll GET /executions/{id} ◄── state rows: PENDING→RUNNING→terminal
```

Key property: the FastAPI request handler **returns 202 before the container
starts**. No execution API call can block a worker thread for the duration of a run.

## 3. Interface Contract — [TARGET]

```python
# backend/app/execution/base.py  — stable; changes here are teacher-review-only
@dataclass(frozen=True)
class RunSpec:
    execution_id: int
    workspace: Path          # materialized, read-only-at-source, per-run copy
    command: tuple[str, ...] # e.g. ("python", "main.py") — argv, never a shell string
    env: dict[str, str]      # allowlisted; user env NEVER inherited
    limits: ResourceLimits   # below
    output_cap: int = 65536  # bytes per stream

@dataclass(frozen=True)
class ResourceLimits:
    cpu: float = 1.0         # in docker --cpus units (1.0 = one core)
    memory: str = "512m"
    pids: int = 128
    timeout: int = 60        # wall-clock seconds
    tmpfs_size: str = "64m"

class Executor(Protocol):
    def run(self, spec: RunSpec, on_output: Callable[[bytes, bytes], None] | None = None)
           -> RunResult: ...

@dataclass(frozen=True)
class RunResult:
    status: ExecutionStatus  # succeeded|failed|timeout|error|cancelled
    exit_code: int | None
    stdout: bytes            # already capped
    stderr: bytes            # already capped
    truncated: bool
```

`NullExecutor` (returns canned results) lets unit tests verify state machine,
persistence, and caps with **no Docker dependency** — required for CI lanes that
lack Docker. `DockerExecutor` integration tests run only when
`RUNNER_BACKEND=docker` + healthz reports docker ok; CI provisions it (§10).

## 4. Isolation Model — [TARGET] (the hardening contract)

Each knob below is a `docker create` argument, asserted by a unit test that builds
the expected config dict (`test_docker_spec.py` — CI-verifiable without Docker).

| Concern | Contract | Rationale |
|---------|----------|-----------|
| Network | `network_mode=Disabled` (`--network none`) | Student code gets zero egress: no exfil, no scanning the teacher's LAN, no pip-installing mid-run |
| Filesystem | Rootfs read-only (`read_only=True`); workspace mounted rw **as a copy** in a per-run tmpdir; `/tmp` = tmpfs 64m (`--tmpfs /tmp:size=64m,mode=1777`) | The *only* writable student surface is their own workspace copy + tmpfs; the real project dir is never mounted; a run that writes garbage loses it on cleanup |
| Capabilities | `cap_drop=["ALL"]`, `security_opt=["no-new-privileges"]`, **never** `privileged`, never `--pid=host`, never `--net=host`, no `devices` | Default-deny; any future image needing more gets an explicit review |
| User | Image runs `USER 1000:1000`; `user: "1000:1000"` set explicitly; host uid mapped — files the run creates in the workspace copy are chowned/deleted by the app uid | Non-root in-container is mandatory; root-in-container + kernel vuln = host root |
| CPU | `nano_cpus = cpu * 1e9` (`--cpus`) | A `while True: pass` costs one core, not the machine |
| Memory | `mem_limit` 512m default, no swap (`memswap_limit == mem_limit`) | OOM is contained: kernel kills the container's process → status `failed`, exit 137 |
| Process count | `pids_limit=128` | Fork bombs die at the cgroup limit |
| Timeout | Hard wall-clock: worker waits `timeout+5s` grace, then `container.kill()` + record `timeout`; enforced **server-side**, never trusts in-container signals | No run can wedge the single worker forever |
| Output | Reader thread per stream; stops at `output_cap` (64 KiB), sets `truncated`; infinite output is *also* covered by timeout | Protects DB + memory + UI (DATABASE.md executions.stdout cap) |
| Env | Container sees only `env` from spec (allowlist: nothing by default + `PYTHONUNBUFFERED=1`) | `JWT_SECRET`, DB creds, teacher's shell env never cross the boundary |
| Image | Pinned digest of locally-built `studentlab-runner:py3.12` (Dockerfile in `runner/`, reviewed like backend core). **Students never supply images/Dockerfiles**; allowlist of interpreters = {python3.12}; node etc. later via new images, not new flags | Image supply chain is the one remaining trust root — keep it small and versioned |
| Command | `argv` tuple only; `shell=False` everywhere, including the materialize/copy step (uses `shutil.copytree`, not `rsync`/git-archive-with-shell) | Command injection via title/entrypoint/filename is the classic sandbox-adjacent bug |

Workspace materialization (`workspace.py`): for v1, project code lives under
`WORKSPACE_ROOT/<workspace_dir>` (teacher provisions from the student's git branch;
future: automated `git archive <source_ref>` from `source_repo` per §7 ownership).
The copy step excludes `.git`, symlink-escapes (refuse on detection — validated, not
just excluded), files > 1 MiB/total > 10 MiB (quota: 100 files), and non-allowlisted
extensions inside the *run tree*. Quota numbers are config (`EXECUTION_*` env keys),
defaults as above.

## 5. Lifecycle — [TARGET]

1. Route → service: ownership ✓, project active ✓, quota ✓, same-project double-run
   guard (one RUNNING per project) → insert `PENDING`, `queue.put(execution_id)` → 202.
2. Worker: mark `RUNNING` (`started_at`) → materialize workspace copy → build
   `RunSpec` → `DockerExecutor.run` (threadpool) → stream/cap output →
   on completion: persist terminal status, `exit_code`, capped streams,
   `finished_at` → **always** `finally:` remove container (`force=True`) and delete
   temp workspace (`ignore_cleanup_errors` + sweeper).
3. Cancel: `runner.cancel(execution_id)` → kill + `CANCELLED`; racing completion wins
   only if terminal already set (state machine below).
4. Crash recovery: on app start, worker marks leftover `PENDING/RUNNING` rows
   `ERROR("orphaned: server restarted mid-run")` — simple, no distributed promises.
5. Sweeper: hourly, deletes orphan `WORKSPACE_ROOT/.runs/*` older than 1 h and
   `docker ps -aq --filter label=studentlab.execution` leftovers.

State machine (DB CHECK mirrors DATABASE.md):

```
PENDING ─► RUNNING ─► SUCCEEDED   (exit 0)
                 ├──► FAILED       (exit != 0, incl. OOM 137)
                 ├──► TIMEOUT      (wall-clock kill)
                 └──► CANCELLED    (user/teacher cancel)
PENDING/RUNNING ──► ERROR          (platform-side: image missing, docker down,
                                    workspace materialization failed, quota race)
```

`ERROR` vs `FAILED` distinction is a contract with the UI: "your code failed" vs
"the platform could not run you" — do not collapse them.

## 6. Fairness & Capacity (classroom reality) — [TARGET]

- Single worker lane per student concurrency: `EXECUTION_MAX_CONCURRENT=2`
  (global), queue depth 50 ⇒ one student's tight loop can't starve the class.
- Quota: max 10 runs / 10 min / student (in-memory sliding window; reset with
  restart — acceptable for classroom; persistent quota is a later option).
- A full queue ⇒ 429 with `Retry-After`.

## 7. Project Ownership & Future Git Flow — [TARGET, forward-looking]

Today: teacher clones the class repo; each student's project code lives on their
branch (`GIT_WORKFLOW.md`). `Project` stores the reference (`source_repo`,
`source_ref`), never the code — the DB stays a metadata store, the git history stays
the truth. Flow the schema already supports: materialization becomes `git -C
<repo> archive student/<name>/<branch> -- <project subdir>` → temp copy.
Ownership invariant across that change: **an execution's visibility is derived
solely from `execution → project → student_id`** (DATABASE.md §3), so the security
check never has to learn about git.

## 8. Why Direct `subprocess`/`exec` Inside the API Server Is Unacceptable — [DECISION]

Enumerate the "simple" alternative (`asyncio.create_subprocess_exec("python",
main.py, cwd=workspace)` next to FastAPI) and why each mitigation is itself harder
than Docker:

1. **Same trust domain**: the child inherits the server's uid — student code can
   read `backend/studentlab.db` (every student's data), `JWT_SECRET`, the teacher's
   `~/.ssh`, `~/.aws`; delete or rewrite anything; `git push` to main with the
   teacher's own credentials. The whole SECURITY.md threat model collapses on
   purpose.
2. **Filesystem**: `cwd` is not a boundary. `chroot` needs root;
   Landlock/seccomp/`ulimit` per-process sandboxing on Linux is
   kernel-version-dependent, untestable across teacher laptops (Windows/macOS
   Docker Desktop vs Linux CI), and a project of its own. Python can `os.walk("/")`
   from the same uid regardless.
3. **Resources**: `RLIMIT_*` misses process-tree cases without cgroups; no clean
   per-run cpu accounting; killing a timeout reliably requires a fresh process
   *group* and still races; memory limits on macOS/Windows via Python: effectively
   none.
4. **Environment leak**: every env var and open file descriptor unless you rebuild
   `env={}` and `close_fds` and re-audit the stdlib (temp dir, PYTHONPATH injection
   from cwd, `sitecustomize.py` import hijack from the workspace!). Docker's
   defaults get the boundary right by construction.
5. **Blast radius = teacher's machine**: this platform runs *on the teacher's*
   laptop/desktop by design (local-first) — unsandboxed execution hands the class
   root-adjacent code execution on it. That is disqualifying on its own.
6. **Auditability**: containers give one clean artifact per run (image digest,
   limits as created, removal on exit) matchable to the DB row; host processes give
   none.
7. **Testability**: DockerExecutor is testable without Docker via config
   assertions (§3); a host-subprocess "sandbox" can only be tested by attempting
   escapes on the dev machine.

Conclusion recorded as a law of the codebase: student code never executes in the
API process or as a child of it. `subprocess` module usage in `backend/app/`
outside `execution/` (and outside teacher-run admin `git fetch`) is a
**review-blocking finding**.

## 9. Runner Image — [TARGET]

`runner/Dockerfile` (owned by teacher, CODEOWNERS): base `python:3.12-slim` pinned
by digest, non-root `USER 1000:1000`, no pip installs from students, no network at
build beyond pinned base, `ENTRYPOINT` unset (command comes from spec argv),
labels `studentlab.runner=1`. Built in CI, sha-pinned in config, digest recorded on
the `executions.image_digest` column.

## 10. Configuration Surface — [TARGET]

```
EXECUTION_ENABLED=true|false        # master switch; false ⇒ endpoints return 503+clear msg
EXECUTION_BACKEND=docker|null
EXECUTION_IMAGE=studentlab-runner:py3.12@sha256:...
EXECUTION_TIMEOUT_SECONDS=60
EXECUTION_MEMORY_LIMIT=512m
EXECUTION_CPU_LIMIT=1.0
EXECUTION_PIDS_LIMIT=128
EXECUTION_MAX_CONCURRENT=2
EXECUTION_OUTPUT_CAP_BYTES=65536
EXECUTION_WORKSPACE_ROOT=./workspaces
EXECUTION_QUOTA_RUNS=10
EXECUTION_QUOTA_WINDOW_SECONDS=600
```

All through `Settings` (pydantic-settings), every key unit-testable; startup
validates limits (e.g. timeout ≤ 300, memory ≤ 2g) so a fat-fingered env can't
silently create an unlimited sandbox.

## 11. Implementation Definition of Done (for the future step) — [TARGET]

- [ ] `Executor` protocol + `DockerExecutor` + `NullExecutor`; no FastAPI imports in `execution/`.
- [ ] Config-assertion tests for every §4 row (runs in any CI).
- [ ] State-machine tests (transitions, orphan recovery, cancel race) with NullExecutor.
- [ ] Quota/queue-depth tests.
- [ ] Integration lane (`RUNNER_BACKEND=docker`): hello-world succeeds; infinite loop → TIMEOUT; fork bomb → PIDs-capped fail; `/etc/passwd` open → fails inside container; egress `ping` → fails; output 10 MB → capped+truncated; container+tmpdir gone after run.
- [ ] Playwright: student runs project → status flips → output visible in ExecutionHistory.
- [ ] `docker_unavailable` surfaces as `ERROR` + healthz red, app otherwise healthy.

## 12. Extensibility (future, do not pre-build) — [DECISION]

Other runtimes = new images + allowlist entries in `RunSpec` validation (same
`Executor`). Multiple lanes/parallelism = replace in-process queue with a table-based
poller (schema already durable). Post-containerd/Kata/Firecracker = swap
`DockerExecutor` for a stricter one behind the same protocol. "Trusted projects"
(opt-in teacher network) would need a new explicit flag in the image policy — never
an absence of policy.
