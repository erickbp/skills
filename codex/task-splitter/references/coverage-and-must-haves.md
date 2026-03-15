# Coverage and Must-Haves

Use this reference when building the `Coverage Map` and the `Must-Haves` section for each card.

## Coverage Map

The coverage map ties the source plan to the card set.

Track:

- source requirements
- observable behaviors
- failed truths in gap-closure mode

For each item, map it to task IDs.

## Coverage Rules

1. Every committed requirement or observable behavior must map to at least one task.
2. Every task must name what it addresses.
3. If an item maps to too many cards, check whether the work is too fragmented.
4. If one card addresses too many distinct items, check whether it is overloaded.

## Depth of Coverage

A requirement is mapped but **shallow** when a task ID appears in the coverage map but the task's Goal or In Scope delivers less than the source requires.

Signals of shallow coverage:

- task Goal says "initial structure", "basic scaffold", "stub", "shell", "placeholder", or "not full flow" for a source-required behavior
- the full behavior is pushed to Follow-up Tasks or deferred without source authorization
- the task delivers types or interfaces but not the runtime behavior the source specifies

When shallow coverage is detected:

- expand the task to deliver the full layer contribution, or
- split into multiple required tasks that together deliver full coverage
- never downgrade source-required behavior to Follow-up Tasks

## Unmapped Items

An item is `Unmapped` when:

- it is in scope
- it is not deferred
- no card clearly exists to satisfy it

When this happens:

- split or add cards if the item is required
- move it to `Deferred / Out of Scope` only if the input supports that move
- never ignore it silently

## Overloaded Tasks

A task is overloaded when it tries to satisfy multiple distinct behaviors or requirements that would be safer as separate reviewable changes.

Common signals:

- the task mixes schema, compatibility, and client adoption
- the task title naturally needs "and"
- the card would be hard to verify in one focused pass
- the card touches several unrelated risk boundaries

When overloaded, split by:

- deployability
- compatibility boundary
- rollout boundary
- subsystem boundary
- verification boundary

## Must-Haves

Every implementation card must include:

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
- derive them from the source requirement or failed truth

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
- use these especially for integration-heavy cards and gap closure

## Gap-Closure Mapping

In `gap_closure` mode, treat `failed truth` as the source unit for coverage.

For each gap:

1. state the failed truth
2. infer the likely root cause
3. list affected artifacts
4. derive the missing must-haves

Example:

- failed truth: "User sees updated billing address immediately after save."
- root-cause guess: cache invalidation missing on success path
- affected artifacts: settings form, mutation handler, cache update helper, test
- key links: mutation success -> cache invalidation -> refreshed read path

This keeps repair work anchored to the missing behavior, not just the symptom report.
