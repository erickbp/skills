# Modes

Choose one mode before writing task cards.

## `standard_decomposition`

Use for:

- PRDs
- RFCs
- epics
- roadmap items
- implementation plans
- large bugfix plans that are still mostly forward-looking

Shape:

- normal repo grounding
- normal decision ledger
- requirement-to-task coverage map
- standard task-card output

Use this unless another mode clearly fits better.

## `brownfield_extension`

Use when:

- the user is adding to an existing codebase
- established repo conventions strongly affect the task split
- existing modules, contracts, migrations, or test patterns are part of the planning constraint

Extra expectations:

- verify representative existing patterns before card writing
- prefer cards that extend existing seams over inventing new abstractions
- call out compatibility with existing interfaces and rollout expectations
- include exact affected paths when verified

Red flags:

- decomposing as if the repo were greenfield
- inventing abstractions that conflict with local conventions
- hiding migration or adoption work inside a generic feature card

## `gap_closure`

Use when the input is:

- a verifier report
- QA findings
- UAT findings
- a cluster of post-implementation bugs
- a "phase went wrong" summary

Primary job:

- normalize gaps into failed truths
- infer likely root causes
- turn them into the fewest focused repair cards

Normalization model:

- `failed truth`: the behavior or invariant that is not true
- `root-cause guess`: the most plausible underlying problem
- `affected artifacts`: files, modules, endpoints, jobs, configs, dashboards
- `missing work`: wiring, tests, compatibility, rollout guardrails, or substantive implementation

Preferred grouping:

- group by root cause, not by symptom
- separate "repair behavior" from "add coverage" when they can land independently
- keep follow-up hardening work separate from required repair work

Do not:

- mirror the bug list one-to-one into shallow cards
- bury missing tests inside a broad behavior-repair card when test work is independently reviewable

## `migration_heavy`

Use when the primary complexity is:

- schema change
- data movement
- backfill
- read or write cutover
- compatibility period
- destructive cleanup planning

Required decomposition bias:

1. schema preparation or additive change
2. write-path compatibility
3. backfill or migration runner
4. read-path cutover
5. cleanup or removal later

Always call out:

- idempotency expectations
- rollout order
- rollback path
- data validation checks
- performance and load concerns
- reversibility

Do not combine destructive cleanup with first-pass feature delivery.

## Mode Selection Heuristics

Use the most specific valid mode:

1. If the input is primarily failed verification or QA findings, use `gap_closure`.
2. Else if data rollout and compatibility are the primary risk, use `migration_heavy`.
3. Else if existing repo conventions materially constrain the work, use `brownfield_extension`.
4. Else use `standard_decomposition`.

If two modes seem relevant:

- choose the one that drives the highest-risk sequencing
- mention the secondary concern in `Highest-Risk Changes`
- do not mix output rules from multiple modes in a confusing way
