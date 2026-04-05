# Architecture Details

## Information Barriers

The holdout barrier is what makes the verification model work. Coders can't teach to the test if they can't see the test.

```
Coder agents       CAN access: plan.md, nlspec.md, project code, convergence feedback
                   CANNOT access: scenarios/

Scenario Runner    CAN access: scenarios/, live app (browser only)
                   CANNOT access: plan.md, project source code, coder output

Scenario Designer  CAN access: nlspec.md
                   CANNOT access: plan.md, coder output
```

**Primary barrier**: Physical repo separation. Scenarios live in `beastmode/runs/{project-name}/{run-id}/scenarios/`. Coders work in the application repo. They're separate Git repositories on separate GitHub remotes.

**Defense-in-depth**: PreToolUse hooks in `hooks.json` intercept any Read/Grep/Glob/Bash call that targets `scenarios/` from non-privileged agents.

## Task Backend Abstraction

The daemon uses the `TaskBackend` abstract interface to interact with the board.

**BeastMode Board** — a built-in FastAPI + SQLite board with WebSocket real-time updates. Runs as a Docker container alongside the daemon. Features:
- Kanban UI with dark/light theme, drag-and-drop status changes
- HTML-formatted pipeline updates (the `board_messages.py` module centralizes all update templates)
- File attachments with inline image thumbnail rendering
- Proof screenshots from Playwright prod verification uploaded automatically
- Zero external dependencies — everything runs in Docker Compose

The board backend implements: `poll_all_items()`, `update_status()`, `post_update()`, `get_updates()`, `create_item()`, `set_column_value()`, `upload_attachment()`. Pipeline code is backend-agnostic.

## Daemon Priority System

The daemon processes tasks in strict priority order:

| Priority | Status | Action |
|----------|--------|--------|
| **0** | **Critical priority** (any actionable status) | Tasks with "Critical" priority (determined by the board item's priority field) jump the queue. Routed to the correct handler based on current status. |
| 1 | Approved & Merge to Main | Merge PR, wait for CI, deploy, verify production |
| 1.5 | Verifying Prod with Tests | Re-run Playwright scenarios against production |
| 1.6 | Working on it (prod-fix re-entry) | Tasks re-entering build phase after prod verification failure (identified by `prod-fix-attempts` file) |
| 1.75 | Ready For Review | Automated PR review (NLSpec + security + quality) |
| 2 | Waiting for Spec & Scenarios Approval | Check for human approval/feedback, start build phase |
| 2.2 | Waiting for Epic Approval | Check for human approval/feedback on epic/deep-plan decomposition |
| 2.5 | Awaiting Input | Check if human answered Q&A question, resume appropriate phase |
| 3 | Ready | New task — route by type (code/epic/deep-planning/infra). Dependency-aware: skips tasks whose `depends_on` items aren't Done. |
| 4 | Stale Working on it | Tasks stuck > 2h — auto-correct based on run artifacts |
| 5 | Stuck | Self-healing — diagnose and fix automatically |
| 6 | Epic Breakdown Posted | Auto-close parent epics when all child stories are Done |

The daemon uses **dynamic N-slot concurrency** — configurable parallel execution slots (default 2, adjustable via `max_slots` in config). Slots are named `slot-0`, `slot-1`, etc. This means a spec phase and a build phase (or any two phases) can execute simultaneously. The scheduler dispatches to whichever slot is free, prioritizing higher-priority work.

Some priorities (2.2 Epic Approval, 2.5 Awaiting Input) run **inline checks** during the poll cycle — cheap board API calls to detect human responses — and only dispatch to a slot when action is needed. This avoids wasting slots on polling.

## Task Lifecycle

**Standalone code tasks:**
```
New → Ready → Working on it (Phase 1) → Waiting for Spec & Scenarios Approval
  → (human provides feedback) → Working on it (revision) → Waiting for Spec & Scenarios Approval → ...
  → (human approves) → Working on it (Phase 2) → Ready For Review
  → (auto-review approves) → Approved & Merge to Main → Verifying Prod with Tests
  → (scenarios pass in production) → Done

Failure paths:
  → (auto-review rejects) → Working on it (rebuild)
  → (prod verification: migration) → run migrations → re-verify → if migration fails → rebuild
  → (prod verification: code/mixed/unknown) → Working on it (prod-fix attempt 1/3) → rebuild → re-verify
     → still failing → Working on it (attempt 2/3) → rebuild → re-verify
     → still failing → Working on it (attempt 3/3) → rebuild → re-verify
     → still failing → satisfaction ceiling check:
        → satisfaction >= 0.50 → Done (Playwright limitations, not code bugs)
        → satisfaction < 0.50 → Stuck
  → (prod verification: infra) → Stuck (rebuild won't help)
  → (pre-flight / SRE gate fails) → stays in Verifying, retries next cycle
  → (5 consecutive failures) → Stuck → Tier 0 fix → Tier 1 Claude heal → satisfaction ceiling → human reset
```

**Epic story code tasks** (with `parent_epic` set):
```
New → Ready → Working on it (Phase 1) → Working on it (Phase 2)
  (skips "Waiting for Spec & Scenarios Approval" — parent epic already approved)
  → Ready For Review → Approved & Merge to Main → Verifying Prod with Tests → Done
  (same failure paths as standalone tasks)
```

**Epic / Deep Planning tasks:**
```
New → Ready → In Progress (decomposition / Q&A rounds) → Waiting for Epic Approval
  → (optional Q&A: In Progress → Awaiting Input → human answers → In Progress resume)
  → (human provides feedback) → In Progress (revision) → Waiting for Epic Approval → ...
  → (human approves) → stories created (with depends_on column) → Epic Breakdown Posted
  → Stories execute in dependency order (blocked stories skipped until deps are Done)
  → All stories Done → parent epic auto-closes to Done
```

## Agent Roster

13 agents serving a pipeline, not an org chart.

| Agent | Stage | Responsibility |
|-------|-------|----------------|
| **Spec Refiner** | 1 | Conversational NLSpec creation. Resolves all ambiguity before producing the spec. |
| **Planner** | 2 | Dependency-aware task graph from NLSpec. Each task is atomic with acceptance criteria. |
| **Scenario Designer** | 3 | Holdout scenarios from NLSpec. Covers functional, security, accessibility, edge cases. |
| **Coder** | 4 (Build) | Implementation from plan. Works in git worktrees. Multiple parallel instances. |
| **Scenario Runner** | 4 (Verify) | Playwright MCP browser verification. Screenshots as evidence. |
| **Convergence Controller** | 4 (Loop) | Iterate-or-ship decisions. Feedback without leaking scenarios. |
| **COO** | All | Ops: git, dependencies, environment, worktrees, dev server, merge. |
| **DTU Builder** | Pre-4 | Digital Twin Universe — behavioral clones of third-party services for testing. |
| **Summarizer** | All | Pyramid summaries (L0/L1/L2) for context window management. |
| **Dashboard Agent** | All | Live HTML dashboard showing pipeline progress. |
| **Brownfield Analyst** | Pre-1 | Existing codebase mapping for modification tasks. |
| **PR Reviewer** | 5.5 | Automated review: NLSpec compliance + security + code quality. |
| **Infra Planner** | Infra | AWS infrastructure planning. Creates board sub-tasks for provisioning. |

## Run Directory Structure

Each pipeline execution creates a structured run directory. Run directories are per-project: `runs/{project-name}/{run-id}/` (single-project installs use `runs/{run-id}/`). Board databases are per-project: `board/data/boards/{project-name}.db`.

```
beastmode/runs/{project-name}/{run-id}/
  manifest.json            # Pipeline config and satisfaction threshold
  checkpoint.json          # Serialized state for crash recovery
  nlspec.md                # Engineering-grade specification
  plan.md                  # Dependency-aware task graph
  scenarios/               # Holdout set (information-isolated from coders)
  iterations/
    001/
      coder-prompt.md      # What the coder was told
      coder-response.md    # What the coder produced
      satisfaction.json    # Per-scenario pass/fail with scores
      feedback.md          # Convergence controller feedback
      screenshots/         # Playwright verification evidence
    002/
      ...
  review-result.json       # PR review verdict (APPROVE/REJECT)
  prod-verification.json   # Production Playwright results
  prod-fix-attempts        # Counter: how many prod-fix rebuild attempts (max 3)
  prev-prod-failures.txt   # Previous cycle's failure signature (for "try different approach" prompt)
  screenshots/prod/        # Production verification Playwright screenshots (uploaded to board as proof)
  artifacts/               # Large outputs (generated code, DTU clones)
  summaries/               # Pyramid summaries per artifact

beastmode/daemon/logs/           # Daemon state directory (Docker volume)
  {project-name}/
    .run_map/{item_id}           # Maps item IDs to run IDs
    .fail_counts/{item_id}       # Per-task failure counters
    .heal_counts/{item_id}       # Per-task healing attempt counters
    .current_item                # Currently in-progress item ID
    .recent_items                # Recently completed items
  deploy-history.jsonl           # Append-only ledger of successful deploys (for auto-rollback)
```

## Multi-Project Support

BeastMode supports multiple projects per factory. Each project has its own board (SQLite database), run directory, and deploy config. Projects are registered under `.beastmode/projects/{name}/` with `project.json` for config.

The daemon uses round-robin fairness across projects with pending work. Per-project slot caps are configurable. The UI includes a project selector for switching between boards.
