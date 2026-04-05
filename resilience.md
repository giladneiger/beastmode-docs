# Resilience Mechanisms

## Pre-Verification Quality Gates

Three layers of early detection catch problems before the expensive Scenario Runner ($2.50-10.00):

1. **Coder self-check** — the coder prompt includes a mandatory self-check section: run `npm run build`, verify every NLSpec requirement is implemented, and sanity-check the diff. Catches build failures and NLSpec gaps within the same Claude session at zero extra cost.
2. **Speculative build check** — external `npm run build` after the coder exits. If it fails, the verifier is skipped entirely.
3. **NLSpec compliance pre-check** — a cheap Haiku model call (~$0.01) compares the diff against the NLSpec. If gaps are found, writes synthetic satisfaction (0.0) and feedback for the coder, skipping the verifier. Fails open on crash (safe default). Controlled by `cost.nlspec_precheck_enabled`.

## Fail-Fast Spec Review

If iteration 1 satisfaction is below 0.3 (configurable via `convergence.fail_fast_threshold`), the build loop aborts and sends the task back for spec review ("Waiting for Spec & Scenarios Approval"). This prevents burning 4 more iterations when the spec or plan itself is wrong — the coder fundamentally misunderstood the task. The failure counter is NOT incremented. Set threshold to 0.0 to disable.

## Infrastructure Failure Retry

When verification fails due to transient infrastructure issues, the daemon retries the same iteration without incrementing the iteration counter. Detected patterns: Bedrock unavailability, API rate limits (`ThrottlingException`), connection refused (`ECONNREFUSED`), 503 Service Unavailable, DNS failures, timeouts (`ETIMEDOUT`), overloaded errors, and API internal server errors. Capped at `convergence.infra_retry_max` (default 3) retries per iteration. Code failures (TypeError, build errors, assertion failures) are never misclassified as infra.

## PR Review Rejection Recovery

When the automated PR reviewer rejects a PR, the task returns to "Working on it". The daemon detects the rejection via `review-result.json` with `verdict=REJECT`, clears the stale review result, and forces a fresh build iteration. This prevents an infinite loop where the `already_converged` guard would skip the build and re-ship the same unchanged PR.

## Automatic Retry

Every failure increments a per-task counter. Below the limit (default 5), the task is automatically retried by moving it back to the appropriate pipeline status. At the limit, it escalates to "Stuck" with a detailed error message.

## Token/Rate-Limit Handling

When Claude hits API rate limits (429/529), the daemon detects the pattern in the log output, pauses the task **without incrementing the failure counter**, and posts to the board. On the next poll cycle, the daemon retries automatically.

## Tiered Self-Healing

Tasks that reach "Stuck" status are automatically picked up at the lowest priority. Healing uses a two-tier system to minimize token spend:

**Tier 0 — Deterministic fixes (0 tokens):**
Before spawning Claude, the daemon reads the failure classification and attempts cheap, known fixes:
- `migration` class → check DB connectivity → run migrations → if success, resume verification
- `infra` class → curl health endpoint → if service self-recovered (HTTP 200), resume verification

Each skipped Claude session saves $2-5.

**Tier 1 — Claude healing (enhanced):**
Only runs if Tier 0 couldn't fix it. Claude receives:
- Error context from board updates and run logs
- Build satisfaction score (so it knows "code was 100%, focus on infra")
- **Heal attempt journal** — a cumulative record of every previous attempt, what was tried, and why it failed

The journal prevents Claude from repeating approaches that already failed. On later attempts (2+/3), the prompt escalates: *"Previous approaches all failed. Try something RADICALLY different or declare UNFIXABLE."*

Self-healing has its own attempt counter (default 3) to prevent infinite loops. After healing attempts are exhausted, a **last resort** attempt runs via a different code path. If that also fails, the satisfaction ceiling check runs — tasks with satisfaction >= `prod_accept_floor` (default 0.65, configurable) are auto-accepted as Done when no production rollback occurred. If production was rolled back, the task goes to Stuck regardless of satisfaction (the code isn't in production).

A human can comment "reset" on a Stuck task to clear everything (including the journal) and restart from scratch.

## Satisfaction Ceiling

When prod-fix attempts are exhausted (3/3), the daemon checks the current satisfaction score before escalating. The behavior depends on whether deploy (rollback) is enabled:

- **Deploy disabled** (no rollback): If satisfaction >= `prod_accept_floor` (config, default 0.65), the task is auto-accepted as Done. The remaining failures are likely Playwright testing limitations (untestable scenarios: API-only endpoints, hover tooltips, drag-and-drop, cron jobs, "skipped to avoid modifying prod data"), not real code issues. If satisfaction < `prod_accept_floor`, the task escalates to Stuck.

- **Deploy enabled** (rollback occurred): The task ALWAYS escalates to Stuck, regardless of satisfaction score. This is because auto-rollback restores the previous deploy before the ceiling check runs — the task's code is NOT running in production. Marking Done would be contradictory. The Stuck message explains that the ceiling was met and a human should decide whether to redeploy.

The satisfaction calculation excludes scenarios with `skip_reason_class` of `dev_only`, `code_review`, or `touch_only` from the testable denominator. This prevents untestable scenarios from unfairly dragging the score down.

Both the verify_production module and the scheduler's ceiling check use the same formula and the same configurable `prod_accept_floor` from `config.verification.prod_accept_floor`.

## Silent Migration Failure Detection

`drizzle-kit push` exits with code 0 even when the database is unreachable (ECONNREFUSED). The daemon scans migration output for connection errors (`ECONNREFUSED`, `ENOTFOUND`, `ETIMEDOUT`, `password authentication failed`) and overrides the exit code to failure when detected. This applies to both local and ECS exec migration strategies.

## DATABASE_URL Priority Chain

Database connection resolution follows a strict priority chain:
1. `deploy.database_url` from config (most reliable — explicitly set for production)
2. `$DATABASE_URL` environment variable
3. `.env` files (last resort)

If the resolved URL points to `localhost` or `127.0.0.1` in daemon mode, the daemon logs a warning — this almost certainly means migrations are targeting the dev DB instead of production RDS.

## Status Auto-Correction

When the daemon detects a task in an impossible state (e.g., "Ready For Review" with no PR, or a story in "Waiting for Epic Approval"), it diagnoses the correct pipeline stage from run artifacts and auto-corrects without incrementing the failure counter:

- **No run_id** → Reset to "Ready" (start from scratch)
- **Has stories file but no build** → Route to "Waiting for Epic Approval"
- **Has satisfaction results but no PR** → Re-enter "Working on it" to create PR
- **Has run_id but no satisfaction** → Route to "Waiting for Spec & Scenarios Approval"
- **Story in epic approval** → Auto-correct to "Ready" (stories don't need epic approval)

## Stale Run ID Clearing

When a human resets a task to "Ready", the daemon clears any existing `run_id` from the task before dispatching it. This prevents the auto-correction logic from seeing a stale converged checkpoint and fast-tracking the task back to Done without re-running the pipeline. The in-memory state store's run mapping is also cleared.

## SRE Health Check Gating

Before dispatching production verification, the daemon runs an **SRE health check** module that probes:
- **HTTP health endpoint** — the configured health check URL must return 200
- **ECS service status** — checks the ECS service is ACTIVE with running tasks via AWS API
- **Database connectivity** — TCP check against the configured DATABASE_URL

If any probe fails, the expensive Playwright session ($3-10) is skipped entirely and the task retries on the next cycle. The ECS probe **gracefully handles missing services** — if the configured ECS service/cluster doesn't exist (common during initial setup), the probe returns healthy with a "skipped" flag rather than blocking the entire pipeline. The SRE module also supports **auto-remediation**: when it detects a failed ECS deployment, it can force a new deployment before retrying.

## Config Auto-Reload

The daemon reloads `beastmode.daemon.json` from disk on every poll cycle without requiring a restart. Changed fields are logged. This allows operators to tune thresholds, timeouts, and feature flags on a running daemon — just edit the config file and wait up to 60 seconds.

## Auto-Rollback on Production Failure

When production verification fails, the daemon immediately rolls back ECS to the last known-good deploy before starting the fix cycle. The deploy history ledger (`daemon/logs/deploy-history.jsonl`) records image tag, commit SHA, task definition ARN, item ID, and item name for each successful deployment. On failure, the daemon reads this ledger, finds the most recent good deploy that isn't the current failing one, and reverts ECS to that image. This restores service while the prod-fix rebuild cycle runs in parallel.

## Update Spam Prevention

Pipeline phases (review, merge, prod verification) post a "starting" message only on the first attempt. On retries (fail count > 0), the message is suppressed. This prevents dozens of identical "Automated PR review starting..." messages when a phase crashes and retries repeatedly.

## Proof Screenshots

After production verification completes, Playwright screenshots from the run's `screenshots/prod/` directory are uploaded to the task as file attachments. The board renders images as inline thumbnails. This provides visual proof that verification ran and shows what the app looks like in production — or exactly what went wrong on failure. Screenshots cost zero LLM tokens.

## Phase Entry Messages

When the daemon dispatches a task to a slot, it posts an entry message to the board so operators have real-time visibility:
- *"Attempting self-healing diagnosis..."* — Stuck task being healed
- *"Human answer detected — resuming..."* — Q&A answer detected, resuming phase
- *"Processing epic approval..."* — Epic approval being processed

Pipeline phases (build, verify-prod, review, merge) post their own entry messages internally.

## Crash Recovery

On daemon restart, orphaned tasks (left in "Working on it" by a crashed run) are restored to the appropriate pipeline status based on what phase they were in when the crash occurred.

**Verify-crash loop protection**: If production verification crashes 3 consecutive times (likely resource exhaustion — Claude + Playwright exceeds EC2 memory), the daemon stops retrying and escalates to Stuck instead of endlessly requeueing.

## Database Migration Safety Gate

Before deployment, the daemon can run a schema safety check that analyzes SQL migration files for breaking changes. The gate detects six breaking patterns (DROP COLUMN, DROP TABLE, RENAME COLUMN, ALTER TYPE, ADD NOT NULL without DEFAULT, SET NOT NULL) and classifies them as simple (auto-decomposable) or complex (requires human approval).

Simple breaking patterns are auto-decomposed into safe multi-step tasks. Complex patterns are posted to the board for human review with a suggested decomposition plan. Humans can comment "proceed anyway" to bypass the gate.

The gate supports quoted identifiers (Prisma, Drizzle, TypeORM), schema-qualified names, and IF EXISTS syntax. Controlled by `migration_safety.enabled` (default: true) in daemon config.
