# F06: Unit Test Generator — Repair-Failures Mode (`unit-test-generator --repair-failures <run>`)

## 1. Purpose

Consume a failure analysis report (`.unit-test-agent/runs/<run-id>/failure-analysis.json` produced by `unit-test-failure-analyzer`) and repair the failing test suite strictly within `testRoots`:

- Fix outdated tests (`stale_test`), broken mocks/fixtures (`broken_mock_or_fixture`), flaky tests (`flaky_or_nondeterministic`), and test import/syntax errors (`syntax_or_import_error`).
- Preserve valid assertions for production bugs (`source_regression_bug`), mark those tests as skipped/expected-fail using the framework's native mechanism, and record the bugs in `suspectedDefects`.
- Write `.unit-test-agent/runs/<run-id>/failure-repair.json`.

---

## 2. Invocation and Arguments

| Runtime            | Interactive Invocation                            | Headless / CLI Invocation                                                      | Arguments                                                               |
| :----------------- | :------------------------------------------------ | :----------------------------------------------------------------------------- | :---------------------------------------------------------------------- |
| **Claude Code**    | `/unit-test-generator --repair-failures <run-id>` | `claude -p "/unit-test-generator --repair-failures 20261020-100500-failures"`  | `--repair-failures <run-id>`: Run ID containing `failure-analysis.json` |
| **Codex CLI**      | `$unit-test-generator --repair-failures <run-id>` | `codex exec "$unit-test-generator --repair-failures 20261020-100500-failures"` | `--repair-failures <run-id>`                                            |
| **Gemini CLI**     | `/unit-test-generator --repair-failures <run-id>` | `gemini -p "/unit-test-generator --repair-failures 20261020-100500-failures"`  | `--repair-failures <run-id>`                                            |
| **GitHub Copilot** | `/unit-test-generator --repair-failures <run-id>` | N/A (VS Code Chat Agent mode)                                                  | `--repair-failures <run-id>`                                            |

---

## 3. Inputs and Outputs

### Preconditions & Inputs

1. `.unit-test-agent/project.json` exists, has supported `schemaVersion`, and has zero `blocking: true` items in `missing`.
2. `.unit-test-agent/runs/<run-id>/failure-analysis.json` exists and has supported `schemaVersion`.

### Outputs

- **Modified test files:** Only the failing test files listed in `failure-analysis.json` (never source files).
- **Repair report:** `.unit-test-agent/runs/<run-id>/failure-repair.json`.
- **Schema:** `.unit-test-agent/schemas/failure-repair.schema.json` (defined in [F07](F07-json-contracts.md)).

### Complete JSON Example (`.unit-test-agent/runs/20261020-100500-failures/failure-repair.json`)

```json
{
  "schemaVersion": "1.0.0",
  "agent": "unit-test-generator",
  "mode": "repair-failures",
  "runId": "20261020-100500-failures",
  "failureAnalysisFile": "failure-analysis.json",
  "createdAt": "2026-10-20T10:12:00Z",
  "testFilesModified": ["tests/order.test.ts", "tests/pricing.test.ts"],
  "resolutions": [
    {
      "failureId": "FAIL-001",
      "testFile": "tests/order.test.ts",
      "testName": "formats order confirmation code with ORD- prefix",
      "category": "stale_test",
      "recommendedAction": "update_test",
      "resolutionStatus": "fixed",
      "changesMade": "Updated expected order code in tests/order.test.ts:18 from 'ORD-0042' to 'ORD-2026-0042' to match formatOrderCode(id, year) contract."
    },
    {
      "failureId": "FAIL-002",
      "testFile": "tests/pricing.test.ts",
      "testName": "calculates 10% loyalty discount for gold members",
      "category": "source_regression_bug",
      "recommendedAction": "skip_and_report_defect",
      "resolutionStatus": "skipped_defect",
      "changesMade": "Preserved expect(calculateMemberPrice(100, 'gold')).toBe(90) assertion, annotated test with it.skip('calculates 10% loyalty discount for gold members [DEF-001]'), and logged DEF-001 in suspectedDefects."
    }
  ],
  "suspectedDefects": [
    {
      "defectId": "DEF-001",
      "failureId": "FAIL-002",
      "targetFile": "src/pricing.ts",
      "targetSymbol": "calculateMemberPrice",
      "line": 31,
      "testFile": "tests/pricing.test.ts",
      "description": "calculateMemberPrice adds 10% instead of subtracting 10% for gold tier members.",
      "expectedBehavior": "calculateMemberPrice(100, 'gold') should return 90",
      "actualBehavior": "calculateMemberPrice(100, 'gold') returns 110"
    }
  ],
  "verification": {
    "commandExecuted": "npx vitest run tests/order.test.ts tests/pricing.test.ts",
    "exitCode": 0,
    "passed": 11,
    "failed": 0,
    "skipped": 1,
    "allResolved": true
  },
  "repairAttemptsUsed": 1,
  "maxRepairAttempts": 3
}
```

---

## 4. Step-by-Step Behavior

1. **Check Preconditions:**
   - Verify `.unit-test-agent/project.json` and `.unit-test-agent/runs/<run-id>/failure-analysis.json` exist, have supported `schemaVersion` (`1.0.0`), and `project.json` has no `blocking: true` items in `missing`.
2. **Process Each Failure Entry in `failure-analysis.json`:**
   - **Case A: `recommendedAction` is `update_test`, `fix_mock_or_setup`, or `stabilize_test` (`stale_test`, `broken_mock_or_fixture`, `flaky_or_nondeterministic`, or test-level `syntax_or_import_error`):**
     - Edit only the failing test in `testFile`.
     - Update outdated arguments, mocks, imports, or expected values to match the verified intentional contract, or inject deterministic clocks/seeds for flaky tests.
     - **Guardrail:** Never delete an assertion or replace a concrete assertion with a tautological one (`toBeDefined()`).
     - Record `resolutionStatus: "fixed"`.
   - **Case B: `recommendedAction` is `skip_and_report_defect` (`source_regression_bug` or Low-Confidence Ambiguous Failure):**
     - **Do not** alter the test's expected value to match the buggy output, and **do not** modify the production source file.
     - Keep the valid assertion intact inside the test function.
     - Mark the test as skipped or expected-fail using the test framework's native mechanism (`it.skip` / `test.fails`, `@pytest.mark.xfail` / `@pytest.mark.skip`, `@Disabled`, `#[ignore]`).
     - Add an entry to `suspectedDefects` linking `defectId` and `failureId`.
     - Record `resolutionStatus: "skipped_defect"`.
   - **Case C: `recommendedAction` is `fix_environment` (`environment_or_dependency`):**
     - Do not modify test files. Record `resolutionStatus: "blocked_env"` and explain the required environment step in `changesMade`.
3. **Verify Repaired Suite (up to `maxRepairAttempts`):**
   - Run `commands.test.value` scoped to `testFilesModified` (plus `typecheck` / `lint` if configured).
   - If any repaired test still fails, iterate up to `policy.maxRepairAttempts`. If still failing after `maxRepairAttempts`, mark `resolutionStatus: "unresolved"`.
4. **Write Output File:**
   - Write `.unit-test-agent/runs/<run-id>/failure-repair.json` and print a summary of fixed tests, skipped defects, and verification results.

---

## 5. Edge Cases

- **Zero failures in `failure-analysis.json`:** Write `failure-repair.json` with empty `testFilesModified: []` and `resolutions: []`, and exit cleanly.
- **Low-confidence `stale_test` in `failure-analysis.json`:** If a `stale_test` entry has `confidence: "low"`, treat it conservatively as `skip_and_report_defect` so a potential regression is never masked.
- **Unflagged tests in the same file:** Only the specific failing test blocks inside `testFile` may be modified; all other passing tests in the same file must remain untouched.

---

## 6. Acceptance Criteria Checklist

- [ ] Stops immediately if `project.json` or `runs/<run-id>/failure-analysis.json` is missing or invalid.
- [ ] Modifies only the failing test files listed in `failure-analysis.json`; never modifies files under `sourceRoots`.
- [ ] Repairs `stale_test`, `broken_mock_or_fixture`, `flaky_or_nondeterministic`, and test-level `syntax_or_import_error` items while keeping strict assertions.
- [ ] For `source_regression_bug`, keeps the valid assertion, marks the test skipped/expected-fail, and logs the defect in `suspectedDefects`.
- [ ] Re-runs the test suite to confirm exit code 0 (passing tests + skipped defects) within `maxRepairAttempts`.
- [ ] Writes a valid `.unit-test-agent/runs/<run-id>/failure-repair.json` conforming to `schemas/failure-repair.schema.json`.
