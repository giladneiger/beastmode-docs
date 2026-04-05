# Getting Started with BeastMode

BeastMode is a Dark Factory that turns task descriptions into verified, deployed software — autonomously. You describe what you want, BeastMode writes the spec, implements it, verifies it works, and ships it.

## Quick Start

### 1. Access the Board

Open the BeastMode Board UI in your browser. Ask your admin for the URL and password.

### 2. Add Your Project

Go to **Projects** in the sidebar and click **Add Project**. Point it at your Git repository. BeastMode auto-detects the tech stack (Node.js, Python, Go, Rust, Java, etc.).

### 3. Create a Task

Click **+** on the board or drag to the **Ready** column. Write a clear description of what you want:

**Good task descriptions:**
- "Add a /health endpoint that returns {status: 'ok', uptime: process.uptime()}"
- "Bug: the login button doesn't work on mobile — it's hidden behind the navbar"
- "Add pagination to the /api/users endpoint — 20 items per page, cursor-based"

**Bad task descriptions:**
- "Fix stuff" (too vague)
- "Rewrite the entire frontend" (too large — use an Epic)

### 4. Watch It Work

Once your task is in **Ready**, BeastMode picks it up within 60 seconds. Watch the status change:

```
Ready → Working on it → Waiting for Approval → Working on it → Ready for Review
→ Approved & Merge → Verifying Prod → Done
```

### 5. Review the Spec

When the task reaches **Waiting for Approval**, BeastMode has written a spec and test scenarios. Read the update on the task and:

- Comment **"approved"** to proceed to implementation
- Provide feedback ("change X to Y", "remove the Redis part") to revise

## Task Types

| Type | How to create | What happens |
|------|--------------|--------------|
| **Code task** | Any normal task description | Full pipeline: spec → build → verify → ship |
| **Bug fix** | Use words like "fix", "broken", "overflow", "not working" | Focused pipeline: narrow spec, original intent verification |
| **Epic** | Include "epic" in the name | Decomposes into ordered stories, each runs independently |
| **Deep Planning** | Include "deep planning" in the name | Iterative Q&A before decomposition — for complex/ambiguous tasks |

## Tips

- **Be specific.** "The sparkline charts overflow their card containers" is better than "dashboard looks broken."
- **Attach screenshots.** For visual bugs, attach a screenshot — BeastMode uses AI vision to analyze it.
- **One task = one feature.** Don't combine "add search AND fix the header AND update the footer" in one task. Split them.
- **For large features, use Epics.** "Epic: User management system" gets decomposed into ordered stories automatically.
- **Reset stuck tasks.** Comment **"reset"** on any Stuck task to clear all counters and restart from scratch.
- **Check the Learnings page.** BeastMode learns from every task. The Learnings page shows what it discovered about your project.

## Pipeline Stages Explained

| Stage | What's happening |
|-------|-----------------|
| **Ready** | Waiting for BeastMode to pick up |
| **Working on it** | Writing spec, building code, or running verification |
| **Waiting for Approval** | Spec + scenarios ready — needs your review |
| **Ready for Review** | PR created, being auto-reviewed |
| **Approved & Merge** | PR approved, merging, running CI, deploying |
| **Verifying Prod** | Running test scenarios against live production |
| **Done** | Completed and verified |
| **Stuck** | Failed after max retries — comment "reset" or ask for help |

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Task stuck in "Working on it" for hours | The daemon may have restarted. It auto-recovers, or comment "reset" |
| Spec doesn't match what I wanted | Provide feedback in the approval step — be specific about what to change |
| Wrong code generated | The spec probably drifted. Reset the task and write a clearer description |
| "Daemon: stopped" in the status bar | Contact your admin — the daemon service needs to be restarted |
| Can't see my project's board | Switch projects using the dropdown in the sidebar |

## FAQ

**How long does a task take?**
Simple tasks (add an endpoint, fix a CSS bug): 5-15 minutes. Medium features: 30-60 minutes. Complex features via Epic: 2-4 hours (split across stories).

**Can I create tasks from my phone?**
Yes. The board UI works on mobile — tap the hamburger menu (top-left) to access navigation.

**What if BeastMode gets the spec wrong?**
Provide feedback in the approval step. Be specific: "The NLSpec says to redesign the template system, but I just need the CSS overflow fixed." BeastMode will revise.

**What if I want to deploy but infrastructure isn't set up?**
BeastMode auto-creates an "Infrastructure Assessment" task when it detects no deploy config. Approve it to provision infrastructure, or ignore it to work in PR-only mode.

**Can multiple people work on the same project?**
Yes. BeastMode handles concurrent tasks with isolated git worktrees. Each task gets its own branch.
