# F11: Automated Output Checks (`evals/`)

## 1. Purpose

Provide a developer-only TypeScript + Vitest verification package in `evals/` (no UI, never installed into user repos, never called by agents) that validates the JSON files and repository state left behind after running the agents against a fixture project in a temporary directory. Because it only inspects post-run outputs and Git state, the exact same test suite verifies runs from **Claude Code**, **OpenAI Codex CLI**, **Gemini CLI**, and **GitHub Copilot**.

---

## 2. Invocation and Arguments

### Step 1: Run Step (Manual or Nightly, Never on Every Code Edit)

Copy a fixture (`examples/ts-vitest` or `examples/py-pytest`) to a temp directory (`EVAL_TARGET_DIR`) and run the agent chain:

- **Claude Code (headless):** `claude -p "/codebase-analyzer"` etc.
- **Codex CLI (headless):** `codex exec "$codebase-analyzer"` etc.
- **Gemini CLI (headless):** `gemini -p "/codebase-analyzer"` etc.
- **GitHub Copilot (manual):** Run the slash commands in VS Code Agent Mode inside `EVAL_TARGET_DIR`.

### Step 2: Check Step (Automated via Vitest)

```bash
# Single run verification:
EVAL_TARGET_DIR=/tmp/ut-eval-run-1 EVAL_FIXTURE=ts-vitest npm --prefix evals test

# Multi-run pass-rate aggregation (when an agent is run N times across /tmp/ut-eval-run-*):
EVAL_RUN_DIRS="/tmp/ut-eval-run-1,/tmp/ut-eval-run-2,/tmp/ut-eval-run-3" EVAL_FIXTURE=ts-vitest npm --prefix evals run test:passrate
```

---

## 3. Inputs and Outputs

### Inputs

- `EVAL_TARGET_DIR` (or `EVAL_RUN_DIRS` for multi-run pass-rate calculation): Path to the temporary fixture directory where the agent chain ran.
- `EVAL_FIXTURE`: `"ts-vitest"` | `"py-pytest"` (selects the expected fixture baseline assertions).
- `schemas/*.schema.json`: The 5 JSON Schema files from [F07](F07-json-contracts.md).

### Outputs

- Vitest console report (pass/fail per property check) and optional JSON summary report (`evals/reports/latest.json`) with pass rate across runs when multiple run directories are provided.

### Complete JSON Example (`evals/reports/latest.json` Multi-Run Pass-Rate Summary)

```json
{
  "evaluatedAt": "2026-10-20T11:00:00Z",
  "runtime": "claude",
  "fixture": "ts-vitest",
  "runsEvaluated": 3,
  "overallPassRate": 1.0,
  "checks": [
    {
      "checkId": "CHK-01",
      "name": "Schema & envelope validation",
      "passedRuns": 3,
      "totalRuns": 3,
      "passRate": 1.0
    },
    {
      "checkId": "CHK-02",
      "name": "project.json matches fixture ground truth",
      "passedRuns": 3,
      "totalRuns": 3,
      "passRate": 1.0
    },
    {
      "checkId": "CHK-03",
      "name": "git diff clean outside testRoots",
      "passedRuns": 3,
      "totalRuns": 3,
      "passRate": 1.0
    },
    {
      "checkId": "CHK-04",
      "name": "Deliberate bug recorded in suspectedDefects & test skipped/xfail",
      "passedRuns": 3,
      "totalRuns": 3,
      "passRate": 1.0
    },
    {
      "checkId": "CHK-05",
      "name": "Reported coverage matches actual re-run coverage",
      "passedRuns": 3,
      "totalRuns": 3,
      "passRate": 1.0
    },
    {
      "checkId": "CHK-06",
      "name": "Every review.json caseId exists in generation.json",
      "passedRuns": 3,
      "totalRuns": 3,
      "passRate": 1.0
    },
    {
      "checkId": "CHK-07",
      "name": "Fix mode touched only flagged files and removed no assertions",
      "passedRuns": 3,
      "totalRuns": 3,
      "passRate": 1.0
    },
    {
      "checkId": "CHK-08",
      "name": "Failure analyzer classifies stale_test vs source_regression_bug",
      "passedRuns": 3,
      "totalRuns": 3,
      "passRate": 1.0
    },
    {
      "checkId": "CHK-09",
      "name": "Repair-failures mode fixes stale test, skips regression bug, touches only failing tests",
      "passedRuns": 3,
      "totalRuns": 3,
      "passRate": 1.0
    },
    {
      "checkId": "CHK-10",
      "name": "Analyzer re-run preserves manual user edits in project.json",
      "passedRuns": 3,
      "totalRuns": 3,
      "passRate": 1.0
    },
    {
      "checkId": "CHK-11",
      "name": "Blocking missing item halts downstream agents",
      "passedRuns": 3,
      "totalRuns": 3,
      "passRate": 1.0
    }
  ]
}
```

---

## 4. Step-by-Step Property Checks (`evals/tests/`)

Agent output is non-deterministic, so `evals/` checks semantic properties and invariants rather than exact string snapshots:

1. **`CHK-01` Schema & Envelope Validation (`schema.eval.test.ts`):**
   - Validates `project.json`, `generation.json`, `generation.r*.json`, `review.json`, `failure-analysis.json`, and `failure-repair.json` against `schemas/*.schema.json` using Ajv (`ajv` + `ajv-formats`).
   - Confirms `schemaVersion`, `agent`, `createdAt`, and `runId` (where applicable) are present.
2. **`CHK-02` `project.json` Ground-Truth Match (`analyzer.eval.test.ts`):**
   - For `ts-vitest`: `language.value === "typescript"`, `testFramework.value === "vitest"`, `packageManager.value === "npm"`, `commands.test.value` contains `"vitest"`.
   - For `py-pytest`: `language.value === "python"`, `testFramework.value === "pytest"`, `commands.test.value` contains `"pytest"`.
3. **`CHK-03` Source File Integrity (`git-diff.eval.test.ts`):**
   - Runs `git status --porcelain` / `git diff --name-only` in `EVAL_TARGET_DIR` and asserts that every modified or untracked file is inside `testRoots` or `.unit-test-agent/`. Zero changes in `sourceRoots`.
4. **`CHK-04` Deliberate Bug Detection & Skip Guardrail (`defects.eval.test.ts`):**
   - Asserts `suspectedDefects.length >= 1` referencing the fixture's deliberate bug (`src/pricing.ts` or `src/inventory/reorder.py`).
   - Reads the corresponding test file and asserts the test case is marked skipped/expected-fail (`it.skip`, `test.skip`, `test.fails`, `@pytest.mark.skip`, or `@pytest.mark.xfail`) and **does not** assert the buggy return value as the expected result.
5. **`CHK-05` Coverage Accuracy (`coverage.eval.test.ts`):**
   - Executes the fixture's coverage command directly inside `EVAL_TARGET_DIR`, parses the resulting coverage summary report, and verifies that `generation.json`'s `coverage.achieved.lines` and `branches` match within `±2.0%`.
6. **`CHK-06` Reviewer Referential Integrity (`reviewer.eval.test.ts`):**
   - Asserts every `finding.caseId` in `review.json` exists in `generation.json`'s `testCases[].id`.
   - Asserts `flakinessCheck.reruns === 3`.
7. **`CHK-07` Fix Mode Scope & Assertion Preservation (`fix-mode.eval.test.ts`):**
   - Compares `generation.json` and `generation.r1.json`: asserts `testFilesWritten` in `r1` is a subset of files flagged in `review.json`, and counts assertions (`expect(` / `assert `) in the modified test file to confirm assertion count did not decrease (unless a `duplicate_case` was explicitly removed).
8. **`CHK-08` Failure Analyzer Root-Cause Classification (`failure-analyzer.eval.test.ts`):**
   - Checks `failure-analysis.json` against the fixture's seeded failing tests: verifies the outdated format test is classified as `stale_test` (`recommendedAction: "update_test"`) and the deliberate production bug failure is classified as `source_regression_bug` (`recommendedAction: "skip_and_report_defect"`).
9. **`CHK-09` Repair-Failures Mode Verification (`failure-repair.eval.test.ts`):**
   - Checks `failure-repair.json`: verifies `stale_test` has `resolutionStatus: "fixed"`, `source_regression_bug` has `resolutionStatus: "skipped_defect"` and appears in `suspectedDefects`, only failing test files were modified, and re-running the repaired test files exits with code `0`.
10. **`CHK-10` User Edit Preservation (`user-edit.eval.test.ts`):**
    - Verifies that after a custom value (e.g., `policy.coverageThresholds.lines = 91` or custom `commands.lint.value`) was written to `project.json` prior to re-running `codebase-analyzer`, the post-run `project.json` still retains that exact user value.
11. **`CHK-11` Precondition Blocking Check (`preconditions.eval.test.ts`):**
    - Verifies that when `project.json` contains a `missing` item with `blocking: true` (or an unsupported `schemaVersion: "99.0.0"`), invoking downstream agents writes **no** `generation.json`/`review.json`/`failure-analysis.json` and leaves `testRoots` untouched.

---

## 5. Edge Cases

- **Partial workflow evaluations:** Individual test files in `evals/tests/` can be run selectively (e.g., `npx vitest run tests/analyzer.eval.test.ts`) when testing a single agent during development, or skipped automatically if a run directory only exercised one workflow.
- **Multi-run pass rate:** When `EVAL_RUN_DIRS` contains multiple comma-separated directories from repeated runs, `evals` runs the checks against each directory and prints the pass rate (`passedRuns / totalRuns`) per check.

---

## 6. Acceptance Criteria Checklist

- [ ] `evals/` is a standalone TypeScript + Vitest package with zero coupling to user-installed agent files.
- [ ] Implements all 11 property checks (`CHK-01` through `CHK-11`) covering all 4 agents and all 3 generator modes.
- [ ] Includes unit self-tests (using mock run directories) so `npm --prefix evals test:self` verifies the evaluator logic itself.
- [ ] Supports evaluating multiple run directories and reporting a pass rate across runs.
