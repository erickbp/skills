# Claude Code Skills

Skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## Available Skills

| Skill | Description |
|-------|-------------|
| [iterative-builder](iterative-builder/) | Plan one task at a time against the actual codebase, build/review/merge it, then plan the next — ideal when the sequence of work should emerge organically |

## Installation

Copy a skill folder into your project's `.claude/skills/` directory or your user-level `~/.claude/skills/` directory:

```bash
# Project-level (recommended for team-shared skills)
cp -r iterative-builder /path/to/your/project/.claude/skills/

# User-level (available across all projects)
cp -r iterative-builder ~/.claude/skills/
```

---

## Iterative Builder

### Overview

The iterative builder plans and implements one task at a time against the **actual codebase state**. After each task is built, reviewed, and merged, it reads the real codebase again to design the next task. This means every task card reflects what the code actually looks like — not a projected future state.

### Quick Start

**Starting fresh** — pass your plan, PRD, or requirements doc:

```
/iterative-builder @PLAN.md
```

**Resuming** — pass the manifest that was created during a previous session:

```
/iterative-builder @TASKS/MANIFEST.md
```

That's it. The skill reads the manifest, sees which requirements are done, in-progress, or pending, and picks up where it left off.

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
Resume with:
```
/iterative-builder @TASKS/MANIFEST.md
```
The manifest tracks all progress. Claude Code reads it, sees where things stand, and continues from the right point.

**"I ran `/clear` at the wrong time"**
Same resume command — `/iterative-builder @TASKS/MANIFEST.md`. The workflow is designed around `/clear` happening between tasks, so all state lives on disk in the manifest and task cards, not in conversation context.

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
