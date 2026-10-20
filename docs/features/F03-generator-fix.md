# F03: Unit Test Generator — Fix Mode (`unit-test-generator --fix <run>`)

## 1. Purpose

Address reviewer findings from `.unit-test-agent/runs/<run-id>/review.json` by updating only the flagged test cases or adding missing branch/mutant-killing tests, re-running tests and coverage, and writing `.unit-test-agent/runs/<run-id>/generation.r<N>.json`.

---

## 2. Invocation and Arguments

| Runtime            | Interactive Invocation                | Headless / CLI Invocation                                         | Arguments                                                               |
| :----------------- | :------------------------------------ | :---------------------------------------------------------------- | :---------------------------------------------------------------------- |
| **Claude Code**    | `/unit-test-generator --fix <run-id>` | `claude -p "/unit-test-generator --fix 20261020-091500-pricing"`  | `--fix <run-id>`: Run ID containing `generation.json` and `review.json` |
| **Codex CLI**      | `$unit-test-generator --fix <run-id>` | `codex exec "$unit-test-generator --fix 20261020-091500-pricing"` | `--fix <run-id>`                                                        |
| **Gemini CLI**     | `/unit-test-generator --fix <run-id>` | `gemini -p "/unit-test-generator --fix 20261020-091500-pricing"`  | `--fix <run-id>`                                                        |
| **GitHub Copilot** | `/unit-test-generator --fix <run-id>` | N/A (VS Code Chat Agent mode)                                     | `--fix <run-id>`                                                        |

---

## 3. Inputs and Outputs

### Preconditions & Inputs

1. `.unit-test-agent/project.json` exists, has supported `schemaVersion`, and has no `blocking: true` items in `missing`.
2. `.unit-test-agent/runs/<run-id>/generation.json` (or latest `generation.r<N-1>.json`) exists.
3. `.unit-test-agent/runs/<run-id>/review.json` exists and has supported `schemaVersion`.

### Outputs

- **Updated test files:** Only the test files flagged in `review.json`.
- **Revision report:** `.unit-test-agent/runs/<run-id>/generation.r<N>.json` (where `<N>` starts at `1` for the first fix pass, `2` for the second, etc., determined by scanning existing `generation.r*.json` files in `<run-id>`).
- **Schema:** `.unit-test-agent/schemas/generation.schema.json` (defined in [F07](F07-json-contracts.md)).

### Complete JSON Example (`.unit-test-agent/runs/20261020-091500-pricing/generation.r1.json`)

```json
{
  "schemaVersion": "1.0.0",
  "agent": "unit-test-generator",
  "mode": "fix",
  "runId": "20261020-091500-pricing",
  "revision": 1,
  "parentGenerationFile": "generation.json",
  "reviewFile": "review.json",
  "createdAt": "2026-10-20T09:30:00Z",
  "targets": ["src/pricing.ts"],
  "testFilesWritten": ["tests/pricing.test.ts"],
  "addressedFindings": [
    {
      "caseId": "TC-001",
      "issue": "weak_assertion: used toBeDefined() instead of exact numeric comparison",
      "actionTaken": "Replaced expect(result).toBeDefined() with expect(result).toBeCloseTo(108.5, 2)"
    },
    {
      "caseId": "SURVIVING-MUTANT-1",
      "issue": "Surviving mutant at src/pricing.ts:28 (changed >= 100 to > 100 for bulk tier)",
      "actionTaken": "Added TC-005 testing exact boundary quantity of 100"
    }
  ],
  "testCases": [
    {
      "id": "TC-001",
      "testFile": "tests/pricing.test.ts",
      "testName": "calculates standard subtotal for positive quantities",
      "targetFile": "src/pricing.ts",
      "targetSymbol": "calculateTotal",
      "category": "happy_path",
      "scenario": "Valid items array with standard tax rate returns exact expected total 108.50",
      "status": "passed"
    },
    {
      "id": "TC-002",
      "testFile": "tests/pricing.test.ts",
      "testName": "returns zero when cart item list is empty",
      "targetFile": "src/pricing.ts",
      "targetSymbol": "calculateTotal",
      "category": "edge",
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
      "scenario": "Negative item quantity triggers validation error",
      "status": "passed"
    },
    {
      "id": "TC-005",
      "testFile": "tests/pricing.test.ts",
      "testName": "applies bulk tier discount when quantity is exactly 100",
      "targetFile": "src/pricing.ts",
      "targetSymbol": "calculateTotal",
      "category": "boundary",
      "scenario": "Boundary quantity === 100 triggers bulk tier discount",
      "status": "passed"
    }
  ],
  "coverage": {
    "thresholds": {
      "lines": 80,
      "branches": 70
    },
    "achieved": {
      "lines": 96.0,
      "branches": 92.0
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
  "suspectedDefects": [
    {
      "defectId": "DEF-001",
      "targetFile": "src/pricing.ts",
      "targetSymbol": "applyDiscount",
      "line": 42,
      "testCaseId": "TC-003",
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

---

## 4. Step-by-Step Behavior

1. **Check Preconditions:**
   - Confirm `project.json`, `runs/<run-id>/generation.json`, and `runs/<run-id>/review.json` exist and have valid `schemaVersion`.
   - If `review.json` has `verdict: "approved"` and zero findings/surviving mutants/flaky tests, inform the user that no changes were requested and exit cleanly without writing a new revision.
2. **Determine Revision Number `<N>`:**
   - Check `.unit-test-agent/runs/<run-id>/` for existing `generation.r*.json` files. Set `N = highest + 1` (or `1` if none exist).
3. **Apply Scoped Fixes Only:**
   - For each finding in `review.json` (`findings`, `survivingMutants`, `flakyTests`):
     - Modify **only** the specific test cases or files flagged by the reviewer.
     - Do **not** touch unflagged test files or unflagged passing test cases.
     - Do **not** remove assertions to resolve a finding (replace weak assertions with stronger assertions, remove duplicate tests only when explicitly flagged as `duplicate_case`, isolate non-deterministic dependencies for `flakyTests`, and add boundary/branch assertions to kill `survivingMutants`).
4. **Re-run Tests & Coverage:**
   - Run the test suite and repair any syntax/setup errors up to `maxRepairAttempts`.
   - Run target-scoped coverage and update `coverage` and `uncoveredLines`.
5. **Write Output:**
   - Write `.unit-test-agent/runs/<run-id>/generation.r<N>.json`.

---

## 5. Edge Cases

- **`review.json` missing for `<run-id>`:** Stop immediately and instruct the user to run `/unit-test-reviewer <run-id>` first.
- **Multiple fix iterations (`r1`, `r2`, ...):** Each fix pass reads the latest `generation.r<N-1>.json` and latest `review.json` and increments `<N>`.
- **Mutant reveals a source bug:** If a new test added to kill a surviving mutant fails because the production code has a bug, follow the standard guardrail: keep the assertion, mark the test skipped/expected-fail, and add it to `suspectedDefects`.

---

## 6. Acceptance Criteria Checklist

- [ ] Stops with a clear message if `review.json` or `generation.json` does not exist for `<run-id>`.
- [ ] Modifies only the test files and test cases flagged in `review.json` (plus any new test cases required for missing branches or surviving mutants).
- [ ] Removes zero assertions (unless replacing a weak/tautological assertion with a stricter assertion, or removing an explicitly flagged `duplicate_case`).
- [ ] Re-runs tests and scoped coverage and records updated results in `.unit-test-agent/runs/<run-id>/generation.r<N>.json`.
- [ ] Never modifies any file outside `testRoots` and `.unit-test-agent/runs/<run-id>/`.
