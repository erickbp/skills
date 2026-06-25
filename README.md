# Iterative Builder

A skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that plans and implements **one task at a time against the actual codebase state**. After each task is built, reviewed, and merged, the skill reads the real codebase again to design the next task — so every task card reflects what the code actually looks like, not a projected future state.

Invoke it with `/iterative-builder`.

## What makes it different

Most planning tools decompose all the work up front, then execute against a plan that drifts from reality as code lands. Iterative Builder instead designs **one task card at a time**, each against the current `main` (including everything merged so far), and only plans the next task after the previous one is built, reviewed, and merged.

It also **fans the read- and verify-heavy steps out across parallel sub-agents** (foreground by default, escalating to background orchestration on the heaviest steps) to widen coverage without weakening any gate. The manifest and validation gates stay intact, the per-task card gate pauses only for sensitive or complex cards, `feature-dev:code-reviewer` remains the single authoritative review gate, and a Codex second opinion runs automatically after a reviewer PASS (advisory only — it never gates the merge).

After each task is merged, it **stops for a context reset**: it prints a handoff and you run `/clear` + `/iterative-builder @TASKS/MANIFEST.md`, so the next task is designed from a clean, disk-backed context (the manifest and task cards on disk hold all the state — nothing is lost). Cohesive, low-risk, uniform work can also land as a single larger (`L`) card instead of being force-split.

## Installation

Copy the skill folder into your project's `.claude/skills/` directory or your user-level `~/.claude/skills/` directory:

```bash
# Run from the repository root (the folder that contains iterative-builder/).

# Project-level (recommended for team-shared skills):
cp -r iterative-builder /path/to/your/project/.claude/skills/

# User-level (available across all projects):
cp -r iterative-builder ~/.claude/skills/
```

## Prerequisites

This skill delegates card design and code review to external agents:

- **`feature-dev` plugin (required).** Provides the `code-architect` (task-card design) and `code-reviewer` (the authoritative review gate). Install it before running, or the workflow stops at the first task card.
- **`codex` plugin (optional).** Powers the advisory Codex second opinion after a reviewer PASS. Without it that step is skipped and the build is never blocked.

---

## Overview

The iterative builder plans and implements one task at a time against the **actual codebase state**. After each task is built, reviewed, and merged, it reads the real codebase again to design the next task. This means every task card reflects what the code actually looks like — not a projected future state.

Seven read- and verify-heavy steps fan out across parallel sub-agents — repo grounding, decision ledger, requirement extraction, card design, card quality-check, review, and the success-criteria check. These run as foreground sub-agents by default and escalate to true background orchestration (the Workflow tool) on the three heavy steps (review, repo grounding, success-criteria check) when the diff or repo is large enough. This widens coverage on large or risky work without weakening any gate: tasks are still designed, built, reviewed, and merged one at a time; the manifest and validation gates stay, and the per-task card gate pauses only for sensitive or complex cards. After each task is merged, the skill stops and hands off a full `/clear` + re-invoke (`/iterative-builder @TASKS/MANIFEST.md`) so the next task starts from a clean, disk-backed context — the manifest and task cards on disk hold all the state, so nothing is lost across the reset; cohesive, low-risk, uniform work may also land as a single larger `L` card.

### Quick Start

**Starting fresh** — pass your plan, PRD, or requirements doc:

```
/iterative-builder @PLAN.md
```

**Resuming** — pass the manifest that was created during a previous session:

```
/iterative-builder @TASKS/MANIFEST.md
```

The skill reads the manifest, sees which requirements are done, in-progress, or pending, and picks up where it left off. It prints this exact resume command in the handoff block after every task, when it stops for the context reset.

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
                        card decision            │
                              ↓                  │
                   implement in worktree         │
                              ↓                  │
                     code review + fix           │
                              ↓                  │
                     merge to main               │
                              ↓                  │
                     update manifest             │
                              ↓                  │
                   more requirements?            │
                              ↓                  │
            yes → stop: /clear + re-invoke ──────┘
                              │
                              no
                              ↓
Phase 3: Validation
  Check success criteria → handle gaps → finalize manifest
```

The seven read- and verify-heavy steps fan out internally, which widens coverage *within* a step but never changes the structure shown here: tasks are still designed, built, reviewed, and merged one at a time; the manifest and validation gates stay, and the per-task card gate pauses only for sensitive or complex cards.

**Key user interaction points**: You approve the manifest (once) and any sensitive or complex task card (before implementation). Routine cards proceed automatically with a non-blocking note you can reply to. Everything else is automated.

### What You'll Be Asked

You'll be asked to approve the **manifest** once (always), and a **task card** only when it's sensitive or complex — routine cards proceed automatically:

**1. Manifest approval** (Phase 1, once)

The manifest contains:
- A goal summary
- Discovered facts about your repo (topology, frameworks, conventions)
- A decision ledger (locked decisions, agent discretion, deferred items, open questions)
- Success criteria
- Numbered requirements with statuses

When presented, you can: approve as-is, add/remove/modify requirements, change locked decisions, or clarify open questions.

**2. Task card approval** (Phase 2, only for sensitive or complex cards)

Each task card contains:
- What the task accomplishes and which requirement(s) it addresses
- Context anchor (why this task exists, what to read first)
- Scope, non-goals, and constraints
- Must-haves (observable behaviors, concrete artifacts, critical wiring)
- Acceptance criteria and verification commands

The orchestrator auto-approves and starts implementing a **routine** card (low or medium risk; additive, reversible, or feature-flagged; not high-blast-radius; no blocked decision), surfacing a one-line non-blocking note you can reply to. It pauses for your explicit approval only when a card is **sensitive or complex** — high risk, high-blast-radius (a schema/migration/breaking/security-boundary/cross-protocol change, Rule 7), a change that isn't safe to land blind (coordinated rollout, deferred cleanup, or unassessed rollback safety), or blocked on an open decision. When it pauses, you can: approve, request changes, defer the requirement, or inject a new requirement.

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
Resume by passing the manifest:
```
/iterative-builder @TASKS/MANIFEST.md
```
The manifest tracks all progress. Claude Code reads it, sees where things stand, and continues from the right point.

**"I ran `/clear` at the wrong time"**
Same resume command — re-run with `@TASKS/MANIFEST.md`. All state lives on disk in the manifest and task cards (not in conversation), so a `/clear` at any point is safe — re-running picks up from the manifest. (The skill already stops for a `/clear` + re-invoke after every task, so an out-of-band `/clear` just brings the next resume forward.)

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
