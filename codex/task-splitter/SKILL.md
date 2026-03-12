---
name: task-splitter
description: Decompose large PRDs, RFCs, epics, roadmap items, migration plans, bugfix plans, and implementation plans into atomic task cards that Codex or another coding agent can execute safely and independently. Use when the user wants to break down a large change, turn a spec into actionable tasks, split work into tickets or cards, plan implementation steps, or make a pasted plan executable.
---

# Task Splitter

Break a large delivery plan into the smallest complete set of task cards that a coding agent can execute safely in fresh threads.

## Objective

- Produce the fewest cards that fully deliver the plan.
- Keep each card independently understandable, reviewable, testable, and as close as possible to independently shippable.
- Optimize for execution, not discussion.

## Card Standard

Every task card must have:
- one primary objective
- one clear risk boundary
- explicit scope
- explicit non-goals
- observable acceptance criteria
- concrete verification steps
- explicit behavior, API, dependency, rollout, and compatibility constraints when relevant

## Decomposition Rules

1. Decompose for execution.
   - Prefer the smallest complete set of tasks.
   - Do not create artificial tasks just to increase count.
   - Do not keep tasks large just to reduce count.

2. Prefer vertical slices over layer buckets.
   - If a thin end-to-end slice can ship safely, prefer that over separate backend/frontend/test tasks.
   - If the change crosses risky boundaries, split it.

3. Split on risk boundaries.
   - Separate schema migration from backfill.
   - Separate backfill from read-path switch.
   - Separate read-path switch from write-path switch when needed.
   - Separate cleanup from first-pass rollout.
   - Separate API contract changes from client adoption when they can ship independently.
   - Separate refactors from behavior changes unless the refactor is strictly required.
   - Separate discovery from implementation when repo facts are unknown.

4. Keep each card single-purpose.
   - Every card should have one primary reason to exist.
   - If the title naturally needs "and", split it unless the work is inseparable.

5. Verify repo facts with tools; do not guess.
   - Use glob_file_search, rg, and read_file to verify repo structure, conventions, file locations, test frameworks, migration patterns, and build systems before writing cards.
   - If the PRD is unclear, surface uncertainties as questions to the user before proceeding.
   - Do not silently turn guesses into scope.
   - Do not invent exact files, modules, tables, endpoints, commands, or environment prerequisites unless verified by tools or explicitly provided in the input.
   - If a fact cannot be verified with tools (requires runtime, external service, staging environment), label it as "unverified" and explain why.
   - Prefer asking one round of clarifying questions over producing cards built on assumptions.

6. Make acceptance criteria observable.
   - Describe externally visible behavior, contract guarantees, data invariants, or other testable outcomes.
   - Avoid vague criteria such as "works correctly" or "production ready".

7. Make verification executable.
   - Include concrete commands whenever possible.
   - Prefer repo-realistic commands if known.
   - If commands are inferred, mark them as best-effort.
   - List required tools, services, env vars, fixtures, or local infrastructure when they are known prerequisites.

8. Preserve stated constraints.
   - Carry forward requirements on behavior, latency, compatibility, dependency choices, rollout, security, compliance, and reliability.
   - If a constraint changes task boundaries, split accordingly.

9. Surface dependencies clearly.
   - Identify blockers.
   - Mark cards that can run in parallel.
   - Sequence the safest landing order.

10. Prefer additive and reversible sequencing.
    - Favor changes that can land safely before all consumers switch.
    - Prefer feature flags, compatibility shims, tolerant readers, and staged rollout when applicable.
    - Include rollback or mitigation notes for risky changes.

11. Separate required work from optional work.
    - Put cleanup, hardening, and nice-to-have improvements in `Follow-up Tasks`.

12. Do not write implementation code.
    - This skill only decomposes work.

## Fresh-Thread Handoff

Write every task card so it can be handed to a fresh coding-agent thread with no hidden context. The card must stand on its own.

Because upfront discovery verifies repo facts before cards are written, task cards contain concrete verified file paths and conventions — not placeholders or estimates. The implementing agent should not need to re-discover what planning already verified.

When a task card depends on a discovery card, it must reference the corresponding `DISCOVERY-NN.md` file as required reading. The implementing agent should read that file before starting work to obtain the context the planning session could not resolve upfront.

## Upfront Discovery Phase

Before writing any task cards, complete all four steps below. This phase is mandatory.

**Step 1: Understand the goal.** Read the PRD/plan carefully. Identify the product goal, scope boundaries, and success criteria.

**Step 2: Explore the repo with tools.** Use glob_file_search, rg, and read_file to verify:
- Repo topology: monolith, service, monorepo, package, app
- Affected subsystems: UI, API, domain logic, persistence, jobs, infra
- Existing patterns: read 1–2 representative examples of similar features or modules to understand conventions
- Data layer: ORM, migration framework, schema definitions, database type
- API layer: framework, routing conventions, middleware stack
- Test infrastructure: test framework, config, directory layout, naming conventions
- Build/CI: build system, CI pipeline, deploy process
- Dependencies: package manager, key libraries, version constraints

You must use tools to verify these facts. Do not rely on memory or assumptions.

**Step 3: Surface uncertainties as questions.** Identify anything the PRD leaves ambiguous, available evidence cannot resolve, or requires user judgment. Present these as numbered questions and wait for answers before proceeding. Do not guess.

**Step 4: Summarize discovered facts.** Produce a "Discovered Facts" block in the Decomposition Summary output (see Required Output Format below).

## Discovery Rules

### Default: upfront tool-based discovery

By the time you write cards, most repo facts should be verified through the upfront discovery phase. Cards should contain concrete paths and conventions from verified discovery, not placeholders.

### Fallback: discovery cards with DISCOVERY-NN.md output

Only create discovery cards when planning genuinely cannot resolve an unknown upfront. Valid reasons include:
- Requires running the application to observe runtime behavior
- Requires calling an external service or API
- Requires performance measurement under realistic load
- Requires staging or production environment access
- Codebase too large for a single planning pass to audit a subsystem

Each discovery card must:
- State what is unknown and why upfront discovery could not resolve it
- Define the concrete output format
- Name blocked follow-on cards and explain what decision depends on the result
- **Write its outputs to a `DISCOVERY-NN.md` file** (e.g., `DISCOVERY-01.md`, `DISCOVERY-02.md`) at the repo root or a designated workspace location. The file must contain the discovery findings in a structured format that a fresh agent can consume.
- Follow-on cards that depend on a discovery card must **reference the `DISCOVERY-NN.md` file as required reading** in their Context Anchor or In Scope section (e.g., "Read `DISCOVERY-01.md` for the confirmed list of tables with tenant_id columns before proceeding.")

This closes the handoff gap: even though the follow-on agent runs in a fresh thread, it knows exactly where to find the discovery outputs.

A discovery card that could have been resolved by reading files during planning is a defect.

Do not let discovery swallow implementation.

## Migration and Data Rules

For schema or data changes, classify the work as one of:
- none
- additive schema
- schema change requiring dual-read or dual-write
- backfill required
- read or write cutover
- destructive cleanup later
- unknown; requires repo or data-model review

Usually split migration work into:
1. schema preparation or additive change
2. application write compatibility
3. backfill or migration runner
4. application read cutover
5. cleanup or legacy removal

For migration-related cards, explicitly include:
- idempotency expectations
- rollout order
- rollback considerations
- data validation checks
- performance or load concerns
- whether the change is reversible

Never assume destructive migrations are safe in the same card as feature work.

## API and Contract Rules

For endpoints, events, queue payloads, RPC, SDKs, shared types, or public interfaces, classify the change as:
- additive backward-compatible
- behavior change but contract stable
- breaking change
- unknown

Prefer additive fields, tolerant readers, compatibility shims, and staged producer/consumer rollout when possible. If the plan implies a breaking change, isolate it into its own sequence and call out the migration strategy.

## Test Rules

Every implementation card must specify:
- which existing test suites should be updated
- whether new tests are required
- whether CI coverage is sufficient
- any manual verification required because automation is insufficient

If test work is broad and cross-cutting, create a dedicated test sweep card instead of hiding it inside implementation cards.

## Rollout and Observability Rules

For user-facing, operationally risky, or production-sensitive changes, explicitly consider:
- logging
- metrics
- alerts
- tracing
- dashboards
- feature flags
- staged rollout
- kill switch or rollback path

Do not assume observability is optional for risky changes.

## Sizing Rules

Use these buckets only as an internal decomposition check:
- XS: tiny, surgical, very low risk
- S: small, focused, standard PR
- M: moderate, still reviewable in one sitting

Do not create L or XL cards. Split them further.

## Required Output Format

Use deterministic sequential task IDs everywhere they appear. The first task must be `[TASK-01]`, the second `[TASK-02]`, the third `[TASK-03]`, and so on. If numbering exceeds 99, continue as `[TASK-100]`, `[TASK-101]`, etc. Always use the literal prefix `TASK`. Never use custom or project-specific prefixes such as `PA`, `AUTH`, `API`, `DB`, or similar. Reuse these exact bracketed IDs consistently in task card headers, dependencies, execution waves, and any examples.

```md
# Decomposition Summary
- Goal: <one short paragraph>
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
- Assumptions: <bullet list, or "None">
- Open Questions: <bullet list, or "None">
- Execution Waves:
  - Wave 1: <[TASK-01], [TASK-02]> <brief label, e.g. "discovery and scaffolding">
  - Wave 2 (after <[TASK-01]>): <[TASK-03], [TASK-04]> <brief label>
  - Wave 3 (after <[TASK-03], [TASK-04]>): <[TASK-05]> <brief label>
  - <...continue as needed>
  - Within each wave, independent tasks can run in parallel.

# Task Cards

For each task, use exactly this template. Number task cards sequentially with `[TASK-01]`, `[TASK-02]`, `[TASK-03]`, ... in the order they appear. Never substitute another prefix.

## [TASK-01] <Short action-oriented title>
- Goal: <what this task accomplishes>
- Context Anchor: <one sentence linking this task to the current wave and the larger delivery goal>
- In Scope: <verified file paths, directories, modules from discovery phase; label any remaining estimates as '(estimated)'>
- Non-Goals: <what is explicitly out of scope>
- Dependencies: <[TASK-01], [TASK-02], ... or "None">
- Parallelizable: <"Yes", "No", or "Yes with ...">
- Can Land Independently: <"Yes", "No", or "Behind flag">
- Change Type: <discovery / scaffolding / schema / backend / frontend / infra / tests / docs / rollout / cleanup>
- Risk Level: <low / medium / high>
- Compatibility:
  - API: <additive backward-compatible / behavior change but contract stable / breaking / none / unknown>
  - Data: <none / additive schema / dual-read-write / backfill / cutover / destructive later / unknown>
  - Rollout: <independent / feature-flagged / staged / coordinated / unknown>
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
- Rollout / Rollback Notes:
  - <only when materially relevant; otherwise "None">
- Notes/Risks:
  - <only if materially useful; otherwise "None">
- Scope Check: <fits one focused PR; if not, split further>

# Follow-up Tasks
- Only include tasks that are explicitly optional, post-launch, cleanup, or nice-to-have.
- If none, write "None".
```

## Quality Bar

Before finalizing, check that:
- upfront discovery phase was completed and Discovered Facts populated with tool-verified evidence
- every card uses verified file paths and conventions, not guesses
- remaining assumptions are explicitly labeled and minimal
- open questions were surfaced to the user before cards were written
- discovery cards are only present when upfront discovery genuinely could not resolve the unknown, with stated justification
- every card is atomic and single-purpose
- every card is understandable in a fresh coding-agent thread
- the context anchor explains why the task exists in the sequence
- dependencies are forward-only and parallel work is called out
- no exact filenames, commands, or prerequisites were invented without evidence
- discovery work is separated and produces concrete outputs
- migrations are safely isolated when needed
- contract changes are classified
- risky changes include rollout or rollback notes
- verification prerequisites are listed when needed
- each card fits one focused PR; if not, split it further
- cards are split by deployability and risk, not arbitrary architecture buckets

Return the result as ready-to-execute task cards. Do not re-plan the plan.
