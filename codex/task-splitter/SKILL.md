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
3. For greenfield projects with layered architectures, initial waves that establish foundational
   layers (domain, interfaces, persistence) may be horizontal. After the foundational layers
   exist, transition to vertical slices for feature-area delivery. The test for "foundational
   vs. feature" is: can this layer be sliced by feature area without creating duplicate
   infrastructure? If not, it's foundational and horizontal is justified.
4. Split on risk boundaries, contract boundaries, migration boundaries, and rollout boundaries.
5. Keep each card single-purpose.
6. Prefer additive and reversible sequencing.
7. Surface dependencies and parallel work clearly.
8. Separate required work from follow-up work.
9. Prevent file collisions in parallel tasks. If two parallel tasks would both create the same file, refactor the decomposition — either assign the file to one task and list it as a predecessor output for the other, or extract it into a predecessor task. Flagging a known collision in wave notes is not sufficient.

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
   - For greenfield projects where no code exists yet, paths from the input document (project names, endpoint routes) are treated as verified. Internal paths that follow established framework conventions (e.g., Controllers/, Entities/, Repositories/) should be labeled `(convention-based)` in Discovered Facts. Artifact paths for files that will be created are acceptable when they follow stated conventions.
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
   - Verification should include both structural checks (build, file existence) and at least one behavioral check when feasible (test execution, endpoint response, output validation). For test tasks, verification commands must always include running the test suite (e.g., `dotnet test`, `pytest`, `npm test`). For non-test implementation tasks, if Artifacts include substance constraints (`contains`, `min_lines`, `exports`), verification commands should include at least one `grep` or content check that validates a substance constraint — not just build and file existence. Prefer stack-native commands (e.g., `dotnet test`, `pytest`, `npm test`) over raw shell utilities when both can verify the same thing.
   - Verification commands must be executable when the task completes (i.e., after the task's wave, given all predecessor waves). Do not include commands that depend on future-wave outputs. If a cross-wave check is needed, add it to the later wave's task or the Wave Merge Protocol.

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

14. **Co-locate tests with their implementation wave.**
    - Test cards should appear in the same wave as or the wave immediately after the code they test.
    - Deferring all tests to a final wave is a defect — it delays bug discovery across every prior wave.
    - Exception: integration tests and architecture tests that span multiple implementation cards may appear in a later wave, but unit tests for a specific layer should follow that layer immediately.

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

Splitting heuristics (these are guidelines, not hard rules — justify exceptions if needed):

- A card that creates more than ~12 new files or modifies more than ~8 existing files is likely L. Consider splitting by sub-domain, sub-feature, or artifact type.
- A card that implements more than ~5 independent use cases, handlers, or controllers is likely L. Split by functional area.
- A card that combines fundamentally different work types (e.g., REST endpoints + WebSocket orchestration, or DbContext + repositories + migration) should be split along the type boundary.
- Formulaic CRUD across many resource types (e.g., 6 admin resources × 4 operations = 24 use cases) exceeds the ~5 use case guideline even though each operation is simple. Split by resource sub-group (e.g., 2-3 related resources per task) rather than by operation type. This maintains vertical cohesion while keeping tasks reviewable. Do not treat "it's all CRUD" as an exception to the sizing guideline — count the total operations.
- Persistence-layer tasks that create one configuration/mapping file per entity plus one repository per interface follow the same counting logic. For example, 11 entity configurations + 10 repository implementations + 1 DbContext + migration files = ~23 files, which exceeds the ~12 guideline. Split by entity domain (e.g., restaurant-config persistence vs. reservation persistence) or by artifact type (configurations in one task, repositories in another).

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

Patterns to Follow and Predecessor Outputs must reference only:
- Files that exist in the repo before the current wave starts
- Outputs from explicitly completed predecessor waves
- Never same-wave task outputs (parallel tasks can't read each other's code)

If a same-wave task's output would be a useful pattern reference, inline the pattern in the card as a Shared Pattern block instead. Conditional references ("if merged — otherwise...") are fragile and should not appear.

If parallel tasks need to follow the same pattern, reference a shared existing file or describe the pattern inline in each card. When 3+ parallel tasks implement the same artifact type (e.g., controllers, repositories, service handlers), extract the shared pattern into a labeled block (e.g., '## Shared Controller Pattern') and copy it verbatim into each card. The checker should verify textual consistency across parallel pattern blocks.

### Execution Model: One Task, One Fresh Session

Every task card must be executed in its own fresh coding-agent session. This is not optional — it applies to all tasks, including the first one. The planning agent that runs this skill must not execute any task cards itself.

- Each task file (`TASKS/TASK-NN.md`) is self-contained. The user starts a fresh coding-agent session per task and points the agent at the task file as its operating instructions.
- Do not carry conversation context from the planning session into task execution.
- Do not batch multiple task cards into a single agent session.
- Respect wave ordering: only start a wave's tasks after all predecessor waves have completed.

## Checkpoint Types

Assign checkpoint types to tasks that need human interaction.

- `None` (default): Task can be fully verified with automated commands. Most tasks use this.
- `human-verify: <what>`: Visual output, UX, or behavior that cannot be tested programmatically (e.g., layout correctness, animation smoothness, "does the flow feel right?").
- `decision: <what>`: Alternatives that depend on user preference or business context (e.g., which auth provider, which color scheme).
- `human-action: <what>`: A physical human action with no CLI/API equivalent (e.g., clicking an email verification link, completing 2FA, approving an OAuth browser flow).

Calibration: ~90% of tasks should be `None`, ~9% `human-verify`, ~1% `decision` or `human-action`. If more than 10% of cards have a non-None checkpoint, re-examine whether those checks can be automated.

## Execution Guardrails

Three default guardrails are embedded in each task file. Override with task-specific guardrails only for high-risk tasks.

1. **Analysis paralysis guard.** 5+ consecutive read/search operations without writing code = stop and either write code or report blocked.
2. **Deviation classification.** Auto-fix: bugs, missing critical functionality, blocking deps. Stop and ask: architectural changes, new tables, breaking contracts. After 3 failed auto-fix attempts, stop and report.
3. **Self-check before completion.** Verify own claims before declaring done. Run the verification commands in the card.

Git commit is Step 5 of the Execution Protocol in each task file, not a guardrail.

## Anti-Stub Patterns

When a task creates files that are high-risk for stubbing (API routes, data-fetching components, business logic modules), include at least one substance constraint in the Artifacts field of Must-Haves. Good constraints: `min_lines: 30`, `contains: prisma.*.findMany`, `exports: [handleSubmit, validateInput]`. The constraint should match the real work, not just file existence.

Read [references/anti-stub-patterns.md](references/anti-stub-patterns.md) for the full stub indicator reference.

## Required Output Format

Use deterministic sequential task IDs everywhere they appear. The first task must be `[TASK-01]`, the second `[TASK-02]`, the third `[TASK-03]`, and so on. If numbering exceeds 99, continue as `[TASK-100]`, `[TASK-101]`, etc. Always use the literal prefix `TASK`. Never use custom or project-specific prefixes.

Output is written as a `TASKS/` directory containing two types of files:

### MANIFEST.md template

```md
# Task Manifest

## Decomposition Summary
- Goal: <one short paragraph>
- Mode: <standard_decomposition / brownfield_extension / gap_closure / migration_heavy>
- Likely Affected Areas: <bullet list>
- Main Workstreams: <bullet list>
- Highest-Risk Changes: <bullet list>

## Discovered Facts
  - Repo topology: <verified>
  - Relevant paths: <verified>
  - Data layer: <verified>
  - Test setup: <verified>
  - Build/CI: <verified>
  - Key conventions: <verified>
  - <other verified facts as needed>

## Decision Ledger
  - Locked Decisions: <bullet list, or "None">
  - Coding Agent's Discretion: <bullet list, or "None">
  - Deferred / Out of Scope: <bullet list, or "None">
  - Open Questions: <bullet list, or "None">

## Coverage Map
  - Source Requirements / Behaviors:
    - <requirement or behavior> -> **[TASK-NN]**, [TASK-01], [TASK-02] (bold = primary deliverer)
  - Unmapped Items: <bullet list, or "None">
  - Overloaded Tasks: <bullet list, or "None">

## Assumptions
<bullet list, or "None">

## Execution Waves
  - Wave 1: [TASK-01], [TASK-02] <brief label>
  - Wave 2 (after [TASK-01]): [TASK-03] <brief label>
  - ...
  - Within each wave, independent tasks can run in parallel.

## Checker Summary
- Dimensions checked: <list all checked dimensions>
- Revisions made: <brief notes on what was changed after checking, or "None — first pass clean">

## Per-Task Checklist

For each task, follow wave ordering:

1. Ensure main branch is clean.
2. Create branch: `git checkout -b task/TASK-NN`
   (For parallel wave tasks, use worktrees: `git worktree add .worktrees/TASK-NN task/TASK-NN`)
3. Start a fresh coding-agent session.
4. Point the agent at `TASKS/TASK-NN.md` as its operating instructions.
   Also provide `PLAN.md` if it exists in the project root.
5. The agent executes autonomously (analyze → implement → verify → commit).
6. End session (`/clear`).
7. (Optional) Start a new session to run code-reviewer on the modified files.
8. Repeat for remaining tasks in the current wave.

## Wave Merge Protocol

After all tasks in a wave complete:

1. Switch to main branch.
2. For each task branch: `git merge --squash task/TASK-NN`
3. Resolve merge conflicts by preserving additions from both sides. Deduplicate any overlapping boilerplate (imports, dependency registrations, route declarations) and verify the merged result is syntactically valid.
4. Run build verification: `<build command from Discovered Facts, or "None">`
5. If the build fails, fix before proceeding.
6. Run test verification: `<test command from Discovered Facts, or "None — no test tasks completed yet">`
7. If tests fail, fix before proceeding — a merge-induced regression caught now prevents cascading failures in later waves.
8. Commit: `[Wave N] <brief summary>`
9. Clean up branches: `git branch -d task/TASK-NN`

## Failure Handling

- If a task fails, do not merge its branch. Log the failure and continue with remaining wave tasks.
- Do not start the next wave if a required task in the current wave failed.

## Follow-up Tasks
- <only explicitly optional, post-launch, cleanup, or nice-to-have items>
- If none, "None".
```

### TASK-NN.md template

```md
# [TASK-NN] <Short action-oriented title>

## Execution Protocol

1. **Setup**: Create branch `git checkout -b task/TASK-NN` (if not already on a task branch).
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
- Addresses: <requirement IDs, observable behaviors, or failed truths>
- Context Anchor:
  - Why: <one sentence linking this task to the delivery goal>
  - Required Reading: <verified file paths or DISCOVERY-NN.md, or "None">
  - Predecessor Outputs: <specific files/artifacts from prior tasks, or "None">
  - Patterns to Follow: <existing repo files to match, or "None">
- In Scope: <verified file paths, directories, or estimated areas labeled '(estimated)'>
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
- Checkpoint: <"None" | "human-verify: <what>" | "decision: <what>" | "human-action: <what>">
- Notes / Risks:
  - <only if materially useful; otherwise "None">
- Scope Check: <fits one focused PR; if not, split further>
```

## Output Delivery

Write the output as a `TASKS/` directory in the project root containing `MANIFEST.md` and individual `TASK-NN.md` files. Do not only print the output in the conversation — it must be persisted to these files so that fresh coding-agent sessions can read them.

If a `TASKS/` directory already exists, confirm with the user before overwriting.

## Final Quality Bar

Before finalizing, check that:

- repo grounding was completed and `Discovered Facts` use tool-verified evidence
- the `Decision Ledger` is present and enforced
- the `Coverage Map` shows every committed requirement or behavior mapped to task IDs, with the primary deliverer bolded when multiple tasks contribute
- every card names what it addresses via the `Addresses` field
- every implementation card includes must-haves (Truths, Artifacts, Key Links)
- discovery cards exist only when truly necessary, with stated justification
- dependencies are forward-only and parallel work is called out
- risky changes include rollout or rollback thinking
- verification commands and prerequisites are realistic and temporally valid — each command is executable after the task's wave completes, not dependent on future-wave outputs
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
- MANIFEST.md includes Per-Task Checklist and Wave Merge Protocol with build verification command and test verification command (or explicit "None" for each)
- MANIFEST.md includes a Checker Summary section between Execution Waves and Per-Task Checklist
- each TASK-NN.md includes Execution Protocol and Guardrails sections before the Task Card
- no task Goal or In Scope uses scope-reducing language ("initial structure", "not full flow", "stub", "shell", "placeholder", "skeleton") for a source-required behavior
- every Follow-up Task traces to an explicitly optional, post-launch, or cleanup item — no source-required behavior is deferred there
- test cards for a layer appear in the same wave or the immediately following wave — not deferred to a final testing wave
- parallel tasks in the same wave do not reference each other's outputs in Patterns to Follow or Predecessor Outputs
- parallel waves with 3+ tasks touching the same directory acknowledge merge risk in the wave description
- greenfield projects label convention-based paths accordingly in Discovered Facts
- every implementation task's primary deliverables are covered by at least one test task — no production artifact is left untested across the entire plan
- when multiple tasks implement the same kind of artifact, they follow the same architectural pattern unless the decision ledger explicitly justifies divergence
- every type, interface, or artifact referenced in a task's Must-Haves, In Scope, or Constraints is traceable to the task itself, a predecessor task's artifacts, or an existing repo file
- the result is tighter and stronger, not just longer

Return the result as ready-to-execute task files. Do not write implementation code.
