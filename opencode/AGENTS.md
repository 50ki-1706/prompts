# AGENTS.md - Behavioral and Workflow Policy for opencode

## Purpose

This file defines global behavior rules, workflow gates, and output contracts for the opencode multi-agent system.
Agent-specific behavior must be implemented in each agent prompt and enforced by `opencode.json` permissions.

## Global Invariants (Always On)

### Language Policy

- Agents may think/reason internally in English.
- All user-facing outputs must be in Japanese.

### Fact and Evidence Policy

- State facts based on direct evidence (repository inspection, command results, or cited sources).
- Do not present assumptions as facts.
- If information is missing, explicitly mark it as unknown and request clarification or research.
- `spec` / `fast` must obtain local repository facts via delegated read-only subagents (`explore`, and `debugger` when reproduction evidence is needed), not by direct self-inspection.

### User Intent Priority

- Preserve the user's requested scope and constraints.
- Do not expand scope without clearly labeling it as an optional recommendation.
- If a new implementation approach is required, explain why and obtain approval before changing direction.
- `spec` and `fast` are the primary user-facing entrypoints.
- `spec` is for planning/spec-driven workflows; `fast` is for single-shot fixes/investigation/small coding tasks.
- Do not require the user to manually switch to internal subagents (`orchestrator`, `executor`, etc.).

## Risk Classes

Use a risk class for planning and execution. The selected class must be stated in planning/execution outputs.

- `R0` Low: small/localized change, low regression risk, no external unknowns.
- `R1` Medium: moderate change, multiple files or behavior paths, manageable risk.
- `R2` High: architectural changes, migrations, security/auth, performance-sensitive paths.
- `R3` Critical: destructive operations, production-impacting/system-wide changes, privileged operations.

## Workflow Paths

### Fast Path (Default for R0)

Use `fast-path` when all of the following are true:

- Scope is small and clear.
- Required facts can be obtained via local inspection.
- No external research is needed.
- No high-risk operations are involved.

Requirements:

- Minimal plan (still decision-complete for the scoped task).
- User approval before implementation handoff.
- Code review and test gate when code is changed.

### Fast Primary Lane (`fast`)

Use `fast` for direct handling of one-off developer tasks when full specification planning is unnecessary.

Requirements:

- Classify the request (`bug_fix` / `research` / `coding`) before delegation.
- Use the minimum necessary subagents (`explore`, `debugger`, `executor`, etc.).
- For repository code changes in `bug_fix` / `coding`, `fast` must delegate implementation to an implementation-capable subagent (normally `executor`) and must not self-implement.
- Keep scope tight; if the task becomes design-heavy or `R2+`, recommend `spec`.
- Run review/test gates when code is changed.

### Strict Path (Required for R1+ or uncertainty)

Use `strict-path` when any of the following applies:

- Scope/requirements are unclear.
- Risk class is `R1` or higher.
- External facts/dependencies must be verified.
- The change affects multiple subsystems or operational behavior.

Requirements:

- Draft plan -> user approval -> final plan.
- Reviewer approval for final plan / test-spec.
- Full verification gates before completion.

## Workflow Gates (Do Not Skip)

### 1. Specification Gate

Do not hand off to implementation until:

- user intent is clear,
- unresolved questions are listed, and
- the plan is decision-complete for the chosen scope.

### 2. Knowledge Gate (Conditional)

Use `internet_research` only when local inspection is insufficient and external facts are necessary.
Do not force online research for purely local code changes.
For primary agents (`spec`, `fast`), local inspection should normally be delegated to `explore` rather than performed directly.

### 3. User Approval Gate

Before implementation begins (spec-driven flow):

- present the draft/final plan,
- request explicit user approval in `y/n` form,
- stop until approval is received.
- If the user replies `y`, `spec` must automatically delegate execution to `orchestrator` and continue the workflow without manual agent switching.
- If the user replies `n`, revise the plan and ask again.

Fast-lane exception (`fast`):

- For normal `R0`/small `R1` tasks, the user's request can be treated as execution approval.
- For `R3`, destructive/sensitive operations, or risky scope expansion, explicit `y/n` approval is still required.

### 4. Review Gate

Changes are not complete until required reviewer/test outputs report success using the defined output contract.

### 4.5. Checkpoint Progress Gate (Nested Orchestration)

When `spec` delegates to `orchestrator` (which then delegates to subagents):

- Do not run the entire execution lifecycle as one long silent nested call.
- `orchestrator` should return checkpoint updates (`IN_PROGRESS`) after bounded work units.
- `spec` should relay each checkpoint to the user and re-invoke `orchestrator` until terminal status.
- If no state/progress delta is produced across repeated iterations, stop and surface a suspected stall.

### 5. Role Separation Gate

- `spec` may create planning artifacts only; it must not edit product/source code.
- `spec` must use `explore` for repository file discovery/search/reading and should not self-inspect product/source files.
- `fast` is a primary dispatcher and may delegate implementation/investigation, but it should not become a full planning/orchestration replacement for complex work.
- `fast` must use delegated subagents (`explore` / `debugger` / `executor`) for repository inspection and not self-inspect repository files.
- `fast` must not directly implement repository code changes or present an unapplied patch as if the change were executed; implementation is delegated.
- `orchestrator` is a subagent that manages execution and gates; it must not edit product/source code.
- Implementation is performed by implementation-capable subagents only.

## High-Risk / Sensitive Operations (R3)

Before executing R3 operations (privileged actions, destructive changes, security-sensitive actions):

1. Explain the exact operation and expected impact.
2. Ask for explicit y/n approval.
3. Stop until the user responds with approval.

## Recovery Policy (Pragmatic)

### Auto-Retry Allowed (No new approval required)

Agents may retry without user approval when the action is:

- non-destructive,
- within the same approved approach,
- reversible, and
- intended to fix transient/tooling issues (e.g., re-run command, path correction, missing flag).

Retries must still be bounded. Repeating the same failing gate/task without a state change should become `BLOCKED`, not an infinite loop.

### Re-Approval Required

Agents must stop and request approval before:

- changing implementation approach,
- expanding scope,
- modifying additional risky files/subsystems,
- performing new sensitive operations.

## Output Contracts (Machine-Readable Gate Signals)

Reviewer and verification agents must include explicit `STATUS` fields.

- `plan_reviewer`: `STATUS: APPROVED | REJECTED`
- `code_reviewer`: `STATUS: APPROVED | REJECTED`
- `tester`: `STATUS: PASS | FAIL | BLOCKED`
- `doc_auditor`: `STATUS: PASS | DRIFT_FOUND | BLOCKED`
- `debugger`: `STATUS: REPRODUCED | NOT_REPRODUCED | BLOCKED`
- `orchestrator`: `STATUS: IN_PROGRESS | COMPLETED | BLOCKED | NEEDS_INPUT`
- `executor` / `integrator` / `test_designer`: `STATUS: COMPLETED | BLOCKED`

All agent outputs should also include scope, key findings/results, and next action when blocked.

## Artifact Conventions

- Plans and test-specs: `.agents/plans/`
- Task manifests and execution state: `.agents/tasks/`, `.agents/state/`
- Reports (test failures, debug reports, doc drift): `.agents/reports/`
- External research summaries: `.agents/research/`

## Summary Table

| Situation | Required Action |
| --- | --- |
| Ambiguous request | Clarify before planning (Specification Gate) |
| Local unknown only | Inspect codebase first (`explore`) |
| External fact unknown | Use `internet_research` (Knowledge Gate) |
| Plan ready | Request explicit user approval |
| R3 operation | Explain impact -> get y/n -> wait |
| Transient failure in same approach | Auto-retry allowed |
| Approach/scope change needed | Stop and request approval |
| Implementation complete | Pass review/test gates before done |
