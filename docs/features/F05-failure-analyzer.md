# F05: Unit Test Failure Analyzer (`unit-test-failure-analyzer`)

## 1. Purpose

Diagnose failing unit or regression tests—either by executing the test suite directly (all tests or scoped targets) or by parsing an existing test failure log file—and classify each failure's root cause (`stale_test`, `broken_mock_or_fixture`, `source_regression_bug`, `flaky_or_nondeterministic`, `environment_or_dependency`, or `syntax_or_import_error`) with concrete code/git evidence. Write `.unit-test-agent/runs/<run-id>/failure-analysis.json` as input for `/unit-test-generator --repair-failures <run-id>` ([F06](F06-generator-repair-failures.md)).

---

## 2. Invocation and Arguments

| Runtime            | Interactive Invocation                                  | Headless / CLI Invocation                  | Arguments                                                            |
| :----------------- | :------------------------------------------------------ | :----------------------------------------- | :------------------------------------------------------------------- |
| **Claude Code**    | `/unit-test-failure-analyzer [targets \| --log <file>]` | `claude -p "/unit-test-failure-analyzer"`  | Optional: `[targets]` (files/folders/globs) OR `--log <path-to-log>` |
| **Codex CLI**      | `$unit-test-failure-analyzer [targets \| --log <file>]` | `codex exec "$unit-test-failure-analyzer"` | Optional: `[targets]` OR `--log <path-to-log>`                       |
| **Gemini CLI**     | `/unit-test-failure-analyzer [targets \| --log <file>]` | `gemini -p "/unit-test-failure-analyzer"`  | Optional: `[targets]` OR `--log <path-to-log>`                       |
| **GitHub Copilot** | `/unit-test-failure-analyzer [targets \| --log <file>]` | N/A (VS Code Chat Agent mode)              | Optional: `[targets]` OR `--log <path-to-log>`                       |

---

## 3. Inputs and Outputs

### Preconditions & Inputs

1. `.unit-test-agent/project.json` exists, has supported `schemaVersion`, and has zero `blocking: true` items in `missing`.
2. Either:
   - **Direct execution mode** (default when `--log` is not passed): Runs `commands.test.value` from `project.json` (scoped to `[targets]` if provided, or the full test suite if omitted).
   - **Log ingestion mode** (`--log <file>`): Reads the specified test output/CI log file (must exist and be readable).

### Output

- **Path:** `.unit-test-agent/runs/<run-id>/failure-analysis.json` (where `<run-id>` is `<YYYYMMDD-HHMMSS>-failures`, e.g., `20261020-100500-failures`).
- **Schema:** `.unit-test-agent/schemas/failure-analysis.schema.json` (defined in [F07](F07-json-contracts.md)).

### Complete JSON Example (`.unit-test-agent/runs/20261020-100500-failures/failure-analysis.json`)

```json
{
  "schemaVersion": "1.0.0",
  "agent": "unit-test-failure-analyzer",
  "runId": "20261020-100500-failures",
  "createdAt": "2026-10-20T10:05:00Z",
  "inputSource": {
    "mode": "command",
    "commandExecuted": "npx vitest run",
    "logFile": null,
    "targets": []
  },
  "summary": {
    "totalTestsRun": 12,
    "passed": 10,
    "failed": 2,
    "skipped": 0,
    "analyzedFailures": 2,
    "byCategory": {
      "stale_test": 1,
      "broken_mock_or_fixture": 0,
      "source_regression_bug": 1,
      "flaky_or_nondeterministic": 0,
      "environment_or_dependency": 0,
      "syntax_or_import_error": 0
    }
  },
  "failures": [
    {
      "failureId": "FAIL-001",
      "testFile": "tests/order.test.ts",
      "testName": "formats order confirmation code with ORD- prefix",
      "targetFile": "src/order.ts",
      "targetSymbol": "formatOrderCode",
      "errorMessage": "AssertionError: expected 'ORD-2026-0042' to be 'ORD-0042'",
      "stackTraceLocation": "tests/order.test.ts:18:32",
      "category": "stale_test",
      "confidence": "high",
      "rootCause": "formatOrderCode(id, year) in src/order.ts:12 was updated to include the 4-digit year segment ('ORD-<year>-<paddedId>'), as documented in its JSDoc and type signature, but tests/order.test.ts:18 still asserts the legacy 'ORD-0042' format.",
      "evidence": [
        "src/order.ts:8-14 (JSDoc and implementation specify ORD-<year>-<id>)",
        "tests/order.test.ts:18 (passes year=2026 but asserts 'ORD-0042')"
      ],
      "recommendedAction": "update_test",
      "proposedChangeSummary": "Update expected value in tests/order.test.ts:18 from 'ORD-0042' to 'ORD-2026-0042' to match the documented year-prefixed contract."
    },
    {
      "failureId": "FAIL-002",
      "testFile": "tests/pricing.test.ts",
      "testName": "calculates 10% loyalty discount for gold members",
      "targetFile": "src/pricing.ts",
      "targetSymbol": "calculateMemberPrice",
      "errorMessage": "AssertionError: expected 110 to be 90",
      "stackTraceLocation": "tests/pricing.test.ts:45:28",
      "category": "source_regression_bug",
      "confidence": "high",
      "rootCause": "src/pricing.ts:31 adds the discount (`price + price * 0.10`) instead of subtracting it (`price - price * 0.10`). The test expectation of 90 for a $100 item with 10% discount is mathematically and contractually correct.",
      "evidence": [
        "src/pricing.ts:27-33 (JSDoc states 'Applies 10% discount for gold tier', line 31 uses '+' operator)",
        "tests/pricing.test.ts:45 (asserts calculateMemberPrice(100, 'gold') === 90)"
      ],
      "recommendedAction": "skip_and_report_defect",
      "proposedChangeSummary": "Do NOT change expected value 90 to 110. Mark the test as skipped/expected-fail in tests/pricing.test.ts and record DEF-001 in suspectedDefects."
    }
  ]
}
```

---

## 4. Step-by-Step Behavior

1. **Check Preconditions:**
   - Confirm `.unit-test-agent/project.json` exists, has a supported `schemaVersion`, and has zero `blocking: true` items in `missing`.
   - If `--log <file>` is passed, verify `<file>` exists on disk.
2. **Collect Failure Data:**
   - **If `mode == "command"`:** Run `commands.test.value` (scoped to `[targets]` if supplied). Capture stdout, stderr, exit code, and failing test names/locations.
   - **If `mode == "logFile"`:** Read and parse the provided log file to extract failing test files, test case names, assertion messages, and stack traces.
   - If 0 tests failed, write `failure-analysis.json` with `analyzedFailures: 0` and `failures: []`, inform the user that all tests passed, and exit.
3. **Investigate Each Failure:**
   - Read the failing test file around the stack trace line.
   - Identify the `targetFile` and `targetSymbol` being tested and read its source implementation, type signatures, and docstrings/comments.
   - Run `git diff` and `git log -n 5 -p -- <targetFile> <testFile>` (when inside a Git repo) to see whether source or test changed recently and why.
4. **Classify Root Cause & Recommended Action:**
   Map each failure to exactly one `category` and `recommendedAction`:
   | Category                    | Meaning                                                                                                                                | `recommendedAction`                                                   |
   | :-------------------------- | :------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------- |
   | `stale_test`                | Intentional production contract/signature/return format changed (backed by types, docs, or commit history), leaving the test outdated. | `update_test`                                                         |
   | `broken_mock_or_fixture`    | Mock return shape, spy target, or test fixture setup is out of sync with valid collaborator interfaces.                                | `fix_mock_or_setup`                                                   |
   | `source_regression_bug`     | The test assertion is valid according to domain logic/docs/types, and the production source code has a bug.                            | `skip_and_report_defect`                                              |
   | `flaky_or_nondeterministic` | Failure caused by timing/race conditions, unseeded randomness, timezone/clock dependency, or test order pollution.                     | `stabilize_test`                                                      |
   | `environment_or_dependency` | Missing dependency, missing env var, unbuilt package, or unavailable external service.                                                 | `fix_environment`                                                     |
   | `syntax_or_import_error`    | Broken import path, renamed module, or TypeScript/syntax compilation error in the test file.                                           | `update_test` (if in test) or `skip_and_report_defect` (if in source) |
5. **Write Output File:**
   - Write `.unit-test-agent/runs/<run-id>/failure-analysis.json`.
   - Print a human-readable summary table of diagnosed failures and tell the user they can run `/unit-test-generator --repair-failures <run-id>` next.

---

## 5. Edge Cases

- **All tests pass:** Writes `failure-analysis.json` with empty `failures: []` and `failed: 0`; does not fail.
- **Ambiguous whether source or test is wrong:** If confidence is `low` and the agent cannot prove an intentional contract change, default to `source_regression_bug` (`skip_and_report_defect`) rather than risking corrupting a valid regression test.
- **Suite crashes during collection/import before running tests:** Classify under `syntax_or_import_error` or `environment_or_dependency` with `testName: "<suite-collection>"`.
- **Read-only enforcement:** `unit-test-failure-analyzer` never modifies source or test files.

---

## 6. Acceptance Criteria Checklist

- [ ] Supports both direct test execution (`/unit-test-failure-analyzer [targets]`) and log file parsing (`/unit-test-failure-analyzer --log <file>`).
- [ ] Accurately distinguishes between a `stale_test` (intentional contract change) and a `source_regression_bug` (bug in production code).
- [ ] Records `failureId`, `testFile`, `testName`, `targetFile`, `targetSymbol`, `category`, `confidence`, `rootCause`, `evidence`, `recommendedAction`, and `proposedChangeSummary` for every failure.
- [ ] Defaults to `source_regression_bug` / `skip_and_report_defect` when a failure looks like a production bug.
- [ ] Writes a valid `.unit-test-agent/runs/<run-id>/failure-analysis.json` conforming to `schemas/failure-analysis.schema.json`.
- [ ] Modifies zero files outside `.unit-test-agent/runs/<run-id>/`.
