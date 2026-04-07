# Setting Up BeastMode — From Zero to Running Factory

## Prerequisites

| Tool | Install |
|------|---------|
| Node.js 20+ | [nodejs.org](https://nodejs.org) |
| Docker | [docker.com](https://get.docker.com) |
| GHCR access | GitHub PAT with `read:packages` scope (ask your admin) |

## Setup

```bash
# Install the CLI
npm install -g @beastmode-develeap/beastmode

# Authenticate to pull private Docker images
docker login ghcr.io
# Username: your GitHub username
# Password: your GitHub PAT (with read:packages scope)

# Initialize — generates docker-compose.yml, .env, pulls images
beastmode init

# Start all services
beastmode up
```

`beastmode init` is interactive — it walks you through setting your GitHub token, optional Anthropic API key, and board password. After init, `beastmode up` starts three containers (board, UI, daemon) and the board UI is available at `http://localhost:8420`.

## Managing Services

```bash
beastmode up              # Start all services
beastmode down            # Stop all services
beastmode logs            # Stream all logs
beastmode logs daemon     # Stream daemon logs only
beastmode update          # Pull latest images and restart
beastmode update --tag 1.2.3  # Pin to a specific version
beastmode doctor          # Health check (includes GHCR auth)
```

## Upgrading

```bash
npm update -g @beastmode-develeap/beastmode   # Update the CLI
beastmode update                               # Pull latest Docker images and restart
```

## Adding Your Project

After BeastMode is running:

1. Open the board UI at `http://localhost:8420`
2. Go to **Projects** in the sidebar
3. Click **Add Project** or **Connect Existing Codebase**
4. Enter the path to your Git repository
5. BeastMode auto-detects the tech stack

Or via CLI:
```bash
beastmode project add /path/to/your/project
```

BeastMode supports: Node.js (Next.js, Vite, React), Python (Django, FastAPI), Go, Rust, Java (Maven, Gradle), and more.

## What Happens Next

Once your project is connected:

1. **Infrastructure Assessment** — If no deploy config exists, BeastMode auto-creates an "Infrastructure Assessment" task that analyzes your project and recommends what AWS resources it needs (ECR, ECS, RDS, etc.) with cost estimates. Approve to provision, or ignore for PR-only mode.

2. **Create tasks** — Write task descriptions on the board. BeastMode picks them up within 60 seconds.

3. **Review specs** — When a task reaches "Waiting for Approval", review the spec and comment "approved" to proceed.

4. **Watch it work** — BeastMode implements, verifies, creates PRs, and (if infra is set up) deploys to production.

## Configuration

Configuration is managed via the **Settings** page in the board UI — changes apply on the next daemon poll cycle (no restart needed).

Key settings:

| Setting | Default | What it does |
|---------|---------|-------------|
| `poll_interval_seconds` | 60 | How often the daemon checks for new tasks |
| `max_slots` | 2 | Parallel pipeline slots (tasks running simultaneously) |
| `verification.satisfaction_threshold` | 0.85 | Minimum satisfaction score to ship |
| `convergence.max_iterations` | 5 | Max build-verify iterations before halting |
| `stack.build_command` | auto-detected | Build command for your project |
| `stack.auth_credentials` | "" | Login credentials for verification (if app has auth) |
| `migration_safety.enabled` | true | Check for breaking schema changes before deploy |
| `alerts.webhook_url` | "" | Slack/Discord webhook for notifications |

## Authentication & Keys

| What | How | What it's for |
|------|-----|---------------|
| Claude Code | `claude login` (inside the daemon container) | **All AI** — every agent runs through the Claude Code CLI (subscription) |
| `GITHUB_TOKEN` | GitHub → Settings → Developer Settings → Personal Access Tokens | Creating PRs, reading repos |
| `ANTHROPIC_API_KEY` (optional) | [console.anthropic.com](https://console.anthropic.com) | If set, used for faster direct API calls. Falls back to CLI when absent. |

No `ANTHROPIC_API_KEY` needed — everything works with just `claude login`. Setting it is a performance optimization, not a requirement.

## Cost Estimates

| Component | Cost | Notes |
|-----------|------|-------|
| EC2 (t3.xlarge) | ~$120/month | BeastMode instance (4 vCPU / 16 GB — recommended for cloud) |
| Claude Code subscription | Included in plan | Max/Team plan covers all AI usage |
| Idle polling | $0 | No AI cost when no tasks are running |
| ECR | ~$1/month | Container image storage (if deploying to AWS) |
| ECS + RDS (if deployed) | ~$80/month | Your application's infrastructure |

## Next Steps

- Read the [Getting Started](../getting-started.md) guide for using BeastMode
- Check the [Pipeline](../pipeline.md) docs for how the pipeline works
- See [Configuration](../configuration.md) for all settings
- Review [Resilience](../resilience.md) for failure handling
