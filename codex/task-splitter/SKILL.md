---
name: task-splitter
description: Decompose PRDs, RFCs, epics, roadmap items, migration plans, implementation plans, verifier reports, and UAT findings into execution-ready task cards with decision capture, requirement traceability, must-haves, and preflight self-checking. Use when the user wants fresh-thread-safe implementation cards or wants failed verification turned into repair cards.
---

# Task Splitter

Break a delivery plan into the smallest complete set of task cards that a coding agent can execute safely in fresh threads.

## Objective

- Produce the fewest cards that fully deliver the goal.
- Keep each card independently understandable, reviewable, testable, and as close as possible to independently shippable.
- Optimize for execution, not discussion.
- Make the decomposition decision-complete before returning it.

## What This Skill Adds

Compared to a plain task-card splitter, this skill also:

- captures user decisions before decomposition
- maps source requirements and behaviors to task IDs
- derives goal-backward must-haves for each card
- runs an internal checker pass before final output
- supports gap-closure replanning from verification or UAT failures

## Workflow

Always work through these stages in order.

### Stage 1: Ground the goal and the repo

Before writing cards:

1. Read the input carefully and identify:
   - product goal
   - scope boundaries
   - likely success criteria
   - likely risk classes
2. Explore the repo with tools. Use rg, file search, and file reads to verify:
   - Repo topology: monolith, service, monorepo, package, app
   - Affected subsystems: UI, API, domain logic, persistence, jobs, infra
   - Existing patterns: read 1–2 representative examples of similar features or modules to understand conventions
   - Data layer: ORM, migration framework, schema definitions, database type
   - API layer: framework, routing conventions, middleware stack
   - Test infrastructure: test framework, config, directory layout, naming conventions
   - Build/CI: build system, CI pipeline, deploy process
   - Dependencies: package manager, key libraries, version constraints
3. Do not invent repo facts that tools can verify. You must use tools to verify these facts.
4. If the repo cannot answer something and the answer is product intent, ask the user before proceeding.

### Stage 2: Build a Decision Ledger

Before decomposition, create a `Decision Ledger` with:

- `Locked Decisions`
- `Coding Agent's Discretion`
- `Deferred / Out of Scope`
- `Open Questions`

Rules:

- Ask questions only for user-judgment gaps, not repo facts.
- Treat `Locked Decisions` as binding.
- Treat `Deferred / Out of Scope` as forbidden scope.
- Use `Coding Agent's Discretion` only for real implementation freedom.
- If important questions remain unanswered, do not silently guess.

Read [references/decision-ledger.md](references/decision-ledger.md) when building or validating the ledger.

### Stage 3: Select the decomposition mode

Choose exactly one mode before writing cards:

- `standard_decomposition`
- `brownfield_extension`
- `gap_closure`
- `migration_heavy`

Read [references/modes.md](references/modes.md) after selecting the mode.

### Stage 4: Build the decomposition

Decompose for execution:

1. Prefer the smallest complete set of cards.
2. Prefer vertical slices over layer buckets when they can land safely.
3. Split on risk boundaries, contract boundaries, migration boundaries, and rollout boundaries.
4. Keep each card single-purpose.
5. Prefer additive and reversible sequencing.
6. Surface dependencies and parallel work clearly.
7. Separate required work from follow-up work.

Then produce:

- `Discovered Facts`
- `Decision Ledger`
- `Coverage Map`
- `Execution Waves`
- `Task Cards`

Read [references/coverage-and-must-haves.md](references/coverage-and-must-haves.md) when building the coverage map or card must-haves.

### Stage 5: Run the checker loop

Before finalizing, run a preflight review across:

- decision compliance
- requirement coverage
- dependency correctness
- scope sanity
- must-have completeness
- rollout safety
- verification realism

If the first pass fails, revise and re-check before returning output.

Read [references/checker-rubric.md](references/checker-rubric.md) for the review dimensions and revision loop.

## Core Rules

1. **Verify repo facts with tools; do not guess.**
   - Use rg, file search, and file reads before naming exact files, modules, tables, endpoints, commands, or conventions.
   - If something cannot be verified with tools (requires runtime, external service, staging environment), mark it `unverified` and explain why.
   - Do not invent exact filenames, modules, tables, endpoints, commands, or environment prerequisites unless verified by tools or explicitly provided in the input.
   - Prefer asking one round of clarifying questions over producing cards built on assumptions.

2. **Preserve intent before decomposition.**
   - Do not let implementation convenience override locked user decisions.
   - Do not quietly reintroduce deferred scope.
   - Do not reduce required scope when splitting by layer. If the source requires full behavior, every task that maps to that behavior must deliver its layer's full contribution — not a structural shell, stub, skeleton, placeholder, or "initial structure."
   - Scope-reducing language ("initial structure", "not full flow", "stub", "shell", "placeholder", "skeleton", "basic scaffold") in a task's Goal or In Scope is a defect when the mapped source requirement demands full behavior.
   - Carry forward stated constraints on behavior, API compatibility, latency, data model, dependency choices, rollout, security, compliance, and backward compatibility.
   - If a constraint changes task boundaries, split tasks accordingly.

3. **Anchor cards to outcomes, not just work.**
   - Every committed requirement or observable behavior must map to at least one task.
   - Every task must say what requirement, behavior, or failed truth it addresses.

4. **Use goal-backward must-haves.**
   - Every implementation card must include `Truths`, `Artifacts`, and `Key Links`.
   - These are for execution safety, not documentation theater.

5. **Keep discovery honest.**
   - Resolve what you can during planning.
   - Only create discovery cards when the unknown genuinely requires runtime behavior, external systems, or broader investigation than the planning pass can support.

6. **Keep cards fresh-thread safe.**
   - A new agent should be able to execute a card without rediscovering the plan.
   - If a card depends on discovery output, reference the relevant `DISCOVERY-NN.md` file in `Required Reading`.

7. **Separate risky rollout work.**
   - Split schema preparation, compatibility, backfill, cutover, and cleanup when applicable.
   - Isolate breaking changes and coordinated rollouts.

8. **Make verification executable.**
   - Include concrete commands whenever possible.
   - If commands are inferred, label them best-effort.
   - List known prerequisites for running them.

9. **Make acceptance criteria observable.**
   - Criteria must describe externally visible behavior, contract guarantees, data invariants, or testable internal outcomes.
   - Avoid vague criteria like "works correctly" or "is production ready".

10. **Surface dependencies clearly.**
    - Identify which cards block others.
    - Mark cards that can run in parallel.
    - Sequence tasks in the safest implementation order.

11. **Separate required work from optional work.**
    - Nice-to-have improvements, future cleanup, and post-launch hardening belong in a separate "Follow-up Tasks" section.
    - A behavior the source marks as required must not appear in Follow-up Tasks. If it cannot fit in the current card set, add or expand a required card — do not downgrade the behavior to optional.

12. **Keep the result compact.**
    - Do not add ceremony for tiny, low-risk changes.
    - For small work, keep the ledger and must-haves terse but still present.

13. **Do not write implementation code.**
    - This skill only plans and decomposes work.

## Discovery Cards

Create discovery cards only when planning cannot resolve an unknown up front.

Each discovery card must:

- state what is unknown
- state why planning could not resolve it
- produce a concrete output
- name the follow-on cards it unblocks
- write findings to `DISCOVERY-NN.md`

Follow-on cards that depend on a discovery card must reference the `DISCOVERY-NN.md` file as required reading in their Context Anchor.

A discovery card that could have been replaced by normal repo inspection is a defect.

## Migration and Contract Rules

For migration-heavy or contract-sensitive work:

- classify data change shape and rollout shape explicitly
- prefer additive and reversible sequencing
- separate schema prep, write compatibility, backfill, read cutover, and cleanup when needed
- isolate breaking or coordinated contract changes
- include rollback or mitigation notes for risky cards

For any schema or data change, classify the work as one of:
- none
- additive schema
- schema change requiring dual-read or dual-write
- backfill required
- read or write cutover
- destructive cleanup later
- unknown; requires repo or data-model review

For any endpoint, event, queue payload, RPC, SDK, shared type, or public interface change, classify it as:
- additive backward-compatible
- behavior change but contract stable
- breaking change
- unknown

## Gap-Closure Rules

In `gap_closure` mode:

- treat the input as failed truths, not just bug bullets
- normalize each gap into:
  - failed truth
  - root-cause guess
  - affected artifacts
  - missing wiring, missing tests, missing compatibility work, or missing rollout guardrails
- group repair cards by root cause when that reduces redundant work
- split behavior fixes from coverage-only fixes when they can land independently

## Sizing Rules

Use these buckets only as an internal check:

- `XS`: tiny, surgical, very low risk
- `S`: small, focused, standard PR
- `M`: moderate, still reviewable in one sitting

Do not create `L` or `XL` cards. Split them further.

## Task ID Rules

- Always use the prefix `TASK-` followed by a zero-padded two-digit number starting at `01`: `[TASK-01]`, `[TASK-02]`, ..., `[TASK-99]`.
- If there are more than 99 tasks, continue with three digits: `[TASK-100]`, `[TASK-101]`, etc.
- Never use project-specific prefixes like `[PA-01]`, `[AUTH-01]`, or `[DB-01]`. The prefix is always `TASK-`.
- Number tasks sequentially in the order they appear in the output.
- Apply this format everywhere: card headers, dependency references, and execution waves.

## Fresh-Thread Handoff Rule

Each task card must be usable as a standalone handoff in a fresh coding-agent thread.

The implementing agent should treat the task card as the task-specific operating instructions for the session and prioritize the card's constraints, non-goals, compatibility requirements, and rollout notes over generic defaults.

Do not assume access to prior task discussions or hidden context.

Because upfront discovery verifies repo facts before cards are written, task cards contain concrete verified file paths and conventions — not placeholders or estimates. The implementing agent should not need to re-discover what planning already verified.

When a task card depends on a discovery card, it must reference the corresponding `DISCOVERY-NN.md` file as required reading. The implementing agent should read that file before starting work to obtain the context the planning session could not resolve upfront.

The Context Anchor fields (Why, Required Reading, Predecessor Outputs, Patterns to Follow) replace implicit assumptions about what the agent "should know." If a task depends on prior wave outputs, name the specific artifacts, not just the task ID.

### Execution Model: One Task, One Fresh Sub-Agent

Every task card must be executed in its own fresh sub-agent or thread. This is not optional — it applies to all tasks, including the first one. The planning agent that runs this skill must not execute any task cards itself.

- Launch a new sub-agent for each task card.
- Pass the task card content (or point the agent to the relevant section in `TASKS.md`) as the agent's operating instructions.
- Do not carry conversation context from the planning session into task execution.
- Do not batch multiple task cards into a single agent session.
- Respect wave ordering: only launch a wave's tasks after all predecessor waves have completed.
- Every task runs in its own git worktree for isolation — even single-task waves. The Execution Protocol section in TASKS.md contains the full orchestrator instructions for worktree creation, wave merging, and commit discipline.
- When writing TASKS.md, include the Execution Protocol section and fill in the build verification command from Discovered Facts.

## Checkpoint Types

Assign checkpoint types to tasks that need human interaction.

- `None` (default): Task can be fully verified with automated commands. Most tasks use this.
- `human-verify: <what>`: Visual output, UX, or behavior that cannot be tested programmatically (e.g., layout correctness, animation smoothness, "does the flow feel right?").
- `decision: <what>`: Alternatives that depend on user preference or business context (e.g., which auth provider, which color scheme).
- `human-action: <what>`: A physical human action with no CLI/API equivalent (e.g., clicking an email verification link, completing 2FA, approving an OAuth browser flow).

Calibration: ~90% of tasks should be `None`, ~9% `human-verify`, ~1% `decision` or `human-action`. If more than 10% of cards have a non-None checkpoint, re-examine whether those checks can be automated.

## Execution Guardrails

Four default guardrails apply to any agent executing a task card. Most cards use these as-is (marked `Execution Guardrails: Standard`). Override with task-specific guardrails only for high-risk tasks.

1. **Analysis paralysis guard.** 5+ consecutive reads without writing code = stop and write or report blocked.
2. **Deviation classification.** Auto-fix: bugs, missing critical functionality, blocking deps. Stop and ask: architectural changes, new tables, breaking contracts. After 3 failed auto-fix attempts, stop and report.
3. **Self-check before completion.** Verify own claims before declaring done. Run the verification commands in the card.
4. **Git commit after completion.** After self-check passes, stage and commit changes with message `[TASK-NN] <title>`. Only if inside a git repo. Do not push.

Read [references/execution-guardrails.md](references/execution-guardrails.md) for full details.

## Anti-Stub Patterns

When a task creates files that are high-risk for stubbing (API routes, data-fetching components, business logic modules), include at least one substance constraint in the Artifacts field of Must-Haves. Good constraints: `min_lines: 30`, `contains: prisma.*.findMany`, `exports: [handleSubmit, validateInput]`. The constraint should match the real work, not just file existence.

Read [references/anti-stub-patterns.md](references/anti-stub-patterns.md) for the full stub indicator reference.

## Required Output Format

Use deterministic sequential task IDs everywhere they appear. The first task must be `[TASK-01]`, the second `[TASK-02]`, the third `[TASK-03]`, and so on. If numbering exceeds 99, continue as `[TASK-100]`, `[TASK-101]`, etc. Always use the literal prefix `TASK`. Never use custom or project-specific prefixes.

```md
# Decomposition Summary
- Goal: <one short paragraph>
- Mode: <standard_decomposition / brownfield_extension / gap_closure / migration_heavy>
- Likely Affected Areas: <bullet list or short line>
- Main Workstreams: <bullet list or short line>
- Highest-Risk Changes: <bullet list or short line>
- Discovered Facts:
  - Repo topology: <verified>
  - Relevant paths: <verified directories/files>
  - Data layer: <verified ORM, migrations, DB>
  - Test setup: <verified framework, config, conventions>
  - Build/CI: <verified>
  - Key conventions: <verified patterns>
  - <other verified facts as needed>
- Decision Ledger:
  - Locked Decisions: <bullet list, or "None">
  - Coding Agent's Discretion: <bullet list, or "None">
  - Deferred / Out of Scope: <bullet list, or "None">
  - Open Questions: <bullet list, or "None">
- Coverage Map:
  - Source Requirements / Behaviors:
    - <requirement or behavior> -> <[TASK-01], [TASK-02]>
  - Unmapped Items: <bullet list, or "None">
  - Overloaded Tasks: <bullet list, or "None">
- Assumptions: <bullet list, or "None">
- Execution Waves:
  - Wave 1: <[TASK-01], [TASK-02]> <brief label>
  - Wave 2 (after <[TASK-01]>): <[TASK-03], [TASK-04]> <brief label>
  - Wave 3 (after <[TASK-03], [TASK-04]>): <[TASK-05]> <brief label>
  - <...continue as needed>
  - Within each wave, independent tasks can run in parallel.

# Execution Protocol

These instructions are for the orchestrator — the agent that reads this file and spawns sub-agents. Individual sub-agents should read only their task card and the references listed in it.

## Setup

- Read `PLAN.md` in the project root (if it exists) for project-level context before launching any tasks.
- Confirm the repo is on the main branch with a clean working tree before starting Wave 1.
- Ensure `.worktrees/` is listed in the project's `.gitignore` (add it if missing, commit the change) before creating any worktrees.

## Per-Task Execution

Every task runs in a fresh sub-agent in its own git worktree, regardless of wave size:

1. Create a worktree and branch for the task (e.g., `git worktree add .worktrees/TASK-NN task/TASK-NN`).
2. Launch a fresh sub-agent whose working directory is the new worktree.
3. Pass the sub-agent its task card and the Standard Execution Guardrails section below as operating instructions. If `PLAN.md` exists, include it as required reading.
4. The sub-agent executes per its Execution Guardrails (Standard unless the card specifies overrides).
5. When the sub-agent finishes, note its completion status.

Within a wave, launch all independent tasks in parallel. Do not start the next wave until the current wave is fully complete.

## Wave Merge Protocol

After all tasks in a wave have completed successfully:

1. Return to the main working tree on the main branch.
2. For each completed task branch in the wave, merge it one at a time: `git merge --squash task/TASK-NN`
3. If a merge conflict occurs, resolve it by keeping additions from both sides.
4. After all branches in the wave are merged, run the build verification command: `<build verification command from Discovered Facts, or "None (no build step detected)">`
5. If the build fails, fix compilation errors before proceeding.
6. Commit the merged wave with a descriptive message: `git commit -m "[Wave N] <brief summary of what this wave accomplished>"`
7. Clean up worktrees and branches: `git worktree remove .worktrees/TASK-NN && git branch -d task/TASK-NN`

## Failure Handling

- If a sub-agent reports failure on a task, do not merge that task's branch. Remove its worktree, log the failure, and continue with the remaining tasks in the wave. Report the failure to the user after the wave completes.
- If the build fails after merging, identify which task caused the failure and resolve before proceeding.
- Do not proceed to the next wave if any required task in the current wave failed.

## Completion

After all waves are merged and the final build passes, the deliverable is complete. Do not push to a remote unless explicitly instructed.

## Standard Execution Guardrails

These guardrails apply to any task card marked `Execution Guardrails: Standard`. The orchestrator must include this section in each sub-agent's operating instructions.

1. **Analysis paralysis guard.** 5+ consecutive read/search operations without writing code = stop and either write code or report blocked.
2. **Deviation classification.** Auto-fix: bugs, missing critical functionality, blocking deps. Stop and ask: architectural changes, new tables, breaking contracts. After 3 failed auto-fix attempts, stop and report.
3. **Self-check before completion.** Verify own claims: do files exist? do they contain expected content? Run the verification commands in the card before declaring done.
4. **Git commit.** Stage only files created or modified by this task. Commit with message `[TASK-NN] <title>`. Do not push.

# Task Cards

## [TASK-01] <Short action-oriented title>
- Goal: <what this task accomplishes>
- Addresses: <requirement IDs, observable behaviors, or failed truths>
- Context Anchor:
  - Why: <one sentence linking this task to the current wave and the larger delivery goal>
  - Required Reading: <verified file paths or DISCOVERY-NN.md to read before starting, or "None">
  - Predecessor Outputs: <specific files/artifacts from prior wave this task needs, or "None">
  - Patterns to Follow: <existing repo files that exemplify the pattern this task should match, or "None">
- In Scope: <verified file paths, directories, modules, services, or estimated areas labeled '(estimated)'>
- Non-Goals: <what is explicitly out of scope>
- Dependencies: <[TASK-01], [TASK-02], ... or "None">
- Parallelizable: <"Yes", "No", or "Yes with ...">
- Can Land Independently: <"Yes", "No", or "Behind flag">
- Change Type: <discovery / scaffolding / schema / backend / frontend / infra / tests / docs / rollout / cleanup>
- Risk Level: <low / medium / high>
- Locked Decisions: <only user-binding decisions that constrain this card, or "None">
- Compatibility:
  - API: <additive backward-compatible / behavior change but contract stable / breaking / none / unknown>
  - Data: <none / additive schema / dual-read-write / backfill / cutover / destructive later / unknown>
  - Rollout: <independent / feature-flagged / staged / coordinated / unknown>
- Must-Haves:
  - Truths:
    - <observable behavior or invariant>
  - Artifacts:
    - <file, module, endpoint, job, migration, config, dashboard with substance constraints>
  - Key Links:
    - <critical connection that must be wired>
- Acceptance Criteria:
  - <criterion 1>
  - <criterion 2>
- Verification Prerequisites:
  - <None, or known tools / services / env / setup needed to run verification commands>
- Verification Commands:
  - `<command 1>`
  - `<command 2>`
- Constraints:
  - <behavior, API, dependency, performance, security, reliability, or rollout constraints>
- Change Safety: <additive / reversible / feature-flagged / coordinated / cleanup-later / unknown>
- Rollout / Rollback Notes:
  - <only when materially relevant; otherwise "None">
- Failure Signals:
  - <signs this card is underspecified, likely to regress, or likely to fail in a fresh thread; otherwise "None">
- Checkpoint: <"None" | "human-verify: <what>" | "decision: <what>" | "human-action: <what>">
- Execution Guardrails: <"Standard" or task-specific overrides>
- Notes / Risks:
  - <only if materially useful; otherwise "None">
- Scope Check: <fits one focused PR; if not, split further>

# Follow-up Tasks
- Only include tasks that are explicitly optional, post-launch, cleanup, or nice-to-have.
- If none, write "None".
```

## Output Delivery

Write the full output (Decomposition Summary + Execution Protocol + Task Cards + Follow-up Tasks) to a `TASKS.md` file in the project root directory. Do not only print the output in the conversation — it must be persisted to this file so that fresh sub-agents can read it.

If a `TASKS.md` file already exists, confirm with the user before overwriting.

## Final Quality Bar

Before finalizing, check that:

- repo grounding was completed and `Discovered Facts` use tool-verified evidence
- the `Decision Ledger` is present and enforced
- the `Coverage Map` shows every committed requirement or behavior mapped to task IDs
- every card names what it addresses via the `Addresses` field
- every implementation card includes must-haves (Truths, Artifacts, Key Links)
- discovery cards exist only when truly necessary, with stated justification
- dependencies are forward-only and parallel work is called out
- risky changes include rollout or rollback thinking
- verification commands and prerequisites are realistic
- the checker loop was applied and weak cards were revised
- every card is atomic, single-purpose, and understandable in a fresh thread
- the context anchor explains why the task exists in the sequence
- no exact filenames, commands, or prerequisites were invented without evidence
- contract changes are classified
- all task IDs follow the sequential `[TASK-NN]` format starting at `[TASK-01]`
- every Truths entry describes observable behavior, not implementation steps
- every task that creates files has at least one Artifacts entry with a substance constraint
- key links are specified for tasks where 2+ artifacts must be connected
- tasks with visual/UX output or external services have appropriate checkpoint types (not "None")
- locked decisions are captured in the Decomposition Summary and propagated to relevant card Constraints
- Context Anchor includes Required Reading for tasks that depend on existing code or prior wave outputs
- `Change Safety` and `Failure Signals` are present on every card
- the Execution Protocol section is present with a valid build verification command (or explicit "None")
- no task Goal or In Scope uses scope-reducing language ("initial structure", "not full flow", "stub", "shell", "placeholder", "skeleton") for a source-required behavior
- every Follow-up Task traces to an explicitly optional, post-launch, or cleanup item — no source-required behavior is deferred there
- the result is tighter and stronger, not just longer

Return the result as ready-to-execute task cards. Do not write implementation code.
