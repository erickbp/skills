# Manifest Template

The manifest is a living document that evolves throughout the iterative build process. It starts with requirements only (Phase 1) and grows as tasks are designed, implemented, and completed (Phase 2).

## Template

```md
# Iterative Build Manifest

## Goal
<One short paragraph describing the delivery goal>

## Discovered Facts
- Repo topology: <verified>
- Relevant paths: <verified>
- Data layer: <verified>
- Test setup: <verified>
- Build/CI: <verified>
- Key conventions: <verified>
- <other verified facts as needed>

## Decision Ledger
- Locked Decisions:
  - <bullet list, or "None">
- Coding Agent's Discretion:
  - <bullet list, or "None">
- Deferred / Out of Scope:
  - <bullet list, or "None">
- Open Questions:
  - <bullet list, or "None">

## Success Criteria
- SC-01: <observable condition that proves the goal is met>
- SC-02: <observable condition>
- ...

## Requirements
- REQ-01: <requirement description> — `pending`
- REQ-02: <requirement description> — `pending`
- REQ-03: <requirement description> — `done (TASK-01)`
- REQ-04: <requirement description> — `in-progress (TASK-02)`
- ...

## Completed Tasks
- [TASK-01] <title>: <brief summary of what was built, key files created/modified>
- [TASK-02] <title>: <brief summary>
- ...

## Adjustments Log
<Chronological log of changes made during the build>
- <date/task context>: <what changed and why>
- ...
<or "None">

## Remaining Work
<Summary of what is left to build, based on pending requirements>
- REQ-NN: <brief description of remaining work>
- ...
<or "All requirements addressed">

## Follow-up Items
<Only explicitly optional, post-launch, cleanup, or nice-to-have items>
- <item>
- ...
<or "None">
```

## Update Rules

### After Bootstrap (Phase 1)
The manifest is created with:
- Goal, Discovered Facts, Decision Ledger, Success Criteria filled in
- All requirements listed with status `pending`
- Completed Tasks, Adjustments Log empty
- Remaining Work mirrors the full requirement list
- Follow-up Items contains any items identified during bootstrap

### After Each Task Completes (Phase 2)
Update the manifest with:

1. **Requirements**: Change status from `in-progress (TASK-NN)` to `done (TASK-NN)` for addressed requirements. If partial, note what remains.
2. **Completed Tasks**: Add entry with task ID, title, and brief summary of what was built.
3. **Adjustments Log**: Record any scope changes, new discoveries, requirement modifications, or approach changes that occurred during the task.
4. **Remaining Work**: Update to reflect current state of pending requirements.
5. **Follow-up Items**: Add any non-critical improvements discovered during implementation.

### When Requirements Change
1. Update the requirement text in the Requirements section.
2. Log the change in the Adjustments Log with before/after and rationale.
3. If a new requirement is added, assign a new `REQ-NN` identifier.
4. If a requirement is removed, mark it as `deferred` and move the description to Deferred / Out of Scope in the Decision Ledger.

### After Validation (Phase 3)
1. Verify each Success Criterion is satisfied by noting which task(s) addressed it.
2. Record any gaps and their resolution (additional tasks, accepted as-is, or deferred).
3. Ensure Remaining Work says "All requirements addressed" or lists any outstanding items.
4. The final manifest serves as the complete record of the iterative build.
