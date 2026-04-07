# Cost Analysis

BeastMode is a 13-agent pipeline that spawns Claude Code CLI subprocesses for each stage (spec, coder, verifier, PR reviewer, production verifier, healer, etc.).

## How BeastMode Uses Claude

BeastMode runs Claude through the **Claude Code CLI** (`claude`), not direct API calls. Each pipeline stage spawns a `claude` subprocess with a prompt and tools. The CLI handles authentication, prompt caching, and token management internally.

### Billing Models

| Method | How it works | Best for |
|--------|-------------|----------|
| **Claude Code subscription** (default) | `claude login` — usage included in your Max/Team plan | Most users. No per-token billing. |
| **API key** (optional) | `ANTHROPIC_API_KEY` in `.env` — billed per token via Anthropic API | High-volume factories, precise cost tracking |

Most users should use the Claude Code subscription. The API key is an optional optimization for faster direct API calls and granular cost visibility.

### Model Tiering

**Model tiering is enabled by default** (`cost.model_tiering_enabled: true`). The pipeline uses different models per role to balance capability and cost:

- **Opus 4.6** — spec phase (Refiner/Planner/Scenario Designer) and coder iteration 1
- **Sonnet 4.6** — coder iterations 2+, verifier, PR reviewer, self-healing
- **Haiku 4.5** — NLSpec pre-check (diff-vs-spec compliance)

See `models.*` in `config/beastmode.daemon.json` for the per-role model assignments.

## Per-Step Resource Usage

Each Claude subprocess incurs a ~23K token system prompt overhead (Claude Code base + tool definitions). Prompt caching reduces this on subsequent calls within the cache window. Usage below assumes default model tiering.

| Step | Approx. Input Tokens | Approx. Output Tokens | Notes |
|------|----------------------|----------------------|-------|
| **Phase 1: Spec + Plan + Scenarios** | 50K-200K | 20K-60K | Reads codebase, generates 3 artifacts. Many tool calls (Read, Grep, Write). |
| **Phase 2: Build-Verify Loop** (per iteration) | 100K-300K | 50K-150K | Reads plan, writes code, runs Playwright scenarios. Heaviest step. |
| **PR Review** | 30K-100K | 2K-5K | Reads PR diff + NLSpec + satisfaction. Mostly input-heavy. |
| **Production Verification** | 50K-200K | 10K-30K | Drives Playwright browser for each scenario. Many tool calls. |
| **Comment Interpretation** | 15K-25K | 50-200 | Classification only. Runs every poll cycle when awaiting approval. |
| **Spec/Epic Revision** | 30K-100K | 10K-30K | Re-reads artifacts + feedback, rewrites. |
| **Self-Healing (Tier 1)** | 30K-80K | 5K-20K | Reads error context + journal, diagnoses + fixes. |
| **CI Failure Fix** | 30K-100K | 5K-20K | Reads CI logs + PR diff, applies fix. |
| **Epic Decomposition** | 30K-100K | 10K-30K | Reads codebase, generates stories. |
| **Deep Planning** | 40K-150K | 15K-40K | Multi-round Q&A. Longest planning sessions. |

## Typical Pipeline Usage Per Code Task

```
Phase 1 (spec + plan + scenarios)           ~150K tokens
Approval polling (3-5 comment checks)       ~60K tokens
Phase 2 (build, 1-3 iterations)             ~300K-900K tokens
PR review                                   ~50K tokens
Merge + optional CI fix                     ~0-100K tokens
Production verification                     ~100K tokens
                                            ─────────────
Total per code task (1 iteration)           ~600K-800K tokens
Total per code task (2 iterations)          ~1M-1.5M tokens
```

With Claude Code subscription, this usage is included in your plan. With API key billing, multiply by the per-token rates for each model tier.

## Cost Savings from Pipeline Optimizations

These features reduce wasted Claude sessions regardless of billing model:

| Feature | What It Prevents | Relative Savings |
|---------|-----------------|-----------------|
| Coder self-check | Build failures caught in same session | Free (zero extra cost) |
| NLSpec compliance pre-check | Full verifier session when NLSpec gaps exist | ~1 full verifier session (Haiku vs Opus/Sonnet) |
| Fail-fast (iter 1 < 0.3) | Wasted iterations when spec is fundamentally wrong | Up to 4 skipped build-verify cycles |
| Infra failure retry (no counter) | Wasted iterations on transient rate-limit/503 issues | 1 build-verify cycle per prevented false iteration |
| Pre-flight health checks | Wasted Playwright session on unreachable service | 1 verifier session per skip |
| Tier 0 healing | Claude session for deterministic fixes (migrations, health) | 1 healing session |
| Failure classification | Rebuild on infra issues (never helps) | 1 build-verify cycle |
| Dependency ordering | Building story N before prerequisites are done | 1 full cycle per wasted attempt |
| Satisfaction ceiling | Infinite Stuck → reset → build → verify loops | All rebuild + heal costs per cycle |
| Epic story auto-approval | Human reviewing every story spec | Polling sessions + human time |
| SRE health gating | Wasted Playwright sessions when infra is down | 1 verifier session per skip |
| Incremental verification | Re-running passing scenarios on iterations 2+ | Partial verifier session |

## Cost Optimization Interactions

**Incremental verification + build check** — when a speculative build check fails, it writes a pseudo-satisfaction file with a single "build_check" scenario. The incremental verification system skips these pseudo-iterations when looking for the last real verifier run, ensuring that iteration N+1 correctly re-runs only the scenarios that failed in the last actual verification.

**Model tiering** — Phase 2 uses Opus for the first coder iteration (highest capability for initial implementation) and Sonnet for iterations 2+ (~40% cheaper, the coder already has feedback). The NLSpec pre-check uses Haiku (5x cheaper than Opus) since it's a simple diff-vs-spec comparison.

## Infrastructure Costs

| Component | Cost | Notes |
|-----------|------|-------|
| EC2 (t3.xlarge) | ~$120/month | BeastMode instance (4 vCPU / 16 GB — recommended) |
| Claude Code subscription | Included in Max/Team plan | Primary AI auth — no per-token billing |
| Idle polling | $0 | No AI cost when no tasks are running |
| ECR | ~$1/month | Container image storage (if deploying to AWS) |
| ECS + RDS (if deployed) | ~$80/month | Your application's infrastructure |

## Measuring Actual Usage

The Claude CLI provides token usage data via `--output-format json`, which returns token breakdowns for each session:

```bash
# Measure a single Claude session's usage
echo "hello" | claude --output-format json | jq '.[-1] | .modelUsage'
```

The daemon logs every Claude subprocess to `daemon/logs/`. Each run log contains the session output including token counts.
