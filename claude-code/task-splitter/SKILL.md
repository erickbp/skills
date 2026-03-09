---
name: task-splitter
description: >
  Decompose large PRDs, RFCs, epics, migration plans, roadmap items, bugfix plans, or implementation
  plans into atomic task cards that a coding agent (Claude Code, Codex, etc.) can execute safely and
  independently. Use this skill whenever the user wants to break down a plan into tasks, split work
  into cards, decompose a PRD, create an implementation plan from a spec, or organize a large change
  into safe, shippable units -- even if they don't explicitly say "decompose" or "task card." Also
  trigger when users paste a large spec or plan and ask to "make this actionable", "turn this into
  tickets", "plan the implementation", or "what are the steps to build this."
---

# Task Splitter

Take any large PRD, RFC, roadmap item, epic, migration plan, bugfix plan, or implementation plan and break it into atomic task cards that a coding agent can execute safely and independently.

Optimize for execution, not discussion.

## Primary objective

Produce the smallest complete set of task cards that fully delivers the plan while keeping each card:
- independently understandable
- independently reviewable
- independently testable
- as close as possible to independently shippable
- small enough for one focused implementation unit
- safe for a fresh coding-agent thread with no hidden context

## Definition of a good task card

A good task card has:
- one primary objective
- one clear risk boundary
- explicit scope
- explicit non-goals
- observable acceptance criteria
- concrete verification steps
- explicit constraints on behavior, API, dependencies, rollout, and compatibility when relevant

## Core rules

1. **Decompose for execution.**
   - Prefer the smallest complete set of tasks.
   - Do not create artificial tasks just to increase task count.
   - Do not keep tasks large just to reduce task count.

2. **Prefer vertical slices over layer-based buckets.**
   - If a thin end-to-end slice can ship safely, prefer that over separate "backend", "frontend", and "tests" tasks.
   - If the change crosses risky boundaries, split it.

3. **Split on risk boundaries.**
   - Separate schema migration from backfill.
   - Separate backfill from read-path switch.
   - Separate read-path switch from write-path switch when needed.
   - Separate cleanup/removal from first-pass rollout.
   - Separate API contract changes from client adoption when they can ship independently.
   - Separate refactors from behavior changes unless the refactor is strictly required.
   - Separate discovery from implementation when repo facts are unknown.

4. **Keep each card single-purpose.**
   - Every card should have one primary reason to exist.
   - If the title naturally needs "and", it is probably too large unless the pieces are inseparable.

5. **Do not invent requirements or repo facts.**
   - If the PRD is unclear, capture assumptions and open questions explicitly.
   - Do not silently turn guesses into scope.
   - Do not invent exact filenames, modules, tables, endpoints, commands, or environment prerequisites unless the input or repo evidence provides them.
   - If exact repo details are unknown, name the subsystem and expected file areas, and label them as estimated.

6. **Make acceptance criteria observable.**
   - Criteria must describe externally visible behavior, contract guarantees, data invariants, or testable internal outcomes.
   - Avoid vague criteria like "works correctly" or "is production ready".

7. **Make verification executable.**
   - Include concrete commands whenever possible.
   - Prefer repo-realistic commands if known.
   - If commands must be inferred, mark them as best-effort.
   - If verification depends on tools, services, environment variables, fixtures, or local infrastructure, list them as prerequisites.
   - Only include prerequisites that are known from repo evidence or clearly implied by the plan.

8. **Preserve constraints.**
   - Carry forward stated constraints on behavior, API compatibility, latency, data model, dependency choices, rollout, security, compliance, and backward compatibility.
   - If a constraint changes task boundaries, split tasks accordingly.

9. **Surface dependencies clearly.**
   - Identify which cards block others.
   - Mark cards that can run in parallel.
   - Sequence tasks in the safest implementation order.

10. **Prefer additive and reversible sequencing.**
    - Favor changes that can land safely before all consumers switch.
    - Prefer feature flags, compatibility shims, tolerant readers, and staged rollout when applicable.
    - Every risky change should include rollback or mitigation notes.

11. **Separate required work from optional work.**
    - Nice-to-have improvements, future cleanup, and post-launch hardening belong in a separate "Follow-up Tasks" section.

12. **Do not write implementation code.**
    - This is a decomposition pass only.

## Task ID rules

- Always use the prefix `TASK-` followed by a zero-padded two-digit number starting at `01`: `[TASK-01]`, `[TASK-02]`, ..., `[TASK-99]`.
- If there are more than 99 tasks, continue with three digits: `[TASK-100]`, `[TASK-101]`, etc.
- Never use project-specific prefixes like `[PA-01]`, `[AUTH-01]`, or `[DB-01]`. The prefix is always `TASK-`.
- Number tasks sequentially in the order they appear in the output.
- Apply this format everywhere: card headers, dependency references, and execution waves.

## Fresh-thread handoff rule

Each task card must be usable as a standalone handoff in a fresh coding-agent thread.

The implementing agent should treat the task card as the task-specific operating instructions for the session and prioritize the card's constraints, non-goals, compatibility requirements, and rollout notes over generic defaults.

Do not assume access to prior task discussions or hidden context.

## Repo-aware planning checks

Before writing tasks, briefly identify:
- product goal
- repo topology: monolith, service, monorepo, package, app
- likely affected layers or subsystems: UI, API, domain logic, persistence, jobs, infra
- main workstreams
- main risk classes: performance, security, privacy, reliability
- release boundaries and rollout strategy: independent, dark launch, feature flag, staged rollout, cutover
- data model and migration impact: migrations, backfills, idempotency, rollback
- affected contracts: public APIs, internal interfaces, events, schemas
- test surface impact: unit, integration, contract, e2e, migration, smoke
- observability implications
- areas that require discovery before implementation

## Discovery rules

If repo details are missing or uncertain, create discovery cards first.

Discovery cards must:
- stay tightly scoped
- produce concrete outputs
- unblock specific follow-on cards

Concrete outputs should include some combination of:
- list of files, modules, or services involved
- confirmed constraints or conventions
- identified test suites or migration framework
- recommended next task cards

Do not let discovery swallow implementation.

## Migration and data rules

For any schema or data change, classify the work as one of:
- none
- additive schema
- schema change requiring dual-read or dual-write
- backfill required
- read or write cutover
- destructive cleanup later
- unknown; requires repo or data-model review

Usually split migration work into separate cards for:
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

## API and contract rules

For any endpoint, event, queue payload, RPC, SDK, shared type, or public interface change, classify it as:
- additive backward-compatible
- behavior change but contract stable
- breaking change
- unknown

When possible, prefer:
- additive fields
- tolerant readers
- compatibility shims
- staged producer and consumer rollout

If the plan implies a breaking change, isolate it into its own card sequence and call out migration strategy.

## Test rules

Every implementation card must specify expected test impact:
- which existing test suites should be updated
- whether new tests are required
- whether CI coverage is sufficient
- any manual verification required because automation is insufficient

If test work is broad and cross-cutting, create a dedicated test sweep card instead of hiding it inside implementation cards.

## Rollout and observability rules

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

## Sizing rules

Use these buckets only as an internal decomposition check, not as required per-card output:
- XS: tiny, surgical, very low risk
- S: small, focused, standard PR
- M: moderate, still reviewable in one sitting

Do not create L or XL cards. If work would be L or XL, split it.

## Decomposition heuristics

When decomposing, apply these heuristics:
- If a task would likely require multiple commits touching unrelated concerns, split it.
- If a task would require multiple reviewers from distinct domains, split it unless it is still one shippable unit.
- If rollback strategy differs across parts of the work, split it.
- If one part can be feature-flagged and another cannot, consider splitting.
- If one part changes persistent data and another changes UI, split unless the change is a clearly bounded vertical slice.
- If docs, metrics, alerts, or dashboards are required for safe shipping, include them as explicit cards.
- If the input is already granular, normalize and tighten it rather than re-expanding it.

## Required output format

Task IDs must use the `[TASK-NN]` format (e.g. `[TASK-01]`, `[TASK-02]`), numbered sequentially starting at 01. Never use project-specific prefixes.

```
# Decomposition Summary
- Goal: <one short paragraph>
- Likely Affected Areas: <bullet list or short line>
- Main Workstreams: <bullet list or short line>
- Highest-Risk Changes: <bullet list or short line>
- Assumptions: <bullet list, or "None">
- Open Questions: <bullet list, or "None">
- Execution Waves:
  - Wave 1: TASK-01, TASK-02 <brief label, e.g. "discovery and scaffolding">
  - Wave 2 (after TASK-01, TASK-02): TASK-03, TASK-04 <brief label>
  - Wave 3 (after TASK-04): TASK-05 <brief label>
  - <...continue as needed>
  - Within each wave, independent tasks can run in parallel.

# Task Cards

## [TASK-01] <Short action-oriented title>
- Goal: <what this task accomplishes>
- Context Anchor: <one sentence linking this task to the current wave and the larger delivery goal>
- In Scope: <exact files / directories / modules / services / subsystems, or estimated areas if unknown>
- Non-Goals: <what is explicitly out of scope>
- Dependencies: TASK-NN or "None"
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

## Quality bar

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
- all task IDs follow the sequential `[TASK-NN]` format starting at `[TASK-01]` with no project-specific prefixes

The result should be ready for a coding agent to execute card by card without re-planning the entire PRD.
