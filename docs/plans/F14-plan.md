# Implementation Plan: F14 — Run Reporter (`unit-test-reporter`)

## 1. Dependencies on Other Features

- **Depends on:**
  - [F07: JSON Contracts](../features/F07-json-contracts.md): every field the report renders.
  - [F02](F02-plan.md)–[F06](F06-plan.md), [F13](F13-plan.md): the agents whose outputs it reads.

## 2. Files to Create or Change

- `agents/unit-test-reporter.md` — Canonical Markdown agent instructions.
- Runtime wrappers are created in [F08](F08-plan.md).

## 3. Ordered Tasks

### Task F14-T1: Draft `agents/unit-test-reporter.md`

- **Description:** Write the canonical instructions: preconditions, input file selection (latest generation revision), the render-only rules table, the recommendation table, the section order with a Markdown template, the no-requirements banner, and failure-workflow rendering.
- **Acceptance Criteria:**
  - [ ] Writes only `runs/<run-id>/report.md` and runs no tests or tools.
  - [ ] Contains the recommendation table verbatim from [F14](../features/F14-reporter.md).
  - [ ] Contains no review, classification, or test-writing rules (those stay in their agents).
- **Files Touched:** `agents/unit-test-reporter.md`

### Task F14-T2: Verify on Fixtures

- **Description:** Run `/unit-test-reporter <run-id>` on a requirement-driven run, a run without requirements, and a `--failures` run, for both fixtures.
- **Acceptance Criteria:**
  - [ ] `evals/` `CHK-15` passes (sections present, IDs and counts match the JSON, recommendation matches the table).
  - [ ] The no-requirements run shows the banner and recommendation row 5 (or a higher row when it matches).
- **Files Touched:** `PROGRESS.md`

## 4. How to Verify Manually

1. After a requirement-driven run on `examples/ts-vitest`, run `/unit-test-reporter <run-id>`.
2. Confirm the headline is "Do not merge. Fix production code" with the `discount.ts` defects, and the traceability table lists every `SCN` from `requirements.json`.
3. Run it on a `--failures` run and confirm only Summary, Actions, Failure triage, and Reproduce appear.
