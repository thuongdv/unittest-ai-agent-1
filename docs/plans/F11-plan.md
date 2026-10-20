# Implementation Plan: F11 — Automated Output Checks (`evals/`)

## 1. Dependencies on Other Features

- **Depends on:**
  - [F07: JSON Contracts (`schemas/`)](F07-plan.md)
  - [F09: Example Fixtures (`examples/`)](F09-plan.md)
    _(Per `docs/prompt.md` Step 3: Build F11 after F07 (schemas) and F09 (fixtures) so it can verify agent runs.)_

## 2. Files to Create or Change

- `evals/package.json`
- `evals/tsconfig.json`
- `evals/vitest.config.ts`
- `evals/src/loader.ts` (loads target dir JSON files, schemas, and git diff state)
- `evals/src/passrate.ts` (aggregates multi-run pass rates across `EVAL_RUN_DIRS`)
- `evals/tests/schema.eval.test.ts` (`CHK-01`)
- `evals/tests/analyzer.eval.test.ts` (`CHK-02`, `CHK-10`, `CHK-11`)
- `evals/tests/generator-and-review.eval.test.ts` (`CHK-03`, `CHK-04`, `CHK-05`, `CHK-06`, `CHK-07`)
- `evals/tests/failure-workflow.eval.test.ts` (`CHK-08`, `CHK-09`)
- `evals/tests/self-check.test.ts` (unit tests with synthetic valid/invalid run fixtures verifying the eval suite itself)

## 3. Ordered Tasks

### Task F11-T1: Scaffold `evals/` Package, Schema Validator & Loader (`CHK-01` to `CHK-05`)

- **Description:** Initialize `evals/` with TypeScript, Vitest, `ajv`, and `ajv-formats`. Implement `loader.ts` and property checks `CHK-01` (schema & envelope), `CHK-02` (`project.json` ground truth), `CHK-03` (`git diff` clean outside `testRoots`), `CHK-04` (deliberate bug in `suspectedDefects` + skipped/xfail test), and `CHK-05` (reported coverage vs re-run coverage).
- **Acceptance Criteria:**
  - [ ] `CHK-01` through `CHK-05` pass against synthetic valid runs and fail against synthetic invalid runs.
- **Files Touched:** `evals/package.json`, `evals/tsconfig.json`, `evals/vitest.config.ts`, `evals/src/loader.ts`, `evals/tests/schema.eval.test.ts`, `evals/tests/analyzer.eval.test.ts`, `evals/tests/generator-and-review.eval.test.ts`

### Task F11-T2: Implement Reviewer, Fix, Failure-Analysis, Repair & Precondition Checks (`CHK-06` to `CHK-11`) + Pass-Rate Reporter

- **Description:** Implement `CHK-06` (`review.json` caseId integrity), `CHK-07` (fix mode scope & assertion count preservation), `CHK-08` (`failure-analysis.json` `stale_test` vs `source_regression_bug`), `CHK-09` (`failure-repair.json` verification), `CHK-10` (user edit preservation on analyzer re-run), `CHK-11` (blocking `missing` item stops downstream agents), `passrate.ts` multi-run reporter, and `self-check.test.ts`.
- **Acceptance Criteria:**
  - [ ] `npm --prefix evals run test:self` passes all 11 checks (`CHK-01`–`CHK-11`).
  - [ ] Multi-run pass-rate reporter outputs per-check and overall pass rates across multiple run directories.
- **Files Touched:** `evals/src/passrate.ts`, `evals/tests/generator-and-review.eval.test.ts`, `evals/tests/failure-workflow.eval.test.ts`, `evals/tests/self-check.test.ts`

## 4. How to Verify Manually

1. Run `npm --prefix evals install && npm --prefix evals run test:self` to verify all 11 checks against synthetic mock run directories.
2. Run an agent workflow in a temp fixture copy and execute `EVAL_TARGET_DIR=/tmp/<run> EVAL_FIXTURE=ts-vitest npm --prefix evals test`.
