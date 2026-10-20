# Implementation Plan: F03 — Unit Test Generator: Fix Mode (`unit-test-generator --fix <run>`)

## 1. Dependencies on Other Features

- **Depends on:**
  - [F07: JSON Contracts (`schemas/generation.schema.json`, `schemas/review.schema.json`)](../features/F07-json-contracts.md)
  - [F02: Generator — Generate Mode](../features/F02-generator-generate.md)
  - [F04: Unit Test Reviewer](../features/F04-reviewer.md)

## 2. Files to Create or Change

- `agents/unit-test-generator.md` — Add the **Fix Mode (`--fix <run-id>`)** section.

## 3. Ordered Tasks

### Task F03-T1: Add Fix Mode (`--fix <run-id>`) Instructions to `agents/unit-test-generator.md`

- **Description:** Extend `agents/unit-test-generator.md` with step-by-step instructions for `--fix <run-id>`: checking `review.json` preconditions, computing revision `<N>`, applying surgical fixes only to flagged cases/files without deleting assertions, killing surviving mutants, stabilizing flaky tests, re-running coverage, and writing `generation.r<N>.json`.
- **Acceptance Criteria:**
  - [ ] Explicitly forbids modifying unflagged test files or removing assertions to pass tests.
  - [ ] Specifies `generation.r<N>.json` fields (`revision`, `parentGenerationFile`, `reviewFile`, `addressedFindings`).
- **Files Touched:** `agents/unit-test-generator.md`

### Task F03-T2: Verify Fix Mode End-to-End on Fixtures

- **Description:** Run `/unit-test-generator --fix <run-id>` on a run where `review.json` requested changes.
- **Acceptance Criteria:**
  - [ ] Writes `generation.r1.json` conforming to `schemas/generation.schema.json`.
  - [ ] Only flagged test files are modified, and all findings in `review.json` are addressed in `addressedFindings`.
- **Files Touched:** `PROGRESS.md`

## 4. How to Verify Manually

1. In a temp fixture directory containing a completed `generation.json` and a `review.json` with `verdict: "changes_requested"`, run `/unit-test-generator --fix <run-id>`.
2. Check that `.unit-test-agent/runs/<run-id>/generation.r1.json` is written and all tests pass.
