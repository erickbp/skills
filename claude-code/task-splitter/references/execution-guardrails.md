# Execution Guardrails

Execution-time quality controls are split into two levels: orchestrator steps (executed by the agent reading TASKS.md) and sub-agent guardrails (passed to each task's coding agent).

## Orchestrator Steps

These are executed by the orchestrator directly as part of the Per-Task Execution protocol. They are NOT relayed to sub-agents as instructions — the orchestrator runs them itself.

### Architecture Pre-Analysis

Before launching a sub-agent for a task, invoke a `feature-dev:code-architect` agent with the task card content. Ask it to produce an implementation blueprint given the current codebase state in the worktree.

- **Universal** — invoke for every task, regardless of complexity. For simple tasks the blueprint will naturally be brief.
- Pass the blueprint to the sub-agent as additional context alongside the task card.
- Task card Locked Decisions and Constraints always take precedence over the blueprint.
- The blueprint fills a gap the task card cannot: it reads the actual current codebase state at execution time and designs against what exists, not just what was planned.

### Post-Task Code Review

After a sub-agent completes and commits, invoke a `feature-dev:code-reviewer` agent scoped to the files created or modified by the task in the worktree.

- **Code quality only** — the reviewer focuses on bugs, logic errors, security vulnerabilities, code quality issues, and convention adherence.
- If the reviewer reports high-confidence issues: launch a fix sub-agent in the same worktree with the review feedback, then re-invoke the reviewer on updated files.
- **Max 5 review cycles.** If issues persist after 5 cycles, note unresolved concerns and proceed.
- Reviewer suggestions that conflict with the task card's Locked Decisions or Constraints are noted but not applied.
- **Exit criteria:** the reviewer reports no high-confidence issues, OR 5 review cycles have been exhausted.

## Sub-Agent Guardrails (Standard)

These four guardrails are passed to every sub-agent executing a task card. They are behavioral rules — no tool invocations required.

### 1. Analysis Paralysis Guard

If an executing agent makes 5+ consecutive read/search operations without writing code, it should stop and either write code (it has enough context) or report that it is blocked with the specific missing information. Continued reading without action is a stuck signal.

### 2. Deviation Classification

While executing, the agent will discover work not in the card. Apply automatically: auto-fix bugs, missing critical functionality, and blocking dependencies without asking. Stop and ask for architectural changes (new database tables, switching libraries, breaking API contracts). After 3 failed auto-fix attempts on a single issue, stop and report — do not keep retrying.

### 3. Self-Check Before Completion

Before marking a task done, the executing agent must verify its own claims: do created files exist? do they contain expected content? do verification commands pass? Do not declare completion without running the verification commands listed in the card.

### 4. Git Commit

After the self-check passes, the executing agent must commit its changes if the working directory is inside a git repository:

1. Check: `git rev-parse --is-inside-work-tree`. If not a git repo, skip this step.
2. Stage only the files created or modified by this task.
3. Commit with the message format: `[TASK-NN] <task title>` (e.g., `[TASK-03] Add user authentication endpoint`).
4. Do not push to a remote — only commit locally.
5. If the commit fails, report the failure but do not block task completion.
