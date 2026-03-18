---
name: iterative-builder
description: Iterative build workflow — plan one task at a time against actual codebase state. Each task card is designed by code-architect reading the real codebase, approved by the user, implemented in a worktree, reviewed, and merged before the next task is planned. Use when upfront planning would diverge from reality or when the sequence of work should emerge organically.
---

# Iterative Builder

Plan one task at a time against the actual codebase, build it, review it, commit it, then plan the next.

## Objective

- Produce task cards that are designed against the real codebase state, not a projected future state.
- Let the sequence of work emerge organically from completed work and remaining requirements.
- Keep the user in the loop — every task card is approved before implementation begins.
- Deliver fully reviewed, tested, committed code after each task before moving on.

## Workflow

### Phase 1: Bootstrap

Establish the goal, capture decisions, extract requirements, and create the living manifest.

#### Step 1: Ground the goal and the repo

Before designing any task:

1. Read the input carefully and identify:
   - product goal
   - scope boundaries
   - likely success criteria
   - likely risk classes
2. Explore the repo with tools to verify:
   - Repo topology: monolith, service, monorepo, package, app
   - Affected subsystems: UI, API, domain logic, persistence, jobs, infra
   - Existing patterns: read 1-2 representative examples of similar features or modules to understand conventions
   - Data layer: ORM, migration framework, schema definitions, database type
   - API layer: framework, routing conventions, middleware stack
   - Test infrastructure: test framework, config, directory layout, naming conventions
   - Build/CI: build system, CI pipeline, deploy process
   - Dependencies: package manager, key libraries, version constraints
3. Do not invent repo facts that tools can verify. You must use tools to verify these facts.
4. If the repo cannot answer something and the answer is product intent, ask the user before proceeding.
5. For greenfield projects where no code exists yet, paths from the input document are treated as verified. Internal paths that follow established framework conventions should be labeled `(convention-based)` in Discovered Facts.

#### Step 2: Build a Decision Ledger

Create a `Decision Ledger` with four buckets:

- `Locked Decisions` — binding user choices and hard constraints
- `Coding Agent's Discretion` — real implementation freedom areas
- `Deferred / Out of Scope` — explicitly excluded work
- `Open Questions` — missing user judgment that materially changes the plan

Rules:

- Ask questions only for user-judgment gaps, not repo facts.
- Treat `Locked Decisions` as binding.
- Treat `Deferred / Out of Scope` as forbidden scope.
- Use `Coding Agent's Discretion` only for real implementation freedom.
- If important questions remain unanswered, do not silently guess.

Read [references/decision-ledger.md](references/decision-ledger.md) when building or validating the ledger.

#### Step 3: Extract requirements and success criteria

From the input and grounding, extract:

- **Numbered requirements**: `REQ-01`, `REQ-02`, ... — each a distinct deliverable or behavior the goal demands.
- **Success criteria**: `SC-01`, `SC-02`, ... — observable conditions that prove the goal is met.

Requirements should be atomic enough that each maps to roughly one task, but coarse enough to avoid ceremony. If a requirement clearly needs multiple tasks, that will emerge naturally during the per-task loop.

#### Step 4: Write the manifest

Write `TASKS/MANIFEST.md` using the manifest template. At this stage the manifest contains:

- Goal summary
- Discovered Facts
- Decision Ledger
- Success Criteria
- Requirements with status (all `pending`)
- Empty sections for: Completed Tasks, Adjustments Log, Remaining Work, Follow-up Items

Read [references/manifest-template.md](references/manifest-template.md) for the manifest format.

#### Step 5: Present manifest for approval

Present the manifest to the user. The user may:

- Approve as-is
- Add, remove, or modify requirements
- Add or change locked decisions
- Clarify open questions

Incorporate feedback and update `TASKS/MANIFEST.md` before proceeding.

### Phase 2: Per-Task Loop

Repeat until all requirements are addressed.

#### Step 1: Select next requirement(s)

Choose the next pending requirement(s) from the manifest. Selection criteria:

- Prerequisites satisfied (prior tasks have landed on main)
- Logical ordering (foundational before dependent)
- User priority if expressed

A single task may address one or more related requirements. Mark selected requirements as `in-progress` in the manifest.

#### Step 2: Design the task card

Invoke `feature-dev:code-architect` with:

- The selected requirement(s) and their context
- The decision ledger
- The task card template ([references/task-card-template.md](references/task-card-template.md))
- Summaries of completed tasks (what was built, what files exist now)
- Coverage and must-haves guidance ([references/coverage-and-must-haves.md](references/coverage-and-must-haves.md))
- Any relevant anti-stub patterns ([references/anti-stub-patterns.md](references/anti-stub-patterns.md))

The code-architect reads the **current codebase** (including all previously merged work) and produces a task card. This is the key advantage over upfront planning — the card is designed against reality.

#### Step 3: Quality-check the card

Before presenting to the user, verify:

- No scope-reducing language in Goal or In Scope ("initial structure", "stub", "placeholder", "skeleton", "shell", "basic scaffold") for source-required behavior
- Substance constraints present in Artifacts for high-risk files
- Verification commands are concrete and executable
- Must-Haves include Truths (observable behavior), Artifacts (concrete deliverables), and Key Links (critical wiring)
- Truths describe behavior or invariants, not implementation steps
- Locked decisions from the ledger are honored
- The card does not implement deferred/out-of-scope items
- The card fits sizing guidelines (XS/S/M — not L or XL)

If the card fails quality checks, re-invoke code-architect with specific feedback.

#### Step 4: Present task card for approval

Present the task card to the user. The user may:

- Approve the card
- Request changes (re-invoke code-architect with feedback)
- Defer the requirement (move to Deferred, update manifest)
- Inject a new requirement (add to manifest, design card for it)

#### Step 5: Create worktree

```bash
git checkout main
git pull
git branch task/TASK-NN
git worktree add .worktrees/TASK-NN task/TASK-NN
```

#### Step 6: Implement

Launch an implementation sub-agent in the worktree with the approved task card as its operating instructions. The sub-agent follows the Execution Protocol and Guardrails embedded in the task card.

#### Step 7: Review

Invoke `feature-dev:code-reviewer` on the modified files in the worktree. If the reviewer identifies issues:

1. Fix the issues in the worktree.
2. Re-run the reviewer to confirm all issues are resolved and no new issues were introduced by the fixes.
3. Repeat for up to 5 fix cycles.
4. If issues persist after 5 cycles, escalate to the user (see Edge Cases).

**Do not skip the re-review.** A passing build is necessary but not sufficient — the reviewer checks correctness, patterns, and quality beyond compilation. Fixes can introduce new issues (e.g., changing an API call may require updating its callers). Only proceed to Step 8 when the reviewer returns a clean report or the user explicitly accepts known debt.

**The orchestrator must not self-dismiss reviewer findings.** The reviewer is an independent check — the orchestrator evaluating findings and deciding they are "non-blocking" or "false positives" defeats the purpose. If you believe findings are false positives:

1. Explain why to the user and let them decide whether to accept the findings or skip re-review.
2. Do not merge until the user explicitly approves or the reviewer returns a clean report.

The only two exit conditions from Step 7 are: (a) the reviewer returns a clean report, or (b) the user explicitly accepts known issues.

#### Step 8: Merge to main

```bash
cd /path/to/project
git merge --squash task/TASK-NN
# Run build verification
<build command from Discovered Facts>
# Run test verification
<test command from Discovered Facts>
# Commit
git commit -m "[TASK-NN] <title>"
# Clean up worktree
git worktree remove .worktrees/TASK-NN
git branch -d task/TASK-NN
```

If build or test verification fails, attempt to fix. If the fix fails, escalate to the user.

#### Step 9: Update manifest

After successful merge:

- Mark addressed requirements as `done (TASK-NN)`
- Add entry to Completed Tasks with summary of what was built
- Log any adjustments (scope changes, new discoveries, requirement modifications) in the Adjustments Log
- Update Remaining Work
- If new follow-up items were identified, add to Follow-up Items

#### Step 10: Reset context

**This step is mandatory — do not skip it.** Clear the conversation context after merging a task and before designing the next one. The manifest and task cards on disk contain all state needed to continue.

After clearing, re-read `TASKS/MANIFEST.md` to re-anchor on:
- Which requirements are done, in-progress, or pending
- What was built in completed tasks (summaries in Completed Tasks section)
- Any adjustments or follow-up items logged

If the user is driving the conversation, ask them to run `/clear` before continuing. If operating autonomously, issue the clear yourself.

#### Step 11: Continue or finish

If all requirements are `done` or `deferred`, proceed to Phase 3. Otherwise, return to Step 1.

### Phase 3: Validation

#### Step 1: Check success criteria

For each success criterion (`SC-01`, `SC-02`, ...):

- Verify it is satisfied by the completed tasks
- Note which task(s) addressed it
- Flag any gaps

#### Step 2: Handle gaps

If success criteria have gaps:

- Present gaps to the user
- User decides: create additional tasks (loop back to Phase 2), accept as-is, or defer

#### Step 3: Finalize

- Write final manifest state to `TASKS/MANIFEST.md`
- Report summary: tasks completed, requirements met, any deferred items, follow-up work

## Core Rules

1. **Verify repo facts with tools; do not guess.**
   - Use Glob, Grep, and Read before naming exact files, modules, tables, endpoints, commands, or conventions.
   - If something cannot be verified with tools (requires runtime, external service, staging environment), mark it `unverified` and explain why.
   - Do not invent exact filenames, modules, tables, endpoints, commands, or environment prerequisites unless verified by tools or explicitly provided in the input.
   - For greenfield projects where no code exists yet, paths from the input document (project names, endpoint routes) are treated as verified. Internal paths that follow established framework conventions (e.g., Controllers/, Entities/, Repositories/) should be labeled `(convention-based)` in Discovered Facts. Artifact paths for files that will be created are acceptable when they follow stated conventions.
   - Prefer asking one round of clarifying questions over producing cards built on assumptions.

2. **Preserve intent across iterations.**
   - Do not let implementation convenience override locked user decisions.
   - Do not quietly reintroduce deferred scope.
   - Do not reduce required scope when designing a task. If the requirement demands full behavior, the task card must deliver full behavior — not a structural shell, stub, skeleton, placeholder, or "initial structure."
   - Scope-reducing language ("initial structure", "not full flow", "stub", "shell", "placeholder", "skeleton", "basic scaffold") in a task's Goal or In Scope is a defect when the mapped requirement demands full behavior.
   - Carry forward stated constraints on behavior, API compatibility, latency, data model, dependency choices, rollout, security, compliance, and backward compatibility.

3. **Anchor cards to outcomes, not just work.**
   - Every requirement must be addressed by at least one task.
   - Every task must say what requirement(s) it addresses.

4. **Use goal-backward must-haves.**
   - Every task card must include `Truths`, `Artifacts`, and `Key Links`.
   - These are for execution safety, not documentation theater.

5. **Keep discovery honest.**
   - Resolve what you can during bootstrap and task design.
   - Only create discovery cards when the unknown genuinely requires runtime behavior, external systems, or broader investigation than the planning pass can support.

6. **Keep cards fresh-thread safe.**
   - A new agent should be able to execute a card without rediscovering the plan.
   - Include all necessary context in the card itself.

7. **Separate risky rollout work.**
   - Split schema preparation, compatibility, backfill, cutover, and cleanup when applicable.
   - Isolate breaking changes and coordinated rollouts.

8. **Make verification executable.**
   - Include concrete commands whenever possible.
   - If commands are inferred, label them best-effort.
   - List known prerequisites for running them.
   - Verification should include both structural checks (build, file existence) and at least one behavioral check when feasible (test execution, endpoint response, output validation).
   - For test tasks, verification commands must always include running the test suite.
   - For non-test implementation tasks, if Artifacts include substance constraints (`contains`, `min_lines`, `exports`), verification commands should include at least one content check that validates a substance constraint.
   - Prefer stack-native commands (e.g., `dotnet test`, `pytest`, `npm test`) over raw shell utilities.

9. **Make acceptance criteria observable.**
   - Criteria must describe externally visible behavior, contract guarantees, data invariants, or testable internal outcomes.
   - Avoid vague criteria like "works correctly" or "is production ready".

10. **Separate required work from optional work.**
    - Nice-to-have improvements, future cleanup, and post-launch hardening belong in Follow-up Items.
    - A behavior the requirement marks as required must not appear in Follow-up Items. If it cannot fit in the current task, create a new requirement for it.

11. **Keep the result compact.**
    - Do not add ceremony for tiny, low-risk changes.
    - For small work, keep the ledger and must-haves terse but still present.

12. **Do not write implementation code during planning.**
    - The bootstrap phase and task card design only plan and decompose work.
    - Implementation happens in the worktree during the per-task loop.

13. **Design each task against the real codebase.**
    - The code-architect must read the current state of main (including all previously merged tasks) when designing each card.
    - Never design a card against a projected future state.

14. **Reset context between tasks.**
    - Always clear the conversation context after merging a task and before designing the next one.
    - Do not rationalize skipping the reset ("still fresh", "context is small", "just one more task").
    - The manifest and task cards on disk are the source of truth — the context window is not.

## Discovery Cards

Create discovery cards only when planning cannot resolve an unknown up front.

Each discovery card must:

- state what is unknown
- state why planning could not resolve it
- produce a concrete output
- name which requirements it unblocks
- write findings to `DISCOVERY-NN.md`

Follow-on tasks that depend on a discovery card must reference the `DISCOVERY-NN.md` file in their Context Anchor.

A discovery card that could have been replaced by normal repo inspection is a defect.

## Sizing Rules

Use these buckets only as an internal check:

- `XS`: tiny, surgical, very low risk
- `S`: small, focused, standard PR
- `M`: moderate, still reviewable in one sitting

Do not create `L` or `XL` cards. Split the requirement further.

Splitting heuristics (guidelines, not hard rules — justify exceptions if needed):

- A card that creates more than ~12 new files or modifies more than ~8 existing files is likely L. Consider splitting by sub-domain, sub-feature, or artifact type.
- A card that implements more than ~5 independent use cases, handlers, or controllers is likely L. Split by functional area.
- A card that combines fundamentally different work types (e.g., REST endpoints + WebSocket orchestration, or DbContext + repositories + migration) should be split along the type boundary.
- Formulaic CRUD across many resource types (e.g., 6 admin resources x 4 operations = 24 use cases) exceeds the ~5 use case guideline even though each operation is simple. Split by resource sub-group rather than by operation type.

## Task ID Rules

- Always use the prefix `TASK-` followed by a zero-padded two-digit number starting at `01`: `[TASK-01]`, `[TASK-02]`, ..., `[TASK-99]`.
- If there are more than 99 tasks, continue with three digits: `[TASK-100]`, `[TASK-101]`, etc.
- Never use project-specific prefixes like `[PA-01]`, `[AUTH-01]`, or `[DB-01]`. The prefix is always `TASK-`.
- Number tasks sequentially in the order they are created during the per-task loop.

## Anti-Stub Patterns

When a task creates files that are high-risk for stubbing (API routes, data-fetching components, business logic modules), include at least one substance constraint in the Artifacts field of Must-Haves. Good constraints: `min_lines: 30`, `contains: prisma.*.findMany`, `exports: [handleSubmit, validateInput]`. The constraint should match the real work, not just file existence.

Read [references/anti-stub-patterns.md](references/anti-stub-patterns.md) for the full stub indicator reference.

## Edge Case Handling

### User rejects a task card

When the user rejects or requests changes to a task card:

1. Collect the user's feedback.
2. Re-invoke code-architect with the original requirements plus the user's feedback.
3. Present the revised card.
4. If the requirement itself is the issue, offer to defer it (move to Deferred in the ledger, update manifest).

### User injects a new requirement

When the user wants to add work mid-build:

1. Add the new requirement to the manifest with a new `REQ-NN` identifier.
2. Log the addition in the Adjustments Log with rationale.
3. Set status to `pending`.
4. Design a task card for it when it becomes the next logical piece of work.

### Requirements change mid-build

When existing requirements change:

1. Update the requirement text in the manifest.
2. Log the change in the Adjustments Log with before/after and rationale.
3. If a completed task partially addressed the old requirement, note what delta remains.
4. Create new requirements for the delta work if needed.

### Code-reviewer unfixable issues (5 cycles)

When the code reviewer identifies issues that cannot be fixed after 5 cycles:

Present to the user with three options:

1. **Proceed with known debt** — document the issues in the manifest Adjustments Log and continue.
2. **Abandon the task** — remove the worktree, revert the requirement to `pending`, and redesign.
3. **Pause for manual fix** — the user fixes the issues manually, then resume the workflow.

### Implementation failure

When the implementation sub-agent fails to complete the task:

1. Collect the failure context (what was attempted, what failed, error messages).
2. Re-invoke code-architect with the failure context to redesign the approach.
3. If the requirement is too large, split it into smaller requirements.
4. If the requirement is blocked by an external factor, defer it and log the blocker.

### Build or test failure on merge

When the merge to main fails build or test verification:

1. Attempt to fix the failure in the worktree.
2. Re-run verification.
3. If the fix fails, present to the user with the failure details and options:
   - Fix manually and continue
   - Abandon the task and redesign
   - Proceed with the failure acknowledged (only for non-critical test failures)

## Output Delivery

Write the output as a `TASKS/` directory in the project root containing:

- `MANIFEST.md` — the living manifest (updated after every task)
- `TASK-NN.md` — individual task cards (created as each task is designed)

If a `TASKS/` directory already exists, confirm with the user before overwriting.

The manifest is a living document. It starts with requirements only (Phase 1) and grows as tasks are designed, implemented, and completed (Phase 2). By the end, it provides a complete record of what was built, what changed, and what remains.

Read [references/manifest-template.md](references/manifest-template.md) for the manifest format.
Read [references/task-card-template.md](references/task-card-template.md) for the task card format.

## Final Quality Bar

### Per-task checks (before presenting each card)

- repo grounding was completed and Discovered Facts use tool-verified evidence
- the card names what requirement(s) it addresses via the `Addresses` field
- must-haves include Truths (observable behavior), Artifacts (concrete deliverables), Key Links (critical wiring)
- Truths describe observable behavior or invariants, not implementation steps
- no scope-reducing language in Goal or In Scope for source-required behavior
- substance constraints present in Artifacts for files that are high-risk for stubbing
- verification commands and prerequisites are realistic and executable after the task completes
- the card is single-purpose and fits sizing guidelines
- locked decisions from the ledger are honored
- no deferred/out-of-scope items are implemented
- the context anchor explains why this task exists and what to read first
- no exact filenames, commands, or prerequisites were invented without evidence
- Change Safety and Failure Signals are present
- every type, interface, or artifact referenced is traceable to the task itself, a predecessor task, or an existing repo file
- the card is understandable in a fresh thread without access to this conversation

### Per-manifest checks (after each update)

- every requirement has a status (`pending`, `in-progress (TASK-NN)`, or `done (TASK-NN)`)
- completed tasks have summaries of what was built
- adjustments are logged with rationale
- remaining work accurately reflects what is left
- follow-up items contain only explicitly optional or post-launch work
- success criteria are tracked against completed tasks
- the manifest provides a complete record usable by someone joining the project

Return the result as ready-to-execute task files. Do not write implementation code during the planning and card design phases.
