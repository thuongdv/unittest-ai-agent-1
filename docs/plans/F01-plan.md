# Implementation Plan: F01 — Codebase Analyzer (`codebase-analyzer`)

## 1. Dependencies on Other Features

- **Depends on:** [F07: JSON Contracts (`schemas/project.schema.json`)](../features/F07-json-contracts.md) (must be created first so the analyzer instructions reference the exact schema).
- **Recommended before manual verification:** [F08: Multi-Runtime Packaging](../features/F08-multi-runtime-packaging.md) and [F09: Example Fixtures](../features/F09-example-fixtures.md).

## 2. Files to Create or Change

- `agents/codebase-analyzer.md` — Canonical Markdown agent instructions.

## 3. Ordered Tasks

### Task F01-T1: Draft `agents/codebase-analyzer.md` Detection Rules & Ecosystem Reference Table

- **Description:** Write the canonical instruction file `agents/codebase-analyzer.md` covering existing `project.json` preservation rules, the 6-ecosystem detection reference table (JS/TS, Python, Java/Kotlin, C#/.NET, Go, Rust), style exemplar selection, baseline test execution, and `missing` array classification (`blocking: true` vs `false`).
- **Acceptance Criteria:**
  - [ ] Includes concise, imperative instructions for every field in `schemas/project.schema.json`.
  - [ ] Explicitly forbids overwriting non-null user-edited values on re-run.
  - [ ] Includes the reference table for manifests, test runners, coverage tools, and mutation tools across JS/TS, Python, Java/Kotlin, C#/.NET, Go, and Rust.
- **Files Touched:** `agents/codebase-analyzer.md`

### Task F01-T2: Verify `codebase-analyzer` End-to-End on Fixtures

- **Description:** Run `/codebase-analyzer` against temporary copies of `examples/ts-vitest` and `examples/py-pytest` (including a re-run after manually editing `project.json`).
- **Acceptance Criteria:**
  - [ ] `.unit-test-agent/project.json` validates against `schemas/project.schema.json`.
  - [ ] Baseline test counts match actual fixture execution.
  - [ ] Manual edit to `policy.coverageThresholds.lines` survives re-run.
- **Files Touched:** `PROGRESS.md`

## 4. How to Verify Manually

1. Copy `examples/ts-vitest` to a temp directory, install `.unit-test-agent/{agents,schemas}` and `.claude/`.
2. Run `/codebase-analyzer`.
3. Inspect `.unit-test-agent/project.json` for `language.value == "typescript"`, `testFramework.value == "vitest"`, and `baseline.passed > 0`.
4. Modify `policy.coverageThresholds.lines` to `90`, re-run `/codebase-analyzer`, and confirm `90` is unchanged.
