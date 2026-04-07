# Setting Up BeastMode — From Zero to Running Factory

Four ways to run BeastMode, from simplest to most production-ready.

## Option 0: Image Mode (Recommended)

Run BeastMode from pre-built Docker images — no source clone needed. You need: Node.js 20+, Docker, and GHCR access (GitHub PAT with `read:packages` scope).

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

### Managing Services

```bash
beastmode up              # Start all services
beastmode down            # Stop all services
beastmode logs            # View daemon logs
beastmode update          # Pull latest images and restart
beastmode doctor          # Health check
```

### Upgrading

```bash
npm update -g @beastmode-develeap/beastmode   # Update the CLI
beastmode update                     # Pull latest Docker images and restart
```

## Option 1: One-Command Cloud Deploy

Deploys BeastMode to AWS with a single command. You need: AWS CLI configured, Claude Code authenticated (`claude login`), GitHub token.

```bash
# Clone BeastMode
git clone https://github.com/develeap/beastmode.git
cd beastmode

# Authenticate Claude Code (uses your subscription — no API key needed)
claude login

# Set your GitHub token
export GITHUB_TOKEN=ghp_...

# Deploy to AWS
npx beastmode deploy --cloud aws
```

**What this creates on AWS:**

| Resource | Purpose | Estimated Cost |
|----------|---------|---------------|
| EC2 t3.xlarge (4 vCPU / 16 GB) | Runs BeastMode (board + daemon + UI) | ~$120/month |
| VPC + public subnet | Networking and internet access | Free |
| Security group | Allows SSH (22) and board UI (8420) | Free |
| Elastic IP | Stable public IP address | Free (while attached) |
| Secrets Manager | Stores GitHub token and API keys | ~$0.40/month |
| IAM role + instance profile | Allows EC2 to access AWS services | Free |

The command uses **Terraform** under the hood — all infrastructure is defined in `infra/` and can be inspected, modified, or destroyed with `terraform destroy`. No hidden resources.

Wait 3-5 minutes. BeastMode prints the URL and auto-generated password:

```
BeastMode Deployed!
  BoardURL: http://3.92.xxx.xxx:8420
  PublicIP: 3.92.xxx.xxx

  To retrieve the board password:
    ssh into the instance and run: cat /home/ubuntu/.beastmode-password
```

Open the URL, enter the password, and you're running. Skip to [Adding Your Project](#adding-your-project).

**To tear down:** `npx beastmode deploy --destroy` runs `terraform destroy` and removes all AWS resources. You can also run `terraform destroy` directly in the `infra/` directory.

## Option 2: Docker Compose (Local or Server)

> **Note:** For contributors working from source. If you don't need the source code, use Option 0 instead.

Run BeastMode on any machine with Docker.

```bash
git clone https://github.com/develeap/beastmode.git
cd beastmode

# Authenticate Claude Code (uses your subscription — no API key needed)
claude login

# Configure
cp .env.example .env
# Edit .env — fill in:
#   GITHUB_TOKEN=ghp_...
#   PROJECT_DIR=/path/to/your/project
# (Optional: ANTHROPIC_API_KEY — if set, used for faster direct API calls)

# Start
docker compose up -d

# Open the board UI
open http://localhost:8420
```

The board UI runs at `:8420`, the board API at `:8080`. No password needed when running locally.

## Option 3: Bare Metal (EC2 / Linux Server)

> **Note:** For contributors working from source. If you don't need the source code, use Option 0 instead.

For full control, install directly on a Linux server.

### Prerequisites

| Tool | Install |
|------|---------|
| Node.js 24+ | `curl -fsSL https://deb.nodesource.com/setup_24.x \| bash && apt install nodejs` |
| Python 3.12+ | `apt install python3 python3-venv` |
| Git | `apt install git` |
| Docker | `curl -fsSL https://get.docker.com \| sh` |
| GitHub CLI | `apt install gh` |
| Claude Code CLI | `npm install -g @anthropic-ai/claude-code` |

### Setup

```bash
# Clone
git clone https://github.com/develeap/beastmode.git
cd beastmode

# Initialize (interactive wizard)
npx beastmode init

# Or non-interactive:
cp .env.example .env
# Edit .env with your API keys

# Install Python dependencies
cd board && python3 -m venv .venv && .venv/bin/pip install -e . && cd ..
cd daemon && python3 -m venv .venv && .venv/bin/pip install -e . && cd ..

# Build CLI
cd cli && npm install && npm run build && cd ..

# Deploy all 3 services (systemd)
npx beastmode deploy
```

This creates and starts three systemd services:
- `beastmode-ui` — Board UI at :8420 (password-protected)
- `beastmode-board` — Board API at :8080 (localhost only)
- `beastmode` — Pipeline daemon (polls board, runs pipeline)

### Verify

```bash
# Check all services
beastmode deploy --status

# Run health check
beastmode doctor

# View daemon logs
sudo journalctl -u beastmode -f
```

## Adding Your Project

After BeastMode is running:

1. Open the board UI
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

All configuration lives in `config/beastmode.daemon.json`. Key settings:

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

Edit via the **Settings** page in the board UI — changes apply on the next daemon poll cycle (no restart needed).

## Managing Services

```bash
# Status of all services
beastmode deploy --status

# Stop everything
beastmode deploy --stop

# Restart after config change
sudo systemctl restart beastmode

# View logs
sudo journalctl -u beastmode -f          # daemon
sudo journalctl -u beastmode-ui -f       # board UI
sudo journalctl -u beastmode-board -f    # board API
```

## Authentication & Keys

| What | How | What it's for |
|------|-----|---------------|
| Claude Code | `claude login` | **All AI** — every agent runs through the Claude Code CLI (subscription) |
| `GITHUB_TOKEN` | GitHub → Settings → Developer Settings → Personal Access Tokens | Creating PRs, reading repos |
| `ANTHROPIC_API_KEY` (optional) | [console.anthropic.com](https://console.anthropic.com) | If set, used for faster direct API calls (no subprocess overhead). Falls back to CLI when absent. |
| `AWS_ACCESS_KEY_ID` (optional) | AWS IAM Console | Only needed for deploy and infra provisioning |

No `ANTHROPIC_API_KEY` needed — everything works with just `claude login`. Setting it is a performance optimization, not a requirement.

## Cost Estimates

| Component | Cost | Notes |
|-----------|------|-------|
| EC2 (t3.xlarge) | ~$120/month | BeastMode instance (4 vCPU / 16 GB — recommended) |
| Claude Code subscription | Included in plan | Max/Team plan covers all AI usage |
| Idle polling | $0 | No AI cost when no tasks are running |
| ECR | ~$1/month | Container image storage |
| ECS + RDS (if deployed) | ~$80/month | Your application's infrastructure |

## Next Steps

- Read the [Getting Started](../getting-started.md) guide for using BeastMode
- Check the [Pipeline](../pipeline.md) docs for how the pipeline works
- See [Configuration](../configuration.md) for all settings
- Review [Resilience](../resilience.md) for failure handling
