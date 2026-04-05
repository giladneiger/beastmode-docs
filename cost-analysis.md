# Cost Analysis

BeastMode is a 13-agent pipeline that spawns Claude subprocesses for each stage (spec, coder, verifier, PR reviewer, production verifier, healer, etc.). [Claude API pricing](https://platform.claude.com/docs/en/about-claude/pricing) per million tokens:

| Model | Input | Output | Relative to Opus 4.6 |
|-------|------:|-------:|:---------------------|
| Claude Opus 4.6 | $5 | $25 | — |
| Claude Sonnet 4.6 | $3 | $15 | ~40% cheaper (1.67x) |
| Claude Haiku 4.5 | $1 | $5 | 80% cheaper (5x) |

Prompt caching reads are 0.1x the base input rate, and 5-minute cache writes are 1.25x — a cache hit after one subsequent call already pays for the write.

**Model tiering is enabled by default** (`cost.model_tiering_enabled: true`). The pipeline uses:
- **Opus 4.6** — spec phase (Refiner/Planner/Scenario Designer) and coder iteration 1
- **Sonnet 4.6** — coder iterations 2+, verifier, PR reviewer, self-healing
- **Haiku 4.5** — NLSpec pre-check (diff-vs-spec compliance)

See `models.*` in `config/beastmode.daemon.json` for the per-role model assignments.

## Per-Step Token Estimates

Each Claude subprocess incurs a ~23K token system prompt overhead (Claude Code base + tool definitions). With prompt caching, this costs ~$0.14 on first call and ~$0.01 on subsequent calls within the cache window. Costs below assume the default model tiering.

| Step | Input Tokens | Output Tokens | Est. Cost | Notes |
|------|-------------|--------------|-----------|-------|
| **Phase 1: Spec + Plan + Scenarios** | 50K-200K | 20K-60K | $1.00-5.00 | Reads codebase, generates 3 artifacts. Many tool calls (Read, Grep, Write). |
| **Phase 2: Build-Verify Loop** (per iteration) | 100K-300K | 50K-150K | $2.50-10.00 | Reads plan, writes code, runs Playwright scenarios. Heaviest step. |
| **PR Review** | 30K-100K | 2K-5K | $0.20-0.65 | Reads PR diff + NLSpec + satisfaction. Mostly input-heavy. |
| **Production Verification** | 50K-200K | 10K-30K | $0.75-3.50 | Drives Playwright browser for each scenario. Many tool calls. |
| **Comment Interpretation** | 15K-25K | 50-200 | $0.08-0.13 | Classification only. Runs every poll cycle when awaiting approval. |
| **Spec/Epic Revision** | 30K-100K | 10K-30K | $0.50-3.00 | Re-reads artifacts + feedback, rewrites. |
| **Self-Healing (Tier 1)** | 30K-80K | 5K-20K | $0.30-2.00 | Reads error context + journal, diagnoses + fixes. |
| **CI Failure Fix** | 30K-100K | 5K-20K | $0.30-2.50 | Reads CI logs + PR diff, applies fix. |
| **Epic Decomposition** | 30K-100K | 10K-30K | $0.50-3.00 | Reads codebase, generates stories. |
| **Deep Planning** | 40K-150K | 15K-40K | $0.75-5.00 | Multi-round Q&A. Longest planning sessions. |

## Typical Cost Per Code Task (Full Pipeline)

```
Phase 1 (spec + plan + scenarios)           $1.00 - $5.00
Approval polling (3-5 comment checks)       $0.25 - $0.65
Phase 2 (build, 1-3 iterations)             $2.50 - $30.00
PR review                                   $0.20 - $0.65
Merge + optional CI fix                     $0.00 - $2.50
Production verification                     $0.75 - $3.50
                                            ─────────────
Total per code task                         $4.70 - $42.30
Typical (1 iteration, no failures)          ~$6-8
Typical (2 iterations, smooth pipeline)     ~$12-18
```

## Cost Savings from Resilience Features

| Feature | What It Prevents | Savings Per Occurrence |
|---------|-----------------|----------------------|
| Coder self-check | Build failures and NLSpec gaps caught in same session | $0 (zero extra cost — same Claude session) |
| NLSpec compliance pre-check | Full verifier session when NLSpec gaps exist | $2.50-10.00 (cost: ~$0.01 Haiku call) |
| Fail-fast (iter 1 < 0.3) | 4 wasted iterations when spec is fundamentally wrong | $10.00-40.00 (prevents iterations 2-5) |
| Infra failure retry (no counter) | Wasted iterations on transient Bedrock/rate-limit/503 issues | $2.50-10.00 per prevented false iteration |
| Pre-flight checks | Wasted Playwright session on unreachable service | $0.75-3.50 |
| Prod-fix loop (max 3) | Instant escalation to Stuck for fixable prod issues | Saves human intervention time |
| Tier 0 healing | Claude session for deterministic fixes (migrations, health) | $0.30-2.00 |
| Failure classification | Rebuild on infra issues (never helps) | $2.50-10.00 |
| Dependency ordering | Building story N before its prerequisites are done | $2.50-10.00 per wasted cycle |
| Satisfaction ceiling | Infinite Stuck → reset → build → verify → Stuck loops | All rebuild + heal costs per cycle |
| Epic story auto-approval | Human reviewing every story spec (already covered by epic) | ~$0.08-0.13 per approval poll + human time |
| SRE health gating | Wasted Playwright sessions when infra is down | $0.75-3.50 per skipped session |

## Cost Optimization Interactions

**Incremental verification + build check** — when a speculative build check fails, it writes a pseudo-satisfaction file with a single "build_check" scenario. The incremental verification system (`_find_last_real_verification()`) skips these pseudo-iterations when looking for the last real verifier run, ensuring that iteration N+1 correctly re-runs only the scenarios that failed in the last actual verification, not just the build-check pseudo-scenario.

**Model tiering** — Phase 2 uses Opus for the first coder iteration (highest capability for initial implementation) and Sonnet for iterations 2+ (~40% cheaper, the coder already has feedback). The NLSpec pre-check uses Haiku (5x cheaper than Opus) since it's a simple diff-vs-spec comparison.

## CI: Self-Hosted Runner

GitHub Actions CI runs on a **self-hosted runner** on the BeastMode EC2 instance (`beastmode-ec2`). This consumes zero GitHub Actions minutes. The runner is installed as a systemd service (`actions.runner.giladneiger-journey-v2.beastmode-ec2.service`) and auto-starts on reboot.

## Measuring Actual Costs

The Claude CLI provides exact cost data via `--output-format json`, which returns `total_cost_usd` and per-model token breakdowns for each session. To enable cost tracking:

```bash
# Test: measure a single Claude session cost
echo "hello" | claude --output-format json | jq '.[-1] | {total_cost_usd, usage: .modelUsage}'
```

The daemon logs every Claude subprocess to `daemon/logs/`. Future improvement: switch to `--output-format json` with a cost-extraction wrapper to log per-step costs and accumulate per-task totals.
