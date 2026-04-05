# Pipeline — Step by Step

## Code Tasks

### 1. Task Intake

A human creates a task on the BeastMode Board: *"Add dark mode support"* or *"Build a Kanban board for operations management"*. They set the status to "Ready". The daemon, polling every 60 seconds, picks it up.

If the task has file attachments (images, documents, specs), BeastMode downloads them and includes them as context for the spec phase. **Image attachments** (logos, mockups, screenshots) are read visually by Claude — it sees the actual image content, not just a file path. **URLs** mentioned in the task description are detected and the Spec Refiner is instructed to screenshot them for visual reference.

### 2. Phase 1 — Spec + Plan + Scenarios

The daemon spawns a Claude process that runs three stages sequentially:

**Spec Refiner** takes the task title and description and produces `nlspec.md` — an engineering-grade specification precise enough that any developer (human or AI) could implement from it without further clarification. It includes data models, API contracts, state machines, error handling, and explicit out-of-scope items.

**Planner** reads the NLSpec and produces `plan.md` — a dependency-aware task graph. Each task is atomic, has acceptance criteria, and references the specific NLSpec sections it implements.

**Scenario Designer** reads the NLSpec (NOT the plan — this prevents implementation bias) and produces holdout scenarios in `scenarios/`. Each scenario is an end-to-end user story with setup steps, actions, and expected outcomes. Scenarios cover functional requirements, security, accessibility, edge cases, and integration points.

All three artifacts are posted as task updates (HTML-formatted for readability), and the task moves to "Waiting for Spec & Scenarios Approval". The factory pauses until the human responds.

**Epic stories skip this gate entirely.** When a story has `parent_epic` set (created by epic decomposition), its spec phase completes and immediately proceeds to the build phase. The parent epic was already human-approved, so re-approving each story is redundant. This reduces human-in-the-loop to epics and standalone tasks only, aligning with dark factory principles.

Human comments are interpreted agentically by Claude — not regex matching. Both top-level updates and **thread replies** are read, so humans can respond inline to BeastMode's posts. The human can:
- **Approve** with any phrasing ("approved", "lgtm", "looks good", "go ahead")
- **Provide feedback** ("change X to Y", "remove the Redis section") — BeastMode revises the spec/plan/scenarios in-place, re-posts the updated artifacts, and waits for another review
- If the spec is ambiguous, the Spec Refiner may ask clarifying questions via the board before producing the spec, using the "Awaiting Input" status

### 3. Phase 2 — Build-Verify Convergence Loop

After approval, the daemon spawns another Claude process for the build-verify loop:

**Coder agents** implement the plan. They work in git worktrees to prevent conflicts during parallel work. They can see the plan and the NLSpec, but they **cannot see the scenarios**. This is the information barrier — physical repo separation plus hooks enforcement. Before exiting, each coder runs a **mandatory self-check**: `npm run build` must pass, every NLSpec requirement must be implemented (not stubbed), and a sanity check of the diff (no debug logs, no TODOs, no import errors). This catches build failures and NLSpec gaps within the same Claude session at zero extra cost.

After the coder finishes, two quality gates run before the expensive Scenario Runner:

1. **Speculative build check** — runs `npm run build` externally. If it fails, the verifier is skipped entirely (saves ~$2.50-10.00 per iteration).
2. **NLSpec compliance pre-check** — a cheap Haiku model call (~$0.01) compares the git diff against the NLSpec. If it finds gaps (missing requirements, incomplete implementations), the verifier is skipped and the coder gets targeted feedback about exactly which NLSpec requirements are missing. This catches the exact issues that would later cause PR review rejection, saving a full verifier session. The pre-check **fails open** — if it crashes or times out, the verifier runs anyway. Controlled by `cost.nlspec_precheck_enabled` (default: true).

**Scenario Runner** executes every holdout scenario against the running application using **Playwright MCP browser automation**. It navigates pages, clicks buttons, fills forms, reads DOM state, and takes screenshots as evidence. It produces a satisfaction score:

```
satisfaction = passed_scenarios / total_scenarios
```

**Convergence Controller** reads the satisfaction score. If it's below the threshold (default 0.85), it generates structured feedback for the coders — explaining WHAT failed and WHY, but never leaking the actual scenario definitions. The coders iterate. The loop repeats.

**Fail-fast on catastrophic first iteration** — if iteration 1 satisfaction is below 0.3 (configurable via `convergence.fail_fast_threshold`), the build loop aborts immediately and sends the task back to "Waiting for Spec & Scenarios Approval" for human review. This prevents burning 4 more iterations when the spec or plan itself is wrong. The failure counter is NOT incremented (this is a spec quality issue, not a code failure). Setting the threshold to 0.0 disables fail-fast.

**Infrastructure failure detection** — when verification fails due to infrastructure issues (Bedrock unavailability, API rate limits, connection refused, 503 Service Unavailable, DNS failures, timeouts), the daemon retries the **same iteration** without incrementing the iteration counter. This prevents wasting iterations on transient environment problems. Infra retries are capped at `convergence.infra_retry_max` (default 3) per iteration. The detection uses pattern matching against known infra error signatures — code failures (TypeError, build errors, assertion failures) are never classified as infra.

If satisfaction >= 0.85, the loop exits. If the maximum iteration count (default 5) is reached, the pipeline halts and reports the current state.

### 4. Ship + Automated PR Review

On convergence, the daemon pushes a feature branch and creates a PR. The task moves to "Ready For Review".

If migration safety is enabled (`migration_safety.enabled`), the daemon analyzes SQL migration files for breaking schema changes before proceeding. Breaking changes are either auto-decomposed into safe multi-step tasks or flagged for human review.

The **PR Reviewer** agent automatically reviews the diff against:
- **NLSpec compliance** — every requirement has corresponding implementation
- **Security scan** — secrets, SQL injection, XSS, eval, CORS
- **Code quality** — empty catches, dead code, console.log, TODOs

Decision: APPROVE if satisfaction >= threshold, no HIGH security findings, no missing requirements. REJECT sends it back to the build loop with feedback — the daemon detects the rejection via `review-result.json`, clears it, and forces a fresh build iteration (preventing the re-ship-same-PR infinite loop). On crash, the task stays in "Ready For Review" for human fallback.

### 5. Merge + CI + Deploy

On APPROVE, the daemon merges the PR via `gh pr merge`, waits for GitHub Actions CI to pass (Docker build + ECR push), waits for ECS to deploy the new image, and polls the health check endpoint. CI runs on a **self-hosted GitHub Actions runner** on the EC2 instance — zero GitHub Actions minutes consumed.

If CI fails, the daemon spawns Claude to read the failure logs and attempt a fix (up to 2 attempts). If the fix works, it retries the merge.

### 6. Production Verification

After deployment, the daemon runs the holdout scenarios one more time — this time against **live production** using Playwright MCP. This catches deployment-specific issues (environment config, database migrations, CDN caching).

**Pre-flight checks** run before spawning the expensive Playwright session ($3-10 per run):
1. **Service health** — HTTP 200 from the health endpoint
2. **DB connectivity** — TCP/pg_isready check against the configured DATABASE_URL
3. **Database migrations** — runs `drizzle-kit push` with output validation (catches silent failures — drizzle-kit exits 0 even on ECONNREFUSED)
4. **Deployment lag** — verifies the latest commit SHA matches the running ECS image tag, waits up to 5 minutes for rollout

If any pre-flight check fails, Playwright is skipped entirely and the task retries on the next cycle.

On completion, Playwright screenshots are uploaded to the task as **proof attachments** — visual evidence of the working feature. The board renders these as inline image thumbnails. Screenshots capture the dashboard after login, each major feature being verified, and any failures showing the broken state. This costs zero LLM tokens — just HTTP file uploads of existing PNGs.

**Failure classification** — when production verification fails, the daemon classifies failures to determine the right remediation:

| Class | Indicators | Action |
|-------|-----------|--------|
| `migration` | HTTP 500, "relation does not exist" | Run migrations, re-verify. If migration fails, fall back to code rebuild. |
| `infra` | ECONNREFUSED, timeouts, 502/503 | Escalate to Stuck (rebuild won't help) |
| `code` | Logic errors, wrong responses, CSS/font issues | Up to 3 rebuild attempts, then escalate |
| `mixed`/`unknown` | Multiple failure types | Up to 3 rebuild attempts, then escalate |

**Auto-rollback** — when production verification fails, the daemon immediately rolls back to the last known-good deploy before starting the fix cycle. It reads the deploy history ledger (which records image tag, commit SHA, and task ID for each successful deploy) and reverts ECS to the previous image. This restores service while the fix cycle runs in parallel.

**Autonomous prod-fix loop** — after rollback, the daemon re-enters the build phase (Phase 2) with production failure details injected into the coder prompt. This happens up to 3 times before escalating. The coder sees exactly which scenarios failed and why, plus guidance on common prod-specific root causes (CSS/font loading differences, missing migrations, runtime environment differences).

Key design decisions:
- **No "build-awareness" shortcut** — high build satisfaction (>= 0.95) does NOT prevent rebuild attempts. CSS, font, and runtime issues regularly pass local verification but fail in production (e.g., `geist/font/sans` package import works locally but the font doesn't load in the Docker container).
- **Identical failures don't instant-Stuck** — even if the same scenarios keep failing, the daemon allows fix attempts. On each retry, the prompt tells Claude that previous fixes didn't work and it must try a different approach.
- **Direct routing** — tasks go to "Working on it" (not back through the approval gate), where the daemon picks them up via Priority 1.6 and calls `run_build_phase()` directly. The `prod-verification.json` file is already read by the build phase prompt.
- **Attempt tracking** — a `prod-fix-attempts` counter file persists across daemon restarts. Cleared on success or human reset.

**Satisfaction ceiling** — when prod-fix attempts are exhausted (3/3), the daemon checks the current satisfaction score before escalating. If satisfaction >= 0.65 (`prod_accept_floor`, configurable), the task is **auto-accepted as Done**. The remaining failures are likely Playwright testing limitations (untestable scenarios: API-only endpoints, hover tooltips, drag-and-drop, cron jobs, "skipped to avoid modifying prod data"), not real code issues. If satisfaction < 0.65, the task genuinely needs human help and escalates to Stuck. This prevents the infinite Stuck → reset → build → verify → Stuck loop that occurred when Playwright couldn't test enough scenarios to reach 0.85.

**Auto-approve for restarted tasks** — when a task that was previously human-approved gets sent back through the pipeline (by self-healing or failure recovery), the daemon auto-approves the spec phase instead of waiting for another human review.

If production satisfaction >= threshold: **Done**. The task is complete.
If production satisfaction < threshold: routed by failure classification (see above).
If prod-fix attempts exhausted and satisfaction >= 0.65: **Done** (satisfaction ceiling).
If prod-fix attempts exhausted and satisfaction < 0.65: **Stuck** (genuinely broken).

## Epic Decomposition

For large features, create a task with "epic" in the name (e.g., *"Epic: Organization Management"*). Instead of the code pipeline, BeastMode runs the epic decomposition flow:

1. **Brownfield analysis** — scans the target codebase to understand existing architecture
2. **Scope analysis** — determines story boundaries aligned with architectural patterns
3. **Clarifying questions** (if needed) — asks up to 2 questions via board updates ("Awaiting Input" status), waits for answers, then resumes
4. **Epic NLSpec** — writes `epic-nlspec.md` covering the entire epic scope
5. **Story decomposition** — breaks the epic into 3-8 ordered stories with dependencies
6. **Posts plan for approval** — moves to "Waiting for Epic Approval"

On approval, the daemon creates each story as a separate board item (with `parent_epic` column set, status "Ready"). Each story independently enters the code pipeline with the epic NLSpec as context. **Stories skip the human approval gate** — the parent epic's approval covers all its children, so stories go directly from spec to build.

On feedback, BeastMode revises the plan and re-posts for another review cycle.

**Story dependency management** — when the epic decomposition defines dependencies between stories (e.g., "DB schema" before "API endpoints" before "UI"), the daemon:

1. Uses the `depends_on` column on the board (idempotent)
2. Resolves dependency references (e.g., "Story 1", "Story 3") to board item IDs
3. Populates the `depends_on` column with comma-separated item IDs
4. Detects circular dependencies via topological sort — breaks cycles with a warning rather than deadlocking
5. Posts a dependency chain summary showing execution groups

The daemon's task selection (Priority 0 and Priority 3) checks `depends_on` before picking a task. Stories whose prerequisites aren't "Done" yet are skipped. This ensures stories execute in the correct order without manual intervention.

**Auto-close parent epics** — when all child stories of an epic reach "Done", the daemon automatically advances the parent epic from "Epic Breakdown Posted" to "Done" (Priority 6 check).

```
Epic lifecycle:
Ready → In Progress → Waiting for Epic Approval
  → (feedback) → In Progress (revision) → Waiting for Epic Approval → ...
  → (approved) → stories created (with depends_on) → Epic Breakdown Posted
  → Each story: Ready → spec → build → ship → deploy → Done
      (skips human approval — parent epic covers it; respects dependency order)
  → All stories Done → parent epic auto-closes to Done
```

## Deep Planning Mode

For complex or ambiguous tasks where you want collaborative planning before committing to a decomposition, create a task with "deep planning" in the name (e.g., *"Deep Planning: Multi-tenant Analytics Platform"*).

Deep planning removes the 2-question limit of epic decomposition. BeastMode enters an **iterative Q&A loop**:

1. **Brownfield analysis** — scans the target codebase
2. **Posts batch of 3-5 questions** to the board covering functional scope, technical constraints, UX, integrations, edge cases, and priorities
3. **Sets "Awaiting Input"** — waits for the human to answer
4. **Resumes with follow-up questions** — drills deeper based on answers
5. **Repeats** until confident (or the human says "that's enough, plan it")
6. **Produces comprehensive plan** — with phased stories and flexible granularity
7. **Posts for approval** — moves to "Waiting for Epic Approval"

The approval/revision/story-creation flow is identical to epic decomposition.

## Infrastructure Self-Provisioning

BeastMode can provision its own production infrastructure. A single task like *"Plan and provision production infrastructure"* triggers the infra-plan pipeline:

1. **Infra Planner** checks AWS state, GitHub repos, and creates missing repos (Infra, GitOps)
2. Creates board sub-tasks for each resource (ECR, ECS, RDS, Redis)
3. Human reviews and sets sub-tasks to "Ready"
4. Each sub-task triggers `infra-execute`: Terraform plan → apply → verification scenarios
5. Infrastructure scenarios verify resources work (AWS CLI checks, connectivity, HTTP endpoints)

Infrastructure playbooks in `infra/playbooks/` and CI templates in `ci-templates/` provide reference modules.
