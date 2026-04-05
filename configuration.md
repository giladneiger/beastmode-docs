# Configuration

Base config: `config/beastmode.json`. Daemon overrides: `config/beastmode.daemon.json` (bare metal) or `config/beastmode.docker.json` (Docker Compose).

The Docker config uses `${ENV_VAR:-default}` interpolation — values are resolved from the `.env` file at startup. This keeps secrets out of the config file and in the environment.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `satisfaction_threshold` | `0.85` | Minimum satisfaction to exit convergence loop |
| `max_iterations` | `5` | Maximum build-verify iterations before halting |
| `max_slots` | `2` | Maximum parallel pipeline execution slots (dynamic pool) |
| `poll_interval_seconds` | `60` | Daemon polling interval |
| `stale_task_hours` | `2` | Hours before a "Working on it" task is considered stale |
| `retry.max_failures` | `5` | Consecutive failures before escalating to Stuck |
| `healing.max_attempts` | `3` | Self-healing attempts per stuck task |
| `prod_fix.max_attempts` | `3` | Prod-fix rebuild attempts before satisfaction ceiling check |
| `verification.prod_accept_floor` | `0.65` | Minimum satisfaction to auto-accept at prod-fix exhaustion |
| `review.timeout_minutes` | `15` | PR review timeout |
| `ci.timeout_minutes` | `30` | CI pipeline timeout |
| `deploy.health_check_timeout_seconds` | `120` | Post-deploy health check timeout |
| `deploy.database_url` | `null` | Production DATABASE_URL for migrations (highest priority) |
| `deploy.ecs_cluster` | `null` | ECS cluster name for deployment lag checks and ECS exec migrations |
| `deploy.ecs_service` | `null` | ECS service name for deployment lag checks |
| `convergence.fail_fast_threshold` | `0.3` | Abort build loop if iteration 1 satisfaction below this. Set to 0.0 to disable. |
| `convergence.infra_retry_max` | `3` | Max infra-failure retries per iteration (no iteration counter increment) |
| `convergence.max_precheck_retries` | `2` | NLSpec precheck retries before advancing iteration |
| `cost.nlspec_precheck_enabled` | `true` | Enable cheap NLSpec compliance check before expensive verifier |
| `cost.nlspec_precheck_model` | `claude-haiku-4-5-20251001` | Model for NLSpec pre-check (Haiku is ~25x cheaper than Opus) |
| `board.url` | `http://board:8080` | BeastMode Board URL (Docker internal) |
| `deploy.ecr_repository` | `null` | ECR repository URI for Docker images |
| `deploy.ecs_cluster` | `null` | ECS cluster for deployment and rollback |
| `deploy.ecs_service` | `null` | ECS service name |
| `deploy.health_check_url` | `null` | Production health endpoint URL |
| `migration_safety.enabled` | `true` | Check for breaking schema changes before deploy |
| `migration_safety.auto_decompose` | `true` | Auto-split breaking changes into safe steps |
| `migration_safety.migration_paths` | `["migrations/", "drizzle/", "prisma/migrations/", "sql/"]` | Directories to scan for SQL migration files |
| `migration_safety.allow_override` | `true` | Let humans bypass blocks with "proceed anyway" |
| `stack.name` | `"node"` | Tech stack (node, python, go, rust, java) |
| `stack.build_command` | `"npm run build"` | Build command for the target project |

## Project Structure

```
beastmode/
  CLAUDE.md                            # Pipeline constitution (agent rules)
  config/
    beastmode.json                     # Base config (all modes)
    beastmode.daemon.json              # Daemon mode config (bare metal)
    beastmode.docker.json              # Docker Compose config (env var interpolation)
    playwright-mcp.json                # Playwright MCP server config for Claude
  agents/                              # 13 agent definition files
  skills/
    beastmode/                         # Main pipeline orchestrator (code tasks)
    beast-status/                      # Progress dashboard
    epic-plan/                         # Epic decomposition into ordered stories
    deep-planning/                     # Iterative Q&A deep planning mode
    frontend-design/                   # Visual design guidelines for CSS/UI tasks
    infra-plan/                        # Infrastructure planning
    infra-execute/                     # Terraform execution
  hooks/
    hooks.json                         # Holdout barrier hooks + quality gates
  daemon/
    beastmode_daemon/                  # Python daemon package (~1000 lines scheduler)
      __main__.py                      #   Entry point
      scheduler.py                     #   Priority-based N-slot scheduler
      config.py                        #   Typed configuration with auto-reload
      slots.py                         #   Dynamic slot pool manager
      healing.py                       #   Tiered self-healing (Tier 0 + Tier 1)
      recovery.py                      #   Crash recovery + status auto-correction
      registry.py                      #   Multi-project registry + round-robin dispatcher
      clients/
        task_backend.py                #   Abstract TaskBackend interface
        board.py                       #   BeastMode Board client (HTTP/httpx)
        claude.py                      #   Claude CLI subprocess runner
        github.py                      #   GitHub CLI wrapper (PR, CI, merge)
        aws.py                         #   AWS SDK client (ECS, ECR)
        docker_client.py               #   Docker operations
      pipeline/
        spec_phase.py                  #   Phase 1: spec + plan + scenarios
        build_phase.py                 #   Phase 2: build-verify convergence loop
        review.py                      #   Automated PR review
        merge.py                       #   PR merge + CI wait
        ship.py                        #   Ship orchestration
        verify_production.py           #   Production Playwright verification + proof screenshots
        rollback.py                    #   Auto-rollback on prod failure + deploy history ledger
        schema_safety.py               #   Migration safety gate (SQL analysis)
      sre/
        health_check.py                #   SRE probes (HTTP, ECS, DB)
        gate.py                        #   Pre-dispatch health gating
        remediator.py                  #   Auto-remediation (force deploy, restart)
        failure_tracker.py             #   Failure pattern tracking
      tasks/
        router.py                      #   Task type detection + dependency checks
        code.py                        #   Code task handler (Q&A, approval)
        epic.py                        #   Epic task handler (decomposition, stories)
      prompts/                         #   Prompt builders for each pipeline phase
        board_messages.py              #   HTML-formatted board update templates
        spec.py                        #   Spec/plan/scenario phase prompts
        coder.py                       #   Coder agent prompts
        verifier.py                    #   Verifier + prod verification prompts
        reviewer.py                    #   PR review prompts
      state/
        file_store.py                  #   File-based state (run_id, fail counts, in-progress)
        models.py                      #   Data models (SlotName, TaskStatus, PipelineState)
    tests/                             #   330+ tests (pytest)
    logs/                              # Daemon + per-run logs
      daemon.log                       #   Main daemon log (structured JSON via structlog)
      {run-id}-*.log                   #   Per-phase logs (spec, build, verify, review)
      migration-*.log                  #   Database migration output (for silent failure detection)
      heal-{item-id}-*.log            #   Self-healing Claude session logs
      heal-journal-{item-id}.md       #   Cumulative heal attempt journal (cross-attempt memory)
      heal-diagnosis-{item-id}.md     #   Latest heal diagnosis (root cause + fix + status)
  board/                                 # Built-in task board (FastAPI + SQLite)
    board_server/
      routes/                          #   REST API (items, updates, attachments, poll)
      static/                          #   Frontend (kanban UI, dark/light theme)
      models.py                        #   Pydantic models
    Dockerfile                         #   Board container image
    tests/                             #   Board tests (pytest + Playwright)
  docker-compose.yml                   # Runs board + daemon containers
  Dockerfile (daemon/)                 # Daemon container (Python + Chrome + Claude CLI)
  .env.example                         # Template for environment variables
  infra/                               # Terraform (VPC, EC2, ECS, RDS, ECR)
    playbooks/                         # Reference Terraform modules (ECR, ECS, RDS, Redis)
  ci-templates/                        # GitHub Actions workflows
  runs/                                # Pipeline run artifacts (gitignored)
```
