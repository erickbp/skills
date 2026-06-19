# Task Card Template

Use this template when designing task cards during the per-task loop. Each card is written to `TASKS/TASK-NN.md`.

## Template

```md
# [TASK-NN] <Short action-oriented title>

## Execution Protocol

1. **Setup**: Confirm you are on branch `task/TASK-NN` in the worktree. The branch and worktree were created by the orchestrator — do not recreate them.
2. **Analyze**: Read the codebase areas relevant to this task. Identify existing patterns, files, and conventions. Design the implementation approach before writing code.
3. **Implement**: Execute the task card below following the guardrails.
4. **Verify**: Run the verification commands listed in the card. Confirm created files exist and contain expected content.
5. **Commit**: Stage only files created or modified by this task. Commit with message `[TASK-NN] <title>`. Do not push.

## Guardrails

1. **Analysis paralysis guard.** 5+ consecutive read/search operations without writing code = stop and either write code or report blocked.
2. **Deviation classification.** Auto-fix: bugs, missing critical functionality, blocking deps. Stop and ask: architectural changes, new tables, breaking contracts. After 3 failed auto-fix attempts, stop and report.
3. **Self-check before completion.** Verify own claims: do files exist? do they contain expected content? Run the verification commands before declaring done.

## Task Card

- Goal: <what this task accomplishes>
- Addresses: <REQ-NN identifiers this task delivers>
- Context Anchor:
  - Why: <one sentence linking this task to the delivery goal>
  - Required Reading: <verified file paths or DISCOVERY-NN.md, or "None">
  - Predecessor Outputs: <specific files/artifacts from prior tasks, or "None — first task">
  - Patterns to Follow: <existing repo files to match, or "None (greenfield)">
- In Scope: <verified file paths, directories, or areas labeled '(convention-based)' or '(estimated)'>
- Non-Goals: <what is explicitly out of scope for this task>
- Change Type: <discovery / scaffolding / schema / backend / frontend / infra / tests / docs / rollout / cleanup>
- Risk Level: <low / medium / high>
- Locked Decisions: <only user-binding decisions that constrain this card, or "None">
- Must-Haves:
  - Truths:
    - <observable behavior or invariant>
  - Artifacts:
    - <file, module, endpoint with substance constraints>
  - Key Links:
    - <critical connection that must be wired>
- Acceptance Criteria:
  - <criterion 1>
  - <criterion 2>
- Verification Prerequisites:
  - <None, or known tools / services / env needed>
- Verification Commands:
  - `<command 1>`
  - `<command 2>`
- Constraints:
  - <behavior, API, dependency, performance, security constraints>
- Change Safety: <additive / reversible / feature-flagged / coordinated / cleanup-later / unknown>
- Rollout / Rollback Notes:
  - <only when materially relevant; otherwise "None">
- Failure Signals:
  - <signs this card is underspecified or likely to regress; otherwise "None">
- Notes / Risks:
  - <only if materially useful; otherwise "None">
```

## Field Descriptions

### Goal
What this task accomplishes. Should be action-oriented and specific. Must not contain scope-reducing language ("initial structure", "stub", "placeholder") when the requirement demands full behavior.

### Addresses
The `REQ-NN` identifier(s) this task delivers. Links the task to the manifest's requirement list.

### Context Anchor
Provides all context a fresh-thread agent needs:
- **Why**: One sentence linking this task to the overall delivery goal.
- **Required Reading**: Verified file paths the agent should read before starting, or `DISCOVERY-NN.md` references. "None" if no specific reading needed.
- **Predecessor Outputs**: Specific files or artifacts created by prior tasks that this task builds on. "None — first task" for the first task.
- **Patterns to Follow**: Existing repo files whose patterns the agent should match. "None (greenfield)" for the first task in a greenfield project.

### In Scope
Verified file paths, directories, or areas this task will create or modify. Label convention-based paths accordingly. Use `(estimated)` for areas that cannot be verified.

### Non-Goals
Explicit boundaries — what this task must NOT do, even if related.

### Change Type
One of: discovery, scaffolding, schema, backend, frontend, infra, tests, docs, rollout, cleanup.

### Risk Level
- **low**: additive, isolated, well-understood
- **medium**: touches shared code, has integration points
- **high**: breaking changes, data migration, security-sensitive

### Locked Decisions
Only the locked decisions from the ledger that directly constrain THIS card. "None" if no locked decisions apply.

### Must-Haves
The compact execution contract:
- **Truths**: Observable behaviors or invariants. NOT implementation steps. If a Truth can be satisfied by an empty file, reframe it.
- **Artifacts**: Concrete files, modules, endpoints that must exist. Include substance constraints for high-risk files (`min_lines`, `contains`, `exports`).
- **Key Links**: Critical connections between artifacts. Especially important for integration work.

### Acceptance Criteria
Specific, testable conditions that prove the task is complete. Must describe externally visible behavior, contract guarantees, or data invariants.

### Verification Prerequisites
Tools, services, or environment requirements needed to run verification commands. "None" if standard dev environment suffices.

### Verification Commands
Concrete commands to verify the task. Should include both structural checks (build, file existence) and behavioral checks (test execution, content validation). Label inferred commands as best-effort.

### Constraints
Behavior, API, dependency, performance, security constraints that the implementation must respect.

### Change Safety
How reversible the change is: additive, reversible, feature-flagged, coordinated, cleanup-later, unknown.

### Rollout / Rollback Notes
Only when materially relevant — migration steps, feature flag configuration, rollback procedures. "None" for most tasks.

### Failure Signals
Signs that this card is underspecified or the implementation is likely to regress. Helps the implementing agent know when to stop and ask. "None" for straightforward tasks.

### Notes / Risks
Additional context only if materially useful. "None" for most tasks.
