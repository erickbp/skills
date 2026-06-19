# Iterative Builder

Skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that plan and implement **one task at a time against the actual codebase state**. After each task is built, reviewed, and merged, the skill reads the real codebase again to design the next task — so every task card reflects what the code actually looks like, not a projected future state.

This folder ships three variants of the same workflow. They share identical phases, approval gates, manifest, and task-card format — they differ only in how much of the work runs in parallel and how hands-off the run is.

## Variants

| Skill | Invoke with | Best for |
|-------|-------------|----------|
| [iterative-builder-base](iterative-builder-base/) | `/iterative-builder-base` | **The default.** Each step runs as a single pass. You approve the manifest and every task card; the optional Codex second opinion is offered through a manual prompt. |
| [iterative-builder-ultracode](iterative-builder-ultracode/) | `/iterative-builder-ultracode` | Larger or higher-risk work. Seven read- and verify-heavy steps fan out across parallel sub-agents — foreground by default, escalating to true background orchestration (the Workflow tool) on the three heavy steps (review, repo grounding, success-criteria check) when the diff or repo is large. Every approval gate stays intact; the Codex prompt auto-defaults to *yes* after one minute. |
| [iterative-builder-ultracode-auto](iterative-builder-ultracode-auto/) | `/iterative-builder-ultracode-auto` | Unattended runs. Same parallel fan-out as `ultracode`, tuned to move through the prompts automatically so a build can proceed with minimal supervision. |

All three read and write the same `TASKS/` files, so you can even resume a build with a different variant than you started with. **When in doubt, use `base`.**

## Installation

Each variant is a standalone skill. Copy the folder(s) you want into your project's `.claude/skills/` directory or your user-level `~/.claude/skills/` directory:

```bash
# Run from this directory. Copy one variant — or all three.

# Project-level (recommended for team-shared skills):
cp -r iterative-builder-base /path/to/your/project/.claude/skills/

# User-level (available across all projects):
cp -r iterative-builder-base ~/.claude/skills/
```

Installing more than one variant is fine — each registers under its own name and is invoked independently.

---

## Overview

The iterative builder plans and implements one task at a time against the **actual codebase state**. After each task is built, reviewed, and merged, it reads the real codebase again to design the next task. This means every task card reflects what the code actually looks like — not a projected future state.

The three variants differ only in execution strategy:

- **`base`** runs each step as a single pass. Simplest to follow, fewest moving parts.
- **`ultracode`** fans the read- and verify-heavy steps out across parallel sub-agents — repo grounding, decision ledger, requirement extraction, card design, card quality-check, review, and success-criteria check. These run as foreground sub-agents by default and escalate to true background orchestration (the Workflow tool) on the three heavy steps when the diff or repo is large enough. This widens coverage on large or risky work without changing any gate.
- **`ultracode-auto`** uses the same parallel fan-out but is geared for unattended runs, advancing through the prompts automatically so the build needs minimal supervision.

### Quick Start

**Starting fresh** — pass your plan, PRD, or requirements doc to whichever variant you want (shown here with `base`):

```
/iterative-builder-base @PLAN.md
```

**Resuming** — pass the manifest that was created during a previous session:

```
/iterative-builder-base @TASKS/MANIFEST.md
```

The skill reads the manifest, sees which requirements are done, in-progress, or pending, and picks up where it left off. Resume with the **same variant you started with** — each one prints its exact resume command in the handoff block after every task. (To switch variants, swap the command, e.g. `/iterative-builder-ultracode @TASKS/MANIFEST.md`.)

### Workflow at a Glance

```
Phase 1: Bootstrap
  Read input → explore repo → build decision ledger → extract requirements
  → write TASKS/MANIFEST.md → present for approval
                                    ↓
                              ┌─ user approves ─┐
                              │                  │
Phase 2: Per-Task Loop        ▼                  │
  Pick next requirement → design task card ──────┤
                              ↓                  │
                        user approves            │
                              ↓                  │
                   implement in worktree         │
                              ↓                  │
                     code review + fix           │
                              ↓                  │
                     merge to main               │
                              ↓                  │
                     update manifest             │
                              ↓                  │
                     /clear context              │
                              ↓                  │
                   more requirements? ───yes─────┘
                              │
                              no
                              ↓
Phase 3: Validation
  Check success criteria → handle gaps → finalize manifest
```

The `ultracode` and `ultracode-auto` variants parallelize seven of these steps internally (the read- and verify-heavy ones listed above), which widens coverage *within* a step but never changes the structure shown here: tasks are still designed, built, reviewed, and merged one at a time, and every approval gate stays.

**Key user interaction points**: You approve the manifest (once) and each task card (before implementation begins). Everything else is automated.

### What You'll Be Asked

You'll be asked for approval at two points:

**1. Manifest approval** (Phase 1, once)

The manifest contains:
- A goal summary
- Discovered facts about your repo (topology, frameworks, conventions)
- A decision ledger (locked decisions, agent discretion, deferred items, open questions)
- Success criteria
- Numbered requirements with statuses

When presented, you can: approve as-is, add/remove/modify requirements, change locked decisions, or clarify open questions.

**2. Task card approval** (Phase 2, once per task)

Each task card contains:
- What the task accomplishes and which requirement(s) it addresses
- Context anchor (why this task exists, what to read first)
- Scope, non-goals, and constraints
- Must-haves (observable behaviors, concrete artifacts, critical wiring)
- Acceptance criteria and verification commands

When presented, you can: approve, request changes, defer the requirement, or inject a new requirement.

### Modifying the Manifest or Task Cards

**Preferred: ask Claude Code to make changes.** Just describe what you want ("add a requirement for rate limiting", "defer REQ-03", "change the locked decision about the ORM"). Claude Code updates the manifest or card and keeps everything internally consistent.

**Alternative: edit files directly.** You can open `TASKS/MANIFEST.md` or any `TASKS/TASK-NN.md` in your editor and make changes. If you do, tell Claude Code what you changed so it can reconcile — e.g., "I edited the manifest to add REQ-06, take a look."

### Files Created

All files live in a `TASKS/` directory at your project root:

| File | Purpose |
|------|---------|
| `MANIFEST.md` | Living state document — goal, requirements, progress, decisions, and adjustments. Updated after every task. This is the single source of truth for the build. |
| `TASK-NN.md` | Individual task card — self-contained instructions for implementing one piece of work. Created as each task is designed. |

### Common Scenarios

**"I closed Claude Code / it crashed mid-workflow"**
Resume with the same variant you started with, passing the manifest:
```
/iterative-builder-base @TASKS/MANIFEST.md
```
The manifest tracks all progress. Claude Code reads it, sees where things stand, and continues from the right point.

**"I ran `/clear` at the wrong time"**
Same resume command — re-run your variant with `@TASKS/MANIFEST.md`. The workflow is designed around `/clear` happening between tasks, so all state lives on disk in the manifest and task cards, not in conversation context.

**"I want to skip a requirement"**
Tell Claude Code to defer it: "defer REQ-04." It moves to the Deferred section of the decision ledger, the manifest is updated, and the workflow continues with the remaining requirements.

**"I want to add a new requirement mid-build"**
Tell Claude Code: "add a requirement for X." It assigns a new `REQ-NN` identifier, adds it to the manifest, logs the addition in the Adjustments Log, and designs a task card for it when it becomes the next logical piece of work.

**"A requirement changed"**
Tell Claude Code what changed: "REQ-02 now needs to support batch uploads too." It updates the requirement text, logs the change in the Adjustments Log, and accounts for the delta in upcoming tasks.

**"The build/tests are failing after merge"**
Claude Code will attempt to fix the failure. If it can't resolve it, you'll be presented with options: fix it manually and continue, abandon the task and redesign, or proceed with the failure acknowledged.

**"I want to review what's been done so far"**
Open `TASKS/MANIFEST.md` and check the **Completed Tasks** section — it has a summary of what each task built and which requirements it addressed. The **Requirements** section shows the status of every requirement at a glance.
