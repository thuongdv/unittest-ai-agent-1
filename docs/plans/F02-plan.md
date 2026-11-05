# Implementation Plan: F02 — Unit Test Generator: Generate Mode (`unit-test-generator`)

## 1. Dependencies on Other Features

- **Depends on:**
  - [F07: JSON Contracts (`schemas/generation.schema.json`)](../features/F07-json-contracts.md)
  - [F01: Codebase Analyzer (`project.json`)](../features/F01-codebase-analyzer.md)
  - [F13: Requirement Analyzer (`requirements.json`)](../features/F13-requirement-analyzer.md)

## 2. Files to Create or Change

- `agents/unit-test-generator.md` — Canonical Markdown agent instructions (Generate Mode section).

## 3. Ordered Tasks

### Task F02-T1: Draft `agents/unit-test-generator.md` Preconditions & Generate Mode Instructions

- **Description:** Create `agents/unit-test-generator.md` with argument routing (`<targets>` vs `--fix` vs `--repair-failures`), strict precondition checks (`project.json` existence, `schemaVersion`, zero blocking `missing` items), test case design rules (`happy_path`, `edge`, `boundary`, `error`), repair loop (`maxRepairAttempts`), target-scoped coverage loop, `uncoveredLines` reporting, `suspectedDefects` skip/xfail guardrails, and `--requirements <run-id>` scenario mapping (expected values from scenarios only, `scenarioId`, `unmappedScenarios`).
- **Acceptance Criteria:**
  - [ ] Enforces writing only under `testRoots` and never modifying `sourceRoots`.
  - [ ] Enforces keeping valid assertions on defective production code, marking tests skipped/expected-fail, and populating `suspectedDefects`.
  - [ ] Forbids taking a requirement-linked expected value from the code's output.
  - [ ] Output matches `schemas/generation.schema.json`.
- **Files Touched:** `agents/unit-test-generator.md`

### Task F02-T2: Verify Generate Mode on Fixtures

- **Description:** Run `/unit-test-generator src/pricing.ts` on `examples/ts-vitest` and `/unit-test-generator src/inventory/reorder.py` on `examples/py-pytest`.
- **Acceptance Criteria:**
  - [ ] Generates `.unit-test-agent/runs/<run-id>/generation.json` valid against `schemas/generation.schema.json`.
  - [ ] Deliberate bug is logged in `suspectedDefects` and its test case is skipped/xfail.
  - [ ] `git diff src/` is empty.
  - [ ] With `--requirements` on `src/discount.ts`, the $100 and $200 scenario tests are skipped as defects with `scenarioId` set; without it, `requirementsFile` is `null`.
- **Files Touched:** `PROGRESS.md`

## 4. How to Verify Manually

1. In a temp copy of `examples/ts-vitest` with `project.json` initialized, run `/unit-test-generator src/pricing.ts`.
2. Confirm `tests/pricing.test.ts` is created and passes when executed via `npx vitest run`.
3. Confirm `.unit-test-agent/runs/<run-id>/generation.json` lists the deliberate bug in `suspectedDefects` and `uncoveredLines` explains the unreachable guard.
