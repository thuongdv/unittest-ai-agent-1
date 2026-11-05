# F02: Unit Test Generator — Generate Mode (`unit-test-generator`)

## 1. Purpose

Generate comprehensive unit tests (happy path, edge, boundary, and error cases) for specified target source files, folders, or globs; iteratively repair test errors up to `maxRepairAttempts`; verify target-scoped code coverage against `project.json` thresholds; and flag any discovered production bugs in `suspectedDefects` without weakening assertions.

With `--requirements <run-id>`, expected results come from the requirement scenarios in `requirements.json` ([F13](F13-requirement-analyzer.md)), not from the code. Without it, the implementation (with its docs and types) is the requirement.

---

## 2. Invocation and Arguments

| Runtime            | Interactive Invocation           | Headless / CLI Invocation                          | Arguments                                                                       |
| :----------------- | :------------------------------- | :------------------------------------------------- | :------------------------------------------------------------------------------ |
| **Claude Code**    | `/unit-test-generator <targets>` | `claude -p "/unit-test-generator src/pricing.ts"`  | `<targets>`: One or more file paths, folder paths, or globs under `sourceRoots` |
| **Codex CLI**      | `$unit-test-generator <targets>` | `codex exec "$unit-test-generator src/pricing.ts"` | `<targets>`: File paths, folders, or globs                                      |
| **Gemini CLI**     | `/unit-test-generator <targets>` | `gemini -p "/unit-test-generator src/pricing.ts"`  | `<targets>`: File paths, folders, or globs                                      |
| **GitHub Copilot** | `/unit-test-generator <targets>` | N/A (VS Code Chat Agent mode)                      | `<targets>`: File paths, folders, or globs                                      |

Optional: `--requirements <run-id>` — a run created by `unit-test-requirement-analyzer` ([F13](F13-requirement-analyzer.md)). Example: `/unit-test-generator src/discount.ts --requirements 20261105-101500-shop-142`.

_(Note: If invoked with `--fix <run>`, see [F03](F03-generator-fix.md). If invoked with `--repair-failures <run>`, see [F06](F06-generator-repair-failures.md).)_

---

## 3. Inputs and Outputs

### Preconditions & Inputs

1. `.unit-test-agent/project.json` must exist.
2. `project.json` `schemaVersion` must be supported (`1.0.0`).
3. `project.json` `missing` array must have **zero** items with `blocking: true`.
4. `<targets>` argument must be provided and resolve to at least one source file under `sourceRoots` (and not excluded by `ignoreGlobs`).
5. With `--requirements <run-id>`: `runs/<run-id>/requirements.json` exists with a supported `schemaVersion`, and `runs/<run-id>/generation.json` does not exist yet.

### Outputs

- **Test files:** Written strictly under `testRoots` following `testFileNamingPattern`.
- **Report:** `.unit-test-agent/runs/<run-id>/generation.json` (where `<run-id>` is `<YYYYMMDD-HHMMSS>-<short-slug>`, e.g., `20261020-091500-pricing`). With `--requirements <run-id>`, the generator writes into that existing run instead of creating a new one.
- **Schema:** `.unit-test-agent/schemas/generation.schema.json` (defined in [F07](F07-json-contracts.md)).

### Complete JSON Example (`.unit-test-agent/runs/20261020-091500-pricing/generation.json`)

```json
{
  "schemaVersion": "1.0.0",
  "agent": "unit-test-generator",
  "mode": "generate",
  "runId": "20261020-091500-pricing",
  "createdAt": "2026-10-20T09:15:00Z",
  "targets": ["src/pricing.ts"],
  "requirementsFile": null,
  "testFilesWritten": ["tests/pricing.test.ts"],
  "testCases": [
    {
      "id": "TC-001",
      "testFile": "tests/pricing.test.ts",
      "testName": "calculates standard subtotal for positive quantities",
      "targetFile": "src/pricing.ts",
      "targetSymbol": "calculateTotal",
      "category": "happy_path",
      "scenarioId": null,
      "scenario": "Valid items array with standard tax rate returns expected total",
      "status": "passed"
    },
    {
      "id": "TC-002",
      "testFile": "tests/pricing.test.ts",
      "testName": "returns zero when cart item list is empty",
      "targetFile": "src/pricing.ts",
      "targetSymbol": "calculateTotal",
      "category": "edge",
      "scenarioId": null,
      "scenario": "Empty items array returns 0 without applying flat shipping fee",
      "status": "passed"
    },
    {
      "id": "TC-003",
      "testFile": "tests/pricing.test.ts",
      "testName": "applies 100% discount cap at boundary",
      "targetFile": "src/pricing.ts",
      "targetSymbol": "applyDiscount",
      "category": "boundary",
      "scenarioId": null,
      "scenario": "Discount percentage of exactly 100 reduces total to 0, not negative",
      "status": "skipped_defect"
    },
    {
      "id": "TC-004",
      "testFile": "tests/pricing.test.ts",
      "testName": "throws RangeError when item quantity is negative",
      "targetFile": "src/pricing.ts",
      "targetSymbol": "calculateTotal",
      "category": "error",
      "scenarioId": null,
      "scenario": "Negative item quantity triggers validation error",
      "status": "passed"
    }
  ],
  "coverage": {
    "thresholds": {
      "lines": 80,
      "branches": 70
    },
    "achieved": {
      "lines": 94.5,
      "branches": 88.0
    },
    "metThresholds": true,
    "commandExecuted": "npx vitest run tests/pricing.test.ts --coverage"
  },
  "uncoveredLines": [
    {
      "file": "src/pricing.ts",
      "lines": [84, 85],
      "reason": "Defensive unreachable default branch in exhaustive switch over Currency enum"
    }
  ],
  "unmappedScenarios": [],
  "suspectedDefects": [
    {
      "defectId": "DEF-001",
      "targetFile": "src/pricing.ts",
      "targetSymbol": "applyDiscount",
      "line": 42,
      "testCaseId": "TC-003",
      "scenarioId": null,
      "testFile": "tests/pricing.test.ts",
      "description": "applyDiscount subtracts (price * (1 + discount/100)) instead of (price * (discount/100)) when discount === 100, returning -price instead of 0.",
      "expectedBehavior": "applyDiscount(100, 100) should return 0",
      "actualBehavior": "applyDiscount(100, 100) returns -100"
    }
  ],
  "repairAttemptsUsed": 1,
  "maxRepairAttempts": 3
}
```

### Requirement-Mode Excerpt (`runs/20261105-101500-shop-142/generation.json`)

The requirement says the discount starts at $100; the code starts it above $200. The tests assert the requirement's values, so they fail and are skipped as defects instead of locking in the code's behavior.

```json
{
  "requirementsFile": "requirements.json",
  "testCases": [
    {
      "id": "TC-002",
      "testFile": "tests/discount.test.ts",
      "testName": "applies 10% discount at exactly $100",
      "targetFile": "src/discount.ts",
      "targetSymbol": "calculateDiscount",
      "category": "boundary",
      "scenarioId": "SCN-002",
      "scenario": "SCN-002: order amount exactly $100 → discount $10; asserts calculateDiscount(100) === 10",
      "status": "skipped_defect"
    },
    {
      "id": "TC-006",
      "testFile": "tests/discount.test.ts",
      "testName": "applies 10% discount to a $300 order",
      "targetFile": "src/discount.ts",
      "targetSymbol": "calculateDiscount",
      "category": "happy_path",
      "scenarioId": null,
      "scenario": "Implementation-derived: covers the discount branch; consistent with REQ-001",
      "status": "passed"
    }
  ],
  "unmappedScenarios": [],
  "suspectedDefects": [
    {
      "defectId": "DEF-001",
      "targetFile": "src/discount.ts",
      "targetSymbol": "calculateDiscount",
      "line": 12,
      "testCaseId": "TC-002",
      "scenarioId": "SCN-002",
      "testFile": "tests/discount.test.ts",
      "description": "calculateDiscount applies the discount only when amount > 200; REQ-001 (SHOP-142) requires it from $100.",
      "expectedBehavior": "calculateDiscount(100) returns 10",
      "actualBehavior": "calculateDiscount(100) returns 0"
    }
  ]
}
```

---

## 4. Step-by-Step Behavior

1. **Check Preconditions:**
   - Validate `.unit-test-agent/project.json` existence, `schemaVersion`, and absence of `blocking: true` items in `missing`. If any check fails, stop immediately and explain what the user must resolve.
2. **Resolve Targets & Read Exemplars:**
   - Resolve `<targets>` to concrete source files under `sourceRoots`.
   - Read the target source files and any directly imported types/interfaces.
   - Read `exemplarTestFiles` from `project.json` to match imports, assertion conventions, describe/test naming, and mocking patterns.
3. **Design Test Cases:**
   - For each exported function, class, or method in the target files, design cases across four categories:
     - `happy_path`: Typical valid inputs and expected state/return values.
     - `edge`: Empty collections, whitespace strings, `null`/`undefined`/`None`, single-element inputs, idempotency.
     - `boundary`: `0`, `-1`, `MAX_SAFE_INTEGER`, threshold boundaries (`<`, `<=`, `===`, `>=`, `>`), off-by-one boundaries.
     - `error`: Invalid arguments, thrown exceptions, rejected promises, error return tuples.
   - Assign stable IDs (`TC-001`, `TC-002`, ...).
   - **With `--requirements`, map scenarios first:**
     - For each `SCN` in `requirements.json`, write at least one test case on the target symbol that implements it, and set its `scenarioId`.
     - If the scenario has `call` and `expectedValue`, use them verbatim. Otherwise translate `given` into concrete inputs and `expected` into an exact assertion. Take the expected value from the scenario text only, never from running or reading the code. Unit or format conversions (e.g., dollars → cents) are allowed and must be stated in `scenario`.
     - If no symbol in the targets implements a scenario, list it in `unmappedScenarios` with a `reason`.
     - Then add implementation-derived cases (`scenarioId: null`) for the remaining branches. They must never assert a value that contradicts any scenario.
   - **Without `--requirements`:** set every `scenarioId` to `null`, `requirementsFile` to `null`, and `unmappedScenarios` to `[]`.
4. **Write Test Files (Strictly Under `testRoots`):**
   - Create or update the corresponding test files under `testRoots`.
   - **Never modify any file under `sourceRoots`.**
5. **Run Tests & Iterative Repair Loop (up to `maxRepairAttempts`):**
   - Run the scoped test command (plus `typecheck` and `lint` if configured).
   - If a test fails due to a **test issue** (wrong import path, mock setup error, syntax/type error in the test), repair the test and increment `repairAttemptsUsed` (up to `policy.maxRepairAttempts`).
   - If a test fails because the **source code behavior contradicts a requirement scenario, its documented contract, types, or obvious domain logic**:
     - **Never** change the expected value in the assertion to the buggy return value or delete the assertion.
     - Keep the correct assertion inside the test body, mark the test as skipped or expected-fail (`it.skip` / `test.fails` in Vitest/Jest, `@pytest.mark.xfail(strict=False)` or `@pytest.mark.skip` in pytest, `@Disabled` in JUnit, `#[ignore]` in Rust), set the test case status to `skipped_defect`, and record a detailed entry in `suspectedDefects` (with the test's `scenarioId`, or `null`).
6. **Run Scoped Coverage & Loop Until Thresholds Met:**
   - Execute `commands.coverage` scoped to the target files.
   - Parse achieved line and branch coverage.
   - If coverage is below `policy.coverageThresholds` (and user may set 100%), add test cases targeting the uncovered lines/branches and re-run.
   - For any line that remains uncovered after exhausting reachable paths (or because a test is skipped due to a suspected defect), record its file, line numbers, and specific reason in `uncoveredLines`.
7. **Write Output File:**
   - Write `.unit-test-agent/runs/<run-id>/generation.json` and report the `<run-id>` and summary to the user.

---

## 5. Edge Cases

- **100% Coverage Threshold with Unreachable Defensive Guards:** When the user sets threshold to 100% and a line is unreachable from the public API (e.g., exhaustive `default:` branch), document the exact line numbers and justification in `uncoveredLines`.
- **Target File Has Existing Tests:** Append or merge new test cases into the existing test file without deleting or breaking pre-existing passing tests.
- **Repair Attempts Exhausted:** If a test still fails after `maxRepairAttempts` due to complex setup issues, leave its status as `failed` in `generation.json` and report the error clearly.
- **Code and its docs agree, but contradict the requirement:** The requirement wins. Keep the scenario's expected value and record a defect; the code's docs are not a reason to change the assertion.
- **Scenario belongs to code outside the targets:** List it in `unmappedScenarios` (e.g., `"reason": "No symbol in src/discount.ts handles shipping fees"`); do not widen the targets.

---

## 6. Acceptance Criteria Checklist

- [ ] Stops immediately with a clear error message if `project.json` is missing, has an unsupported `schemaVersion`, or contains any `blocking: true` item in `missing`.
- [ ] Writes test files only inside `testRoots`; `git diff` on `sourceRoots` is always empty.
- [ ] Generates test cases covering `happy_path`, `edge`, `boundary`, and `error` categories.
- [ ] When encountering a deliberate bug in a target source file, keeps the valid assertion, marks the test as skipped/expected-fail, and records the bug in `suspectedDefects`.
- [ ] Runs the repository's coverage command and records actual achieved line and branch percentages.
- [ ] Lists every remaining uncovered line with an explicit `reason` in `uncoveredLines`.
- [ ] With `--requirements`: every scenario has at least one linked test case or an `unmappedScenarios` entry, and every linked assertion uses the scenario's expected value.
- [ ] Without `--requirements`: `requirementsFile` is `null` and every `scenarioId` is `null`.
- [ ] Writes a valid `.unit-test-agent/runs/<run-id>/generation.json` conforming to `schemas/generation.schema.json`.
