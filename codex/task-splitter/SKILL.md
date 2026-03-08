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

5. Do not invent requirements or repo facts.
   - Capture assumptions and open questions explicitly when the input is unclear.
   - Do not invent exact files, modules, tables, endpoints, commands, or environment prerequisites unless the plan or repo evidence provides them.
   - If exact repo details are unknown, name the subsystem and file areas as estimated.

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

## Repo-Aware Checks

Before writing tasks, briefly identify:
- product goal
- repo topology
- likely affected layers or subsystems
- main workstreams
- main risk classes
- release boundaries and rollout strategy
- data model and migration impact
- affected contracts
- test surface impact
- observability implications
- areas that need discovery before implementation

## Discovery Rules

If repo details are missing or uncertain, create discovery cards first.

Discovery cards must:
- stay tightly scoped
- produce concrete outputs
- unblock specific follow-on cards

Concrete outputs can include:
- files, modules, or services involved
- confirmed constraints or conventions
- identified test suites or migration framework
- recommended next task cards

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

```md
# Decomposition Summary
- Goal: <one short paragraph>
- Likely Affected Areas: <bullet list or short line>
- Main Workstreams: <bullet list or short line>
- Highest-Risk Changes: <bullet list or short line>
- Assumptions: <bullet list, or "None">
- Open Questions: <bullet list, or "None">
- Execution Waves:
  - Wave 1: <task IDs> <brief label, e.g. "discovery and scaffolding">
  - Wave 2 (after <blocking task IDs>): <task IDs> <brief label>
  - Wave 3 (after <blocking task IDs>): <task IDs> <brief label>
  - <...continue as needed>
  - Within each wave, independent tasks can run in parallel.

# Task Cards

For each task, use exactly this template:

## [TASK-ID] <Short action-oriented title>
- Goal: <what this task accomplishes>
- Context Anchor: <one sentence linking this task to the current wave and the larger delivery goal>
- In Scope: <exact files / directories / modules / services / subsystems, or estimated areas if unknown>
- Non-Goals: <what is explicitly out of scope>
- Dependencies: <task IDs or "None">
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
