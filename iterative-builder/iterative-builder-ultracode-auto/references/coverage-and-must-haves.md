# Coverage and Must-Haves

Use this reference when tracking requirement coverage and building the `Must-Haves` section for each task card.

## Requirement Status Tracking

Each requirement in the manifest has a status:

- `pending` — not yet addressed by any task
- `in-progress (TASK-NN)` — currently being addressed by the named task
- `done (TASK-NN)` — fully addressed by the named task

When a requirement is addressed by a task, update its status. If a requirement was partially addressed and needs additional work, keep it as `pending` with a note about what remains, or create a new requirement for the delta.

## Coverage Rules

1. Every committed requirement must be addressed by at least one task.
2. Every task must name what requirement(s) it addresses.
3. If a requirement needs more work after a task completes, either update the requirement status with a note or create a new requirement for the remaining work.
4. Non-functional requirements that cannot be verified by task-level tests or commands (latency targets, throughput, load behavior) should be noted with a `(design-for)` annotation and listed in Follow-up Items for dedicated performance testing.

## Depth of Coverage

A requirement is addressed but **shallow** when a task claims to cover it but delivers less than the requirement demands.

Signals of shallow coverage:

- task Goal says "initial structure", "basic scaffold", "stub", "shell", "placeholder", or "not full flow" for a requirement that demands full behavior
- the full behavior is pushed to Follow-up Items without source authorization
- the task delivers types or interfaces but not the runtime behavior the requirement specifies

When shallow coverage is detected:

- expand the task to deliver the full contribution, or
- split into additional tasks that together deliver full coverage
- never downgrade requirement-demanded behavior to Follow-up Items

## Security Requirements as Truths

Authentication, authorization, and input validation requirements from the source document must appear as Truths in the relevant task's Must-Haves, not only in Constraints or Notes. If the source specifies auth for a class of endpoints, every task that creates endpoints in that class must have an auth truth.

Signals of buried security requirements:

- auth appears only in a Constraints bullet, not in Truths
- webhook signature validation is a "Note" rather than a must-have
- role-based access is mentioned in the source but absent from endpoint tasks

When detected, promote the security requirement to a Truth: e.g., "All reservation endpoints require authenticated user with valid JWT" or "Webhook handler validates ACS signature before processing payload."

## Must-Haves

Every task card must include:

- `Truths`
- `Artifacts`
- `Key Links`

These are not prose extras. They are the compact execution contract.

### `Truths`

These are the observable behaviors or invariants that must be true when the card is done.

Good:

- "User can submit the form without losing existing attachments."
- "Backfill can be rerun without duplicating migrated rows."
- "Consumer accepts both old and new event shapes during rollout."

Bad:

- "API file created."
- "Schema updated."
- "Tests added."

Rules:

- prefer externally meaningful behavior or operational invariants
- make them short and testable
- derive them from the source requirement

Infrastructure and persistence tasks deserve extra care:

- Frame Truths as what the layer guarantees, not what classes exist. Prefer: "queries scoped by tenant ID never return cross-tenant data" over "repository implementations filter by group ID." Prefer: "round-tripping an entity through persistence preserves all fields including value objects" over "DbContext has DbSet for all entities."
- "Build succeeds" is a verification step, not a Truth. It belongs in Verification Commands.

Scaffolding tasks (project setup, solution structure) also need behavioral Truths:

Good:

- "Adding a reference from Application to Infrastructure causes a build failure."
- "All projects compile and produce assemblies under the declared target framework."

Bad:

- "Domain.csproj contains no ProjectReference elements." (structural assertion — can be true of an empty file)
- "`dotnet build` succeeds with zero errors." (verification step, not a Truth)

The test: if the Truth can be satisfied by an empty or trivially wrong file, reframe it as the guarantee that matters.

### `Artifacts`

These are the concrete things that must exist for the truth to hold.

Examples:

- route file
- handler module
- migration
- backfill job
- feature flag
- dashboard or alert
- test file
- shared type or contract

Rules:

- name verified repo paths when known
- use subsystem-level descriptions only when the repo cannot resolve exact paths
- do not list filler artifacts that add no execution value

### `Key Links`

These are the critical connections that often get missed even when artifacts exist.

Examples:

- UI submits to endpoint
- endpoint writes using the new schema path
- backfill updates the field consumed by the read path
- feature flag gates both read and write behavior
- alert watches the rollout metric that actually matters

Rules:

- call out the minimum wiring needed for the truth to hold
- prefer real semantic links over generic "hook it up" language
- use these especially for integration-heavy cards

## Manifest Update Guidance

After each task completes and merges to main:

1. Update the requirement status from `in-progress (TASK-NN)` to `done (TASK-NN)`.
2. Add a Completed Tasks entry summarizing what was built, what files were created/modified.
3. If the task revealed new work needed, either update existing requirements or create new ones.
4. If the task changed the approach for remaining work, log the change in the Adjustments Log.
5. Update Remaining Work to reflect the current state.
6. Review Follow-up Items — add any non-critical improvements discovered during implementation.
