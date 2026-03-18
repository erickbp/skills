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

### Greenfield variant

When the repo is empty or near-empty:

- Repo grounding yields structure FROM the input document rather than FROM existing code.
  Discovered Facts should note "greenfield" and cite the input document as the source.
- The first 1-3 waves will typically be horizontal (scaffold, domain, interfaces) because
  the dependency chain must be established before vertical slices are possible.
- Paths in task cards are convention-based, not tool-verified. Label them accordingly.
- "Patterns to Follow" for the first wave is "None (greenfield)" — subsequent waves reference
  patterns established by predecessor waves.
- Test cards for foundational layers should immediately follow those layers, not accumulate
  into a single final testing wave.
- For host/wiring tasks that register dependencies from multiple predecessor waves (e.g.,
  ASP.NET Program.cs, DI containers, module registrations), split into:
  (a) a host skeleton task (middleware, auth, config, health checks, placeholder DI) that
  lands as soon as its own direct dependencies are met, and
  (b) a DI completion task that wires remaining services after they exist.
  Do not defer the entire host to a late wave just because some registered services are not
  yet implemented. A host that depends on 3+ predecessor waves is a bottleneck — split it.
- Parallel tasks must not modify the same DI registration file (Program.cs, DI extension
  methods, module registrations). If multiple services need registration, either:
  (a) consolidate all DI registration into a single host/wiring task placed after the
  services it registers, or
  (b) use the host skeleton + DI completion split above, where only the DI completion task
  performs registration.
  Do not distribute DI registration across parallel tasks — this is a guaranteed merge conflict.
- For projects using an ORM with migration support (EF Core, Alembic, Django migrations, etc.),
  the decomposition must explicitly address initial schema creation — either as a dedicated task,
  as part of the DbContext/model configuration task, or as an explicit Coding Agent's Discretion
  item in the decision ledger. Do not assume the coding agent will figure out schema creation
  on their own.

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
