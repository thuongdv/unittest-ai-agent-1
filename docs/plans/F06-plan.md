# Implementation Plan: F06 — Unit Test Generator: Repair-Failures Mode (`unit-test-generator --repair-failures <run>`)

## 1. Dependencies on Other Features

- **Depends on:**
  - [F07: JSON Contracts (`schemas/failure-repair.schema.json`, `schemas/failure-analysis.schema.json`)](../features/F07-json-contracts.md)
  - [F02: Generator — Generate Mode](../features/F02-generator-generate.md)
  - [F05: Failure Analyzer](../features/F05-failure-analyzer.md)

## 2. Files to Create or Change

- `agents/unit-test-generator.md` — Add the **Repair-Failures Mode (`--repair-failures <run-id>`)** section.

## 3. Ordered Tasks

### Task F06-T1: Add Repair-Failures Mode (`--repair-failures <run-id>`) Instructions to `agents/unit-test-generator.md`

- **Description:** Extend `agents/unit-test-generator.md` with instructions for `--repair-failures <run-id>`: reading `failure-analysis.json`, repairing `stale_test` / `broken_mock_or_fixture` / `flaky_or_nondeterministic` / test-level `syntax_or_import_error` in place, marking `source_regression_bug` tests as skipped/expected-fail while preserving assertions and recording `suspectedDefects`, reporting `environment_or_dependency` as `blocked_env`, verifying via test execution up to `maxRepairAttempts`, and writing `failure-repair.json`.
- **Acceptance Criteria:**
  - [ ] Never modifies production source files under `sourceRoots`.
  - [ ] Never changes expected values to match buggy production output for `source_regression_bug`.
  - [ ] Output conforms to `schemas/failure-repair.schema.json`.
- **Files Touched:** `agents/unit-test-generator.md`

### Task F06-T2: Verify Repair-Failures Mode End-to-End on Seeded Fixture Failures

- **Description:** Run `/unit-test-generator --repair-failures <run-id>` following a `unit-test-failure-analyzer` run on `examples/ts-vitest` and `examples/py-pytest`.
- **Acceptance Criteria:**
  - [ ] Stale test is updated and passes.
  - [ ] Regression bug test retains its original assertion, is marked skipped/xfail, and is listed in `suspectedDefects`.
  - [ ] Re-running the repaired test file exits with code 0.
- **Files Touched:** `PROGRESS.md`

## 4. How to Verify Manually

1. After running `/unit-test-failure-analyzer` on `examples/ts-vitest`, run `/unit-test-generator --repair-failures <run-id>`.
2. Confirm `.unit-test-agent/runs/<run-id>/failure-repair.json` exists and `verification.exitCode == 0`.
3. Check `git diff` to verify `src/` is untouched and only the failing test file was modified.
