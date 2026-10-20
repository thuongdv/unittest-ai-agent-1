# Implementation Plan: F04 — Unit Test Reviewer (`unit-test-reviewer`)

## 1. Dependencies on Other Features

- **Depends on:**
  - [F07: JSON Contracts (`schemas/review.schema.json`)](../features/F07-json-contracts.md)
  - [F02: Generator — Generate Mode](../features/F02-generator-generate.md)

## 2. Files to Create or Change

- `agents/unit-test-reviewer.md` — Canonical Markdown agent instructions for the reviewer.

## 3. Ordered Tasks

### Task F04-T1: Draft `agents/unit-test-reviewer.md` Rubric, Mutation & Flakiness Instructions

- **Description:** Write `agents/unit-test-reviewer.md` with precondition checks, fresh-context subagent rules, the 7-point review rubric, ecosystem-specific scoped mutation testing table (Stryker, `mutmut`, PIT, Stryker.NET, `gremlins`, `cargo-mutants`) with graceful skip when unconfigured, 3x flakiness reruns, and `review.json` output rules.
- **Acceptance Criteria:**
  - [ ] Requires every `finding.caseId` to match an existing `id` in `generation.json`.
  - [ ] Explicitly instructs the agent not to fail when `mutationTool.value` is `null` (`status: "skipped_not_configured"`).
  - [ ] Enforces 3 consecutive test executions for flakiness detection.
  - [ ] Strictly forbids editing source or test files.
- **Files Touched:** `agents/unit-test-reviewer.md`

### Task F04-T2: Verify `unit-test-reviewer` End-to-End on Fixtures

- **Description:** Run `/unit-test-reviewer <run-id>` on fixture generation runs in `examples/ts-vitest` and `examples/py-pytest`.
- **Acceptance Criteria:**
  - [ ] `.unit-test-agent/runs/<run-id>/review.json` validates against `schemas/review.schema.json`.
  - [ ] `git status` shows zero modified source or test files after review.
- **Files Touched:** `PROGRESS.md`

## 4. How to Verify Manually

1. After running `/unit-test-generator` on a temp fixture copy, invoke `/unit-test-reviewer <run-id>`.
2. Confirm `runs/<run-id>/review.json` exists, `flakinessCheck.reruns == 3`, `mutationTesting.status` is `"executed"` or `"skipped_not_configured"`, and all `caseId` values exist in `generation.json`.
