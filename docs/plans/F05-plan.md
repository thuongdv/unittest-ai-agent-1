# Implementation Plan: F05 — Unit Test Failure Analyzer (`unit-test-failure-analyzer`)

## 1. Dependencies on Other Features

- **Depends on:**
  - [F07: JSON Contracts (`schemas/failure-analysis.schema.json`)](../features/F07-json-contracts.md)
  - [F01: Codebase Analyzer (`project.json`)](../features/F01-codebase-analyzer.md)

## 2. Files to Create or Change

- `agents/unit-test-failure-analyzer.md` — Canonical Markdown agent instructions for failure root-cause analysis.

## 3. Ordered Tasks

### Task F05-T1: Draft `agents/unit-test-failure-analyzer.md`

- **Description:** Write `agents/unit-test-failure-analyzer.md` covering precondition checks, both input modes (direct test execution `[targets]` and log file parsing `--log <file>`), stack-trace + source + `git log`/`git diff` inspection, the 6 root-cause categories (`stale_test`, `broken_mock_or_fixture`, `source_regression_bug`, `flaky_or_nondeterministic`, `environment_or_dependency`, `syntax_or_import_error`), conservative fallback to `source_regression_bug` when uncertain, and writing `failure-analysis.json`.
- **Acceptance Criteria:**
  - [ ] Supports both command execution and `--log <file>` ingestion.
  - [ ] Maps every failure category to its corresponding `recommendedAction`.
  - [ ] Strictly read-only outside `.unit-test-agent/runs/<run-id>/failure-analysis.json`.
- **Files Touched:** `agents/unit-test-failure-analyzer.md`

### Task F05-T2: Verify `unit-test-failure-analyzer` on Seeded Fixture Failures

- **Description:** Run `/unit-test-failure-analyzer` (both via direct execution on `tests/regression/` and via `--log sample-failure.log`) on `examples/ts-vitest` and `examples/py-pytest`.
- **Acceptance Criteria:**
  - [ ] Accurately classifies the seeded outdated test as `stale_test` (`update_test`) and the production bug failure as `source_regression_bug` (`skip_and_report_defect`).
  - [ ] Output validates against `schemas/failure-analysis.schema.json`.
- **Files Touched:** `PROGRESS.md`

## 4. How to Verify Manually

1. In a temp copy of `examples/ts-vitest`, run `/unit-test-failure-analyzer tests/regression/order.failing.ts`.
2. Inspect `.unit-test-agent/runs/<run-id>/failure-analysis.json` and confirm `FAIL-001` is `stale_test` and `FAIL-002` is `source_regression_bug`.
3. Run `/unit-test-failure-analyzer --log sample-failure.log` and confirm log ingestion produces the same root-cause classifications.
